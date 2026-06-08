---
name: audit-callgraph
description: Audit a project's call graph against the actual contract source and propose edge corrections (add a real call Slither missed, remove a false-positive call) as reviewable suggestions. Never edits the graph directly and never guesses — every suggestion must be backed by code you read.
argument-hint: "<project-name> [address-or-addresses]"
allowed-tools: Bash, Read, Write
---

# Call Graph Audit Agent

You audit the **call graph** of project **$0** — the set of external calls Slither resolved between contracts — by comparing it against the **actual Solidity source**, and you propose corrections as **suggestions** for a researcher to review. You never apply changes yourself.

A call-graph edge `A.foo → B.bar` means "function `foo` on contract `A` makes an external call to `bar` on contract `B`." Your job is only to judge whether those edges are **true to the code**: is a resolved edge pointing at the wrong target, and is there a real external call that's missing?

## Absolute rules — read before doing anything

1. **Never suggest based on an assumption.** Not on interface names, not on contract names, not on "this probably calls that." Every suggestion must be justified by a specific piece of source you have **actually read**.
2. **Always read the contract code first.** Confirm the caller function's body, how the target variable is assigned, and what address it resolves to (cross-check `discovered.json` values). If you cannot confirm it from code, **do not suggest it**.
3. **The `reasoning` of every suggestion must cite the evidence** — contract + function + the exact line/snippet that proves it (e.g. `"WrapTokenV3ETH.moveToUnwrapAddress calls unwrapContract.moveFromWrapContract; unwrapContract is set in setUnwrapContract() (line 142) to 0x79973…, not the address Slither resolved"`).
4. **Scope is only callgraph edges.** Only ever propose `edgeType: 'callgraph'` add/remove. Do **not** touch permission or dependency edges — those are curated by the researcher in `functions.json`, not here. If you identify a pattern that could be abstracted away in a rule, propose it to the researcher and explain how it could save the total amount of edge changes specified. 
5. **Suggest, never apply.** You write to `call-graph-suggestions.json` only. You must never write to `call-graph-overrides.json`.

## Prerequisites

The l2b UI server should be running at `http://localhost:2021` (`cd packages/config && l2b ui`). You can also work straight from files on disk under `packages/config/src/projects/$0/`.

## Step 1 — Load the call graph and build a priority queue

Read the raw Slither output — it carries the resolution confidence the API drops (so use this file, **not** the `enhanced-graph-edges` endpoint, which omits it):

```bash
cat packages/config/src/projects/$0/call-graph-data.json
```

Each `contracts[addr].externalCalls[]` entry has: `callerFunction`, `storageVariable`, `interfaceType`, `calledFunction`, `resolvedAddress`, `resolvedContractName`, **`resolutionType`** (`deterministic` | `optimistic`), **`resolutionConfidence`** (0–100), `resolutionHeuristic`, and **`resolutionCandidates[]`** — every `{address, contractName}` Slither weighed.

**Audit these in priority order — this is the whole point of the skill:**
1. **`optimistic` with `resolutionCandidates.length > 1`** — Slither guessed among several contracts and picked one. The pick is often wrong (or the call has no single real target). *Highest priority.*
2. **`optimistic` with low `resolutionConfidence`** (e.g. ≤ 60) — a single guess that may be wrong.
3. Calls with **no `resolvedAddress`** — unresolved; a real edge may be missing.

`deterministic` edges came from a direct `discovered.json` lookup — leave them unless source plainly contradicts them.

If addresses were passed as arguments, restrict to calls whose caller contract is in that set. Work top-down through the priority queue; don't bother with deterministic edges unless time allows.

## Step 2 — Verify against source (the only basis for a suggestion)

For each suspect call, read the caller contract's source and confirm the truth:

```bash
curl -s http://localhost:2021/api/projects/$0/code/<callerAddress>   # → { sources: [{name, code}] }
```
(or read the flattened source under `packages/config/src/projects/$0/.flat/` if present.)

Determine, from the code:
- Does `callerFunction` actually make this external call? Trace the variable (`storageVariable`) to its declaration/assignment (constructor, setter, immutable).
- What address does it really point to? Cross-check the resolved value against `discovered.json` `values`.

For a **multi-candidate** edge: read the code to see which (if any) of the `resolutionCandidates` the variable actually holds. The variable usually pins to **one** of them (via a setter/constructor/immutable you can read) — or to **none** (e.g. the target is a function parameter / freshly-passed implementation, like a UUPS `upgradeTo(newImpl)` where `newImpl` is an argument, so no fixed contract is correct).

Decide one of:
- **Resolution is correct** → no suggestion.
- **Resolved to the wrong candidate** → `removeEdge` the wrong one, and `addEdge` the right candidate **only if** code + `discovered.json` pin it unambiguously.
- **No candidate is the real target** (dynamic/arg-supplied) → `removeEdge` the spurious edge; add nothing.
- **A real external call is missing** (e.g. a low-level `call`/`delegatecall`, or a dynamic target the code pins to a known address) → `addEdge`.

If the code doesn't let you decide with certainty, leave it alone — say so in the report rather than guessing.

## Step 3 — Build the suggestion(s)

Two rule shapes, both `edgeType: 'callgraph'`. Build `from`/`to` as `address.function`, copying the **exact** chain-prefixed addresses and function names from the data (do not hand-retype addresses):

```jsonc
// remove a false-positive edge
{ "id": "r-<slug>", "type": "removeEdge",
  "from": "eth:0xCaller.callerFunction", "to": "eth:0xWrongTarget.calledFunction",
  "edgeType": "callgraph" }

// add a real edge Slither missed
{ "id": "r-<slug>", "type": "addEdge",
  "from": "eth:0xCaller.callerFunction", "to": "eth:0xRealTarget.calledFunction",
  "edgeType": "callgraph" }
```

## Step 4 — Append to the suggestions file (never overwrite)

Read the existing file if present, append your new suggestions to `suggestions[]`, write it back. Each suggestion: a unique `id`, `status: "pending"`, `createdBy: "audit-callgraph"`, an ISO `createdAt`, and the code-cited `reasoning`.

File: `packages/config/src/projects/$0/call-graph-suggestions.json`

```jsonc
{
  "version": "1.0",
  "suggestions": [
    {
      "id": "acg-<unique>",
      "rule": { "id": "r-<unique>", "type": "removeEdge",
                "from": "eth:0x….foo", "to": "eth:0x….bar", "edgeType": "callgraph" },
      "reasoning": "<contract>.<function> — exact code evidence proving the edge is wrong/missing (cite the line).",
      "status": "pending",
      "createdBy": "audit-callgraph",
      "createdAt": "<ISO timestamp>"
    }
  ]
}
```

Preserve any existing entries (pending, accepted, rejected). Generate ids that won't collide with existing ones.

## Step 5 — Report

Summarise what you proposed and the evidence for each, and tell the researcher to review them in the Call Graph Walker's **inbox** tab (they appear inert until accepted — accepting promotes the rule into `call-graph-overrides.json`). If you found nothing wrong, say so plainly — proposing nothing is a valid, expected outcome.
