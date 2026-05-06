---
name: review-protocol
description: Orchestrate an end-to-end DeFiDisco review of an arbitrary DeFi protocol from one or more initial contract addresses. Bootstraps the project, runs discovery + resource gathering, generates the call graph, scans permissions, scores every protocol contract, factually verifies the detected admin impact (adding fieldRef impact caps where the worst case is bounded by observable state), then writes governance + review and finishes with a watch-mode audit.
argument-hint: "<project-slug> <initial-address> [website-url] [--auto]"
allowed-tools: Bash, Read, Write, Edit, WebSearch, WebFetch
---

# Review Protocol

Meta-orchestrator that takes a generic DeFi protocol from "I have an address" to a fully-formed DeFiDisco review. Unlike `/review-morpho-vault` (which pre-bakes one specific protocol's patterns), this skill stays generic and leans on your DeFi knowledge plus the existing skills (`/run-discovery`, `/gather-resources`, `/prune-watch-fields`, `/scan-permissions`, `/score-contract`, `/generate-governance`, `/generate-review`).

## Arguments

- `$0` — project slug (folder name under `packages/config/src/projects/`). Lowercase-hyphen, no spaces.
- `$1` — initial address (chain-prefixed: `eth:0x...`, `base:0x...`, etc.). For multi-anchor protocols, pass a comma-separated list (e.g. `eth:0xA,eth:0xB`).
- `$2` (optional) — official website URL. If omitted, you'll have to web-search to seed `/gather-resources`.
- `--auto` — run downstream skills non-interactively where they support it. Default: pause at key checkpoints.

Parse `$ARGUMENTS` to detect these.

## Prerequisites

The l2b UI server must be running at `http://localhost:2021`. If not, tell the user to start it with `cd packages/config && l2b ui` and stop.

---

## Pipeline overview

```
Step 1  Bootstrap project          → PROJECT_BOOTSTRAP.md
Step 2  Discovery + resources      (run /run-discovery; /gather-resources runs alongside as protocol context)
Step 3  Funds audit                → FUNDS_AUDIT.md
Step 4  Prune watch fields         (/prune-watch-fields)
Step 5  Call graph + permissions   (background curl + /scan-permissions in parallel)
Step 6  Re-run call graph          (cached, picks up new permission edges)
Step 7  Score all protocol contracts (/score-contract --all --interactive)
Step 8  Verify admin impacts       → IMPACT_VERIFICATION.md
Step 9  Governance                 (/generate-governance)
Step 10 Review                     (/generate-review)
Step 11 Watch-mode audit           → WATCH_MODE_AUDIT.md
Step 12 Final report
```

You should expect the whole run to take 30–90 minutes wall-clock, mostly waiting on Slither and discovery. Use background bash (`&`) wherever a step is independent of the next so you don't burn idle time.

---

## Step 1: Bootstrap

Read [PROJECT_BOOTSTRAP.md](PROJECT_BOOTSTRAP.md) and follow it. It validates inputs, refuses to clobber, creates the project folder, writes the minimal `config.jsonc`, initializes the metadata files, and registers the slug in `defidisco-config.json`.

Stop after Step 1 only if bootstrap raised an error.

---

## Step 2: Discovery + resources (concurrent in spirit)

These are independent — `/gather-resources` only needs the project slug + URL, and reads the public web. `/run-discovery` only needs the bootstrapped config. Run them in whichever order is convenient, but **do `/gather-resources` early** because the resources you collect (docs, governance forum, address book) are the same ones you'll lean on later on.

### 2a. Gather resources

```
/gather-resources $0 <website-url>
```

If `$2` was passed, use it. Otherwise web-search the protocol's official site first and pass that URL. If the user already has resources from a prior run, the skill merges — safe to re-run.

In `--auto` mode, just invoke and accept its output. Otherwise, let the skill pause at its review points.

### 2b. Discovery

```
/run-discovery $0 [--auto]
```

`/run-discovery` is the canonical place where external contracts get **leafed** (not removed via `ignoreDiscovery`, but kept as named nodes with every address-typed field listed in `ignoreRelatives`). Hand it any protocol-specific hints you already have from the resource step (which contracts are the protocol's own vs. third-party tokens/oracles, expected entity tags). Let the skill drive its iteration loop.

In non-auto mode, `/run-discovery` pauses between iterations for classification review — answer those prompts using DeFi context you've now built up.

When `/run-discovery` reports the graph stable, move on.

---

## Step 3: Funds audit

Read [FUNDS_AUDIT.md](FUNDS_AUDIT.md) and follow it. Triggers the funds-data fetch, reconciles discovered TVS against an expected TVL range, and fixes tag mismatches — including authoring a new aggregate handler when one is needed. The funds baseline established here feeds every TVS / capital number in the rest of the pipeline. If a new aggregate handler is authored, surface its details in the Step 12 final report.

---

## Step 4: Prune watch fields

```
/prune-watch-fields $0
```

`/run-discovery --auto` already invokes this internally; in interactive mode it asks. Either way, make sure it has run once before scoring — it dramatically reduces noise during Step 11 (the watch-mode audit).

---

## Step 5: Call graph + scan permissions (parallel)

Both feed into Step 7's scoring. The call graph runs Slither on every flattened source — slow, 2–6 minutes typical. `/scan-permissions` reads source + `discovered.json` to identify permissioned functions and write owner paths — also slow but dominated by your own analysis time.

### 5a. Kick off call graph in the background

```bash
curl -sN "localhost:2021/api/terminal/generate-call-graph?project=$0&devMode=false" > /tmp/cg-$0.txt &
CG_PID=$!
echo "call-graph started, pid=$CG_PID"
```

### 5b. Run /scan-permissions in foreground

```
/scan-permissions $0
```

The skill scans every non-external, non-EOA contract in the project. Let it write `ownerDefinitions` for every permissioned function it can verify. Functions it can't resolve get flagged for manual review — note those and come back to them in Step 8 if needed.

If `/scan-permissions` reports missing AccessControl handlers, follow the instructions it prints (add the handler to `config.jsonc`, re-run `l2b discover --dev`, re-scan), then continue.

### 5c. Wait for the call graph

```bash
until [ -f packages/config/src/projects/$0/call-graph-data.json ] \
   || tail -3 /tmp/cg-$0.txt 2>/dev/null | grep -qE "(error|FAIL|done|DONE)"; do
  sleep 10
done
SIZE=$(stat -c%s packages/config/src/projects/$0/call-graph-data.json 2>/dev/null || echo 0)
echo "call-graph size: $SIZE bytes"
test "$SIZE" -gt 10000 || { echo "call-graph too small"; tail -20 /tmp/cg-$0.txt; exit 1; }
```

If absent or under 10 KB, check `/tmp/cg-$0.txt` for Slither errors (usually a missing dependency or an unflattenable contract).

---

## Step 6: Re-run the call graph (cached)

`/scan-permissions` may have added new owner edges via `ownerDefinitions`. The enhanced traversal merges call-graph edges + permission edges, so a re-run picks up any new edges that Slither can re-traverse without re-flattening (cached in sqlite). Cheap insurance — usually under 30 seconds:

```bash
curl -sN "localhost:2021/api/terminal/generate-call-graph?project=$0&devMode=false" \
  > /tmp/cg-$0-rerun.txt
```

If the rerun takes more than a couple of minutes, it isn't actually hitting the cache — investigate before proceeding.

Then refresh funds-data so Step 7's scoring sees correct USD values. Step 3 already established the baseline — this refresh just picks up any contract tags or aggregate-handler changes that landed mid-pipeline:

```bash
curl -s -X POST localhost:2021/api/projects/$0/funds-data/fetch > /tmp/funds-$0.txt
```

---

## Step 7: Score all protocol contracts

```
/score-contract $0 --all --interactive
```

`/score-contract` builds a queue of every contract with at least one unscored permissioned function (skipping externals — they're not the protocol's responsibility) and walks them one at a time. For each contract it triages functions into fast/standard/deep lanes, reads source proportionately, presents a plan, and waits for confirmation.

In `--auto` mode the skill expects you not to pause; let it fast-path obvious functions and only stop on deep-lane items.

When the loop finishes, `/score-contract` prints the aggregate roster (every mit, every cap, every new handler, every token tag) — keep it open for cross-reference in Step 8.

---

## Step 8: Verify admin impacts (factual check)

This is the step that distinguishes a protocol-aware review from a mechanical one. Read [IMPACT_VERIFICATION.md](IMPACT_VERIFICATION.md) and follow it: for every admin whose detected impact is non-trivial, factually check whether the worst case really is what `/admins` reports — and where it isn't, add a fieldRef-backed impact cap or a scoped mitigation.

**Constraint that overrides everything else in this step**: any `impactCap` you add must be a fieldRef. Hardcoded USD caps rot — they don't track price or supply. If the bounding state isn't already in `discovered.json`, add a discovery handler (call/event/computed), re-run `l2b discover $0 --dev`, then write the cap.

---

## Step 9: Governance

```
/generate-governance $0
```

The agent researches the protocol's framework + voting + delays and writes `governance.json`. If you already authored one, the skill asks before overwriting.

---

## Step 10: Generate review

```
/generate-review $0
```

The skill wipes any existing `review-config.json` (preserving `publishedAt` and `verified`) and rebuilds from live API data. The `dataKeys` block is filled with template variables that point at live capital/funds/token-value paths so dollar amounts auto-update on every recompile.

After it finishes, recompile so the compiled review reflects every cap you added in Step 8:

```bash
curl -s -X POST localhost:2021/api/projects/$0/compile-review > /dev/null
```

---

## Step 11: Watch-mode audit

Read [WATCH_MODE_AUDIT.md](WATCH_MODE_AUDIT.md) and follow it. The pattern: snapshot the current `discovered.json`, re-run discovery **without** `--dev` so it hits live RPC, diff against the snapshot, and treat every changed field as a candidate for `ignoreInWatchMode`. This is the only reliable way to catch ticking fields the heuristic-based `/prune-watch-fields` missed (especially on protocol-specific accumulators).

---

## Step 12: Final report

Print a one-screen summary the user can scan in 30 seconds:

```
═══════════════════════════════════════════════════════════════════════════
PROTOCOL REVIEW — $0
═══════════════════════════════════════════════════════════════════════════

Protocol:   <name>  (chain: <chain>)
Initial:    <addresses>

DISCOVERY
  Iterations: <N>      Final contracts: <total>  (core: <X>, external leafed: <Y>)
  EOAs: <W>            Governance: <Z>           Funds-tracked: <V>

FUNDS AUDIT
  Expected TVL:     <range>  (source: <DeFiLlama | protocol dashboard | on-chain | …>)
  Discovered TVS:   $<bal+pos+agg>  (delta: <%>)
    balances:       $<X> across <N> contracts
    positions:      $<X> across <N> contracts (DeBank: <count>)
    aggregate:      $<X> via <handler-name>
  Tags added during audit: <N>
  NEW AGGREGATE HANDLER (if any):
    name:           <new-name>
    file:           packages/defiscan-endpoints/src/services/aggregate/handlers/<file>.ts
    data source:    <DeFiLlama | The Graph | on-chain | protocol API>
    rebuilt:        defiscan-endpoints, l2b, protocolbeat
    restarted:      <yes / how>
    frontend list:  KNOWN_AGGREGATE_HANDLERS in FundsTagsButton.tsx ← updated

PERMISSIONS
  Permissioned functions across protocol: <N>
  Auto-scored: <N>     Researcher unscored / flagged: <N>
  Functions with mitigations: <N>   With impact caps: <N>

ADMIN IMPACT VERIFICATION
  Admins reviewed: <N>
  fieldRef impact caps added: <N>
  Scoped mitigations added: <N>
  Admins re-scoped to no-impact after factual check: <N>

DEPENDENCIES
  External entities: <N>
  Per-dep funds-at-risk after caps:
    <dep1>:  $<amount>
    <dep2>:  $<amount>
    ...

REVIEW
  TVS:        $<positions+balances>
  LoC:        <N>      Coverage: <X>%
  Resources:  <N>      Audits: <N>      Bug bounty: $<amount>
  Governance: <framework> / <onchain|offchain> / delay <duration>

WATCH-MODE AUDIT
  Fields newly ignored: <N> (across <M> contracts)
  Suspicious fields kept under watch: <N>

  Open: http://localhost:2021/ui/projects/$0
═══════════════════════════════════════════════════════════════════════════
NEXT STEPS FOR THE RESEARCHER
═══════════════════════════════════════════════════════════════════════════
  1. <list of /scan-permissions unverified paths, /score-contract unscored functions, etc.>
  2. Toggle the project to "Verified" in the UI once you've audited the AI-generated content.
```

---

## Guidelines

- **Use the existing skills as the building blocks.** Don't re-implement discovery, scanning, scoring, or review generation. If a sub-skill produces something wrong, fix it there — not here.
- **External contracts are leaves, not absent.** `/run-discovery` already enforces this; don't second-guess. A leafed external still appears in dependency analysis and gets entity tags.
- **Trust your DeFi knowledge.** This skill intentionally under-specifies — it expects you to recognize "this is a Compound fork" or "that's a Uniswap V2 factory" without instructions. Use the resources gathered in Step 2 as the protocol-specific manual.
- **fieldRef caps only.** Hardcoded USD impact caps drift. If the bounding state isn't discovered, add a handler and re-discover. Always.
- **Refuse to clobber.** If the project folder already exists, stop and ask — don't merge over an existing review.
- **Funds-data and call-graph are prerequisites for `compile-review`.** Without both, every TVS / capital number resolves to $0. Step 3's funds audit establishes the funds baseline; Step 6's call-graph rerun finalizes the call graph. Confirm both exist before declaring Step 7 done.
- **A new aggregate handler authored in Step 3 is the only code change this skill ships.** It needs four wiring points (handler file, two barrel exports, `server.ts` registration, `KNOWN_AGGREGATE_HANDLERS` in the frontend) plus a rebuild of `defiscan-endpoints` (which `/build` does NOT cover) and `/build` for l2b + protocolbeat. Surface the handler name and file path in the Step 12 final report.
- **Run the watch-mode audit even in --auto mode.** It only takes one extra discovery cycle and produces the only diff that's grounded in live RPC change rather than heuristics.
