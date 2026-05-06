# Funds Audit

The new step that runs right after discovery and before pruning watch fields. The goal: confirm the protocol's discovered fund-holding contracts, when fetched, sum to a USD figure that **matches the protocol's published TVL within reason**. If the numbers don't reconcile, fix the funds tagging — add `fetchBalances` / `fetchPositions` to a missed contract, or build an aggregate handler when a single tag can't represent N similar pools.

This step exists because the funds-data feeds every TVS / capital number in the rest of the pipeline. A wrong baseline here silently wrecks `/score-contract`'s prioritization, the impact-cap caps in Step 7, and the final review's headline numbers.

---

## 0. Establish the expected TVL

Before fetching anything, decide what the answer *should* be. The right source depends on the protocol — pick whichever combination is most credible:

- DeFiLlama protocol page (`/protocol/<slug>`).
- The protocol's own dashboard / analytics page (often the most accurate for a single deployment).
- On-chain reads: a `totalAssets()` on a vault, sum of `aToken.totalSupply() × price` for an Aave market, etc.
- The Graph subgraphs the protocol publishes.

You don't need a single canonical number. A range is fine — **"~$120-140M for this Base deployment"** is enough to spot a 90% undercount or a 5× overcount. Write the expected range down in your scratchpad before reading the API result, otherwise it's tempting to retrofit the expectation.

## 1. List the funds-tagged contracts

```bash
curl -s "localhost:2021/api/projects/$0/contract-tags" | python3 -c "
import json, sys
tags = json.load(sys.stdin)
for t in tags if isinstance(tags, list) else tags.get('tags', []):
    flags = []
    if t.get('fetchBalances'): flags.append('balances')
    if t.get('fetchPositions'): flags.append('positions')
    if t.get('isToken'): flags.append('token')
    if t.get('fetchAggregate'): flags.append(f\"aggregate:{t.get('aggregateHandler','?')}\")
    if flags:
        print(f\"  {t['contractAddress']}  ({','.join(flags)})\")"
```

Cross-check against the contract list you've classified during discovery — every contract you'd expect to hold real capital should be in this list with at least one flag. Common omissions:

- A vault / pool / treasury that wasn't recognized as funds-holding by `/run-discovery` (Category D).
- A factory whose pools collectively hold the TVL — needs `fetchAggregate` on the factory, not `fetchBalances`.
- A protocol-owned contract that holds DeFi positions (e.g. a strategy contract supplying into Aave) — needs `fetchPositions`, not just `fetchBalances`.

If anything is missing, tag it via the API now (same shape `/run-discovery` uses):

```bash
curl -s -X PUT "localhost:2021/api/projects/$0/contract-tags" \
  -H 'Content-Type: application/json' \
  -d '{"contractAddress":"<chain>:0x...","fetchBalances":true}'
```

## 2. Fetch funds-data and read the totals

```bash
curl -s -X POST "localhost:2021/api/projects/$0/funds-data/fetch" > /tmp/fa-$0-fetch.txt
tail -20 /tmp/fa-$0-fetch.txt

python3 -c "
import json
d = json.load(open('packages/config/src/projects/$0/funds-data.json'))
total_bal = total_pos = total_agg = total_tok = 0
rows = []
for addr, c in d.get('contracts', {}).items():
    bal = (c.get('balances') or {}).get('totalUsdValue', 0) or 0
    pos = (c.get('positions') or {}).get('totalUsdValue', 0) or 0
    agg = (c.get('aggregate') or {}).get('totalUsdValue', 0) or 0
    tok = (c.get('tokenInfo') or {}).get('tokenValue', 0) or 0
    if bal or pos or agg or tok:
        rows.append((addr, bal, pos, agg, tok, (c.get('positions') or {}).get('source','')))
    total_bal += bal; total_pos += pos; total_agg += agg; total_tok += tok
rows.sort(key=lambda r: -(r[1]+r[2]+r[3]))
for addr, bal, pos, agg, tok, src in rows:
    print(f'  {addr}  bal=\${bal:,.0f}  pos=\${pos:,.0f}  agg=\${agg:,.0f}  tok=\${tok:,.0f}  src={src}')
print()
print(f'  TOTAL balances:    \${total_bal:,.0f}')
print(f'  TOTAL positions:   \${total_pos:,.0f}')
print(f'  TOTAL aggregate:   \${total_agg:,.0f}')
print(f'  TOTAL token value: \${total_tok:,.0f}  (NOT TVS — informational)')
print(f'  TVS (bal+pos+agg): \${total_bal+total_pos+total_agg:,.0f}')"
```

`tokenInfo.tokenValue` is the protocol token's market cap — it is **not** part of TVS. Don't add it to the reconciliation total; mention it separately in the report.

## 3. Reconcile against the expected range

Three verdicts:

### a. Within ~10% of expected — done

Move on to Step 3 of the main pipeline. Note in the final report what the discovered TVS resolved to and what source you reconciled against.

### b. Significantly under — something's missing

Walk the candidates in this order, stop as soon as the gap closes:

1. **A funds-holding contract is untagged.** Open the protocol's docs / address book and confirm every "the protocol holds X here" statement maps to a tagged contract. Tag and re-fetch.
2. **A vault / strategy holds DeFi positions, not idle balances.** `fetchBalances` reads token balances directly held by the address; `fetchPositions` queries DeBank for protocol positions (Aave deposits, Compound supply, Pendle PT/YT, etc). Add `fetchPositions: true` and re-fetch. **Note**: DeBank doesn't support every chain — if positions stay zero on a non-Ethereum chain, that's why.
3. **The protocol is factory-deployed with N similar pools.** `fetchBalances` on the factory returns nothing (the factory holds no funds; the *pools* do). `fetchPositions` is wrong too (DeBank doesn't track factory-deployed vanity pools). This is the aggregate-handler case — see Section 4.
4. **The aggregate-handler total is right but split across too few rows.** A handler returning a single-row $TVL when there are 200 underlying pools is fine for top-line reconciliation but loses per-pool detail. Acceptable as long as the headline matches.

### c. Significantly over — something's double-counted

Common cause: a token contract tagged `fetchBalances` AND included in another contract's `balances` (e.g. an aToken with `fetchBalances: true` whose underlying is *also* counted in the lending pool's reserves). Decide which side owns the counting and untag the other.

Less common but possible: an aggregate handler returns the protocol-wide TVL but only one of N markets in this project should be counted. Refine the handler's filter or split it.

---

## 4. Building a new aggregate handler

This is the part with real engineering. Reach for it only when no combination of `fetchBalances` / `fetchPositions` produces the right number — typically a factory-style protocol with many pools.

### 4a. Pick a data source

Existing handlers use:
- DeFiLlama protocol JSON (no key, easy, current TVL only).
- The Graph subgraph (best fidelity, requires `THEGRAPH_API_KEY`).
- Direct on-chain calls (factory's `allPoolsLength()` etc.).
- Protocol-specific public APIs.

Match what the protocol itself uses to publish its TVL. If the protocol cites DeFiLlama as canonical, use DeFiLlama; if they have a Dune dashboard fed by their subgraph, the subgraph is more reliable.

### 4b. Author the handler

Read the existing handlers as templates — they show the full shape:

- [packages/defiscan-endpoints/src/services/aggregate/handlers/aerodromeV2Factory.ts](packages/defiscan-endpoints/src/services/aggregate/handlers/aerodromeV2Factory.ts) — DeFiLlama TVL + on-chain pool count.
- [packages/defiscan-endpoints/src/services/aggregate/handlers/uniswapV3Factory.ts](packages/defiscan-endpoints/src/services/aggregate/handlers/uniswapV3Factory.ts) — DeFiLlama via chain map.
- [packages/defiscan-endpoints/src/services/aggregate/handlers/uniswapV2Factory.ts](packages/defiscan-endpoints/src/services/aggregate/handlers/uniswapV2Factory.ts) — The Graph subgraph (constructor takes API key).
- [packages/defiscan-endpoints/src/services/aggregate/handlers/frankencoinMintinghub.ts](packages/defiscan-endpoints/src/services/aggregate/handlers/frankencoinMintinghub.ts) — protocol-specific public API.

Implement the `AggregateHandler` interface from [packages/defiscan-endpoints/src/services/aggregate/handlers/types.ts](packages/defiscan-endpoints/src/services/aggregate/handlers/types.ts):

```ts
export interface AggregateHandler {
  name: string                                              // the string used in contract-tags
  fetch(contractAddress: EthereumAddress, chain: string): Promise<AggregateResponse>
}
```

The `name` value is what goes into `aggregateHandler` on the contract tag. Pick a stable kebab-case slug (`<protocol>-<contractType>`, e.g. `aerodrome-v2-factory`).

### 4c. Wire the handler

Four touch points, all required — miss any one and the handler silently does nothing:

1. **Export from the handlers barrel:** add to [packages/defiscan-endpoints/src/services/aggregate/handlers/index.ts](packages/defiscan-endpoints/src/services/aggregate/handlers/index.ts) (`export { NewProtocolHandler } from './newProtocol'`).
2. **Re-export from the service barrel:** add to [packages/defiscan-endpoints/src/services/aggregate/index.ts](packages/defiscan-endpoints/src/services/aggregate/index.ts).
3. **Register in `server.ts`:** add `new NewProtocolHandler()` to the `AggregateService` constructor list in [packages/defiscan-endpoints/src/server.ts](packages/defiscan-endpoints/src/server.ts) (around line 115). If the handler needs an API key, thread it through `config` like `UniswapV2FactoryHandler` does.
4. **Add to the frontend's known list:** append the same kebab-case slug to `KNOWN_AGGREGATE_HANDLERS` in [packages/protocolbeat/src/apps/discovery/defidisco/FundsTagsButton.tsx](packages/protocolbeat/src/apps/discovery/defidisco/FundsTagsButton.tsx). Without this, the dropdown in the Funds Tags UI won't offer the new handler and a researcher trying to set it via the UI will silently fall back to the first known handler.

### 4d. Rebuild and restart

The defiscan-endpoints service AND the l2b CLI both consume aggregate handler results, and protocolbeat needs the updated frontend list. The existing `/build` skill rebuilds protocolbeat + l2b + defiscan-frontend but not defiscan-endpoints, so do that one explicitly.

```bash
# Build the endpoints service (this is the new one /build doesn't cover)
cd /home/emilien/defidisco/packages/defiscan-endpoints && pnpm build && cd -

# Build l2b + protocolbeat (so the frontend list update lands)
/build
```

Then restart both running processes. How you restart depends on how they were launched in your environment — common options:
- If you started them in foreground terminals: kill (Ctrl-C) and restart.
- If they're under a process manager (`pm2`, `systemd`, `tmux` with named windows): use the manager's restart command.

Don't try to swap the dist file under a running node process — node caches modules at require time and the new handler won't be picked up.

After restart, sanity-check the endpoints service is up and the new handler is registered:

```bash
curl -s "localhost:<endpoints-port>/aggregate/handlers" 2>/dev/null || \
  echo "Confirm endpoints service is running on its configured port and the handler list includes <new-name>"
```

### 4e. Tag the contract and re-fetch

```bash
curl -s -X PUT "localhost:2021/api/projects/$0/contract-tags" \
  -H 'Content-Type: application/json' \
  -d '{"contractAddress":"<chain>:0x<factory>","fetchAggregate":true,"aggregateHandler":"<new-name>","aggregateLabel":"<short-display-label>"}'

curl -s -X POST "localhost:2021/api/projects/$0/funds-data/fetch" > /tmp/fa-$0-refetch.txt
```

Re-run the totals print from Step 2 and confirm the aggregate row populated with a non-zero value.

---

## 5. Final reconciliation and report

After tagging / aggregate-handler work converges:

```
FUNDS AUDIT — $0

  Expected TVL:           <range with source>
  Discovered TVS:         $<bal+pos+agg>  (delta: <%>)
  Sources:
    balances:     $<X>  across <N> contracts
    positions:    $<X>  across <N> contracts  (DeBank source: <count>)
    aggregate:    $<X>  via <handler-name>
    token value:  $<X>  (informational, NOT TVS)

  Tags added/changed during audit:
    <addr>  →  fetchBalances
    <addr>  →  fetchPositions
    <addr>  →  fetchAggregate / handler=<name>

  NEW AGGREGATE HANDLER (if any):
    name:        <new-name>
    file:        packages/defiscan-endpoints/src/services/aggregate/handlers/<file>.ts
    data source: <DeFiLlama | The Graph | on-chain | protocol API>
    rebuilt:     defiscan-endpoints, l2b, protocolbeat
    restarted:   <yes / how>

  Coverage gaps:
    <list anything where the audit could not reconcile and the researcher needs to look>
```

**If you authored a new aggregate handler, repeat its name and wiring summary in the final Step 11 report so the researcher has a single place to find every code change made by this run.** Other steps don't author code; this one does, so the visibility matters.