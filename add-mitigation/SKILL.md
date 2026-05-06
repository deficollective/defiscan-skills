---
name: add-mitigation
description: Analyze a function's source code to find on-chain mitigations (value bounds, delays, rate limits) and add them to functions.json. Reads the flattened Solidity source, identifies require/if constraints, and builds structured mitigation entries.
---

# Add Mitigation

Analyze a permissioned function's source code to find on-chain mitigations and add them to `functions.json`.

## Arguments

```
/add-mitigation <project> <contractAddress> <functionName>
```

- **project** — project folder name (e.g. `aerodrome`, `aave-v3`)
- **contractAddress** — chain-prefixed contract address (e.g. `base:0xB630...`). Add prefix if omitted.
- **functionName** — the function to analyze (e.g. `setDefaultCap`)

## Instructions

### Phase 0: Fetch fund context

Before reading source code, fetch the function's impact data from the API to understand what's at stake:

```bash
curl -s "localhost:2021/api/projects/<project>/function-analysis" | jq '.contracts["<contractAddress>"]["<functionName>"].impact'
```

Present the fund context to the researcher:

- If `totalFundsAtRisk > 0` or `totalTokenValueAtRisk > 0`: "This function can reach **$X in funds** across N contracts (+ $Y in token value). Mitigations for this function constrain the following risk."
- If all zero: "This function has **no detected fund impact**. Mitigations may be less urgent, but you can still add them for documentation purposes."

This helps the researcher prioritize their review effort.

### Phase 1: Locate source code

Find the flattened source in `packages/config/src/projects/<project>/.flat/`. Match by address in the filename. If not found, check `packages/config/crytic-export/etherscan-contracts/` by address.

### Phase 2: Analyze the function

Read the function body and identify **on-chain constraints that meaningfully limit the potential fund impact** of the function. The bar for reporting a mitigation is: *"does this constraint reduce the amount of funds the caller can move, drain, dilute, or freeze?"* If the answer is no, do NOT report it as a mitigation.

Valid mitigation types:

1. **Value range** (`type: "valueRange"`) — `require(value <= MAX)`, `require(value >= MIN)` where the bound actually caps a financial parameter (fee, rate, cap, weight). Also `require(value != 0)` when the field is a financial parameter.
2. **Delay** (`type: "delay"`) — timelocks, cooldown periods, `block.timestamp >= lastAction + DELAY` giving affected users time to react.
3. **Relative value** (`type: "relativeValue"`) — max change per call, e.g. `newRate = oldRate + NUDGE`.
4. **Other** (`type: "other"`) — rate limiting, once-per-epoch caps, formula-driven amount bounds, bytecode-level whitelists that restrict *who can be affected*.

For each constraint found, determine:
- The **bound values** (constants, storage variables, or computed)
- Whether the bound is **hardcoded** (literal in code), **discovered** (a contract field we track in `discovered.json`), or a **fieldRef** (a path expression like `$self.MAX_FEE`)

If a constraint cannot be expressed using the standard mitigation types above, **warn the user**: "Found constraint [describe it] but it cannot be represented in the standard mitigation format. Manual review or a custom `other` type with a descriptive text may be needed."

#### Do NOT report these as mitigations

Many `require`/`if` checks in permissioned functions are **sanity/invariant guards** that don't bound the financial impact. Filter them out — they pollute the risk view and distract the reader from real mitigations.

- **Zero-address / zero-value guards** on address fields (`require(_newAddress != address(0))`). Doesn't limit impact; just prevents the caller from pointing at the zero address.
- **"Same value" guards** (`require(newState != oldState)`, `require(_newAddress != currentAddress)`). Only prevents no-op calls, zero risk-reduction effect.
- **Type/kind filters** that don't narrow the economic scope (`require(escrowType[id] == EscrowType.MANAGED)` when *every* MANAGED NFT holds pooled funds — narrowing to a type doesn't narrow the exposure).
- **Access control** (`require(msg.sender == owner)`). This is captured by `ownerDefinitions`, not as a mitigation.
- **Self-rotation guards** (`require(_newManager != address(this))`). Sanity, not risk-limiting.
- **Initialization one-shot guards** (`require(initializedAt == 0)`). These make the score `no-impact` (the owner is already consumed), not a mitigation.
- **Array length equality checks**, **non-empty array checks**, **reentrancy guards** — protocol-correctness plumbing, not fund-impact bounds.
- **Permission-restating labels of any kind** — `<role>-gated`, `<contract>-gated`, `governance-gated`, `governance-or-multisig`, `only<X>`. These restate `ownerDefinitions` in mitigation form. The smell: the description says "callable only by X" without naming a numeric or temporal bound on what X can do. If the function genuinely *needs* one of these labels, the right action is usually to score `no-impact` (when X is the legitimate caller and there's no extraction path) — not to write the access modifier as a mitigation. A 552-mitigation cleanup in 2026-05 stripped this exact family from a single project; they pollute every entity card without bounding anything.

Rule of thumb: state the mitigation as *"Because of this constraint, the caller cannot move/drain/dilute more than X"*. If the sentence doesn't finish naturally with a number or a duration, it's not a mitigation — drop it.

#### Calibration: when the function looks "future-only" but isn't

The single most common scoring error is calling a state-flag setter `future-only` when the flag is actually checked on a user-exit path. **Don't trust the flag's name** — protocols use `paused` / `frozen` / `halted` / `stopped` / `locked` / `inactive` / `disabled` interchangeably and sometimes invertedly. The only reliable signal is the flag's read sites. Grep the flag across the protocol's `validate*` / `_check*` / inline-`require` paths and classify by what's blocked:

- **Blocks user-exit operations** (withdraw, redeem, repay, claim, transfer, liquidate) → committed funds become inaccessible for the duration → `critical` with a `lock-bound` (or similar) mitigation capped at affected pool/vault TVS.
- **Blocks user-entry operations only** (supply, deposit, borrow, mint) → existing positions wind down cleanly → `no-impact` with `future-only`.
- **Blocks both entry and exit** → same as exit-blocking → `critical`.

*Concrete instance:* in Aave V3, `setReservePause` reads in `validateWithdraw`/`validateRepay`/`validateLiquidationCall` (exit-blocking → `critical`); `setReserveFreeze` reads only in `validateSupply`/`validateBorrow` (entry-only → `no-impact`). Same protocol, same admin, opposite scores. The same shape exists with different naming across many lending and vault protocols — verify by reading the read sites, not by name-matching.

Other shape calibrations worth knowing:

- **Cancel / veto functions** (any DAO `cancelProposal`-style, any timelock `cancel(payloadId)`-style, emergency-veto multisigs). Flip a `Cancelled`/`Vetoed` state and emit an event — they don't execute the cancelled action. Score `no-impact` with a `veto-only` / `payload-veto` / `proposal-veto` label even when call-graph BFS attaches large reach (the cancel and the execute often share a contract).
- **Token-rescue functions with an explicit underlying-asset guard** — `require(token != <underlyingAsset>)` or equivalent — can only sweep ERC-20s mistakenly sent to the contract. Score `no-impact` with a `mistakenly-sent only` label and quote the require. The same shape sometimes exists *without* the require but with an architectural invariant (deposits route directly to per-asset wrappers, so the rescuing contract custodies nothing). Verify the invariant from the deposit-path source before applying.
- **Hook / controller-swap functions** that change which contract receives event hooks (reward-accounting controller, fee-distribution target, accounting bookkeeper). The hook is consulted on transfer/mint/burn for bookkeeping; principal stays in the original contract's own balance accounting. Score `no-impact` with a `no-principal-path` label. Worst-case effects of a malicious hook (revert callback → DoS, divert future rewards) are operational and recoverable; neither extracts committed funds.

### Phase 3: Present findings

Show the user what you found:

```
Found mitigations for setDefaultCap on CLGaugeFactory:

1. [valueRange] _defaultCap must be >= 1 (hardcoded) and <= MAX_BPS (discovered: 10000)
   Source: require(_defaultCap != 0, "ZDC"); require(_defaultCap <= MAX_BPS, "MC");

2. [other] Only callable by emissionAdmin
   Source: require(msg.sender == emissionAdmin, "NA");
```

Ask the user to confirm or adjust before applying. The user may want to:
- Skip some mitigations (e.g. access control is already captured by `ownerDefinitions`)
- Adjust descriptions
- Change value modes (e.g. use `discovered` instead of `hardcoded`)

### Phase 4: Apply

Add mitigations to the function entry via the l2b API (`PUT /api/projects/<project>/functions`).

#### Mitigation format

Each mitigation is an object in the `mitigations` array:

```json
{
  "type": "valueRange",
  "description": "Human-readable description of the constraint",
  "valueRange": {
    "min": { "mode": "...", ... },
    "max": { "mode": "...", ... },
    "unit": "bps"
  },
  "mitigatedField": {
    "contractAddress": "base:0x...",
    "fieldName": "FIELD_NAME"
  }
}
```

#### Writing the `description` field

The `description` is rendered directly in the public DeFiScan report UI (hover tooltip on the mitigation badge). It must read like a plain-English security explanation for a DeFi user, not a developer note.

**NEVER mention in a description:**
- Internal file names (`discovered.json`, `functions.json`, `config.jsonc`, `funds-data.json`)
- Backend/monitoring plumbing terms (`fieldMeta`, `HIGH severity`, `ignoreInWatchMode`, "tracked at", "see the X field")
- Discovery/handler jargon (`CallHandler`, `ignoreRelatives`, `proxyType`)
- Editorial notes addressed to maintainers or future researchers

**DO:**
- Describe the constraint in terms of what the contract does (what limits apply, what trust assumptions exist, what the function can/cannot do)
- Name contracts by their on-chain role ("the pool owner", "the governor") when it adds clarity; include the address only when a user might want to verify it on-chain
- Quote concrete numbers (caps, delays, percentages) in user-readable units

If you want to flag implementation details to researchers, put them in a commit message or a separate doc — **not** in the `description` field.

#### Value modes

Use the most specific mode available:

- **`"hardcoded"`** — literal value in code that cannot change. Use `"value": "123"`.
  ```json
  { "mode": "hardcoded", "value": "500000" }
  ```

- **`"discovered"`** — a field tracked in `discovered.json` whose current value we can read. Use when the bound is a contract storage variable (constant or mutable) that discovery fetches. Provides live tracking.
  ```json
  { "mode": "discovered", "contractAddress": "base:0x...", "fieldName": "MAX_BPS" }
  ```

- **`"fieldRef"`** — a path expression referencing the field on the same or another contract, using the owner path syntax (`$self.FIELD`, `@ref.FIELD`).
  ```json
  { "mode": "fieldRef", "fieldPath": "$self.MAXIMUM_TEAM_RATE" }
  ```

**Choosing between modes:**
- If the value is a Solidity `constant` or `immutable` that appears in `discovered.json` → use `"discovered"`
- If the value is a storage variable on the same contract → use `"fieldRef"` with `$self.FIELD`
- If the value is a literal number in the code not stored in any field → use `"hardcoded"`
- When in doubt, prefer `"discovered"` over `"hardcoded"` since it tracks on-chain changes

#### Mitigation types

**`valueRange`** — for require statements bounding a parameter:
```json
{
  "type": "valueRange",
  "description": "Fee cannot exceed MAX_FEE",
  "valueRange": {
    "min": { "mode": "hardcoded", "value": "0" },
    "max": { "mode": "discovered", "contractAddress": "base:0x...", "fieldName": "MAX_FEE" },
    "unit": "bps"
  },
  "mitigatedField": { "contractAddress": "base:0x...", "fieldName": "MAX_FEE" }
}
```

**`delay`** — for timelock or cooldown constraints:
```json
{
  "type": "delay",
  "description": "48-hour timelock before execution",
  "delaySeconds": 172800,
  "delayRef": { "contractAddress": "base:0x...", "fieldName": "delay" }
}
```

**`relativeValue`** — for max-change-per-call constraints:
```json
{
  "type": "relativeValue",
  "description": "Can only change by 1 bps per epoch",
  "relativeValue": {
    "maxChangePercent": { "mode": "discovered", "contractAddress": "base:0x...", "fieldName": "NUDGE" }
  }
}
```

**`other`** — for constraints that don't fit the above:
```json
{
  "type": "other",
  "description": "Rate limited: once per week",
  "mitigatedField": { "contractAddress": "base:0x...", "fieldName": "WEEK" }
}
```

#### `mitigatedField` (optional)

Points to the contract field that enforces the constraint. Used for display and monitoring. Include when the constraint references a specific on-chain value.

#### Saving via API

Save mitigations via the functions update API. This ensures correct address normalization, attribution stamping, and config severity integration (auto-writes `severity: "HIGH"` for `mitigatedField` references).

1. First, fetch existing mitigations for the function:
   ```bash
   curl -s "localhost:2021/api/projects/<project>/functions" | jq '.contracts["<address>"].functions[] | select(.functionName == "<name>") | .mitigations'
   ```

2. Append new mitigations to the existing array (don't duplicate), then save:
   ```bash
   curl -s -X PUT "localhost:2021/api/projects/<project>/functions" \
     -H "Content-Type: application/json" \
     -d '{
       "contractAddress": "<address>",
       "functionName": "<name>",
       "mitigations": [... existing + new ...]
     }'
   ```

The API replaces the entire `mitigations` array, so always send the full array (existing + new). Only `mitigations` is updated — other fields (`score`, `ownerDefinitions`, etc.) are preserved.

- Do NOT add access control as a mitigation — that's already captured by `isPermissioned` and `ownerDefinitions`

### Phase 5: Report

```
Added 2 mitigations to setDefaultCap on CLGaugeFactory (base:0xB630...):
  Funds at risk: $12.4M across 3 contracts
  + [valueRange] Default cap bounded 1 to MAX_BPS (10000 bps)
  + [other] Cannot set to zero
```