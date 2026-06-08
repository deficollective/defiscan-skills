---
name: sync-role-names
description: Scan a project's flattened Solidity sources for AccessControl/EnumerableRoles role constants, compute their keccak hashes, and keep the roleNames map in config.jsonc complete. Prevents the EnumerableRolesHandler from silently degrading to hash-keyed roles (which breaks role-name owner paths) when .flat/ sources are unavailable, e.g. in CI.
argument-hint: "[project-name]"
---

# Sync Role Names

The `enumerableRoles` discovery handler reads a project's `.flat/` Solidity sources to map on-chain role hashes back to human role names. When `.flat/` is missing (the monitor runs in CI from a clean checkout where `.flat/` is gitignored), the handler falls back to keying roles by raw hash — which silently breaks every `@<contract>.roles.<ROLE_NAME>` owner path in `functions.json`, dropping admins from the analysis.

This skill builds a durable `roleNames` map (hash → role name) directly in `config.jsonc` so the handler resolves names even without `.flat/`. Run it routinely (e.g. after discovering a new project, or after a protocol upgrade adds roles) to keep that map complete.

## Arguments

```
/sync-role-names [project]
```

- **project** — optional project folder name (e.g. `EtherFi-Stake`). If omitted, the skill processes **every** project whose `config.jsonc` declares an `enumerableRoles` handler.

Parse `$ARGUMENTS` to extract the project name as `$0` (may be empty).

## Prerequisites

- The project must have a populated local `.flat/` directory (run discovery first if it doesn't exist — `.flat/` is gitignored, so a fresh clone won't have it).
- `cast` (Foundry) must be available for keccak hashing.
- `jsonc-parser` is resolvable from `packages/l2b/node_modules` — run the config-editing node scripts with `packages/l2b` as the working directory.

---

## Step 1: Identify target projects and handler locations

Find every `config.jsonc` that uses the handler:

```bash
if [ -n "$0" ]; then
  echo "packages/config/src/projects/$0/config.jsonc"
else
  grep -rl '"type": "enumerableRoles"' packages/config/src/projects/*/config.jsonc
fi
```

For each target `config.jsonc`, read it and locate **every** `enumerableRoles` handler block. Each lives at a path like:

```jsonc
"overrides": {
  "eth:0x62247D29B4B9BECf4BB73E0c722cf6445cfC7cE9": {
    "fields": {
      "roles": {
        "handler": {
          "type": "enumerableRoles",
          "flatDir": "src/projects/EtherFi-Stake/.flat",
          "roleNames": { /* hash -> name, may be absent */ }
        }
      }
    }
  }
}
```

Record, per handler: the contract address, the `field` name (the key under `fields`, usually `roles`), the `flatDir`, and the existing `roleNames` object (or empty if absent). A project may have more than one such handler.

If a project has no `enumerableRoles` handler, report that and skip it.

---

## Step 2: Scan flattened sources for role constants

Resolve `flatDir` relative to `packages/config/`. If the directory does not exist or contains no `.sol` files, report it and skip this handler (the skill cannot build the map without sources — tell the user to run discovery for that project first).

Scan recursively for role constants using the **same regex the handler uses** so the result is identical:

```bash
# $FLATDIR = absolute path to the project's .flat directory
grep -rhoE 'bytes32[[:space:]]+public[[:space:]]+constant[[:space:]]+[A-Za-z0-9_]+[[:space:]]*=[[:space:]]*keccak256[[:space:]]*\([[:space:]]*"[^"]+"' "$FLATDIR" \
  | sed -E 's/.*constant[[:space:]]+([A-Za-z0-9_]+)[[:space:]]*=[[:space:]]*keccak256\([[:space:]]*"([^"]+)".*/\1\t\2/' \
  | sort -u
```

Each line is `constantName <TAB> keccakString`. The handler keys the map by **`keccak256(keccakString)`** and stores the **`constantName`** as the value — mirror that exactly. In the common case `constantName === keccakString` (true for all EtherFi-Stake roles), but handle the case where they differ.

Notes / limitations to mention in the final report:
- The regex only matches `bytes32 public constant NAME = keccak256("STRING")`. Roles defined as inline hex literals, `uint256` roles, or computed differently are not captured.
- Non-role constants of the same shape (e.g. rate-limiter `*_LIMIT_ID` keys) are also captured. They are harmless — they simply never appear as roles on-chain — but note them so the reviewer isn't surprised.
- If two different constant names hash to the same value (same `keccakString`), report the collision; the handler would have used last-wins.

---

## Step 3: Compute hashes

For each unique `keccakString`, compute the hash with `cast`:

```bash
cast keccak "EETH_OPERATING_ADMIN_ROLE"
# => 0x06b452e947f709c0549c7a2e857f0d57f53a00c27bb826a3340a48774a76512f
```

`cast keccak "<string>"` hashes the UTF-8 bytes of the string, identical to the handler's `solidityKeccak256(['string'], [s])`.

Build a JSON object `{ "<lowercase-hash>": "<constantName>", ... }` — this is the candidate `roleNames` map. Lowercase all hash keys (the handler lowercases on lookup).

---

## Step 4: Diff against existing roleNames

For each handler, compare the candidate map against the existing `roleNames` in `config.jsonc`:

- **To add** — candidate hashes not present in existing `roleNames`.
- **Already present** — candidate hashes already in `roleNames` with the same name (no-op).
- **Conflict** — candidate hash present in `roleNames` but with a *different* name. Report it; do NOT auto-overwrite — the existing value may be an intentional manual override. Ask the user.
- **Stale (existing-only)** — `roleNames` entries whose hash is not in the candidate map. Do NOT delete — the role's source may simply not be in `.flat/` (e.g. a dependency contract). Just report.

---

## Step 5: Cross-check discovered.json

Read `packages/config/src/projects/<project>/discovered.json`. For the contract that has the `enumerableRoles` handler, inspect its `values.<field>` map (e.g. `values.roles`). Any key starting with `0x` is a role that is **live on-chain right now but unnamed** — exactly the silent-degradation symptom.

For each such hash-keyed entry:
- If its hash is in the candidate map → adding `roleNames` will fix it on the next discovery. Flag as **"will be resolved"**.
- If its hash is NOT in the candidate map → the role exists on-chain but no matching constant was found in `.flat/`. Flag as **"UNKNOWN — needs manual review"** (the source may be missing, the constant uses an unsupported form, or it is an inherited OZ role like `DEFAULT_ADMIN_ROLE = 0x00..00`).

---

## Step 6: Present and confirm

Present per project / per handler:

```
## Project: EtherFi-Stake — handler on eth:0x62247D29... (field: roles)

flatDir: src/projects/EtherFi-Stake/.flat (116 .sol files)
Role constants found in source: 41
Existing roleNames entries: 0

TO ADD (41):
  0x06b452e9...  EETH_OPERATING_ADMIN_ROLE
  0x0e8d9412...  ETHERFI_NODES_MANAGER_ADMIN_ROLE
  ...

ALREADY PRESENT (0)
CONFLICTS (0)
STALE / existing-only (0)

discovered.json cross-check — currently hash-keyed roles: 31
  will be resolved by this sync: 31
  UNKNOWN (no matching constant): 0
```

Then ask:

> Apply these `roleNames` additions to config.jsonc?
> - `apply` — add all TO ADD entries to every handler
> - `apply <project>` — only a specific project
> - `skip` — don't write anything

Conflicts always require explicit per-item user direction before writing.

---

## Step 7: Apply changes

Merge the additions into `config.jsonc` **comment-preserving** using `jsonc-parser`. Write this script to a temp file. `jsonc-parser` is not installed at the repo root — the script resolves it from `packages/l2b/node_modules` (a script in `/tmp` resolves modules relative to its own path, not cwd, so `require('jsonc-parser')` directly would fail):

```js
// /tmp/sync-role-names-apply.js
// run: node /tmp/sync-role-names-apply.js <abs-config-path> <address> <field> <abs-additions-json> packages/l2b
const fs = require('fs')
const jsoncPath = require('child_process')
  .execSync('node -e "console.log(require.resolve(\'jsonc-parser\'))"', {
    cwd: process.argv[6] || process.cwd(),
  })
  .toString()
  .trim()
const { modify, applyEdits, parse } = require(jsoncPath)

const configPath = process.argv[2]            // absolute path to config.jsonc
const address = process.argv[3]               // "eth:0x..." (match casing in config.jsonc)
const field = process.argv[4]                 // "roles"
const additions = JSON.parse(fs.readFileSync(process.argv[5], 'utf8')) // { "<hash>": "<name>", ... }

let text = fs.readFileSync(configPath, 'utf8')
const basePath = ['overrides', address, 'fields', field, 'handler', 'roleNames']

for (const [hash, name] of Object.entries(additions)) {
  const edits = modify(text, [...basePath, hash.toLowerCase()], name, {
    formattingOptions: { insertSpaces: true, tabSize: 2 },
  })
  text = applyEdits(text, edits)
}

fs.writeFileSync(configPath, text)
const errs = []
parse(text, errs)
if (errs.length) { console.error('PARSE ERRORS:', JSON.stringify(errs)); process.exit(1) }
console.log(`Added ${Object.keys(additions).length} roleNames entries to ${address}; parses OK`)
```

Pass the additions as a **file path** (not an inline arg) — the map can be 40+ entries. Use the exact address casing that already appears in `config.jsonc`.

Merge rules:
- **Never remove** existing `roleNames` entries.
- Append/insert only; existing entries with the same hash are left untouched unless the user explicitly resolved a conflict.
- `jsonc-parser`'s `modify` creates the `roleNames` object (and any missing parent objects) if absent, and preserves all surrounding comments.
- Keep `flatDir` in the config — it stays as the local-dev convenience path; `roleNames` is the durable CI-safe source of truth.

After writing, validate the file still parses:

```bash
cd packages/l2b && node -e "const {parse}=require('jsonc-parser'); const e=[]; parse(require('fs').readFileSync(process.argv[1],'utf8'),e); console.log(e.length? 'PARSE ERRORS: '+JSON.stringify(e) : 'config.jsonc OK')" <abs-config-path>
```

---

## Step 8: Re-run discovery (recommended)

For each project whose config changed, re-run discovery so `discovered.json` re-keys its `roles` map by name:

```bash
cd packages/config && l2b discover <project> --dev 2>&1 | tail -5
```

Then re-check the `roles` map in `discovered.json` — there should be no remaining `0x`-prefixed keys (except genuinely unknown roles flagged in Step 5).

---

## Step 9: Summary

```
## Sync Role Names Summary

Projects processed: N
  EtherFi-Stake: +41 roleNames (0 -> 41), 0 conflicts, 0 unknown
  ...
Re-discovered: N projects
Remaining unnamed on-chain roles: N (list any UNKNOWN ones for manual review)
```

---

## Guidelines

- **Mirror the handler exactly.** Use the same regex and the same `keccak256(string)` hashing so the `roleNames` map is byte-for-byte what the handler would have produced from source. Key by hash, value is the Solidity constant name.
- **Never overwrite on conflict.** An existing `roleNames` entry with a different name may be a deliberate manual override — surface it, let the user decide.
- **Never delete stale entries.** A `roleNames` entry without a matching source constant is not necessarily wrong (the source may live outside `.flat/`).
- **`functions.json` paths must use the constant name.** Owner paths are `@<contract>.roles.<CONSTANT_NAME>`. The map's values are constant names, so they line up. If a constant's `keccak256("string")` argument differs from the constant name, the path must use the *constant name*, not the string.
- **Preserve JSONC comments and other overrides.** Only touch the `roleNames` object; leave `flatDir`, `ignoreMethods`, `ignoreRelatives`, etc. intact.
- **Addresses are chain-prefixed** (`eth:0x...`, `arb1:0x...`) and checksummed, matching the keys already in `config.jsonc`.
