---
name: generate-governance
description: Generate a DeFi protocol governance config (governance.json) by researching the protocol and mapping on-chain fields to governance parameters. Sibling of /generate-review but scoped only to governance.
argument-hint: [project-name]
allowed-tools: Bash, Read, Write, WebSearch, WebFetch
---

# Governance Generation Agent

You are a DeFi protocol governance researcher. Your task is to author `governance.json` for the project **$0** by researching the protocol's governance system (voting framework, proposal requirements, periods, delays) and mapping what you find to the `GovernanceConfig` shape.

This skill **writes only to `packages/config/src/projects/$0/governance.json`**. It must NOT touch `review-config.json`, `resources.json`, `permission-overrides.json`, or any other file. It is explicitly designed so `/generate-review` can wipe and regenerate `review-config.json` without losing authored governance.

## Prerequisites

The l2b UI server must be running at `http://localhost:2021`. If not, tell the user to start it with `cd packages/config && l2b ui`.

---

## Step 0: Check existing file

```bash
GOV_PATH="packages/config/src/projects/$0/governance.json"
if [ -f "$GOV_PATH" ]; then
  echo "Existing governance.json found:"
  cat "$GOV_PATH"
  echo ""
  echo "Ask the user: overwrite, merge, or abort?"
fi
```

If the file exists, **read it**, and ask the user whether to overwrite, merge (keep existing non-empty fields, fill in blanks), or abort. Default to merge.

---

## Step 1: Fetch project context from l2b

```bash
curl -s "http://localhost:2021/api/projects/$0" | jq '{
  name: .name,
  entries: [.entries[] | {
    name: .name,
    initialContracts: [.initialContracts[] | {name, address, fields: [.fields[]? | select(.value.type == "number") | {name, description, value}]}],
    discoveredContracts: [.discoveredContracts[] | {name, address, fields: [.fields[]? | select(.value.type == "number") | {name, description, value}]}]
  }]
}' > /tmp/gov-context-$0.json
```

This gives you the contract list + all **numeric fields** (candidates for field-ref durations like timelock delay, voting period, quorum).

---

## Step 2: Research the protocol's governance

Use `WebSearch` / `WebFetch` to learn:

1. **Framework**: Is it Compound Governor Bravo, OpenZeppelin Governor, Aragon, Snapshot, a custom multisig, or something bespoke? Look for governance docs, forum posts, the DAO's Tally page.
2. **Vote execution**: `onchain` (e.g. Governor + Timelock) or `offchain` (e.g. Snapshot voting, multisig enforces).
3. **Voting unit**: What grants voting power? e.g. "COMP token", "veCRV", "1 person 1 vote", "NFT membership".
4. **Proposal requirements**: Who can submit proposals? e.g. "100k COMP delegated", "any address after signer approval", "1 of N multisig signers".
5. **Voting process**: 1-2 sentences describing the flow: propose → vote → queue → execute, quorum rules, timelock, etc. **150 char max.**
6. **Proposal period**: How long is the voting window?
7. **Execution delay**: How long after a vote passes before execution can happen? (Timelock delay.)

Always use primary sources: the protocol's own docs, their governance contract source on Etherscan, the Tally UI for the DAO.

---

## Step 3: Map durations to on-chain fields when possible

For `proposalPeriod` and `executionDelay`, three options:

- **`fieldRef`** — preferred when `voteExecution === 'onchain'`. Points at a numeric field in a discovered contract (e.g. a Timelock's `delay`, a Governor's `votingPeriod`). Keeps the value in sync with reality.
- **`fixed`** — free text like `"~3 days (configurable)"` when there's no on-chain field or the value is off-chain.
- **`none`** — explicitly N/A.

Use `/tmp/gov-context-$0.json` to pick field refs. Match what you read in governance docs (e.g. "timelock delay is 2 days") to a field in the discovered data (e.g. `Timelock.delay = 172800`).

---

## Step 4: Write `governance.json`

Shape (all fields required except where noted):

```json
{
  "framework": "Compound Governor Bravo",
  "voteExecution": "onchain",
  "votingUnit": "COMP token",
  "proposalRequirements": "100k COMP delegated to propose; any holder can vote.",
  "votingProcess": "Propose → 3-day vote → queue in Timelock → 2-day delay → execute. Quorum: 400k COMP.",
  "proposalPeriod": {
    "kind": "fieldRef",
    "ref": {
      "contractAddress": "eth:0xc0Da02939E1441F497fd74F78cE7Decb17B66529",
      "fieldName": "votingPeriod"
    }
  },
  "executionDelay": {
    "kind": "fieldRef",
    "ref": {
      "contractAddress": "eth:0x6d903f6003cca6255D85CcA4D3B5E5146dC33925",
      "fieldName": "delay"
    }
  }
}
```

**Duration kinds:**

- `{ "kind": "fieldRef", "ref": { "contractAddress": "eth:0x...", "fieldName": "..." } }`
- `{ "kind": "fixed", "value": "3 days" }`
- `{ "kind": "none" }`

**votingProcess** must be ≤ 150 characters.

**Addresses** must be chain-prefixed (`eth:`, `arb1:`, `base:`, etc.) and ERC-55 checksummed — copy them exactly from `/tmp/gov-context-$0.json`, do not re-case them.

Write with `Write` tool directly to `packages/config/src/projects/$0/governance.json`.

---

## Step 5: Verify

1. Read back the file you just wrote — confirm valid JSON.
2. Recompile the review for this project: `curl -X POST "http://localhost:2021/api/projects/$0/compile-review"` (or via the UI's compile button).
3. Fetch the compiled review and confirm the `governance` block exists and field-refs resolved to numeric `seconds`:
   ```bash
   curl -s "http://localhost:2021/api/projects/$0/compiled-review" | jq '.governance'
   ```
4. Report to the user: which framework you chose, whether field-refs resolved (and what `seconds` they resolved to), and any fields you left as `"fixed"` because you couldn't find an on-chain source.

---

## Common frameworks reference

- **Compound Governor Bravo** — `onchain`, unit = governance token, uses Timelock with `delay`, Governor has `votingPeriod`
- **OpenZeppelin Governor** — `onchain`, similar shape to Bravo, often with TimelockController
- **Aragon** — varies by version (OS vs OSx), usually `onchain`, DAO + Plugin architecture
- **Snapshot** — `offchain` voting, on-chain execution usually gated by a multisig — proposalPeriod/executionDelay are typically `fixed` or `none`
- **Multisig-only** — `offchain` (no voting), voting unit = "Multisig signers", proposalRequirements = "1 of N signers can queue"
