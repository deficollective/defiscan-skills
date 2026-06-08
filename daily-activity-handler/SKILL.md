---
name: daily-activity-handler
description: Daily research routine — find every project with activity today, judge each field change as noise or real protocol/admin change, flag analysis follow-up (new contracts, upgrades, removals, admin rotations), report everything, then wait for approval before applying anything.
---

# Daily Activity Handler

Walk every project's activity for today, classify each `(contract, field)` change, **report the full picture** — including any analysis follow-up the change creates — and **wait for the user's approval/feedback** before taking any action.

Ask **two** questions per change:

1. **Noise or real?** (drives the digest + what to strip)
   - **Noise** — field that shouldn't be monitored (counters, supply totals, prices, timestamps, internal accounting, etc.). Proposed action: strip from `activity.json` + add to `ignoreInWatchMode`.
   - **Real change** — protocol or admin change. Write a 1-2 sentence impact summary.
2. **Does it leave an analysis artifact stale or missing?** (drives follow-up) — a new contract with no tag/scoring, an upgrade that orphaned the call graph, a removed contract leaving dangling entries, a Safe/owner rotation that stale-dated a description. See **Step 2.5**. **New contracts are the critical case** — they land in `discovered.json` but are invisible in the compiled review until brought into scope.

Use your judgment. You know DeFi: a `totalSupply` rotating is noise, a new `owner` address is real. When unsure, mark UNCERTAIN — don't try to be clever.

**Apply nothing without explicit user approval.** Steps 1–3 are read-only; Step 4 (noise strips) only runs after the user approves. **Follow-up work is detect-and-recommend only** — the skill surfaces it with the exact command/sub-skill to run; it never tags, regenerates, scores, or edits descriptions on its own. The researcher actions follow-up separately (or asks you to run a specific item).

## Prerequisites

l2b UI server at `http://localhost:2021`. The strip flow needs `DATABASE_URL`:

```bash
curl -sf -o /dev/null -w "%{http_code}\n" localhost:2021/api/defidisco/monitor/health
```

If `503`, fall back to manual `config.jsonc` + `activity.json` edits + `l2b discover` at the end.

---

## Step 1 — Find today's activity

```bash
TODAY=$(date -u +%Y-%m-%d)
for f in packages/config/src/projects/*/activity.json; do
  count=$(grep -c "\"timestamp\": \"$TODAY" "$f" 2>/dev/null || echo 0)
  [ "$count" -gt 0 ] && echo "$(basename $(dirname $f)): $count"
done
```

For each hit, extract today's events:

```bash
node -e '
const f = JSON.parse(require("fs").readFileSync(process.argv[1], "utf8"));
const today = new Date().toISOString().slice(0, 10);
console.log(JSON.stringify(f.events.filter(e => e.timestamp.startsWith(today)), null, 2));
' packages/config/src/projects/<project>/activity.json
```

Each event carries `id`, `updateNotifierId`, `address`, `contractName`, `type`, `changes[{field, before?, after?}]`.

---

## Step 2 — Classify (use judgment)

Walk every change and call it noise or real. A few **non-negotiable rules** before judgment kicks in:

- `role-update`, `upgrade`, `contract-added`, `contract-removed` events are **always real**. Never strip.
- `$admin`, `$implementation`, `$pastUpgrades`, `$upgradeCount`, `accessControl.*`, `$members`, `$threshold` are **always real**.
- Fields with `permissions` metadata or `severity: HIGH` in `discovered.json.fieldMeta` → real.

Everything else: read the field name + value + contract context (use `discovered.json` and `contract-tags.json` for entity labels and template). Decide noise vs real on the merits. If a fee field looks like a rate (small integer that an admin tunes) it's real; if it looks like an accumulator (running total, only goes up) it's noise. If a field's value is an Ethereum address you don't recognize, lean toward real — addresses are usually wired-in dependencies. If you genuinely can't tell, mark it UNCERTAIN and surface it.

For deeper taxonomy if you want a sanity check, `/prune-watch-fields` has the full lists — but don't pattern-match line-by-line. Read the field, decide.

---

## Step 2.5 — Detect follow-up work (read-only)

Step 2 decides noise vs real *for the digest*. This step asks a different question: **does the change leave an analysis artifact stale or missing?** A change can need follow-up whether or not it's digest-worthy. **Detect and recommend only — never execute.** For each follow-up item, name the exact command or sub-skill the researcher should run.

Walk the real events through these triggers:

### `contract-added` — the critical case. Flag EVERY new contract.

A newly discovered contract lands in `discovered.json` but has no `contract-tags.json` entry, no call-graph node, no `functions.json` scoring, and no review-config description — so it's **invisible in the compiled review** until a researcher brings it into scope. Don't skip "obvious" externals or mechanical churn (e.g. AssetLists swapped during an upgrade) — flag them all; the researcher decides. For each new contract, gather:

```bash
# Identity + verified status + template
node -e '
const f = JSON.parse(require("fs").readFileSync(process.argv[1], "utf8"));
const e = f.entries.find(x => x.address.toLowerCase() === process.argv[2].toLowerCase());
console.log(JSON.stringify({ name: e.name, type: e.type, template: e.template, unverified: e.unverified === true }, null, 2));
' packages/config/src/projects/<project>/discovered.json <eth:0x...>

# Position in the system: who references it, and does it own/have a role over anything?
node -e '
const f = JSON.parse(require("fs").readFileSync(process.argv[1], "utf8"));
const needle = process.argv[2].toLowerCase().replace(/^.*:/, "");
const refs = [];
for (const e of f.entries) {
  if (!e.values) continue;
  for (const [k, v] of Object.entries(e.values)) {
    const hit = (x) => typeof x === "string" && x.toLowerCase().includes(needle);
    if (hit(v)) refs.push(`${e.name}.${k}`);
    else if (Array.isArray(v)) v.forEach((x, i) => hit(x) && refs.push(`${e.name}.${k}[${i}]`));
  }
}
console.log("Referenced by:", refs.length ? refs.join(", ") : "(nothing — fresh standalone deploy)");
' packages/config/src/projects/<project>/discovered.json <eth:0x...>

# Tagged already? Holds funds?
grep -i "<0x...>" packages/config/src/projects/<project>/contract-tags.json
grep -i "<0x...>" packages/config/src/projects/<project>/funds-data.json
```

Then recommend a classification + the exact next action:

- **External** (third-party oracle / token / bridge / adapter — other contracts point *into* it, it governs nothing): tag `isExternal` + entity. Run `/run-discovery` (tagging) or tag in the UI; prune as a discovery leaf if it dead-ends.
- **Governance** (multisig / timelock / DAO module): tag `isGovernance` + entity via `/run-discovery` or the UI.
- **Internal & verified** (governs protocol contracts, or is reachable as an admin target): tag internal, then **regenerate the call graph**, `/scan-permissions`, `/score-contract` — the full bring-into-scope path.
- **Holds funds** (appears in `funds-data.json` or is a vault/treasury): enable funds tracking via the funds tags.

### `$implementation` swap

The proxy's `call-graph-data.json` + `functions.json` analyzed the **old** impl. Recommend: regenerate the call graph; if the new impl's ABI function-set differs from the old, also `/scan-permissions` + `/score-contract` on the proxy. Check the ABI diff by comparing the function lists under the old vs new impl address in `discovered.json.entries[proxy].abis[]`.

### `contract-removed`

Stale entries may dangle in `functions.json`, `contract-tags.json`, and `review-config.json` (`admins` / `dependencies`). Recommend deleting them. Find them:

```bash
for FILE in functions.json contract-tags.json review-config.json; do
  grep -il "<0x...>" packages/config/src/projects/<project>/$FILE
done
```

### Safe `$members` / `$threshold` change

Any `review-config.json` description that quotes the old `N-of-M` is now wrong. Grep descriptions for the Safe address; flag stale `N-of-M` / `N of M` / `N/M` phrases — including **parent** Safes whose description names this Safe as a nested signer. Recommend updating via `/generate-review` or a direct edit.

### `owner` rotation

A `review-config.json` admin entry naming the old owner is now stale. Recommend updating / transferring the description (`/generate-review` or direct edit). If the old owner was *also* `contract-removed` this cycle, the cleanup above already covers deleting its entry.

---

## Step 3 — Report everything in one go (then STOP)

Process **all** projects with activity today, then emit a single consolidated report. Do not ask anything per-project; do not act yet.

Use globally-numbered items so the user can refer to anything by number across projects.

```
## Daily activity — <today>

<project A>: N events
  REAL (k):
    1. <ContractName> .<field>: <before> → <after>
       Impact: <1-2 sentences>
  NOISE — proposed strip + ignoreInWatchMode (k):
    2. <ContractName> .<field> — <one-line reason>
  UNCERTAIN (k):
    3. <ContractName> .<field> = <value> — <one-line reason>
  FOLLOW-UP NEEDED (k):
    4. NEW CONTRACT <Name> (<eth:0x…>) — <verified?>, <position: referenced by X / governs Y / holds $Z / standalone>.
       → Recommended: <classification + action>. Run: <command or /sub-skill>.
    5. UPGRADE <Proxy> — call graph + scoring analyzed the old impl.
       → Run: regenerate call graph; /scan-permissions + /score-contract if the ABI changed.
    6. STALE DESCRIPTION <Admin/Safe> — review-config quotes "<old N-of-M>", now "<new N-of-M>".
       → Run: /generate-review or edit the description directly.

<project B>: N events
  ...

---

Totals: M real, K proposed noise, J uncertain, F follow-up across P projects.

Awaiting your call. Examples:
  - "approve" / "go" — apply all proposed NOISE strips, leave UNCERTAIN alone
  - "approve, also clean 3,7" — promote those UNCERTAIN items into the strip set
  - "skip 5" — drop item 5 from the strip set
  - "skip <project>" — leave that project untouched
  - "skip all" — apply nothing

FOLLOW-UP items are informational — "approve" does NOT action them. They list the
exact command/sub-skill for the researcher to run separately. If the user says
"do 4" or "tag 4 external", run the named sub-skill then; otherwise leave them be.
```

**Stop here.** Wait for the user to respond. Do not run Step 4 until they approve. Never auto-run a follow-up action — only when the user names it. If they ask follow-up questions about specific items, answer from `discovered.json` / `contract-tags.json` / source — don't apply anything yet.

---

## Step 4 — Apply noise fixes (only after explicit approval)

Group strips by `updateNotifierId` (one POST per row covers every noisy field on it):

```bash
curl -sf -X POST localhost:2021/api/defidisco/monitor/rows/<updateNotifierId>/strip \
  -H 'Content-Type: application/json' \
  -d '{
    "fields": [{ "address": "<eth:0x...>", "key": "<field-from-event>" }],
    "addToIgnoreWatchMode": true
  }'
```

The endpoint cascades: drops the matching `changes[]` entries from `activity.json`, promotes the bare field name to per-contract `ignoreInWatchMode` in `config.jsonc` (the backend strips `values.` and skips structural keys it knows are invalid), and recompiles the review.

**Template-level shortcut:** if a noisy field appears on every contract sharing one template (check `discovered.json.entries[*].template`), edit the template's `config.jsonc` in `packages/config/src/projects/_templates/<template>/config.jsonc` directly instead of letting the API write per-contract — one edit, fleet-wide effect.

If the API is 503: hand-edit the project's `config.jsonc` + `activity.json`, then `cd packages/config && l2b discover <project>`.

---

## Step 5 — Real-change impact

For each real change, one or two sentences. Cross-ref `discovered.json` + `contract-tags.json` for entity labels.

- **Role / admin update** — name the role, name the new holder, say what powers it has.
- **Upgrade** — name the proxy + new impl, flag suspicious patterns (no timelock, unverified impl).
- **Parameter change** — quote `before → after`, translate to user impact (fee % change, cap raised, etc.).
- **Add / remove** — name the contract, where it sits in the system, why it appeared.

Don't fabricate. If you'd need to read source to know what a role or parameter does, say so — flag it for source review rather than guessing.

---

## Step 6 — Confirm what was applied

After the strips run, print one short block: which items were stripped, which were skipped per the user's instructions, **any FOLLOW-UP items still outstanding** (with their recommended command, so the researcher doesn't lose them), and a one-line `git status` of `packages/config/src/projects/` so the user sees what to commit.

**Recompile every project whose activity you edited.** The strip endpoint already recompiles each project it touches, but trigger an explicit recompile per touched project to be safe (and to cover the manual-edit fallback path):

```bash
curl -sf -X POST localhost:2021/api/projects/<project>/compile-review
```

---

## Step 6.5 — Fact-check the digest before writing it

Before drafting Step 7, verify every concrete claim you intend to make. The digest goes to end users; a wrong unit or a stale-state assumption is worse than vague language.

For each REAL change you're about to describe, do **both**:

1. **Read the field directly from `discovered.json`** (don't infer current state from activity history — events can be missing or incomplete). Confirm `before`, `after`, related sibling fields (e.g. all four `is*Paused` flags when describing one of them), and units like `decimals`/`symbol`.
2. **Read the source code** for any parameter you're about to interpret. Specifically:
   - For fees / commissions / amounts: open the contract's flat source and find where the field is *used* — is it compared directly to an amount (absolute units) or multiplied/divided by a denominator like `10000` (basis points)? Don't assume "fee" means "basis points".
   - For roles / permissions: find the modifier or `require` and name what it actually gates.
   - For pause / mode flags: find the function that reads the flag to confirm what behavior it blocks.

If you can't verify a claim from field + source, either drop it or hedge it explicitly ("the unit isn't visible from the available source"). **Never guess a unit.**

---

## Step 7 — Public digest

Write a **user-facing digest** of today's real changes — the output that gets sent to end users.

**Audience:** end users following these protocols. They want the facts, in plain English, and they will form their own opinion.

**Voice:** factual, plain English, conversational. Report what happened, not what it means.

**Hard rules — never include:**

- **Risk verdicts or judgement words.** No "routine", "standard", "long-overdue", "no risk event", "nothing to worry about", "as expected", "worth watching", "heads up", "concerning". State the change; let the reader judge.
- **Protocols with nothing to say.** If a protocol had only noise / monitoring-scope changes today, **do not mention it at all** — leave it out of the digest entirely.
- **Internal monitoring or scope language.** "We added X to monitoring scope", "we cleaned up which contracts we track", "internal cleanup", `ignoreInWatchMode`, `discovered.json`, `activity.json`, "monitor", "diff", "field". The digest is about what protocols did, not what we did.
- **Contract addresses, function names, field paths** (`values.foo`, `accessControl.X`).
- **DeFiDisco / L2BEAT jargon:** "permission overrides", "compiled review", "discovery engine".
- **Speculation dressed as fact.** If you don't know what a role does, say "a role we haven't fully mapped" rather than inventing one.
- **Filler closers.** No "Nothing flagged for follow-up today.", no "Stay safe out there", no "More next time". When you're done stating the facts, stop.

**Do include:**

- Protocol name, in plain English
- What changed, in one or two sentences — translate technical terms (a "Safe" → "multisig wallet"; "$implementation swap" → "the protocol pushed a code update"; "added a market with higher priority in supplyQueue" → "added a new lending market and placed it ahead of the existing markets in the deposit queue")
- The concrete facts a user needs to understand the change: what was added/changed/swapped, in what direction, relative to what was there before
- For multisigs: the old/new signer count and whether the threshold changed (these are facts, not judgements)
- For new markets / parameter changes: the previous state and the new state, so the reader can size the delta themselves
- **Activity-page link** for every protocol section. URL pattern: `https://defiscan.info/protocol/<slug>?view=activity` where `<slug>` is the directory name under `packages/config/src/projects/` (case-sensitive — e.g. `Steakhouse-USDC`, `aave-v3`). Drop it on its own line directly under the protocol's text — Telegram auto-linkifies bare URLs, so no markdown is needed.

**Format — plaintext for Telegram (no markdown).** The digest is copy-pasted directly into the Telegram app, which renders no markdown — `#`, `*`, `**`, `_` all show up literally. Use only blank lines and ALL-CAPS protocol names for visual structure. Never use `#`, `*`, `**`, `_`, backticks, or any other markdown syntax. Don't quote field paths or addresses. Bare `https://…` URLs are fine — Telegram turns them into clickable links automatically.

```
DeFi Digest — <today, written friendly e.g. "May 4, 2026">

<one-line factual intro: "Two protocols changed today: a new lending market on Steakhouse Prime USDC and a signer rotation on Aave." — no tone-setting, no judgement.>

<PROTOCOL NAME IN ALL CAPS>
<1-3 factual sentences. What changed, what it was before, what it is now.>
https://defiscan.info/protocol/<slug>?view=activity

<PROTOCOL NAME IN ALL CAPS>
<...>
https://defiscan.info/protocol/<slug>?view=activity
```

The title is always `DeFi Digest` — never `DeFi Daily` (the routine isn't guaranteed to run daily). Separate sections with one blank line. No bullet points, no dashes-as-bullets, no horizontal rules.

If the day was 100% noise: emit two lines only — `DeFi Digest — <date>` then `No protocol changes today.` Nothing else.

**Length:** one screen total. If you're writing more than ~3 sentences per protocol, you're editorialising.

---

## Guidelines

- **Never apply anything before the user approves.** Steps 1–3 are read-only. Step 4 only runs after explicit go.
- **When uncertain, classify UNCERTAIN.** A noisy alert is annoying; a missed admin change is an incident.
- **Every new contract gets a FOLLOW-UP item.** A new contract is invisible in the compiled review until tagged + brought into scope — this is the most-missed gap. Never assume an unfamiliar contract is harmless background.
- **Follow-up is detect-and-recommend only.** Surface the work and the exact command; never tag, regenerate, score, or edit descriptions unless the user names the item.
- **Group strips by `updateNotifierId`.** One POST per row.
- **Prefer template edits** when a field is fleet-wide noise across one template.
- **One sentence beats a paragraph.** The report is meant to be skimmed in 30 seconds.
