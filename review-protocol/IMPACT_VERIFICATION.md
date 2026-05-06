# Admin Impact Verification

Step 7 of `/review-protocol`. By this point you've run `/score-contract --all --interactive` and every permissioned function has a score + (most have) mitigations. The detected admins are also in place — `/admins` returns each admin with `totalDirectCapital`, `totalReachableCapital`, `uniqueContractsAffected`, and the per-function reach.

The numbers in `/admins` are correct *given the scoring*. They are **not necessarily correct given the protocol** — the score-contract pass is per-function and per-contract; it doesn't reason about whether two admins both gating a critical function actually compose AND or OR, doesn't always notice that the "critical" worst case for an admin is bounded by a state field one indirection away, and can over-flare reach through view-only or DoS-only paths.

This step is where you make the report **factually correct**. For every admin whose detected impact is non-trivial, ask: *if this admin went malicious right now, is the on-chain damage actually $X, or is the real bound something smaller and observable?* Where the answer is "smaller and observable", add a fieldRef-backed `impactCap`. Where the answer is "the path is real but goes through a non-extracting sink", add a scoped mitigation that says so.

---

## What you have to work with

```bash
# The full ranked admin breakdown
curl -s "localhost:2021/api/projects/$0/admins" > /tmp/iv-$0-admins.json

# The dependencies view (cross-reference for what each admin reaches via shared deps)
curl -s "localhost:2021/api/projects/$0/dependencies" > /tmp/iv-$0-deps.json

# Funds-data (so you can sanity-check claimed capital against actual on-chain balances)
curl -s "localhost:2021/api/projects/$0/funds-data" > /tmp/iv-$0-funds.json

# The call graph (resolved + unresolved edges)
ls packages/config/src/projects/$0/call-graph-data.json

# Discovered values for the protocol — the source of truth for fieldRef caps
ls packages/config/src/projects/$0/discovered.json

# Flattened source for every protocol contract (used by the same path /score-contract reads)
ls packages/config/src/projects/$0/.flat/
```

---

## The verification loop

For each admin in `/admins`, sorted by `totalDirectCapital + totalReachableCapital` descending:

### 1. Read the admin's effective surface

Pull every function the admin owns directly:

```bash
python3 -c "
import json
d = json.load(open('/tmp/iv-$0-admins.json'))
admin = '<admin-address>'
a = next(a for a in d['admins'] if a['address'].lower() == admin.lower())
for f in a['functions']:
    rc = f.get('reachableContracts', [])
    print(f\"{f['contractName']}.{f['functionName']}: direct=\${f.get('directFundsUsd',0):,.0f}  reach=\${sum(r.get('fundsUsd',0) for r in rc):,.0f}  ({len(rc)} contracts)\")"
```

Anything not a fast-lane verdict from `/score-contract` (transferOwnership, renounceOwnership, an obvious upgrader) deserves a second look here.

### 2. Look at the source of the high-impact function

Don't trust `/score-contract`'s mitigation prose alone. Open the flattened source and trace the worst-case path yourself. The questions are the same as `/score-contract`'s deep lane, but with the admin's *full* set of functions in context — sometimes one admin holds two functions that compose into a worse path than either alone.

```bash
ls packages/config/src/projects/$0/.flat/ | head
# Find the contract owning the high-impact function, open its source.
```

For each function with non-trivial reach, write down (in your head, not a file) the answer to:

- **What state does this function write?** (Same as Q1 in `/score-contract`.)
- **Where is that state read?** Grep across `.flat/` — not just the owning contract — for the storage variable. The read sites are the only thing that matter for impact; ignore the function name.
- **What's the worst extraction / devaluation / freeze sink that those read sites reach?** Name a concrete sink: `"PoolConfigurator.setReserveBorrowing(asset, false) — disables new borrows on the named reserve"`, `"AToken.transferUnderlyingTo(arbitrary, amount) — pulls from pool reserves"`, etc.
- **What on-chain field bounds the worst case?** This is the cap candidate. See "Picking the right bounding field" below.

### 3. Decide what to do

The decision tree:

| Finding | Action |
|---|---|
| Worst case is a concrete extraction sink with a clear on-chain bound (a balance, a cap, a debt summary) | **Add a fieldRef impact cap** scoped per the placement rules in `/score-contract`'s Phase 3b. |
| Worst case is real but the extraction sink lives in a downstream contract that's **already** capped by an existing mitigation | Verify the existing cap's `effectiveCapUsd` propagates correctly to this admin's reach via forward-BFS. If yes, leave it. If no, the existing cap was placed wrong (probably on the leaf only and not on the direct caller) — fix the placement. |
| Worst case path doesn't reach committed funds (future-only state, view-only call, recovery from accidental sends, hook-swap that doesn't touch principal) | Re-score the function `no-impact` with a specific containment label. Don't write `governance-gated` or `<role>-gated` — that's permission re-stating. Write the actual reason existing funds are safe. |
| `/score-contract` already wrote a mitigation that's vague ("admin can change parameter") | Rewrite the description to name the concrete sink and bound. Replace generic prose with something a reader couldn't derive from the function name alone. |
| Detected reach goes through a clearly non-extracting path (e.g., a `cancelProposal` that flips a state but never executes the cancelled action) | Re-score `no-impact` with a `veto-only` / `payload-veto` label and a one-sentence description naming what's blocked. Don't accept BFS reach as evidence here. |

### 4. Add the cap (fieldRef, never hardcoded)

**Hardcoded USD caps are forbidden in this skill.** The whole point is that the cap auto-updates with on-chain state. If the bounding field doesn't already exist in `discovered.json`, **add a discovery handler** to `config.jsonc`:

```jsonc
"<chain>:0x<anchor-contract>": {
  "fields": {
    "<bound-field-name>": {
      "description": "<what this bounds, for researcher clarity>",
      "handler": {
        "type": "call",
        "method": "function balanceOf(address) view returns (uint256)",
        "args": ["<chain>:0x<holder>"],
        "address": "<chain>:0x<token>"
      }
    }
  }
}
```

Then re-discover and confirm the field populated:

```bash
cd packages/config && l2b discover $0 --dev
python3 -c "
import json
d = json.load(open('packages/config/src/projects/$0/discovered.json'))
e = next(e for e in d['entries'] if e['address'].lower() == '<anchor-lower>')
print(e['values'].get('<bound-field-name>'))"
```

Then write the cap as a mitigation on the appropriate function (per `/score-contract` Phase 3b):

```json
{
  "type": "other",
  "label": "<containment label>",
  "description": "<specific worst-case + bound + what's protected>",
  "impactCap": {
    "value": {
      "mode": "fieldRef",
      "contractAddress": "<chain>:0x<anchor>",
      "fieldName": "<bound-field-name>"
    },
    "unit": {
      "kind": "token",
      "tokenAddress": "<chain>:0x<token>"
    }
  }
}
```

Tag the unit token `isToken: true` if it isn't already, so `funds-data.json` carries its price + decimals:

```bash
curl -s -X PUT "localhost:2021/api/projects/$0/contract-tags" \
  -H "Content-Type: application/json" \
  -d '{"contractAddress":"<chain>:0x<token>","isExternal":true,"entity":"<issuer>","isToken":true}'
```

### 5. Verify the cap resolves

After every cap added, query `/admins` and check `effectiveCapUsd` propagates to the admin's reach. The endpoint reads `ProjectAnalysis` live from `permission-overrides.json` + `discovered.json` + `funds-data.json`, so no `compile-review` is needed here — `compile-review` only refreshes the static `compiled-review.json` consumed by the frontend, and isn't relevant to the verification.

```bash
curl -s "localhost:2021/api/projects/$0/admins" | python3 -c "
import json, sys
d = json.load(sys.stdin)
a = next(a for a in d['admins'] if a['address'].lower() == '<admin-lower>')
for f in a['functions']:
    for rc in f.get('reachableContracts', []):
        if rc.get('effectiveCapUsd') is not None:
            print(f\"  {f['functionName']} -> {rc['contractName']}: funds=\${rc['fundsUsd']:,.0f} cap=\${rc['effectiveCapUsd']:,.0f}\")"
```

`effectiveCapUsd: None` with a cap expected means the cap didn't resolve. Common causes (in order of frequency):

1. Token not tagged `isToken: true` → funds-data has no `tokenInfo` for it.
2. Field missing from `discovered.json` (handler not run, handler errored, address case mismatch).
3. Wrong unit kind: hardcoded amounts in `unit: token` mode are *whole tokens*; fieldRef amounts in the same mode are *raw on-chain units*. Mixing them silently produces wrong USD.
4. Wrong placement (cap on the leaf but the leaf isn't on the BFS path the admin reaches — see Phase 3b's "1/2/3 placement").

Don't end the verification of an admin with caps that show `effectiveCapUsd = None`.

---

## Picking the right bounding field

The right field changes per protocol. The classes that recur:

- **Balance of a specific holder.** A drain function pulls from one named account (a treasury, a strategy contract, a rewards vault). Field: `<token>.balanceOf(<holder>)`. Add via a `call` handler if not discovered.
- **A discovered cap field.** The protocol has an on-chain `supplyCap`, `borrowCap`, `mintLimit`, `bucketCapacity`, etc. that directly bounds extractable amount. Field: the cap itself.
- **The target contract's own holdings.** A drain function pulls from `address(this)`. Field: `<token>.balanceOf(<targetContract>)`.
- **An allowance.** The caller can only move what's been pre-approved by a separate party. Field: `<token>.allowance(<owner>, <spender>)`. Common with rewards-distribution patterns.
- **A computed aggregate.** Total debt of a reserve, TVL of a pool, totalSupply of an aToken. Some protocols expose this as a single getter (`getReserveData`), others need a multi-call aggregator. Use an existing aggregate handler if one fits (`aaveMarketAggregateTvs`, `metaMorphoCap`); write a dedicated handler if not.

Sanity-check the value with `cast` before wiring the cap:

```bash
cast call <TOKEN> "balanceOf(address)(uint256)" <HOLDER> --rpc-url <RPC>
```

Pull-vs-push mismatches are a classic trap: capping at a strategy's own balance reads zero if the strategy pulls from a separate vault on demand. The effective bound is `min(vault.balanceOf, vault→strategy.allowance)` — pick the bound that survives the strategy being re-funded mid-attack.

---

## Don't over-cap

A fieldRef cap works when the relationship "worst-case value moveable ≤ field value" is obviously true from the source. If the cap requires assumptions about attacker behavior, sequencing, or protocol-level invariants that aren't in the source, prefer **uncapped `critical` with a precise descriptive mitigation** over a fragile cap. A wrong cap is worse than no cap — it makes the admin look safer than it is.

Two specific anti-patterns:

- **Don't use a "max possible call" cap as if it were a single-step cap.** If the admin can call the function repeatedly within a tx, the per-call cap × ∞ ≠ a real bound. Use the contract's own balance / supply, not the per-call max.
- **Don't cap at a flow rate when the attacker can drain accumulated state.** If a rate-limit is on *new emissions* but a vault holds 6 months of pre-emitted rewards, the relevant bound is the vault balance, not the rate.

---

## Roster

After all admins are reviewed, print the same Phase 6 roster format that `/score-contract` uses, scoped to the changes you made in this step. Include:

- Every admin re-evaluated, with before/after `totalDirectCapital + totalReachableCapital`.
- Every cap added (fieldRef + resolved USD).
- Every new handler added to `config.jsonc`.
- Every function that was re-scored (e.g. moved from `critical` to `no-impact` after the factual check).
- Any admin you left at high reach because the path is genuinely uncapped — flag those for the researcher.

Then proceed to Step 8 (governance).
