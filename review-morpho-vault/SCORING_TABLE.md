# Canonical MetaMorpho Scoring Table

The MetaMorpho codebase is fixed (`morpho-org/metamorpho` v1). Every permissioned function on the vault has a known impact profile derived from the source contract. **Apply this table directly via the API** instead of re-running the full impact-analysis loop. Only `submitCap` and `submitMarketRemoval` are vault-specific and require researcher attention.

This file is a reference loaded by `/review-morpho-vault` at scoring time. It is the contract: if a function isn't here, it isn't auto-scored.

---

## Placeholders used below

- `<vault>` — vault address (lowercase chain-prefixed, e.g. `eth:0xbeef...`).
- `<vault-checksum>` — vault address in ERC-55 checksum form, used inside `mitigatedField` / `delay` references where it appears in researcher-facing UI.

The API accepts either case for `contractAddress`, but checksums in `mitigatedField` / `delay` keep the JSON readable when the researcher inspects `functions.json` later.

---

## The table

| # | Function | Owner paths | Delay | Score | Mitigation |
|---|---|---|---|---|---|
| 1 | `setCurator` | `$self.owner` | — | `no-impact` | — |
| 2 | `setIsAllocator` | `$self.owner` | — | `no-impact` | — |
| 3 | `setSkimRecipient` | `$self.owner` | — | `no-impact` | — |
| 4 | `submitTimelock` | `$self.owner` | `<vault>.timelock` | `no-impact` | `valueRange` 1..14 Days |
| 5 | `setFee` | `$self.owner` | — | `no-impact` | `valueRange` 0..50 % against `<vault>.fee` |
| 6 | `setFeeRecipient` | `$self.owner` | — | `no-impact` | — |
| 7 | `submitGuardian` | `$self.owner` | `<vault>.timelock` | `no-impact` | — |
| 8 | `submitCap` | `$self.curator`, `$self.owner` | `<vault>.timelock` | `unscored` (researcher) | — |
| 9 | `submitMarketRemoval` | `$self.curator`, `$self.owner` | `<vault>.timelock` | `unscored` (researcher) | — |
| 10 | `setSupplyQueue` | `$self.allocators`, `$self.curator`, `$self.owner` | — | `no-impact` | future-only label optional |
| 11 | `updateWithdrawQueue` | `$self.allocators`, `$self.curator`, `$self.owner` | — | `no-impact` | future-only label optional |
| 12 | `reallocate` | `$self.allocators`, `$self.curator`, `$self.owner` | — | `no-impact` | label `cap-bounded` (per-market caps) |
| 13 | `revokePendingTimelock` | `$self.guardian`, `$self.owner` | — | **(omit `score`)** | `other` "Veto-only" **scopedTo guardian admin** |
| 14 | `revokePendingGuardian` | `$self.guardian`, `$self.owner` | — | **(omit `score`)** | `other` "Veto-only" **scopedTo guardian admin** |
| 15 | `revokePendingCap` | `$self.curator`, `$self.guardian`, `$self.owner` | — | **(omit `score`)** | `other` "Veto-only" **scopedTo guardian admin** |
| 16 | `revokePendingMarketRemoval` | `$self.curator`, `$self.guardian`, `$self.owner` | — | **(omit `score`)** | `other` "Veto-only" **scopedTo guardian admin** |
| 17 | `transferOwnership` | `$self.owner` | — | `critical` | (Ownable2Step `pendingOwner`) |
| 18 | `renounceOwnership` | `$self.owner` | — | `critical` | (uncapped) |
| 19 | `setName` *(V1_1)* | `$self.owner` | — | `no-impact` | ERC20 metadata only |
| 20 | `setSymbol` *(V1_1)* | `$self.owner` | — | `no-impact` | ERC20 metadata only |
| — | `acceptCap`, `acceptGuardian`, `acceptOwnership`, `acceptTimelock` | — | — | `isPermissioned: false` | — |
| — | `skim`, `multicall`, `deposit`, `mint`, `withdraw`, `redeem`, `transfer`, `approve`, `permit` | — | — | `isPermissioned: false` | — |

### Why rows 13-16 omit `score` instead of `no-impact`

If you score `revokePending*` as `no-impact`, the Guardian admin's `totalReachableCapital` is computed as $0 — the Guardian then renders as a $0 admin card, which understates the role. The Guardian holds veto power over the entire vault's capital flow; if compromised, it can freeze every owner/curator submission until the multisig is replaced. That's not "no impact" — it's "full impact, but the only available action is a veto."

The right output is:
1. **Omit the `score` field** entirely on rows 13-16 (default = `critical` for capital purposes).
2. **Add a `type: "other", label: "Veto-only"` mitigation** that **must include `scopedTo: { address: <guardian-address>, type: "admin" }`**.

The `scopedTo` filter is critical: without it, the Veto-only badge bleeds onto the Owner and Curator cards (they can also call `revokePending*` since these accept "guardian OR owner" or "curator OR guardian OR owner"). With `scopedTo` set to the Guardian admin, the badge renders only on the Guardian's card, where it is the correct semantics.

```json
{
  "type": "other",
  "label": "Veto-only",
  "description": "Only revokes pending owner/curator submissions during the timelock window. Cannot extract funds or initiate state changes.",
  "scopedTo": { "address": "<chain>:<guardian-address>", "type": "admin" }
}
```

> **V1_1 detection:** **Do NOT use the `discovered.json` name** — the on-chain contract name is just `MetaMorpho` for both V1 and V1.1. Instead, grep the source for `function setName(`. If present, apply rows 19–20.

---

## Why most setters score `no-impact`

The vault enforces five hardcoded protocol invariants that bound every owner-touchable parameter:

1. **`MAX_TIMELOCK = 14 days` / `MIN_TIMELOCK = 1 day`** — owner cannot move `timelock` outside this range, so every delay-protected setter has at most a 14-day window before the change can be cancelled by the guardian.
2. **`MAX_FEE = 50%`** — `setFee` cannot push the management fee above 50% of yield. This is yield reduction, not fund extraction.
3. **`MAX_QUEUE_LENGTH = 30`** — `setSupplyQueue` and `updateWithdrawQueue` cannot bloat the market list past 30 entries.
4. **Per-market `cap`** — `reallocate` and the supply-side functions can only move funds into markets that already have a non-zero cap submitted via `submitCap`. The cap itself is timelocked, so a malicious reallocate is bounded by the existing cap.
5. **Guardian veto** — `revokePending*` functions let the guardian (or owner) cancel any pending timelock'd change. This means even if the owner submits a malicious change, the guardian can revoke it before it executes.

None of these can drain depositor capital outright. The worst case is a 50%-of-yield fee, which is contained.

## Why revokePending* are NOT scored `no-impact`

Earlier versions of this skill scored rows 13-16 as `no-impact` because they don't move funds. This is technically correct from a "can this function drain capital?" perspective, but it produces a misleading admin card: the Guardian renders as a $0 admin in the UI, which is wrong — the Guardian holds veto reach over the entire vault TVS.

The convention now is:
- **No `score` field** on rows 13-16 → defaults to critical for capital purposes → Guardian's reachable-capital totals match the vault TVS.
- **Scoped Veto-only mitigation** on each function → the Guardian's admin card shows a "Veto-only" badge that clarifies the constraint without zeroing the impact number.
- **`scopedTo: { type: "admin", address: <guardian> }`** on the mitigation → the badge does NOT bleed onto the Owner or Curator cards, even though both can also call `revokePending*` (their cards correctly show no veto-only badge because their other functions actually move funds).

## Why `submitCap` and `submitMarketRemoval` are left `unscored`

These two functions are the actual research surface for a Morpho vault review. They depend on **which markets the curator is allowed to enable**:

- A curator pushing `submitCap` to a malicious market with a broken oracle can drain via bad-debt accounting on Morpho Blue (the borrower drains the supply side using a manipulated price).
- A curator pushing `submitMarketRemoval` on a market with active borrowers triggers liquidation cascades that shift losses to lenders.

The 1–14 day timelock + guardian veto is the upstream mitigation, but it isn't a structural cap. The researcher must inspect:
- The list of markets currently in `supplyQueue` and their per-market caps.
- The curator's track record (cap-setting history via `SubmitCap` events).
- Whether the guardian is independent enough to actually catch a hostile cap submission.

Surface these two as "researcher must score" in the final report and offer to invoke `/score-contract <slug> <vault>` on them.

## Why `transferOwnership` / `renounceOwnership` are `critical`

The vault uses `Ownable2Step`, so `transferOwnership` only sets `pendingOwner` — the new owner must `acceptOwnership` to take over. That's a structural mitigation but not a cap: a compromised owner can transfer ownership to an attacker who immediately accepts. There is no on-chain bound on what the new owner can then do.

`renounceOwnership` permanently breaks the vault's governance: no further `setFee`, `setCurator`, `submitTimelock`, etc. The vault becomes unable to respond to market emergencies or update parameters. Score `critical`, uncapped.

---

## API payload templates

Use these templates against `PUT localhost:2021/api/projects/<slug>/functions`. Replace placeholders before sending.

### Owner-only setter, no delay, no mitigation

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "setCurator",
    "isPermissioned": true,
    "score": "no-impact",
    "ownerDefinitions": [{"path":"$self.owner"}]
  }'
```

Functions: `setCurator`, `setIsAllocator`, `setSkimRecipient`, `setFeeRecipient`.

### `setFee` (owner, capped at 50%)

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "setFee",
    "isPermissioned": true,
    "score": "no-impact",
    "ownerDefinitions": [{"path":"$self.owner"}],
    "mitigations": [{
      "type": "valueRange",
      "description": "Maximum fee is 50%",
      "valueRange": {
        "min": {"mode":"hardcoded","value":"0"},
        "max": {"mode":"hardcoded","value":"50"},
        "unit": "%"
      },
      "mitigatedField": {
        "contractAddress": "<vault-checksum>",
        "fieldName": "fee"
      }
    }]
  }'
```

### `submitTimelock` (owner, delay-protected, capped 1..14 days)

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "submitTimelock",
    "isPermissioned": true,
    "score": "no-impact",
    "ownerDefinitions": [{"path":"$self.owner"}],
    "delay": {"contractAddress":"<vault-checksum>","fieldName":"timelock"},
    "mitigations": [{
      "type": "valueRange",
      "description": "Timelock can only be set in [1, 14] days",
      "valueRange": {"min":"1","max":"14","unit":"Days"}
    }]
  }'
```

### `submitGuardian` (owner, delay-protected, no value cap)

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "submitGuardian",
    "isPermissioned": true,
    "score": "no-impact",
    "ownerDefinitions": [{"path":"$self.owner"}],
    "delay": {"contractAddress":"<vault-checksum>","fieldName":"timelock"}
  }'
```

### `submitCap` / `submitMarketRemoval` (curator OR owner, delay-protected, unscored)

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "submitCap",
    "isPermissioned": true,
    "score": "unscored",
    "ownerDefinitions": [{"path":"$self.curator"},{"path":"$self.owner"}],
    "delay": {"contractAddress":"<vault-checksum>","fieldName":"timelock"}
  }'
```

Same shape with `submitMarketRemoval`.

### Allocator / curator / owner (no delay)

`setSupplyQueue`, `updateWithdrawQueue`, `reallocate`:

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "reallocate",
    "isPermissioned": true,
    "score": "no-impact",
    "ownerDefinitions": [
      {"path":"$self.allocators"},
      {"path":"$self.curator"},
      {"path":"$self.owner"}
    ]
  }'
```

### Guardian / owner revokes (no delay) — Veto-only mitigation, scopedTo guardian

`revokePendingTimelock`, `revokePendingGuardian`. **Note: no `score` field is sent — default treatment = critical for capital, which is correct since the Guardian holds full veto reach. The badge clarification comes from the `scopedTo` mitigation.**

```bash
GUARDIAN="<chain>:<guardian-address>"
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d "$(cat <<EOF
{
  "contractAddress": "<vault>",
  "functionName": "revokePendingTimelock",
  "isPermissioned": true,
  "ownerDefinitions": [{"path":"\$self.guardian"},{"path":"\$self.owner"}],
  "mitigations": [{
    "type": "other",
    "label": "Veto-only",
    "description": "Only revokes pending owner/curator submissions during the timelock window. Cannot extract funds or initiate state changes.",
    "scopedTo": { "address": "$GUARDIAN", "type": "admin" }
  }]
}
EOF
)"
```

### Curator / guardian / owner revokes — same shape

`revokePendingCap`, `revokePendingMarketRemoval`:

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d "$(cat <<EOF
{
  "contractAddress": "<vault>",
  "functionName": "revokePendingCap",
  "isPermissioned": true,
  "ownerDefinitions": [
    {"path":"\$self.curator"},
    {"path":"\$self.guardian"},
    {"path":"\$self.owner"}
  ],
  "mitigations": [{
    "type": "other",
    "label": "Veto-only",
    "description": "Only revokes pending owner/curator submissions during the timelock window. Cannot extract funds or initiate state changes.",
    "scopedTo": { "address": "$GUARDIAN", "type": "admin" }
  }]
}
EOF
)"
```

### Clearing a previously-set `score` (PUT-merge gotcha)

The `/functions` PUT endpoint **merges** instead of replacing — the backend uses `score ?? existing`, so omitting `score` in the payload **keeps the previous value**. Sending `score: null` doesn't help either (still keeps existing).

If you previously scored a function `no-impact` and now want to switch it to default-critical (e.g. moving rows 13-16 over from an old config), you must **edit `functions.json` directly** to remove the `score` field, then re-run discovery / compile-review:

```bash
python3 -c "
import json
path = 'packages/config/src/projects/<slug>/functions.json'
d = json.load(open(path))
revokes = {'revokePendingTimelock','revokePendingGuardian','revokePendingCap','revokePendingMarketRemoval'}
for addr, c in d['contracts'].items():
    for f in c.get('functions', []):
        if f['functionName'] in revokes and 'score' in f:
            f.pop('score')
json.dump(d, open(path, 'w'), indent=2)
print('cleared score on revoke functions')"
```

This caveat applies to any field that uses `??` semantics in the backend merge logic — see `packages/l2b/src/implementations/discovery-ui/defidisco/functions.ts` for the exact list.

### `transferOwnership` / `renounceOwnership` (critical)

```bash
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H "Content-Type: application/json" \
  -d '{
    "contractAddress": "<vault>",
    "functionName": "transferOwnership",
    "isPermissioned": true,
    "score": "critical",
    "ownerDefinitions": [{"path":"$self.owner"}]
  }'
```

### Public functions (mark explicitly non-permissioned)

So the scan agent doesn't re-flag them on a future re-scan:

```bash
for fn in acceptCap acceptGuardian acceptOwnership acceptTimelock skim multicall deposit mint withdraw redeem; do
  curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
    -H "Content-Type: application/json" \
    -d "{\"contractAddress\":\"<vault>\",\"functionName\":\"$fn\",\"isPermissioned\":false}"
done
```

### Merging deps into `submitCap` / `acceptCap` (Step 7)

**PUT replaces** — to add Chainlink deps + per-feed `impactCap` mitigations to `submitCap` (Step 7), re-send the full payload preserving the score, ownerDefinitions, delay, and original delay-mitigation. Use this template:

```bash
VAULT="<chain>:0xVAULT..."
DEPS='[{"contractAddress":"<chain>:0xFeed1..."}, {"contractAddress":"<chain>:0xFeed2..."}, ...]'
# Build per-feed scoped impactCap mitigations (one per feed)
MITS='[
  {
    "type":"other",
    "label":"Per-market cap",
    "description":"Prices market(s) X — exposure bounded by their USDC supply caps.",
    "scopedTo":{"address":"<chain>:0xFeed1...","type":"dependency"},
    "impactCap":{
      "value":{"mode":"fieldRef","contractAddress":"'$VAULT'","fieldName":"feedCap_0xFeed1..."},
      "unit":{"kind":"scaler","factor":"1e6"}
    }
  }
  // ... one entry per feed ...
]'

# submitCap: preserve score + ownerDefinitions + delay, prepend the original delay mitigation
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H 'Content-Type: application/json' \
  -d "$(python3 -c "
import json
mits = json.loads('''$MITS''')
delay_mit = {'type':'delay','description':'Delay before execution','delayRef':{'contractAddress':'$VAULT','fieldName':'timelock'}}
print(json.dumps({
    'contractAddress':'$VAULT',
    'functionName':'submitCap',
    'isPermissioned':True,
    'score':'unscored',
    'ownerDefinitions':[{'path':'\$self.curator'},{'path':'\$self.owner'}],
    'delay':{'contractAddress':'$VAULT','fieldName':'timelock'},
    'dependencies': json.loads('''$DEPS'''),
    'mitigations': [delay_mit] + mits
}))"
)"

# acceptCap: simpler — non-permissioned + deps + mitigations
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H 'Content-Type: application/json' \
  -d "$(python3 -c "
import json
print(json.dumps({
    'contractAddress':'$VAULT',
    'functionName':'acceptCap',
    'isPermissioned':False,
    'dependencies': json.loads('''$DEPS'''),
    'mitigations': json.loads('''$MITS''')
}))"
)"
```

**Critical schema reminder:** `impactCap` accepts only the new shape — `{value: {mode: "hardcoded"|"fieldRef", ...}, unit: {kind: "usd"|"scaler"|"token", ...}}`. The legacy form `{value: "200000000", unit: "usd"}` (plain strings) silently does **not** resolve and the cap won't bind. Use `unit: {kind: "scaler", factor: "1e6"}` for USDC vaults; switch the factor to `"1e8"` for cbBTC, `"1e18"` for WETH/DAI, etc.

---

## Application order

Apply rows in numeric order (1 → 18). Each PUT is independent; the API replaces the function entry. After all are applied, recompile:

```bash
curl -s -X POST localhost:2021/api/projects/<slug>/compile-review > /dev/null
```

Then verify by reading `<slug>/functions.json` and confirming all 16 permissioned + 2 critical entries are present, and that `submitCap` / `submitMarketRemoval` have `score: "unscored"`.
