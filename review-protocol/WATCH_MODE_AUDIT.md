# Watch-Mode Audit

Step 10 of `/review-protocol`. The final sanity check on which fields the monitor is watching.

`/prune-watch-fields` (Step 3) already classifies fields heuristically — names like `nonce`, `totalSupply`, `lastAccrualTime` get auto-promoted to `ignoreInWatchMode`. But heuristics miss protocol-specific accumulators (a custom `cumulativeRewardIndex_X`, a per-asset `lastUpdateTimestamp` map, etc.). The only reliable way to find them is to **observe what actually changes between two real RPC reads** and treat every change-by-default as a candidate.

The pattern: snapshot now → re-discover live → diff → propose ignores.

---

## 0. Confirm the project's discovered.json reflects committed state

`l2b discover --dev` pins to a saved timestamp; everything else uses live RPC. For this step you want `discovered.json` to reflect the most recent live state — usually the `--dev` run from earlier in the pipeline. Check:

```bash
python3 -c "
import json
d = json.load(open('packages/config/src/projects/$0/discovered.json'))
print('timestamp:', d.get('timestamp'))
print('blockNumber:', d.get('blockNumber'))"
```

If the timestamp is more than ~24 hours old or the block lags significantly, run `l2b discover $0 --dev` once to refresh the saved snapshot. The audit only makes sense on a recent baseline.

## 1. Snapshot the current discovered.json

```bash
SNAP="/tmp/wma-$0-before.json"
cp "packages/config/src/projects/$0/discovered.json" "$SNAP"
echo "snapshot saved: $SNAP"
```

## 2. Run discovery WITHOUT --dev (live RPC, current timestamp)

The point of this run is to read every field at *now* instead of at the snapshot's timestamp. Anything that ticks every block, every Chainlink heartbeat, every rebase, every interest accrual will diff:

```bash
cd packages/config && l2b discover $0 2>&1 | tee /tmp/wma-$0-output.txt
```

If discovery errors (RPC rate-limit, transient revert), retry once. If it consistently errors, the audit can't run — fix discovery first.

## 3. Diff the field values

```bash
python3 <<'PY'
import json
before = json.load(open('/tmp/wma-$0-before.json'))
after  = json.load(open('packages/config/src/projects/$0/discovered.json'))

bmap = {e['address'].lower(): e for e in before['entries']}
amap = {e['address'].lower(): e for e in after['entries']}

changed = []  # (address, name, field, before, after)
for addr, ae in amap.items():
    be = bmap.get(addr)
    if not be: continue
    bv = be.get('values', {})
    av = ae.get('values', {})
    for k, av_val in av.items():
        bv_val = bv.get(k, '<MISSING>')
        if json.dumps(bv_val, sort_keys=True) != json.dumps(av_val, sort_keys=True):
            changed.append((addr, ae.get('name','?'), k, bv_val, av_val))

# Group by contract
by_contract = {}
for addr, name, k, b, a in changed:
    by_contract.setdefault((addr, name), []).append((k, b, a))

print(f'\n{len(changed)} field changes across {len(by_contract)} contracts\n')
for (addr, name), fields in sorted(by_contract.items()):
    print(f'\n{name}  ({addr})')
    for k, b, a in fields:
        # Truncate long values
        bs = str(b)[:60] + ('...' if len(str(b))>60 else '')
        as_ = str(a)[:60] + ('...' if len(str(a))>60 else '')
        print(f'  {k}:  {bs}  →  {as_}')
PY
```

The output is your candidate list. Save it to `/tmp/wma-$0-changes.txt` for reference.

## 4. Classify each change

For every changed field, decide:

- **TICKING — add to ignoreInWatchMode** — value moves on its own (block-by-block, heartbeat, interest accrual, rebase). The change between two arbitrary RPC reads is informationally noise, not a security signal.
  - Counters / nonces / sequence numbers
  - Chainlink `latestAnswer` / `latestRoundData` / `aggregator` outputs
  - Index accumulators (`liquidityIndex`, `borrowIndex`, `cumulativeRewardPerToken`)
  - Pool reserves / `totalSupply` / `totalAssets` / `lastTotalAssets`
  - Per-block prices / TWAPs / `sqrtPriceX96`
  - Timestamps that reflect "last action" rather than admin policy (`lastAccrualTime`, `lastUpdateTimestamp`)

- **KEEP WATCHING — security-critical** — change indicates a governance / admin / state action and *should* fire an alert. Examples:
  - `owner`, `pendingOwner`, `admin`, `pauser`, `guardian`
  - `accessControl.<ROLE>.members`
  - Multisig `$members` / `$threshold`
  - `$implementation`, `$admin`, `$pastUpgrades`, `$upgradeCount`
  - Rate parameters (fee rates, LTV, oracle params, caps)
  - `paused` / `frozen` / `live` flags

- **NEEDS THOUGHT — flag for the researcher** — value moved but the semantics are unclear from the field name alone. Print these in the report and don't auto-promote.

You should already know most of these classes from `/prune-watch-fields`'s logic — apply the same Tier 1 / Tier 2 / Tier 3 rules. The difference here is that you're working from *empirical evidence* of what actually moves, not name-based heuristics, so don't second-guess: if a field changed and it isn't in the security-critical list above, default to TICKING.

Cross-check: any TICKING field that's already in a contract's `ignoreInWatchMode` is fine — no action needed. Don't double-add.

## 5. Apply via config.jsonc

For each TICKING field that isn't already ignored, add to the appropriate contract's `ignoreInWatchMode` array in `config.jsonc`. Same merge rules as `/prune-watch-fields` Step 4:

- Never remove existing entries
- Append, don't overwrite (no duplicates)
- Sort alphabetically inside each array
- Preserve JSONC comments

If the contract has no override block, create one with just `ignoreInWatchMode`. If it has an override block but no `ignoreInWatchMode`, add the property.

**Don't bulk-apply.** In interactive mode, present the candidate list grouped by contract first and ask the user to confirm. In `--auto` mode, apply TICKING fields automatically and print the NEEDS THOUGHT list under a dedicated section in the final report.

## 6. Re-run discovery once more to sync

After updating `config.jsonc`, run discovery one final time so `discovered.json` reflects the new `ignoreInWatchMode` settings (which the monitor reads):

```bash
cd packages/config && l2b discover $0 --dev 2>&1 | tail -5
```

`--dev` is fine here — you're not trying to observe more change, just sync the file with the new config.

## 7. Report

```
WATCH-MODE AUDIT — $0

  Snapshot timestamp:  <before>
  Live timestamp:      <after>
  Block delta:         <after-block> - <before-block> = <N> blocks

  Total field changes: N across M contracts

  Auto-promoted to ignoreInWatchMode:
    <ContractName>:  <field1>, <field2>, <field3>
    ...

  Already in ignoreInWatchMode (no action):
    <count> fields

  NEEDS RESEARCHER REVIEW:
    <ContractName>.<field>:  <before> → <after>
       Reason flagged: <one-sentence why this isn't an obvious ticker>
    ...

  KEPT UNDER WATCH (security-critical):
    <ContractName>:  <field1>, <field2>
    ...
```

Cleanup:

```bash
rm -f /tmp/wma-$0-before.json /tmp/wma-$0-output.txt /tmp/wma-$0-changes.txt
```

---

## Why this step is non-negotiable

Without it, the monitor will fire Discord alerts on every Chainlink heartbeat, every interest accrual, every nonce tick — researchers stop reading the alerts, and the *real* governance event (an `owner` change, an unscheduled upgrade) gets buried in noise. The cost of a missing real alert is much higher than the cost of one extra discovery cycle here.

The audit is also the only step in the pipeline that catches protocol-specific accumulators that the heuristic-based `/prune-watch-fields` doesn't recognize by name. Don't skip it on the grounds that `/prune-watch-fields` already ran.
