---
name: review-morpho-vault
description: Orchestrate an end-to-end DeFiDisco review of a Morpho MetaMorpho vault from a single vault address. Creates the project, pre-bakes the standard MetaMorpho config (handlers, hardcoded constants, ignored noise), runs discovery, applies known Morpho tags, scans/scores permissioned functions against the standardized MetaMorpho function set, gathers curator-specific resources on top of the canonical Morpho audits/frontends/license, and authors governance.json with a vault-timelock fieldRef.
argument-hint: "<vault-address> [project-slug] [--auto]"
allowed-tools: Bash, Read, Write, Edit, WebSearch, WebFetch
---

# Review Morpho Vault

You are orchestrating an end-to-end DeFiDisco review for a **Morpho MetaMorpho vault**. The goal is to go from a single vault address (e.g. `eth:0xBEEF01735c132Ada46AA9aA4c54623cAA92A64CB`) to a fully-formed project review with discovered contracts, scored permissions, governance, resources, audits, per-feed impact caps, and a compiled review — leveraging everything we already know about MetaMorpho vaults so the human only spends time on the **curator-specific** pieces.

This skill is a meta-orchestrator that calls into the existing skills (`/run-discovery`, `/scan-permissions`, `/score-contract`, `/gather-resources`, `/generate-governance`, `/generate-review`) but **pre-bakes the standardized Morpho-vault patterns** so most of the work resolves to "fill in the curator's name + URLs."

## Arguments

- `$0` (vault-address, required): chain-prefixed address of the MetaMorpho vault, e.g. `eth:0xBEEF01735c132Ada46AA9aA4c54623cAA92A64CB`. If only `0x...` is given, default to `eth:` (Ethereum mainnet).
- `$1` (project-slug, optional): project folder name. If omitted, derive from the vault's `name()` (uppercase first letter of each word, hyphenated, ASCII-only). Example: vault `name()` = `"Steakhouse USDC"` → slug = `Steakhouse-USDC`.
- `--auto` flag: run downstream skills non-interactively where they support it. Default: pause at key checkpoints.

Parse `$ARGUMENTS` to detect these.

## Prerequisites

The l2b UI server must be running at `http://localhost:2021`. If not, tell the user to start it with `cd packages/config && l2b ui` and stop.

---

## Why this skill exists

A Morpho MetaMorpho vault is an **immutable, standardized** ERC4626 wrapper around Morpho Blue lending markets. Every vault deployed from the MetaMorpho codebase has:

- The same on-chain contract surface (one `MetaMorpho` contract → Morpho Blue → per-market oracles → underlying Chainlink price feeds).
- The same set of permissioned functions with the same role hierarchy (owner / curator / guardian / allocator).
- The same hardcoded protocol invariants (`MAX_TIMELOCK = 14 days`, `MIN_TIMELOCK = 1 day`, `MAX_QUEUE_LENGTH = 30`, `MAX_FEE = 50%`).
- The same upstream audits, bug bounty, frontends, docs, GitHub repo, and license — because the contracts come from `morpho-org/metamorpho`.

The **only** per-vault variables are:
- The asset (USDC, WETH, DAI, …) and the curator (Steakhouse, Gauntlet, Re7, …).
- Which Morpho Blue markets it allocates into (auto-discovered from `withdrawQueue` — the full universe of markets the vault may hold positions in, not `supplyQueue` which is just the active deposit target). See "Why withdrawQueue, not supplyQueue" below.
- The owner / curator / guardian / allocator addresses (different multisigs / DAOs).
- The curator's website, docs, GitHub, X handle.
- The curator's governance setup (some are pure multisigs, some have DAOs).
- The per-market USDC supply caps (read on-chain via the `metaMorphoCap` handler).

This skill encodes everything in the first list as defaults and only researches / asks about the second list.

---

## Why withdrawQueue, not supplyQueue (READ THIS — it has caused a 95% TVS undercount)

A MetaMorpho vault exposes two queue arrays:

- **`supplyQueue`** — the *active deposit target*. The vault's `deposit()` flow only allocates new PYUSD/USDC into markets in this list, in the order shown. Curators routinely shrink supplyQueue down to a single market (often just the idle market) when they want to pause new exposure to a market without removing existing positions.
- **`withdrawQueue`** — the *full universe* of markets the vault holds (or has ever held) positions in. Any market with non-zero supply shares from the vault MUST be in withdrawQueue, because the vault's `withdraw()` flow walks this list to find liquidity. This is the canonical "where does the vault hold capital?" list.

**The undercount mode**: a vault with `supplyQueueLength = 1` (idle only) but `withdrawQueueLength = 10` (idle + 9 markets with real positions) will report **just the idle market's capital** if anything walks supplyQueue. We saw a real case (Sentora-PYUSD) where this produced **$11.18M reported vs $230.69M on-chain — 95% undercount**.

Two places use this list and BOTH must read withdrawQueue:

1. **Discovery's `morphoMarkets` handler** (`marketParams` field) — pass `"queueField": "withdrawQueue"` (the template in `templates/config.jsonc.template` already does this, but if you write the override by hand or migrate an older project, double-check). Without it, only supplyQueue's market collateral/oracle stack gets discovered.

2. **Funds-data fetcher's MorphoVaultService** (`packages/defiscan-endpoints/src/clients/MorphoRpcClient.ts`) — fixed in this codebase to walk `withdrawQueue`. If you ever fork this skill or revert that file, the bug returns.

The bug only manifests when **both** of these conditions hold:
- DeBank doesn't track the vault directly (newer vaults fall through to MorphoVaultService — `source: "onchain/debank"` in funds-data.json), AND
- `supplyQueueLength < withdrawQueueLength` (curator narrowed the active deposit target)

When DeBank knows the vault (`source: "debank"` in funds-data.json), DeBank's positions API aggregates correctly across all Morpho positions and the bug doesn't appear.

**Sanity check, mandatory**: see Step 4b — compare `funds-data.totalUsdValue` against on-chain `totalAssets()`. They must agree within ~1%. A divergence is almost always this bug.

---

## Per-chain reference data

| Chain prefix | RPC default | Morpho Blue | PublicAllocator |
|---|---|---|---|
| `eth` | `https://eth.llamarpc.com` | `eth:0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb` | `eth:0xfd32fA2ca22c76dD6E550706Ad913FC6CE91c75D` |
| `base` | `https://base.llamarpc.com` | `base:0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb` | `base:0xA090dD1a701408Df1d4d0B85b716c87565f90467` |
| `arb1` | `https://arbitrum.llamarpc.com` | `arb1:0x6c247b1F6182318877311737BaC0844bAa518F5e` | (look up) |
| `optimism` | `https://optimism.llamarpc.com` | (look up via `MORPHO()`) | (look up) |
| `polygon` | `https://polygon.llamarpc.com` | (look up via `MORPHO()`) | (look up) |

Morpho Blue is generally CREATE2-deployed at `0xBBBB…EFCb` across chains, but verify by reading `MORPHO()` on the vault. PublicAllocator addresses differ per chain — if the table doesn't list one, find it by inspecting the vault's `allocators` array post-discovery for a Morpho-tagged contract that isn't `MorphoHelper`.

## Canonical contract classifier (for `/run-discovery`)

Pass these hints so the model doesn't classify by hand:

| Template / contract name | Auto-classification |
|---|---|
| `morpho/MetaMorpho`, `MetaMorphoV1_1` | core (the vault — the project's `initialAddresses` entry) |
| `morpho/Morpho`, name `Morpho` | external + entity `Morpho` |
| name `MorphoHelper`, `PublicAllocator` | external + entity `Morpho` |
| name `MorphoChainlinkOracleV2` | external + entity `Morpho` (per-market oracle adapters) |
| name `EACAggregatorProxy`, `AccessControlledOCR2Aggregator`, `ChainlinkOracle` | external + entity `Chainlink` |
| `GnosisSafe` matching the vault's owner/curator/guardian/feeRecipient | core (multisig) |
| `GnosisSafe` belonging to Morpho's owner / Chainlink's owner | external + entity `Morpho` / `Chainlink` |
| `DAO`, `GovernorBravoDelegate`, `OZ Governor`, `TimelockController` (if guardian or owner) | governance (`isGovernance: true`) |

---

## Step 0: Validate and bootstrap

### 0a. Validate inputs

```bash
# Vault address must be chain-prefixed
echo "$0" | grep -qE '^(eth|base|arb1|optimism|polygon):0x[0-9a-fA-F]{40}$' \
  || { echo "Vault address must be chain-prefixed (e.g. eth:0xBEEF...)"; exit 1; }
```

If the user passed a bare `0x...`, prepend `eth:` and tell them.

### 0b. Validate server is running

```bash
curl -sf localhost:2021/api/projects/Steakhouse-USDC > /dev/null 2>&1 \
  && echo "OK" || echo "SERVER_MAYBE_NOT_RUNNING (test endpoint failed but server may still be up)"
```

If the server is not running, stop and tell the user.

### 0c. Read the vault on-chain

Pick the RPC by chain prefix (see the per-chain table above):

```bash
CHAIN=$(echo "$0" | cut -d: -f1)
case "$CHAIN" in
  eth)      RPC="${ETHEREUM_RPC_URL_FOR_DISCOVERY:-https://eth.llamarpc.com}" ;;
  base)     RPC="${BASE_RPC_URL_FOR_DISCOVERY:-https://base.llamarpc.com}" ;;
  arb1)     RPC="${ARBITRUM_RPC_URL_FOR_DISCOVERY:-https://arbitrum.llamarpc.com}" ;;
  optimism) RPC="${OPTIMISM_RPC_URL_FOR_DISCOVERY:-https://optimism.llamarpc.com}" ;;
  polygon)  RPC="${POLYGON_RPC_URL_FOR_DISCOVERY:-https://polygon.llamarpc.com}" ;;
  *) echo "Unsupported chain: $CHAIN"; exit 1 ;;
esac
VAULT_ADDR=$(echo "$0" | sed 's/^[a-z0-9]*://')

cast call "$VAULT_ADDR" "name()(string)"           --rpc-url "$RPC"
cast call "$VAULT_ADDR" "symbol()(string)"         --rpc-url "$RPC"
cast call "$VAULT_ADDR" "asset()(address)"         --rpc-url "$RPC"
cast call "$VAULT_ADDR" "MORPHO()(address)"        --rpc-url "$RPC"
cast call "$VAULT_ADDR" "owner()(address)"         --rpc-url "$RPC"
cast call "$VAULT_ADDR" "curator()(address)"       --rpc-url "$RPC"
cast call "$VAULT_ADDR" "guardian()(address)"      --rpc-url "$RPC"
cast call "$VAULT_ADDR" "timelock()(uint256)"      --rpc-url "$RPC"
cast call "$VAULT_ADDR" "feeRecipient()(address)"  --rpc-url "$RPC"
cast call "$VAULT_ADDR" "supplyQueueLength()(uint256)" --rpc-url "$RPC"
cast call "$VAULT_ADDR" "withdrawQueueLength()(uint256)" --rpc-url "$RPC"
cast call "$VAULT_ADDR" "totalAssets()(uint256)" --rpc-url "$RPC"
```

**Sanity check it's a MetaMorpho vault:** `MORPHO()` must return a non-zero address. If it reverts or returns zero, **stop** — this is not a MetaMorpho vault. Do **not** try to read `MIN_TIMELOCK` / `MAX_TIMELOCK` via cast — those are Solidity `constant` values without view functions and the call will revert.

**Record both queue lengths and `totalAssets()` for the Step 4b sanity check.** If `withdrawQueueLength > supplyQueueLength`, that is the signal the curator narrowed the active deposit target — Step 4b *must* confirm the funds-data fetcher saw all markets. `totalAssets()` returned in raw asset units (divide by the asset's decimals — 6 for USDC/PYUSD/cbBTC, 8 for WBTC, 18 for WETH/DAI/sDAI/wstETH) is the single source of truth for vault TVS.

### 0d. Derive the project slug

If `$1` was provided, use it verbatim. Otherwise transform `name()` deterministically:

```bash
NAME=$(cast call "$VAULT_ADDR" "name()(string)" --rpc-url "$RPC" | tr -d '"')
# Title-case-preserving: replace runs of non-alphanumeric with hyphens, drop leading/trailing
SLUG=$(echo "$NAME" | sed 's/[^A-Za-z0-9]\+/-/g; s/^-//; s/-$//')
echo "Slug: $SLUG"
```

This also gives you the **Morpho App slug** for free: lowercase the result.
Examples: `"Steakhouse Prime USDC"` → slug `Steakhouse-Prime-USDC`, Morpho slug `steakhouse-prime-usdc`.

### 0e. Detect the governance pattern early

Read owner/curator/guardian types directly via `cast` codesize + signature heuristics, or wait until after first discovery and resolve via `discovered.json`. The cleanest path: do this after Step 2 (first discovery), then walk this decision tree once:

| Owner contract type | Guardian contract type | Pattern |
|---|---|---|
| `GnosisSafe` | EOA / `GnosisSafe` / not set | **A — Multisig-only** |
| `GnosisSafe` | DAO contract (Aragon, Compound Governor, OZ Governor, TimelockController) | **B — Guardian-as-DAO** *(Steakhouse pattern)* |
| DAO contract | (any) | **C — DAO-as-owner** |
| Custom timelock + multisig chain | (any) | **D — Custom** |

Write the detected pattern to a scratchpad — it determines:
- Which addresses get `isGovernance: true` in Step 3.
- Which `governance.json` shape is written in Step 8.

### 0f. Refuse to clobber

```bash
if [ -d "packages/config/src/projects/<SLUG>" ]; then
  echo "Project <SLUG> already exists. Pass a different slug or delete the existing folder first."
  exit 1
fi
```

Do not overwrite an existing project folder. Ask the user to confirm a different slug or delete the folder.

---

## Step 1: Create the project with pre-baked Morpho config

### 1a. Make the project folder

```bash
mkdir -p packages/config/src/projects/<SLUG>
```

### 1b. Write `config.jsonc` from the template

The `MetaMorpho` template (`packages/config/src/projects/_templates/morpho/MetaMorpho/template.jsonc`) and `Morpho` template are matched automatically by bytecode hash — you do **not** need to add them. But the vault override below complements the template with handlers and hardcoded constants the template cannot express.

Read [`templates/config.jsonc.template`](templates/config.jsonc.template) and substitute the placeholders:

| Placeholder | Replace with |
|---|---|
| `{{SLUG}}` | the project slug derived in Step 0d (e.g. `Steakhouse-USDC`) |
| `{{VAULT_ADDRESS}}` | `$0`, the chain-prefixed vault address (e.g. `eth:0xBEEF…`) |
| `{{ASSET_ADDRESS}}` | the chain-prefixed underlying asset address from `asset()` in Step 0c |

Write the substituted result to `packages/config/src/projects/<SLUG>/config.jsonc`. Use the chain prefix that matches the vault (`eth:`, `base:`, etc.) — the template doesn't hardcode a chain.

**What the template does:**
- `ignoreMethods` on the vault hides ERC4626 preview helpers — keeps the discovered output focused on governance state. (Note: `withdrawQueue` is now an explicit `array` handler — see below — so it must NOT be in `ignoreMethods`.)
- `supplyQueue` and `withdrawQueue` array handlers enumerate the active deposit queue and the full market universe, respectively. Both are needed: `supplyQueue` documents the curator's current intent, `withdrawQueue` is the canonical list for downstream market-params discovery and per-feed cap aggregation.
- Two `morphoMarkets` handlers: the first fetches full market params from Morpho Blue's `idToMarketParams` — **driven by `withdrawQueue`** (`queueField: "withdrawQueue"`) so all markets the vault holds positions in are discovered, not just the active deposit target. The second extracts only the per-market `oracle` so each oracle gets discovered as a downstream contract.
- `allocators` event handler enumerates allocator additions/removals from `SetIsAllocator`.
- The four hardcoded fields (`MAX_TIMELOCK`, `MIN_TIMELOCK`, `MAX_QUEUE_LENGTH`, `MAX_FEE`) come from the MetaMorpho source as `constant` Solidity values — they have no view function, so `hardcoded` handlers surface them so mitigation `valueRange` entries can fieldRef them.
- `fee` is flagged HIGH severity because changes there directly affect depositor yield.
- `ignoreInWatchMode` silences the three fields that tick every block (`totalAssets`, `totalSupply`, `lastTotalAssets`).

(Per-market `marketCap_oracle_*` and per-feed `feedCap_*` handler entries are added in **Step 6** after discovery surfaces the market list.)

### 1c. Initialize the metadata files

Create empty placeholder files so the API has something to read. The downstream skills will populate them:

```bash
SLUG="<slug>"
DIR="packages/config/src/projects/$SLUG"
echo '{"version":"1.0","contracts":{}}' > "$DIR/functions.json"
echo '{"version":"1.0","tags":[]}'      > "$DIR/contract-tags.json"
echo '{"resources":[],"audits":[]}'     > "$DIR/resources.json"
```

Do **not** create `governance.json` — `/generate-governance` writes it (or you write it directly for pattern B in Step 8).
Do **not** create `review-config.json` — `/generate-review` writes it.

### 1d. Register the project in `defidisco-config.json`

Without this, the project doesn't appear in the gallery / API filter and the frontend never loads it — even after `compile-review` writes the compiled file.

Append the slug to the `defiProjects[]` array in [packages/config/src/defidisco-config.json](packages/config/src/defidisco-config.json):

```bash
python3 -c "
import json
path = 'packages/config/src/defidisco-config.json'
d = json.load(open(path))
slug = '<slug>'
if slug not in d['defiProjects']:
    d['defiProjects'].append(slug)
    json.dump(d, open(path, 'w'), indent=2)
    print(f'Added {slug}')
else:
    print(f'{slug} already present')"
```

This is a one-line array append — keep the file's existing JSON formatting (2-space indent).

---

## Step 2: Run discovery

Invoke `/run-discovery <slug>` (or `/run-discovery <slug> --auto` if `--auto` was passed). Let that skill drive the iterative-deepening discovery loop, prune externals, and tag entities.

**Hand `/run-discovery` the canonical classifier table** (see top of this file) plus the chain-specific addresses:

> Hints for this Morpho vault discovery:
> - Vault: `<vault-address>` (MetaMorpho ERC4626).
> - Underlying asset is already pruned via `ignoreDiscovery`.
> - Expect: Morpho Blue, PublicAllocator, MorphoHelper, several `MorphoChainlinkOracleV2` (one per market), several Chainlink `EACAggregatorProxy`, possibly an `AccessControlledOCR2Aggregator`.
> - **Use the canonical classifier table from review-morpho-vault** — every external contract has a deterministic tag.
> - Tag the vault `fetchBalances: true, fetchPositions: true`.
> - Common Chainlink overrides: `ignoreRelatives: ["aggregator", "proposedAggregator"]` on `EACAggregatorProxy`; `ignoreMethods: ["latestRoundData"]` on `AccessControlledOCR2Aggregator`; `ignoreInWatchMode: ["price"]` on `ChainlinkOracle` AND on `MorphoChainlinkOracleV2`. **Use `ignoreInWatchMode` for `price`, not `ignoreMethods` — the field is genuinely useful at discovery time (sanity-check the oracle is alive) but ticks every Chainlink heartbeat (minutes), so it must not trigger watch-mode alerts.**
> - For Chainlink feeds with overflow on `getTransmitters`, add it to `ignoreMethods`.
> - Owner/curator/guardian/feeRecipient addresses are usually `GnosisSafe` — let the GnosisSafe template match them. If guardian or owner is a DAO/Timelock, tag `isGovernance: true`.

`/run-discovery` will iterate, classify, and tag. Wait for it to finish.

---

## Step 3: Verify Morpho-specific tags + run governance pattern detection

### 3a. Verify tags (don't blindly re-apply)

GET the existing tags first; only PUT what's missing:

```bash
curl -s localhost:2021/api/projects/<slug>/contract-tags > /tmp/tags.json
# Required tags: vault → fetchPositions+fetchBalances; Morpho/PublicAllocator/MorphoHelper → external+Morpho;
# all MorphoChainlinkOracleV2 → external+Morpho; all Chainlink contracts → external+Chainlink;
# guardian (if DAO) → isGovernance: true.
```

Use the per-chain Morpho/PublicAllocator addresses from the table at the top of this file. On non-Ethereum chains, also confirm via `MORPHO()` on the vault; if PublicAllocator isn't in the table, find it in the vault's `allocators` array (it's the Morpho-tagged contract that isn't `MorphoHelper`).

### 3b. Run governance pattern detection

Now that contracts are templated, classify owner/curator/guardian:

```bash
python3 -c "
import json
d = json.load(open('packages/config/src/projects/<slug>/discovered.json'))
v = next(e for e in d['entries'] if e['address'].lower() == '<vault-address-lower>')
def kind(addr):
    e = next((e for e in d['entries'] if e['address'].lower() == addr.lower()), None)
    if not e: return 'UNKNOWN'
    return f\"{e.get('name','?')} (template={e.get('template','-')})\"
print('owner    →', kind(v['values']['owner']))
print('curator  →', kind(v['values']['curator']))
print('guardian →', kind(v['values']['guardian']))"
```

Match against the pattern decision tree (Step 0e). Save the chosen pattern (A/B/C/D) for Step 8.

If pattern is B (Guardian-as-DAO), tag the guardian DAO `isGovernance: true` now.

---

## Step 4: Fetch funds + generate call graph (parallel) + prune watch fields

Funds and call-graph are both **prerequisites** for `compile-review`. Without them, `compile-review` returns `{status: "skipped"}` and every TVS / capital number resolves to $0. They are independent — **start the call-graph in the background first, then run funds in foreground**, so you save 2–5 minutes of wall-clock time.

### 4a. Kick off the call graph in the background

Slither takes 2–6 min depending on contract count:

```bash
curl -sN "localhost:2021/api/terminal/generate-call-graph?project=<slug>&devMode=false" \
  > /tmp/cg.txt &
echo "call-graph started, pid=$!"
```

### 4b. Fetch funds (foreground — usually < 10s) + cross-check vs on-chain `totalAssets()`

```bash
curl -s -X POST localhost:2021/api/projects/<slug>/funds-data/fetch > /tmp/funds.txt
# Verify positions populated AND match on-chain totalAssets (the supplyQueue/withdrawQueue gotcha — see top of skill)
python3 <<PY
import json
d = json.load(open('packages/config/src/projects/<slug>/funds-data.json'))
v = next((val for k, val in d.get('contracts',{}).items() if k.lower() == '<vault-address-lower>'), {})
bal = v.get('balances', {}).get('totalUsdValue', 0)
pos = v.get('positions', {}).get('totalUsdValue', 0)
src = v.get('positions', {}).get('source', '?')
print(f'balances=\${bal:,.2f}  positions=\${pos:,.2f}  source={src}')
assert pos > 0, 'POSITIONS NOT POPULATED — DeBank may not support this chain'

# Compare against on-chain totalAssets() recorded in Step 0c
# Substitute <ON_CHAIN_TOTAL_ASSETS_USD> with the value from Step 0c (totalAssets / 10**asset_decimals)
on_chain = <ON_CHAIN_TOTAL_ASSETS_USD>
delta_pct = abs(pos - on_chain) / on_chain * 100 if on_chain else 0
print(f'on-chain totalAssets: \${on_chain:,.2f}  delta: {delta_pct:.1f}%')
assert delta_pct < 5, f'TVS divergence too large ({delta_pct:.1f}%) — likely the supplyQueue/withdrawQueue bug. Investigate.'
print('OK')
PY
```

If `positions` is missing/zero on a non-Ethereum chain, DeBank may not support that chain's Morpho positions. Flag it in the final report.

**If the cross-check fails (delta > ~1%, often 95%):**
- The vault almost certainly has `supplyQueueLength < withdrawQueueLength` (curator narrowed active deposits — see "Why withdrawQueue, not supplyQueue" at the top of this skill).
- Check `/funds-data.json` for `source: "onchain/debank"` (MorphoVaultService path) vs `source: "debank"` (DeBank's positions API). DeBank's path aggregates correctly across all Morpho positions; the on-chain path only walks whatever queue the MorphoRpcClient walks.
- The current MorphoRpcClient (`packages/defiscan-endpoints/src/clients/MorphoRpcClient.ts`) walks `withdrawQueue` and is correct. If it ever gets reverted to walking `supplyQueue`, this assertion catches it.
- Do **not** proceed to Step 5+ until TVS matches on-chain. The downstream review depends on accurate TVS.

### 4c. Prune watch fields with `/prune-watch-fields --auto`

While the call-graph still runs, invoke `/prune-watch-fields <slug> --auto`. The canonical config pre-bakes the obvious noisy fields (`totalAssets`, `totalSupply`, `lastTotalAssets` on the vault), but Chainlink/Morpho oracle adapters discovered downstream often expose other ticking fields (`price`, `latestAnswer`, `lastValidPrice`, etc.) that need `ignoreInWatchMode` too. Without this pass, the monitor will fire Discord alerts every Chainlink heartbeat.

```bash
# /prune-watch-fields handles the loop internally — applies all HIGH-CONFIDENCE recommendations
# in --auto mode and reports any UNCERTAIN ones in the final report.
```

### 4d. Wait for the call-graph to finish

```bash
until [ -f packages/config/src/projects/<slug>/call-graph-data.json ] \
   || tail -3 /tmp/cg.txt 2>/dev/null | grep -qE "(error|FAIL|done|DONE)"; do
  sleep 10
done
SIZE=$(stat -c%s packages/config/src/projects/<slug>/call-graph-data.json 2>/dev/null || echo 0)
echo "call-graph size: $SIZE bytes"
test "$SIZE" -gt 10000 || { echo "call-graph too small or missing — check /tmp/cg.txt"; tail -20 /tmp/cg.txt; exit 1; }
```

If absent or < 10 KB, check `/tmp/cg.txt` for Slither errors — usually a missing dependency or an unflattenable contract.

---

## Step 5: Scan permissions

Invoke `/scan-permissions <slug> <vault-address>`. The vault is the only contract worth scanning — price feeds, Morpho Blue, and multisigs are external.

Verify the standard MetaMorpho function set was detected (see Step 6 for the canonical list). If any are missing, the scan agent's source-fetch may have failed; re-run on the implementation address directly.

---

## Step 6: Score the vault — apply the canonical MetaMorpho scoring

The MetaMorpho codebase is fixed. Every permissioned function has a known impact profile. **Apply these scores directly via the API instead of re-running `/score-contract`** — `/score-contract` would do the same analysis but slower.

### 6a. Detect V1 vs V1.1 by source grep, not contract name

The on-chain contract name on Etherscan/Blockscout is just `MetaMorpho` for both V1 and V1.1 — **don't rely on the `discovered.json` name**. Grep the fetched source for `setName`/`setSymbol` directly:

```bash
SRC=$(curl -s "localhost:2021/api/projects/<slug>/code/<vault-address>" | python3 -c "import json,sys; print(json.load(sys.stdin)['sources'][0]['code'])")
if echo "$SRC" | grep -qE "function\s+setName\s*\("; then
  echo "V1.1 detected — apply rows 19-20 (setName/setSymbol)"
  IS_V1_1=1
else
  echo "V1.0 — skip setName/setSymbol rows"
  IS_V1_1=0
fi
```

### 6b. Apply the table

**Read [`SCORING_TABLE.md`](SCORING_TABLE.md) now.** It contains:
- The canonical table of MetaMorpho permissioned functions (V1 + V1_1: setName/setSymbol added).
- Per-function rationale for *why* each score holds.
- Ready-to-use `curl` payload templates for each lane.
- Why `submitCap` / `submitMarketRemoval` are left `unscored` for researcher attention.

Apply rows against `PUT localhost:2021/api/projects/<slug>/functions`.

> **PUT MERGES, doesn't replace.** Omitted fields fall back to the existing value (the backend uses `score ?? existing`). To **clear** a score (e.g. switch a function from `no-impact` to default-critical), you must edit `functions.json` directly — sending `score: null` does NOT work. To overwrite an existing entry, send the FULL payload with every field you care about, including ownerDefinitions, delay, dependencies, mitigations.

> When you later add deps or mitigations to `submitCap` / `acceptCap` (Step 7), preserve the score, ownerDefinitions, and delay fields by re-sending the full payload.

### 6c. Validate

After all PUTs, sanity-check the count and scores:

```bash
curl -s "localhost:2021/api/projects/<slug>/functions" | python3 -c "
import json, sys
d = json.load(sys.stdin)
v = list(d.get('contracts', {}).values())[0]
fns = v.get('functions', [])
perm = [f for f in fns if f.get('isPermissioned')]
by_score = {}
for f in perm:
    by_score.setdefault(f.get('score','default'), []).append(f['functionName'])
print(f'permissioned functions: {len(perm)} (expect 16 V1 / 18 V1.1)')
for s, names in sorted(by_score.items()):
    print(f'  {s}: {len(names)}')
assert 'unscored' in by_score and len(by_score['unscored']) == 2, 'Expected 2 unscored: submitCap, submitMarketRemoval'
assert 'critical' in by_score and len(by_score['critical']) == 2, 'Expected 2 critical: transferOwnership, renounceOwnership'
print('OK')"
```

`submitCap` / `submitMarketRemoval` remain `unscored` by default — flag them in the final report.

---

## Step 7: Wire oracle dependencies and per-feed impact caps

Without this step, the per-market Chainlink oracles are discovered but do not appear in the vault's dependency view, OR they appear with `$0` funds at risk, OR they appear with the full vault TVS uniformly (implying any feed failure can drain everything — wrong).

The fix is two-part:
1. **Manual `dependencies[]` on Morpho Blue's `liquidate` AND on the vault's `submitCap` / `acceptCap`** — `liquidate` is the semantically correct attachment (it's the Morpho function that reads the oracle), but `liquidate` is **not** on the vault's forward-BFS frontier (vaults never call liquidate; external liquidators do), so attaching there alone yields $0 funds-at-risk. Attaching the same deps to `submitCap` / `acceptCap` (which ARE on the BFS path and commit capital under each oracle) makes the Chainlink reach inherit the vault TVS.
2. **Per-feed `impactCap` mitigations** scoped to each Chainlink-feed dependency, so each feed's reach is bounded by the cap of its market(s) instead of the whole vault TVS.

### 7a. Build the oracle list

The vault's `marketOracles` field returns the per-market `MorphoChainlinkOracleV2` adapter contracts. The actual underlying Chainlink feeds live inside those adapters as `BASE_FEED_1`/`BASE_FEED_2`/`QUOTE_FEED_1`/`QUOTE_FEED_2` fields:

```bash
python3 -c "
import json
d = json.load(open('packages/config/src/projects/<slug>/discovered.json'))
v = next(e for e in d['entries'] if e['address'].lower() == '<vault-address-lower>')
adapters = [a for a in v['values'].get('marketOracles', []) if a != '<chain>:0x0000000000000000000000000000000000000000']
feeds = set()
for a in adapters:
    e = next((e for e in d['entries'] if e['address'].lower() == a.lower()), None)
    if not e: continue
    for k in ('BASE_FEED_1','BASE_FEED_2','QUOTE_FEED_1','QUOTE_FEED_2'):
        f = e['values'].get(k)
        if f and not f.endswith('0x0000000000000000000000000000000000000000'):
            feeds.add(f)
print('Adapters:', adapters)
print('Distinct feeds:', sorted(feeds))"
```

### 7b. Attach deps to `Morpho.liquidate` and to the vault's `submitCap` / `acceptCap`

Use the chain-specific Morpho Blue address. PUT three entries (one per function):

```bash
MORPHO="<chain>:0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb"  # or chain-specific
DEPS='[{"contractAddress":"<chain>:0xFeed1..."},{"contractAddress":"<chain>:0xFeed2..."}, ...]'

# 1. Morpho.liquidate (semantically correct, even if BFS doesn't reach it)
curl -s -X PUT localhost:2021/api/projects/<slug>/functions \
  -H 'Content-Type: application/json' \
  -d "$(printf '{"contractAddress":"%s","functionName":"liquidate","isPermissioned":false,"dependencies":%s}' "$MORPHO" "$DEPS")"

# 2. submitCap (preserve permissioned config from Step 6 — score, ownerDefinitions, delay, mitigations)
# 3. acceptCap (non-permissioned)
# Use the full-payload pattern from SCORING_TABLE.md "merging deps into submitCap/acceptCap" section.
```

### 7c. Add per-market and per-feed cap fields to `config.jsonc` via the `metaMorphoCap` handler

The `metaMorphoCap` handler (`packages/discovery/src/discovery/handlers/defidisco/MetaMorphoCapHandler.ts`) has two modes:
- `mode: "market"` — takes a `marketId` (bytes32), returns the raw cap from `vault.config(id)`.
- `mode: "feed"` — takes a `feed` address, walks every market's adapter, and returns the **sum** of caps across markets whose adapter consumes that feed (correctly handles shared feeds like a USDC quote feed used by 3 markets).

**CRITICAL parameter formats — easy to get silently wrong:**

| Param | Format | Example | What happens if you get it wrong |
|---|---|---|---|
| `marketId` | bytes32 hex string with `0x` prefix | `"0x9103c3b4...91836"` | Handler reverts; field shows error |
| `feed` | **bare hex address, no chain prefix** | `"0x64c911996D3c6aC71f9b455B1E8E7266BcbD848F"` | Handler silently returns `0`. The internal comparison is `feedAddr.toLowerCase() === targetFeed.toLowerCase()` against the on-chain `BASE_FEED_*` return value, which is bare hex — passing `"base:0x..."` makes the prefix part of the string and no match is ever found. |

Always use **bare hex** (no `eth:`/`base:` prefix) for `feed` and `marketId`. The lookup compares against return values from `eth_call` which are never chain-prefixed.

Generate one `marketCap_oracle_<bareOracleAddr>` entry per non-zero market, plus one `feedCap_<bareFeedAddr>` entry per distinct underlying feed. Add all of them to `ignoreInWatchMode`. Example block to add to the vault's override in `config.jsonc`:

```jsonc
"<vault>": {
  "fields": {
    // ... existing template fields ...
    "marketCap_oracle_<bareOracleAddr>": {
      "handler": {
        "type": "metaMorphoCap",
        "mode": "market",
        "marketId": "0x9103c3b4..."
      }
    },
    // ... one per market ...
    "feedCap_<bareFeedAddr>": {
      "handler": {
        "type": "metaMorphoCap",
        "mode": "feed",
        "feed": "0x64c91199...",   // BARE HEX — no chain prefix
        "queueField": "withdrawQueue"  // Aggregate caps across the FULL universe of markets,
                                       // not just supplyQueue. Without this, decommissioned-but-
                                       // still-funded markets are missed and per-feed caps
                                       // under-report. The handler skips oracle=0x0 markets
                                       // automatically, so the idle market doesn't pollute the sum.
      }
    }
    // ... one per distinct feed ...
  },
  "ignoreInWatchMode": [
    "totalAssets","totalSupply","lastTotalAssets",
    "marketCap_oracle_<...>", "feedCap_<...>", "..."
  ]
}
```

Then re-run discovery (no need to delete anything; new fields populate alongside existing ones):

```bash
cd packages/config && l2b discover <slug> --dev
```

**Validate immediately**: read every `marketCap_*` and `feedCap_*` field. **All-zero is the fail mode.** If at least one entry per market and per feed is non-zero, you're good. A zero entry can mean:
- The market is decommissioned (cap=0 on-chain — see `cast call <vault> "config(bytes32)(uint184,bool,uint64)" <marketId>`). Expected.
- The `feed` parameter was passed with a chain prefix (`eth:0x...` instead of bare `0x...`). Bug — handler silently returns 0 because the string compare against the on-chain return never matches.
- The `queueField` parameter was omitted/wrong, leaving the handler walking `supplyQueue` (default), which only includes the active deposit target. Aggregation misses markets with non-zero caps that were rotated out of supplyQueue.

The validator below flags all-zero results as errors and prints the per-entry breakdown so you can spot which case applies:

```bash
python3 -c "
import json
d = json.load(open('packages/config/src/projects/<slug>/discovered.json'))
v = next(e for e in d['entries'] if e['address'].lower() == '<vault-address-lower>')
caps = {k: v_ for k, v_ in v['values'].items() if k.startswith('marketCap_') or k.startswith('feedCap_')}
nonzero = sum(1 for val in caps.values() if str(val) != '0')
for k, val in caps.items():
    flag = '' if str(val) != '0' else '  (zero — decommissioned market or unused feed)'
    print(f'  {k} = {val}{flag}')
assert nonzero > 0, 'ALL caps are zero — likely chain-prefixed feed/marketId, or queueField missing/wrong, or all markets decommissioned'
print(f'OK ({nonzero}/{len(caps)} non-zero)')"
```

### 7d. Apply per-feed `impactCap` mitigations

For each underlying Chainlink feed, add a mitigation on `submitCap` and `acceptCap` scoped to that feed (`scopedTo: { type: "dependency" }`) with `impactCap.value.fieldRef` pointing at the vault's `feedCap_<addr>` field. Use `unit: { kind: "scaler", factor: "1e6" }` for USDC vaults (the asset's decimal exponent — switch to the right factor for non-USDC vaults: `1e18` for WETH/DAI, `1e8` for cbBTC, etc.).

```json
{
  "type": "other",
  "label": "Per-market cap",
  "description": "Prices market(s) X. Capital exposure under feed failure is bounded by the relevant market USDC supply caps.",
  "scopedTo": { "address": "<chain>:<feedAddress>", "type": "dependency" },
  "impactCap": {
    "value": {
      "mode": "fieldRef",
      "contractAddress": "<vault>",
      "fieldName": "feedCap_<bareFeedAddr>"
    },
    "unit": { "kind": "scaler", "factor": "1e6" }
  }
}
```

Recompile + verify each Chainlink dep shows funds-at-risk equal to the corresponding feed's cap (capped at vault TVS):

```bash
curl -s -X POST localhost:2021/api/projects/<slug>/compile-review > /dev/null
curl -s "localhost:2021/api/projects/<slug>/dependencies" | python3 -c "
import json, sys
d = json.load(sys.stdin)
for x in d['dependencies']:
    if x.get('entity') == 'Chainlink':
        print(f\"  {x['address']}  funds=\${x.get('totalFundsAtRisk',0):,.2f}\")"
```

Each feed should show its own cap, not a uniform vault TVS. If they all show the same number, the impactCap fieldRef didn't resolve — common causes: wrong field name, hardcoded shape used instead of the new schema (`{value: "...", unit: "usd"}` is **not** valid; it must be `{value: {mode: "hardcoded", amount: ...}, unit: {kind: "usd"}}` or the fieldRef equivalent).

---

## Step 8: Gather resources

Invoke `/gather-resources <slug>`. The bulk of resources is **always the same**; only curator-specific resources vary.

**Pre-seed `resources.json` with the canonical Morpho-vault block first** so the resource skill only adds the curator's URLs:

Read [`templates/resources.json.template`](templates/resources.json.template) and substitute:

| Placeholder | Replace with |
|---|---|
| `{{MORPHO_CHAIN}}` | Morpho's chain slug for the App URL: `ethereum`, `base`, `arbitrum`, etc. |
| `{{VAULT_ADDRESS_BARE}}` | the vault address with the chain prefix stripped |
| `{{MORPHO_VAULT_SLUG}}` | the lowercased slug from Step 0d (deterministic from `name()`, no need to fetch) |

The Morpho App URL is deterministic from the vault `name()` — lowercase + hyphenate. You don't need to fetch the redirect; just construct it.

Then invoke `/gather-resources <slug> <curator-website-url>` and tell the resource skill:

> The standard Morpho vault resources (Morpho App, fallback, Lite Apps, Morpho Docs, MetaMorpho GitHub + license, and the four MetaMorpho audits + Immunefi bounty) are already pre-populated. **Do not re-fetch or modify them.** Add only:
> - Curator's website (type `website`, exactly one)
> - Curator's docs (type `docs`)
> - Curator's GitHub org (type `github`)
> - Curator's X handle (type `x`)
> - Any curator-specific audits

`/gather-resources` merges — preserves existing entries.

### 8a. License label note

The MetaMorpho LICENSE file is dual-licensed (`GPL-2.0-or-later OR BUSL-1.1`). The actual license in effect is whatever the SPDX header on a representative `.sol` file declares. As of v1.x this is `GPL-2.0-or-later` — the template hardcodes that. For V2 forks or unusual deployments, verify the SPDX header on the deployed implementation and update the label if it differs.

### 8b. Logo (with Morpho fallback)

Prefer the curator's logo when extractable. Fall back to the Morpho logo otherwise — these vaults are fundamentally Morpho products. After `/gather-resources` returns, if `packages/defiscan-frontend/public/protocols/<slug>.{svg,png}` is missing, fetch from `https://morpho.org` (apple-touch-icon → favicon.svg → og:image) and save under the project slug. Resize PNGs to 128×128. Note in the final report whether you used `curator-logo` or `morpho-fallback`.

---

## Step 9: Generate governance (Patterns B/C/D only — Pattern A has none)

The governance section is for **DAO-style governance**, not multisigs. A pure-multisig setup (Pattern A) is just an admin — the multisig already shows up as a high-impact admin in the review. Adding a "Multisig" governance section duplicates that information and clutters the report.

### 9a. Pattern A (Multisig-only) — DO NOTHING

If Step 3b detected Pattern A, **skip this step entirely**. Do **not** create `governance.json`. The owner/curator/guardian multisigs already render as admin cards in Step 10's `/generate-review` output, which is the correct representation.

The Guardian's veto power is already conveyed by the **scoped Veto-only mitigation** on `revokePending*` (see SCORING_TABLE.md rows 13-16). No further governance modeling is needed.

### 9b. Pattern C (DAO-as-owner) / D (Custom)

Invoke `/generate-governance <slug>` with hints:

> Universal hints (every Morpho vault):
> - 1–14 day vault timelock (`<vault>.timelock`) on owner-initiated changes.
> - Guardian (`$self.guardian` = `<addr>`) can revoke pending timelocked changes via `revokePending*`. Mention this veto in `votingProcess`.
>
> Pattern-specific:
> - **C (DAO owner):** Owner is `<framework>` at `<addr>`. Research voting unit, proposal requirements, `votingPeriod`. `executionDelay`: fieldRef → `<vault>.timelock`.

### 9c. Pattern B (Guardian-as-DAO, Steakhouse pattern)

`/generate-governance` doesn't know about this pattern — its decision tree treats `owner` as the governance entity. **Write `governance.json` directly:**

```json
{
  "framework": "<guardian's framework, e.g. Aragon OSx>",
  "voteExecution": "onchain",
  "votingUnit": "<e.g. Vault shares>",
  "proposalRequirements": "<e.g. Via DAO plugin>",
  "votingProcess": "A <framework> DAO is the vault guardian and can veto the timelocked actions.",
  "proposalPeriod": { "kind": "fixed", "value": "<DAO's voting window>" },
  "executionDelay": { "kind": "fixed", "value": "None" }
}
```

`votingProcess` must be ≤ 150 chars and explicitly mention the **veto** role. `executionDelay = "None"` because the vault timelock applies to *owner* actions, not the guardian's veto.

### 9d. Verify (Patterns B/C/D only)

```bash
curl -s -X POST localhost:2021/api/projects/<slug>/compile-review > /dev/null
python3 -c "
import json
c = json.load(open('packages/defiscan-frontend/public/data/<slug>/compiled-review.json'))
g = c.get('governance', {})
assert g.get('framework'), 'framework missing'
vp = g.get('votingProcess', '')
assert len(vp) <= 150, f'votingProcess too long: {len(vp)}'
print('OK:', g)"
```

---

## Step 10: Generate the review

`/generate-review` is now invocable via the Skill tool (its `disable-model-invocation` was lifted). Invoke `/generate-review <slug>`. It produces `review-config.json` from the live API data.

After it finishes, recompile:

```bash
curl -s -X POST localhost:2021/api/projects/<slug>/compile-review > /dev/null
```

### 10a. Validate the compiled review (catches the silent failures)

```bash
python3 <<'PY'
import json
c = json.load(open('packages/defiscan-frontend/public/data/<slug>/compiled-review.json'))
t = c.get('totals', {})

# 1. TVS must be > 0
assert t.get('totalCapitalAtRisk', 0) > 0, 'TVS is $0 — funds-data or call-graph missing'

# 2. Admin names must NOT be auto-generated ("3/7 43% SafeL2") — they should come from review-config
auto_pattern = ('SafeL2', 'GnosisSafe', '/')
for a in c.get('admins', []):
    name = a.get('name', '')
    if all(p in name for p in auto_pattern):
        print(f'WARN: admin still has auto-generated name: {name}')
        print(f'  → check chain prefix in review-config.json admins keys (must match {a["address"].split(":")[0]}:)')

# 3. Chainlink dependencies must show non-zero funds-at-risk
chainlink_deps = [d for d in c.get('dependencies', []) if d.get('entity') == 'Chainlink']
for d in chainlink_deps:
    assert d.get('totalFundsAtRisk', 0) > 0, f'Chainlink dep {d["address"]} has $0 funds — feedCap_/marketCap_ likely zero (chain-prefix in feed param?)'

# 4. Per-feed impactCap mits should resolve (effectiveCapUsd populated)
# Find acceptCap/submitCap and check their per-feed mitigations
for a in c.get('admins', []):
    for fn in a.get('functions', []):
        if fn.get('functionName') in ('submitCap', 'acceptCap'):
            for m in fn.get('mitigations', []):
                if m.get('label') == 'Per-market cap':
                    cap = m.get('impactCap', {}).get('effectiveCapUsd')
                    assert cap is not None and cap > 0, f'Per-market cap did not resolve on {fn["functionName"]}'

# 5. LoC must be set (compile-review computes it)
assert t.get('linesOfCode', 0) > 0, 'LoC = 0 — compile-review may have skipped'

print(f'TVS: ${t["totalCapitalAtRisk"]:,.0f}')
print(f'admins: {t["adminCount"]}, deps: {t["dependencyCount"]}')
print(f'LoC: {t["linesOfCode"]}, coverage: {t.get("coverage")}%')
print('OK')
PY
```

If any assertion fails, the most common causes are:
- `eth:` chain prefix in review-config keys on a non-Ethereum project → fix `/generate-review`'s output
- `feed` / `marketId` passed with chain prefix in `metaMorphoCap` handler → bare hex
- Funds-data fetch failed silently (DeBank doesn't support this chain)
- Call-graph file missing or < 10 KB (Slither error)

Lines of code is computed inside `compile-review` — no separate command needed (this used to be a manual step via the "Count Lines of Code" button; not a slash command).

---

## Step 11: Final report

Print a summary the user can scan in 30 seconds:

```
═══════════════════════════════════════════════════════════════════════════
MORPHO VAULT REVIEW — <slug>
═══════════════════════════════════════════════════════════════════════════

Vault:      <name>  (<vault-address>)
Asset:      <symbol>  (<asset-address>)
Chain:      <chain>
Owner:      <owner-address>  (<resolved-name>)
Curator:    <curator-address>
Guardian:   <guardian-address>
Timelock:   <timelock-seconds> seconds (~ <human-readable>)
Allocators: <count>
Markets:    <supplyQueueLength> active deposit / <withdrawQueueLength> total in withdrawQueue

DISCOVERY
  Contracts:  <total>  (core: <X>, external: <Y>, governance: <Z>, EOAs: <W>)

PERMISSIONS
  Permissioned vault functions: <N>  (auto-scored: <N>, unscored: 2)
  Left unscored: submitCap, submitMarketRemoval

ORACLE DEPENDENCIES
  Chainlink feeds: <N>
  Per-feed funds-at-risk:
    <feed1>:  $<cap>
    <feed2>:  $<cap>
    ...

RESOURCES & AUDITS
  Standard Morpho block:  6 resources + 4 audits ($1.5M Immunefi bounty)
  Curator-specific added: <N> resources, <N> audits
  Logo: <curator-logo | morpho-fallback>

GOVERNANCE
  Pattern: <A | B | C | D>
  Framework: <…>
  Vote execution: <onchain | offchain>
  Execution delay: <…>

REVIEW
  TVS: $<positions.totalUsdValue>
  LoC: <N>, coverage: <100%>
  Open: http://localhost:2021/ui/projects/<slug>

═══════════════════════════════════════════════════════════════════════════
NEXT STEPS FOR THE RESEARCHER
═══════════════════════════════════════════════════════════════════════════
  1. Score submitCap / submitMarketRemoval. Run:
       /score-contract <slug> <vault-address>
  2. Verify the curator's website / docs / X handle.
  3. Confirm the governance pattern matches expectations.
```

---

## Guidelines

- **Don't redo work the canonical patterns already settle.** The MetaMorpho contract surface is fixed — reading source for `setFee` over and over wastes tokens. Apply the table in Step 6 and move on.
- **Do read source for the two open functions.** `submitCap` / `submitMarketRemoval` genuinely depend on the per-vault market choices.
- **Pre-seed, then merge.** Never call `/gather-resources` first and then patch — write the canonical block first.
- **Per-chain Morpho address.** The Ethereum Morpho Blue address (`0xBBBB…EFCb`) and PublicAllocator (`0xfd32…c75D`) are hardcoded; on Base, Arbitrum, etc., look them up from the per-chain table or `MORPHO()` on the vault.
- **Refuse to clobber.** If the project folder already exists, stop and ask.
- **Sanity-check it's actually a MetaMorpho vault.** If `MORPHO()` reverts or the contract has no `setCurator` / `setIsAllocator` in its ABI, refuse rather than write a broken config. The newer Morpho `Vault V2` factory line has a different ABI and is out of scope.
- **PUT replaces.** Subsequent PUTs to the same `(contract, function)` overwrite. When adding deps/mitigations to `submitCap`/`acceptCap`, preserve the score, ownerDefinitions, delay, and existing mitigations by re-sending the full payload.
- **Funds-data and call-graph are prerequisites for `compile-review`.** Without them, every numeric output is $0. Run them in Step 4 before compiling.
- **Liquidate-only deps don't propagate capital.** Always attach Chainlink deps to BOTH `Morpho.liquidate` (semantic) AND `MetaMorphoV1_1.submitCap` / `acceptCap` (operational — these are on the BFS path).
- **withdrawQueue, not supplyQueue, drives every market enumeration in this skill** — discovery's `marketParams` handler, the per-feed cap aggregator, and the funds-data fetcher (`MorphoRpcClient`). supplyQueue is the *active deposit target*, often a strict subset (sometimes just the idle market). See "Why withdrawQueue, not supplyQueue" at the top of the skill for the full rationale and the 95% TVS undercount bug.
