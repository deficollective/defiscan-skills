---
name: score-contract
description: Score all permissioned functions on a contract — assess fund impact (drain risk, token devaluation, yield reduction), set scores (critical/no-impact), and add on-chain mitigations with impact caps. Combines impact analysis, containment classification, and on-chain-bounded cap derivation into a single batch workflow.
---

# Score Contract

Batch-analyze permissioned functions on a contract: triage by complexity, read source proportionately, classify containment, score, and wire mitigations with on-chain-bounded impact caps where the worst case is bounded by observable state.

API data (`/admins`, `/function-analysis`, `/dependencies`) is for **prioritization** — what reaches funds, at what scale. It is not the final verdict. Source-reading is mandatory, but it scales with ambiguity, not with function count: most functions resolve in seconds from signature + a glance at the body; only the minority with genuine ambiguity need a full storage-trace.

## Arguments

```
/score-contract <project> <contractAddress>
/score-contract <project> <addr1> <addr2> <addr3> ...
/score-contract <project> <addr1,addr2,addr3>
/score-contract <project> --all --interactive
```

- **project** — project folder name (e.g. `aerodrome`, `aave-v3`)
- **contractAddress** — chain-prefixed address (e.g. `base:0xB630...`) or contract name (e.g. `CLGaugeFactory`). Add chain prefix if omitted.
- **Multiple addresses** — positional or comma-separated. Names and addresses may be mixed; normalize by lowercasing when matching.
- **--all --interactive** — score every contract with unscored permissioned functions, one at a time, waiting for user confirmation after each.

When a list is provided, use the `--all --interactive` loop but build the queue from the supplied list; preserve input order.

---

## Instructions

### Phase 1: Fetch structured data

API (`localhost:2021`):

```bash
curl -s "localhost:2021/api/projects/<project>/admins?contract=<address>"
curl -s "localhost:2021/api/projects/<project>/function-analysis"
curl -s "localhost:2021/api/projects/<project>/dependencies"
```

Disk:
- `packages/config/src/projects/<project>/functions.json` — permissioned function list with current scores/mitigations
- `packages/config/src/projects/<project>/discovered.json` — discovered fields, used for fieldRef caps
- `packages/config/src/projects/<project>/call-graph-data.json` — resolved call-graph edges (needed for direct-caller detection and unresolved call details)
- `packages/config/src/projects/<project>/funds-data.json` — token prices/decimals (for `unit: token` cap resolution)
- `packages/config/src/projects/<project>/.flat/` — flattened source code (mandatory input, see Phase 2)

Skip functions that already have a non-`unscored` score AND a non-empty `mitigations` array — they are fully reviewed. Mention the skipped count.

**Output — Function List:**

```
CONTRACT: CLGaugeFactory (base:0xB630...)
PERMISSIONED FUNCTIONS (6):
  #  Function              Score       Funds at Risk    Unresolved
  ── ───────────────────── ─────────── ──────────────── ──────────
  1. setEmissionCap        unscored    $0 direct        0
  2. setDefaultCap         unscored    $0 direct        0
  3. setRedistributor      unscored    $12.4M reach.    0
  4. setEmissionAdmin      critical    (skipped)        -
  5. setNotifyAdmin        unscored    $12.4M reach.    2
  6. setSecondsPerLiq      no-impact   (skipped)        -
```

---

### Phase 2: Triage, then read source proportionately

Reading source remains mandatory, but **not every function needs the same depth**. Most functions resolve in a glance; a minority need a real trace. The goal is to let cost scale with ambiguity, not with function count.

#### Phase 2a — Load context once per contract

Before scoring any function on a contract, do this once:

1. Read the contract's flattened source into working memory (typically one file under `.flat/<ContractName>/` or `.flat/<address>/`). Subsequent function analysis reads from memory, not fresh fetches.
2. Scan for **shared bounds** — patterns that apply to multiple functions:
   - A modifier gating many functions (`wait`, `onlyPoolConfigurator`, `whenNotPaused`).
   - A constant cap used by several setters (`MAX_VALID_LTV`, `MAX_VALID_RESERVE_FACTOR`).
   - A common guard shape (every setter bounded by `require(_x <= MAX)`).
3. Note any contract-wide state flags that change fund semantics (`initialized`, `live`, `paused`, `triggered`).

Resolve these once; reuse across all functions on this contract instead of re-deriving per function.

#### Phase 2b — Triage each function into a lane

Using the API data (`directFundsUsd`, `totalReachableFundsUsd`, `viewOnlyPath`, `impact.reachableContracts[]`), the owner path, the function signature, and a **single glance** at the function body, classify each function into one of three lanes:

**Fast lane** — verdict is high-confidence from the signature, owner, and a one-line body read. No grep across `.flat/`, no Q2/Q3.

Typical fast-lane cases (not exhaustive — the principle is: the shape is unambiguous):
- Ownership / authority transfers → `critical`, uncapped.
- Proxy upgrade functions (`upgradeTo`, `upgradeToAndCall`) → `critical`, reach = full impl (engine handles the propagation).
- Pure view / getter functions marked permissioned by mistake → `no-impact`.
- `renounceOwnership` → `critical`.
- Obvious init-guarded one-shots → `no-impact`.

**Standard lane** — the function body fits on one screen, has one or two storage writes, and the read sites for those writes are clearly local (same contract, obvious timing semantics). Answer Q1 from the body; Q2/Q3 follow directly or are unnecessary.

Typical standard-lane cases:
- Most `set*` parameter functions on self-contained contracts.
- Role grants / revokes on an AccessControl contract.
- Simple pause/unpause toggles where the pause flag's read sites are obvious from a grep within the same file.

**Deep lane** — the fast/standard verdict feels uncertain. Invoke the full three-question walk below. Don't default here for every function; reserve it for cases where:
- A setter's written field is read in **other** contracts (the flattened file's imports cross contract boundaries).
- The function's call-graph reach includes heuristic-resolved edges (`TMP_*` storage variables, or contracts you didn't expect).
- The name suggests future-only but the body writes a field checked on existing-balance paths (or vice versa — name suggests destructive but the path is actually future-bounded).
- A user-supplied parameter feeds directly into an accounting update (the `configureAssets` / fake-`totalSupply` class of issue).
- You can't confidently finish the sentence *"if an attacker controls this, they can move/drain/dilute/freeze ___."*

#### Phase 2c — Three questions (deep-lane only)

For each deep-lane function, answer **in order**. Stop as soon as the answer is settled.

**Q1 — What state does this function write, and where is that state read?**
Grep the storage variable(s) across all flattened sources. If every read lands on a future-only path (new-operation check, next-epoch accounting, next-swap fee computation) and no read touches existing balances, the function is `no-impact`. Stop.

**Q2 — If the caller is malicious, what is the worst-case value movement?**
Trace forward from the writes to a concrete extraction / devaluation / freeze sink. Name it explicitly: *"Pool.rescueTokens sends any ERC20 held by the Pool to an arbitrary address — no inline bound on token or amount."* If no such sentence completes, the function is `no-impact` and you were over-scoring.

**Q3 — What on-chain state bounds that worst case?**
Look for `require(x <= CAP)`, balance checks, thresholds, allowances, or an external value like a specific holder's balance, an accrued user balance, or a vault the contract pulls from. This is what becomes the `impactCap` fieldRef in Phase 3a. If nothing bounds it, it's `critical`, uncapped.

#### When the call graph has unresolved edges

A function with unresolved calls (see `call-graph-data.json`) jumps straight to the deep lane even if the signature suggests otherwise — the unresolved calls are exactly where Q1's "where is that state read" tends to hide answers. Print the unresolved set:

```
⚠ UNRESOLVED CALLS for setNotifyAdmin:
  - storageVariable: externalOracle, interface: IOracle, function: getPrice
```

Then follow `/analyze-impact` to trace storage writes through read sites.

#### Containment labels

When a mitigation is warranted, pick a short containment label that describes *what's actually bounded* — use whatever wording fits the specific situation. Common shapes we've seen:

- **future-only** — effect only applies to new operations (new deposits, next epoch, next swap); existing positions untouched.
- **rewards-only / incentives-only** — effect contained to a separate reward distribution; core deposits unaffected.
- **asset-bounded / pool-bounded** — effect bounded to one specific asset or one pool's balance.
- **timelock-delayed** — routed through a governance timelock with a non-trivial delay.
- **public-trigger-with-threshold** — the function is public but gated by a threshold outside admin control (emergency modules, kill switches).
- **disabled-by-state** — the action is impossible given current on-chain state.
- **claim-DoS** — breaks future claims but doesn't drain accrued balances.
- **per-user theft** — per-call theft bounded to a single targeted user's unclaimed balance.

These are examples, not a fixed vocabulary. Invent a more specific label when it fits the situation better.

#### Scoring rules

- **`critical`** — Q2 yields a concrete worst-case value movement that touches already-committed funds / tokens / yields. Even with a cap, the score stays `critical` and the cap attaches to a mitigation.
- **`no-impact`** — Q2 yields nothing that touches already-committed state; effect is confined to future behavior, configuration with no fund path, init-guarded, or a purely defensive action (e.g. disabling a safety module).

#### Calibration examples (illustrative, not exhaustive)

These are specific mistakes worth keeping in mind because they illustrate *classes* of error — similar surprises exist in other protocols under other names:

- **State-flag setters that gate user operations.** The function flips a boolean / state flag that other functions read; the question is *which* user operations the flag blocks. **Don't trust the flag's name** — protocols use `paused` / `frozen` / `halted` / `stopped` / `locked` / `inactive` / `disabled` interchangeably and sometimes invertedly (a "frozen" reserve in one protocol behaves like a "paused" one in another, and vice versa). The only reliable signal is the flag's read sites. Grep the flag across the protocol's `validate*` / `_check*` / inline-`require` paths and classify by what gets blocked:
  - **Blocks user-exit operations** (withdraw, redeem, repay, claim, transfer, liquidate). Committed funds become inaccessible for the duration — `critical` with a `lock-bound` (or similar) mitigation capped at the affected pool/vault TVS.
  - **Blocks user-entry operations only** (supply, deposit, borrow, mint). Existing positions still wind down cleanly — `no-impact` with `future-only`.
  - **Blocks both entry and exit**: same as exit-blocking — `critical`.
  - **Blocks neither (only protocol-internal hooks like `mintToTreasury`)**: `no-impact`.

  *Concrete instance:* in Aave V3, `setReservePause` reads in `validateWithdraw`/`validateRepay`/`validateLiquidationCall` (exit-blocking → `critical`), while `setReserveFreeze` reads only in `validateSupply`/`validateBorrow` (entry-only → `no-impact`) — both flags are on the same contract, same admin, opposite scores. Same shape exists with different naming in many lending and vault protocols. **Do not write `future-only` on a flag setter without checking the read sites** — the label is wrong any time the flag is checked on an exit path, regardless of what the function or flag is named.
- **"Freeze" vs "deactivate" on a reserve** look similar but differ on whether withdraws still work. *Class:* if a flag is read by a withdraw path, setting it blocks withdraws → `critical`. If it's only checked on supply/borrow, it's `future-only`. Read the flag's call sites.
- **Cancel / veto functions** (any DAO `cancelProposal`-style function, any timelock `cancel(payloadId)`-style function, emergency-veto multisigs). The function flips a `Cancelled`/`Vetoed` state and emits an event — it does NOT execute the cancelled action's downstream calls. Despite call-graph BFS attaching the cancelled action's reach to the function (the cancel and the execute often live on the same contract and reference the same downstream executor), the cancel itself extracts nothing. Score `no-impact` with a `veto-only` / `payload-veto` / `proposal-veto` label and a one-sentence description naming what's blocked. Don't accept the BFS reach as evidence of impact here.
- **Token-rescue functions with an explicit underlying-asset guard.** Many wrapper / vault / lending-pool aToken contracts expose a `rescueTokens(token, ...)` (or `sweep(token, ...)`) that's gated by `require(token != <underlyingAsset>, ...)` or equivalent. The check explicitly disallows the asset users deposited, so the function can only move ERC-20s mistakenly sent to the contract. Score `no-impact` with a `mistakenly-sent only` label and quote the require in the description. *Watch for the same shape without the explicit require but with an architectural invariant that achieves the same end* — e.g. lending pools that route deposits directly to per-asset wrapper contracts so the pool itself custodies nothing under normal conditions. Verify the invariant from the deposit path's source before using this verdict; it isn't universal across protocols.
- **Hook / controller-swap functions** that change which contract receives event hooks (reward accounting controllers, accounting bookkeepers, fee-distribution targets, etc.). The hook is consulted on every transfer/mint/burn for *bookkeeping*, but principal stays in the original contract's own balance accounting and is never touched by the hook. Score `no-impact` with a `no-principal-path` label. Worst-case effects of a malicious hook (revert in the hook callback → DoS on transfers; divert future reward distributions) are operational and recoverable by setting the hook back; neither extracts committed funds. Confirm the hook contract has no write access to balances before applying this verdict.
- **Emergency-module "cage" functions** can mean either "trigger emergency shutdown" or "retire this module." *Class:* when a function name suggests destructive action, still trace the state change — sometimes the `live` flag's semantics are inverted relative to what the name implies.
- **Arming-a-trigger functions** that look like setters. A function that sets an oracle threshold may look benign but arms a public-callable trigger. *Class:* if a function writes a field that gates a public destructive action, the scoring depends on what the destructive action does — not on the setter itself.
- **Functions that take user-supplied state** (supply totals, balances) and use them in an accounting update. *Class:* on-chain checks based on caller-supplied values can bypass the invariant the field normally enforces — check whether the function re-reads from the authoritative source.
- **Rewire-the-pipe admin functions** (e.g. setting a new rewards controller) can be live-path-changing *or* orphan-creating. *Class:* trace downstream — if existing accruals continue working on the old pipe, it's a misconfiguration risk, not a fund-impact event.

Treat this list as examples of where names deceive. Other projects will have their own variants; the judgment is to distrust the name and read the reads.

**Output — Summary Table (include containment and lane):**

```
IMPACT ANALYSIS RESULTS:

  #  Function              Lane      Score       Containment        Worst-case path
  ── ───────────────────── ─────────  ─────────── ────────────────── ─────────────────────────
  1. setDefaultCap         standard  no-impact   future-only         Cap read at next ensureDefault()
  2. setRedistributor      deep      critical    (uncapped)          Sets recipient of excess funds
  3. setNotifyAdmin        deep      critical    rewards-only        Controls who triggers distribute
  4. transferOwnership     fast      critical    (uncapped)          Full admin transfer
```

**Ask the user to confirm before applying. Do not proceed to Phase 3 without explicit confirmation.**

---

### Phase 3: Find mitigations (specific, not generic)

Add a mitigation when any of these holds:
- The function has a non-trivial on-chain constraint that bounds extraction (`valueRange`, `delay`, `relativeValue`).
- The function scores `no-impact` and needs a containment label + one-sentence explanation of the real path.
- The function scores `critical` but is subject to an upstream timelock or a scoped restriction (see Phase 3b).

**Reject as non-mitigations:**
- Zero-address / zero-value guards.
- "Same value" guards.
- Type/kind filters that don't narrow economic scope.
- Self-rotation guards (`require(_new != address(this))`).
- Init one-shot guards (these make the function `no-impact` instead).
- Access control (captured by `ownerDefinitions`).
- **Permission-restating labels of any kind**, e.g. `<role>-gated`, `<contract>-gated`, `governance-gated`, `governance-or-multisig`, `only<X>`, `<X>-only`. The label-smell is "this function is callable by X" — that's `ownerDefinitions`, not a mitigation. A mitigation describes a *bound* on what the listed owner can do (delay, cap, rate limit), not who the listed owner is. If the only "constraint" you found is the function's own access modifier, **do not write a mitigation** — either score `no-impact` (when the listed owner is the legitimate caller and there's no extraction path) or leave it `critical` uncapped (when there is a path but no on-chain bound). A blanket sweep of these labels in 2026-05 stripped 552 such pseudo-mitigations from a single project — they pollute every entity card without bounding anything.

**Write specific descriptions, not boilerplate.** Compare:

> ❌ *"Does not touch already-committed funds — users retain full withdrawal rights."*
>
> ✅ *"Even a maliciously low threshold only enables `KillSwitchOracle.trigger()`, whose sole on-chain effect is calling `PoolConfigurator.setReserveBorrowing(asset, false)` on active reserves. Existing borrow positions, collateral deposits, repayments, and withdrawals remain fully functional."*

Name the specific downstream function, the specific bound, and the specific reason existing funds are safe. Three or four sentences max, but every clause must carry information a researcher couldn't derive from the function name.

**Shared patterns pass** — after analyzing one function, scan siblings on the same contract for the same bound (all setters capped by the same `MAX_VALID_*`, all functions routed through the same `wait` modifier).

---

### Phase 3a: Impact caps (first-class, not optional)

For every `critical` function whose worst case is bounded by observable state, attach an `impactCap` mitigation. **Prefer fieldRef + token-unit over hardcoded USD** — caps that drift with price are worse than no cap.

**The canonical shape:**

```json
{
  "type": "other",
  "label": "<containment label>",
  "description": "<specific worst-case + bound + what's protected>",
  "impactCap": {
    "value": {
      "mode": "fieldRef",
      "contractAddress": "eth:0x<contract that holds the bound field>",
      "fieldName": "<discovered field name>"
    },
    "unit": {
      "kind": "token",
      "tokenAddress": "eth:0x<token the bound is denominated in>"
    }
  }
}
```

The resolver reads the raw uint from `discovered.json`, divides by `10**decimals`, and multiplies by the token price from `funds-data.json`. The cap auto-updates on every discovery cycle and funds-data refresh.

**Before writing a cap, identify what bounds the worst case.** The answer changes per protocol; here are a few shapes it commonly takes:

1. **Balance of a specific holder.** The protocol pulls from one account (a vault, a treasury, a reward strategy); the caller can't extract more than that account holds. Field: `<token>.balanceOf(<holder>)`.
2. **A discovered cap field.** The protocol has an on-chain `supplyCap`, `borrowCap`, `mintLimit`, etc. that directly bounds the extractable amount. Field: the cap itself (may need a unit conversion via `multiplier`).
3. **The target contract's own holdings.** A drain function pulls from `address(this)`. Field: `<token>.balanceOf(<targetContract>)`.
4. **An allowance or approval.** The caller can only move what's been pre-approved. Field: `<token>.allowance(<owner>, <spender>)`.
5. **A computed aggregate.** Total debt of a reserve, TVL of a pool, totalSupply of an aToken. Field: the summary.

For each, sanity-check with `cast` before wiring the cap — actual balances often surprise:

```bash
cast call <TOKEN> "balanceOf(address)(uint256)" <HOLDER> --rpc-url $ETHEREUM_RPC_URL_FOR_DISCOVERY
cast call <TOKEN> "allowance(address,address)(uint256)" <OWNER> <SPENDER> --rpc-url ...
cast call <CONTRACT> "<summaryGetter>()(uint256)" --rpc-url ...
```

A pull-vs-push mismatch is a classic trap: capping at the strategy's own balance reads zero if the strategy pulls from a separate vault on demand. The effective bound is `min(vault.balanceOf, vault→strategy.allowance)`.

**If the bounding field is not already in `discovered.json`, stop and add a handler to `config.jsonc`:**

```jsonc
"eth:0x<anchor contract that will host the field>": {
  "fields": {
    "<bound field name>": {
      "description": "<what this bounds, for researcher clarity>",
      "handler": {
        "type": "call",
        "method": "function <signature> view returns (<type>)",
        "args": [<args>],
        "address": "eth:0x<contract the call targets>"
      }
    }
  }
}
```

The anchor contract is whichever discovered contract is semantically closest to the thing being capped — it doesn't have to own the bounding state. Then run `l2b discover <project> --dev` and confirm the field populated before writing the cap.

**Tag the cap's unit token** with `isToken: true` so its price + decimals populate `funds-data.json`:

```bash
curl -s -X PUT "localhost:2021/api/projects/<project>/contract-tags" \
  -H "Content-Type: application/json" \
  -d '{"contractAddress":"eth:0x<TOKEN>","isExternal":true,"entity":"<issuer>","isToken":true}'
```

Confirm `tokenInfo` is present in `funds-data.json`. If the DeFiScan endpoints server isn't running and the fetch fails, seed `tokenInfo` manually from an on-chain price source the protocol itself uses (its own oracle, an AMM TWAP, a discoverable price feed) + `totalSupply()` via `cast` — don't block the cap on an external service.

**When to use each unit:**
- `unit: token` + `value: fieldRef` — the default. Auto-updates with balance + price.
- `unit: usd` + `value: hardcoded` — only when the bound is a fixed fiat amount nobody expects to move (rare). Flag it as "stale-risk; revisit on <condition>".
- `unit: scaler` — for ratio caps; rarely needed in scoring workflows.
- `multiplier` — use when the field needs a fractional adjustment (e.g. `supplyCap` × LTV when the worst case is the collateralizable portion, not the full cap).

**Don't over-reach with caps.** A fieldRef cap works when the relationship "worst-case value moveable ≤ field value" is obviously true from the source. If the cap requires assumptions about attacker behavior or protocol-level invariants, prefer an uncapped `critical` with a descriptive mitigation over a fragile cap.

---

### Phase 3b: Mitigation placement

Get this wrong and the cap either doesn't propagate or over-propagates. **Two anchoring options, four placement rules.** The first decision is structural; the placement rules apply conditionally based on it.

---

#### Step 0 — Anchoring decision (do this for EVERY mitigation and cap, regardless of type)

The principle, from [`docs/developers/designs/edge-centric-constraints.md`](../../docs/developers/designs/edge-centric-constraints.md): **anchor a constraint where it originates**. This applies to **every** mitigation type (`delay`, `valueRange`, `relativeValue`, `other`) and to caps.

Ask: *"Is this bound true for **every** caller of this function, or does it exist **because of who calls**?"*

- **Function-intrinsic** — bound comes from the function's body (`require(value <= MAX)`, on-chain cooldown checked on every call, a cap derived from observable state of the function's own contract). True for every caller. → **Anchor on the function**. Apply placement rules 1 / 2 below.

- **Caller-relationship** — bound only exists because of a specific caller's path: a vault timelock between curator/owner and `submitCap`, a DAO `executionDelay` between an Executor and its targets, a per-caller cap a downstream router enforces only on one path, a multisig threshold gate specific to one owner. → **Anchor on the edge**. Apply placement rule 4 below.

The same applies to **caps**. A function-intrinsic cap (worst case bounded by observable state, regardless of caller) → `mitigations[N].impactCap` on the function. A caller-relationship cap (only enforced because of who calls) → `setEdgeCap` rule on the edge.

The two anchors compose at compute time — the merge in `getMitigationsForOwner` collects function mits plus edge mits and de-dupes through the shared `mitigationDedupKey` — so there's no double-counting. But mis-anchoring (caller-relationship constraint on the function) silently propagates the bound onto every downstream dependency row, even if you add `scopedTo`. The historic `scopedTo: { type: "admin" }` pattern (rule 3 below) is **legacy** for new authoring of caller-relationship constraints: prefer edge anchoring where the dedup model already handles per-owner display correctly.

> ⚠ **The `func.delay` synthesis trap.** Separately from `mitigations[]`, the function entry can carry a top-level `delay: { contractAddress, fieldName }` field. `buildMergedMitigations` reads it and **auto-synthesizes** a global `delay` mitigation that propagates onto every downstream dependency row as a spurious badge. For **any caller-relationship timelock** (most timelocks in practice), do NOT set `func.delay` — emit the edge mitigation per rule 4. Setting `func.delay` is appropriate only when the function carries its own enforced cooldown for every caller (rare). The Morpho-vault leak we hunted down across 5 projects in 2026-06 was this exact trap; see [`permissions.md § Edge-anchored mitigations & impact caps`](../../docs/developers/features/permissions.md).

---

#### Function-anchored placement (rules 1–3, apply when Step 0 says "function-intrinsic")

**1. Global leaf cap** — on the *called* function of the external dependency, no `scopedTo`.
Example: `LRTOracle.getAssetPrice` or `LRTOracle.rsETHPrice`.
Purpose: the engine uses this as the per-reachable-contract clamp when any upstream function's reach includes that leaf. Also propagates as a transitive mit through forward-BFS.

**2. Scoped-to-dependency self-cap** — on **direct callers** per `call-graph-data.json`, with `scopedTo: { address: <depAddress>, type: "dependency" }`.
Extract the direct caller set:

```bash
python3 -c "
import json
cg = json.load(open('packages/config/src/projects/<p>/call-graph-data.json'))
dep = '<eth:0xDEPADDR>'.lower()
for a, c in cg['contracts'].items():
    for e in c.get('externalCalls', []):
        if (e.get('resolvedAddress') or '').lower() == dep:
            print(f\"{c.get('name','?')}.{e['callerFunction']} -> {e.get('calledFunction')}\")"
```

Do NOT apply this to every function the dependency analysis *lists* — only to the direct callers from the call graph. The engine propagates the cap to transitive reachers via forward-BFS. Over-applying the scoped mit is harmless but noisy.

**3. Scoped-to-admin self-cap** — on functions the admin directly calls, with `scopedTo: { address: <adminAddress>, type: "admin" }`.
Use this to attach upstream timelock or multi-path-gated mitigations (e.g. "all actions via DSPauseProxy are subject to a 1-day DSPause timelock"). Same direct-callee rule applies — put the mit on each function the admin directly owns.

**Caller-specific mitigations need scoping; function-level bounds must stay un-scoped.** Many functions are reachable through more than one admin path — typically a slow governance pipeline (DAO + timelock) AND a fast multisig path (emergency guardian, risk council, gnosis safe with the same role). A "vote + execution delay" mitigation is true only for the timelock caller; the multisig calls directly with no delay. The constraint *must* be expressed per-caller. Two ways to do that:
- **Legacy (rule 3)**: `scopedTo: { address: <timelock-or-executor>, type: "admin" }` on the function mitigation. Still works; existing data is not migrated.
- **Principled new (rule 4)**: `setEdgeMitigation` on the owner→function permission edge. The edge IS the scope; no `scopedTo` needed; the dedup model prevents per-owner display issues. **Prefer this for new authoring.**

A global (un-scoped) caller-specific bound falsely shows on every other admin that holds the same role and misleads readers about that admin's real authority — regardless of which anchoring you choose.

The opposite mistake is just as bad. A *function-level* bound (impactCap from observable state, pause-window scope, value-range max from a require) applies to every caller and **must stay un-scoped** — adding `scopedTo` there hides the cap from admins it should apply to. *Concrete instance from 2026-05:* an emergency-pause function's `pause-bound` cap had been scoped to the timelock caller only; the emergency multisig (the actual fast caller of the function) saw no cap and only a misleading transitive `future-only` mitigation propagated up from a downstream view function — until the scope was removed.

Decide consciously per mitigation per Step 0: is the bound a property of *who calls* (delay, sequencer-confirmation requirement, vote-threshold, per-caller cap) or a property of *the function itself* (numeric cap, locked-window, value-range)? Caller-relationship → edge anchor (rule 4) or scoped function mit (legacy rule 3). Function-intrinsic → un-scoped function mit (rules 1/2).

---

#### Edge-anchored placement (rule 4, apply when Step 0 says "caller-relationship")

**4. Edge-anchored mitigation or cap on the owner→function permission edge.** This works for **any** mitigation type (`delay`, `valueRange`, `relativeValue`, `other`) and for `impactCap`. The edge IS the scope — no `scopedTo` is needed on the mitigation.

For each owner of the function (one rule per `(owner, fn)` permission edge), emit into `call-graph-overrides.json`:

```jsonc
// Mitigation on an owner→fn edge
{
  "id": "<slug>-<role>-<fn>",
  "type": "setEdgeMitigation",
  "from": "<chain>:<ownerAddress>",
  "to":   "<chain>:<fnContract>.<fnName>",
  "edgeType": "permission",
  "mitigations": [ /* any Mitigation object — same shape as functions.json mitigations[N] */ ],
  "note": "Why this constraint belongs on the edge"
}

// Cap on an owner→fn edge (impact cap; relationship-anchored, NOT function-intrinsic)
{
  "id": "<slug>-<role>-<fn>-cap",
  "type": "setEdgeCap",
  "from": "<chain>:<ownerAddress>",
  "to":   "<chain>:<fnContract>.<fnName>",
  "edgeType": "permission",
  "cap": { "value": { "mode": "fieldRef", "contractAddress": "<addr>", "fieldName": "..." }, "unit": { "kind": "..." } }
}
```

For the idempotent end-to-end python template that resolves owners from `discovered.json` and emits one rule per `(owner, fn)` pair (safe to re-run), see [`/review-morpho-vault` SCORING_TABLE.md § Step 7c](../review-morpho-vault/SCORING_TABLE.md) — it's a Morpho-vault example but the script is generic.

**When edge anchoring is the right choice — concrete shapes:**
- **DAO governance pipeline timelock** (e.g. `Executor.execute` is gated by a vote+timelock chain). Author the delay as `setEdgeMitigation` on the (DAO/Executor → target.fn) edge, not as a global mit on every owned function.
- **Per-caller cap a downstream router enforces** (e.g. only one specific upstream owner triggers the capped path; other owners bypass it). `setEdgeCap` on the relevant owner→fn permission edge.
- **Multi-path admin where one path is fast and another is slow** (vote+timelock vs emergency multisig). Edge-anchor the slow path's delay on the timelock's edge; the fast path's edge stays uncapped. Replaces the legacy `scopedTo: { type: "admin" }` approach (rule 3) with the principled new pattern.
- **Veto-only or role-restricted constraints** (`type: "other"` with a `label`) that only apply to one specific owner. Anchor on that owner's edge.

> Rule 3's `scopedTo: { type: "admin" }` is the **legacy** way to express most of these. It still works and existing data is not migrated. For **new** authoring of caller-relationship constraints, prefer rule 4 — the dedup model handles per-owner display correctly without leaking onto dependency rows.

---

**Upgrade functions (`upgradeTo`, `upgradeToAndCall`) are special.** They have no outgoing call-graph edges themselves but semantically grant the full impl reach. The engine now seeds upgrade-function BFS with every caller function on the contract, so a cap on the leaf or on a direct caller of the dependency propagates correctly into upgrade-function views. Just place your caps correctly per rules 1–4 — the engine handles the rest.

---

### Phase 3c: Present the plan

Before applying, print the complete plan for researcher review:

```
MITIGATIONS + CAPS TO APPLY:

setDefaultCap  (no-impact, future-only, $0 funds):
  mit: [other / future-only] "Cap is read when ensureDefault() runs; never touches existing balances."
  cap: none

setRedistributor  (critical, $12.4M reachable, asset-bounded):
  mit: [other / asset-bounded]  "Routes excess WETH to an arbitrary address; bounded to pool WETH balance."
  cap: fieldRef CLGauge.poolWethBalance (unit: token WETH)
  placement: direct, on this function

Pool.borrow  (critical, direct Kelp caller):
  mit: [other / bounded-by-spark-rsETH-pool]  scopedTo: { LRTOracle, dependency }
  cap: fieldRef LRTOracle.sparkRsETHPoolBalance (unit: token rsETH)
  placement: rule 2 — direct Kelp caller per call-graph (resolvedAddress == LRTOracle)

Executor.execute  (critical, caller-relationship timelock):
  mit: [delay / 7d]  via edge mitigation on (DAO → Executor.execute | permission)
  cap: none (relationship constraint; no value bound)
  placement: rule 4 — setEdgeMitigation in call-graph-overrides.json; NO func.delay on the function

vault.submitCap  (unscored, caller-relationship timelock × 2 owners):
  mit: [delay / 7d]  via edge mitigations on (curator → vault.submitCap) AND (owner → vault.submitCap)
  cap: none
  placement: rule 4 — two setEdgeMitigation rules; see /review-morpho-vault SCORING_TABLE.md § Step 7c template
```

**Wait for explicit user confirmation before proceeding.** If a new handler or tokenInfo seeding is needed, list those as separate actions to confirm.

---

### Phase 4: Apply scores and mitigations

```bash
curl -s -X PUT "localhost:2021/api/projects/<project>/functions" \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<address>",
    "functionName": "<name>",
    "score": "critical",
    "mitigations": [...]
  }'
```

Rules:
- Send the **full** mitigations array (API replaces it). To append, GET current function data first and concat.
- Preserve existing `score` if this call is just adding mitigations, and vice versa.
- Do not change scores the user has explicitly overridden.
- Do not add access control as a mitigation (captured by `ownerDefinitions`).

For non-permissioned leaf functions (e.g. `LRTOracle.getAssetPrice` — the external dependency's entry), still use the same endpoint, with `isPermissioned: false` alongside the mitigation. The engine indexes it for transitive propagation.

---

### Phase 5: Verify caps resolve

**Do not skip this phase.** Until you've checked the numbers drop, you don't know the cap is working.

```bash
# 1. Recompile the review
curl -s -X POST "localhost:2021/api/projects/<project>/compile-review" > /dev/null

# 2. Check admin view for the affected function's caps
curl -s "localhost:2021/api/projects/<project>/admins?contract=<address>" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
for a in d['admins']:
    for f in a['functions']:
        for rc in f.get('reachableContracts', []):
            cap = rc.get('effectiveCapUsd')
            if cap is not None:
                print(f\"{a['name']} / {f['functionName']} -> {rc['contractName']}: funds=\${rc['fundsUsd']:,.0f} cap=\${cap:,.0f}\")"

# 3. Check dependency view if you placed dep-scoped caps
curl -s "localhost:2021/api/projects/<project>/dependencies" | python3 -c "
import json, sys
d = json.load(sys.stdin)
for dep in d['dependencies']:
    reach = dep.get('totalFundsAtRisk',0) + dep.get('totalTokenValueAtRisk',0)
    print(f'{dep[\"name\"]}: \${reach:,.0f}')"
```

**Interpret the result:**
- `effectiveCapUsd = None` with a cap expected → cap isn't resolving. Most common causes:
  1. Token not tagged `isToken: true`.
  2. `tokenInfo` missing from `funds-data.json` (endpoints service not running or fetch failed — seed manually).
  3. Field missing from `discovered.json` (handler not run or failed — re-run `l2b discover`).
  4. Address case mismatch (check checksum).
- Dep reach didn't drop → check that your direct-caller detection caught the right set. Run the `resolvedAddress` grep and diff against what you applied.
- Admin direct funds still unbounded → the admin's direct callees may still need scoped self-caps (`scopedTo.type: admin`).

If anything is wrong, fix and re-verify. **Never end the run with caps that don't resolve.**

---

### Phase 6: Explicit mitigation + cap roster (mandatory)

**At the end of every run, list every mitigation written and every cap resolved, so the researcher can audit without re-reading any file.** Never summarize this away. Format:

```
═══════════════════════════════════════════════════════════════════════════════
MITIGATIONS + CAPS APPLIED — <project> / <contract or --all>
═══════════════════════════════════════════════════════════════════════════════

1. Pool.borrow  (eth:0xC13e...)
   score: critical
   mits:
     [other / bounded-by-spark-rsETH-pool]  scopedTo: LRTOracle (dependency)
         cap: fieldRef LRTOracle.sparkRsETHPoolBalance  unit: token rsETH
         resolved: $37,915.39  (15.32 rsETH × $2,475.48)
   placement: direct Kelp caller (call-graph resolvedAddress == LRTOracle)

2. EmissionManager.setClaimer  (eth:0xf09e...)
   score: critical
   mits:
     [other / rewards-only]  global
         cap: fieldRef EmissionManager.rewardsVaultWstETHBalance  unit: token wstETH
         resolved: $76,049.71  (26.47 wstETH × $2,873.10)
     [other / per-user reward theft]  global
         cap: (inherits rewards-only cap)
         resolved: $76,049.71

3. LRTOracle.getAssetPrice  (eth:0x349A...)  [non-permissioned leaf]
   score: n/a
   mits:
     [other / bounded-by-spark-rsETH-pool]  global
         cap: fieldRef LRTOracle.sparkRsETHPoolBalance  unit: token rsETH
         resolved: $37,915.39

4. (... every mit, every cap, with its resolved USD value)

─────────────────────────── NEW HANDLERS ADDED ───────────────────────────

- EmissionManager.rewardsVaultWstETHBalance
    wstETH.balanceOf(0x8076...)  →  26.47 wstETH
- LRTOracle.sparkRsETHPoolBalance
    rsETH.balanceOf(0x856f1Ea7...)  →  15.32 rsETH

─────────────────────────── TOKEN TAGS ADDED ─────────────────────────────

- eth:0x7f39C581... (wstETH)  isToken: true
- eth:0xA1290d69... (rsETH)   isToken: true

─────────────────────────── VERIFICATION ────────────────────────────────

Admin totals (before → after):
  2/3 GnosisSafe (EmissionManager admin):   $1.90B → $76K   ✓
Dependency totals (before → after):
  LRTOracle (Kelp):                          $3.35B → $102K ✓
  RSETHExchangeRateOracle (Spark wrapper):   $1.90B → $76K  ✓

Unresolved caps: 0   (all caps resolved to concrete USD)

═══════════════════════════════════════════════════════════════════════════════
```

Any cap that failed to resolve (`effectiveCapUsd = None`) must appear in an "UNRESOLVED CAPS" section with the reason — never hide it. The roster is the contract of the run: if it isn't in the roster, it wasn't applied.

---

### Phase 7: Short summary (familiar format)

After the roster, a one-screen summary for workflow scanning:

```
═══════════════════════════════════════════════════
CONTRACT SCORING: CLGaugeFactory (base:0xB630...)
═══════════════════════════════════════════════════

Permissioned functions: 6
  Skipped (fully reviewed): 1
  Analyzed: 5

RESULTS:
  Function              Score       Containment      Mits  Cap
  ─────────────────────────────────────────────────────────────────
  setDefaultCap         no-impact   future-only      1     —
  setRedistributor      critical    asset-bounded    1     $12.4M
  setNotifyAdmin        critical    (uncapped)       0     —
  bulkUpdateFees        no-impact   future-only      1     —

Scores set: 4 (2 critical, 2 no-impact)
Mitigations added: 3
Caps resolved: 1 ($12.4M)
New handlers: 0
Token tags: 0
═══════════════════════════════════════════════════
```

---

## `--all --interactive` Mode

When invoked as `/score-contract <project> --all --interactive`:

### Phase 0: Build contract queue

Read `functions.json` and extract contracts with at least one permissioned function where `score == "unscored"` or `mitigations` is missing and the function has fund impact. Sort by name; print the queue.

### Loop: Process each contract

For each contract:
1. Print header: `Processing contract 3/17: CLGaugeFactory (base:0xB630...)`
2. Execute Phases 1–3c
3. Present plan; wait for confirmation
4. On confirm: Phase 4 (apply) → Phase 5 (verify) → Phase 6 (roster for this contract) → progress print
5. On adjust: apply corrections then continue
6. On skip: don't save, move on

### Final aggregate roster

After all contracts, emit **one consolidated Phase 6 roster** spanning the whole run: every mitigation written, every cap resolved, every new handler, every token tag, every before/after admin+dep total. Then the short summary.

---

## Anchor reminders

- **Triage, then read proportionately.** Most functions resolve from signature + a glance. Reserve the full storage-trace for cases where the fast verdict feels uncertain.
- **Load source once per contract.** Scan for shared bounds and global state flags up front; reuse across every function on that contract.
- **Name the containment.** Pick whatever short label describes what's actually bounded. The label is half the researcher value.
- **Cap with fieldRef + token-unit when you can verify the relationship holds.** Hardcoded USD caps rot. Uncapped `critical` with a good description beats a fragile cap.
- **Direct callers only for scoped dep-caps.** The engine propagates transitively from there.
- **Verify before finishing.** A cap that doesn't resolve is worse than no cap.
- **Print the roster.** Mitigations and caps that aren't explicitly listed at end-of-run effectively don't exist from the researcher's perspective.
- **Don't pattern-match from past projects.** The calibration examples are illustrative of classes of mistake; each protocol will have its own variants. Trust the source over the function name.
