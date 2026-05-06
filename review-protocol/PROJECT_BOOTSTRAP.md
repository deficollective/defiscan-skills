# Project Bootstrap

How to create the project folder and stub it for downstream skills. This is Step 1 of `/review-protocol`.

The pattern mirrors `/review-morpho-vault`'s bootstrap but stays generic — no protocol-specific config baked in. The minimal `config.jsonc` written here gives `/run-discovery` enough to start; you'll layer protocol-specific overrides as discovery surfaces them.

---

## 0. Validate inputs

```bash
# $0: project slug — lowercase, hyphens, no spaces
echo "$0" | grep -qE '^[a-z0-9][a-z0-9-]*$' \
  || { echo "Slug must be lowercase alphanumeric+hyphens (got: $0)"; exit 1; }

# $1: at least one chain-prefixed address (comma-separated for multi-anchor protocols)
for addr in $(echo "$1" | tr ',' '\n'); do
  echo "$addr" | grep -qE '^(eth|base|arb1|optimism|polygon|bsc|avalanche|gnosis):0x[0-9a-fA-F]{40}$' \
    || { echo "Address must be chain-prefixed (got: $addr)"; exit 1; }
done
```

If a bare `0x...` is passed, ask the user which chain instead of guessing — guessing on a non-Ethereum protocol silently produces a broken project (admin keys won't match, `/generate-review` falls through to auto-generated names with no error).

## 0a. Validate server is running

```bash
curl -sf localhost:2021/api/projects > /dev/null \
  && echo "OK" || echo "SERVER_NOT_RUNNING — run 'cd packages/config && l2b ui' and retry"
```

If the server is not running, stop and tell the user.

## 0b. Refuse to clobber

```bash
if [ -d "packages/config/src/projects/$0" ]; then
  echo "Project $0 already exists. Pass a different slug or delete the existing folder first."
  exit 1
fi
```

Never overwrite a researcher's existing project. Ask for a different slug.

---

## 1. Create the project folder

```bash
mkdir -p "packages/config/src/projects/$0"
```

## 2. Write the minimal `config.jsonc`

Use the chain prefix from `$1` (the first address if multiple were passed) for the `$schema` path resolution and any chain-specific paths.

```jsonc
{
  "$schema": "../../../../discovery/schemas/config.v2.schema.json",
  "name": "<slug>",
  "import": ["../globalConfig.jsonc"],
  "initialAddresses": [
    "<address1>"
    // add more entries for multi-anchor protocols
  ],
  "maxAddresses": 100,
  "maxDepth": 3
}
```

That's it. No `overrides`, no `ignoreMethods`, no handlers. `/run-discovery` will iteratively add overrides (leafing externals, adding handlers when array methods overflow, etc.) as the graph surfaces. Pre-baking overrides here is wrong — you don't yet know which addresses will appear or which fields will overflow.

The only exception worth pre-baking is when you know in advance that a contract is the protocol's main entry point AND it has a known method-overflow problem (e.g. a factory with a `getAllPools()` getter). In that case add an `ignoreMethods` entry under `overrides` for that one address. Otherwise leave overrides empty.

## 3. Initialize the metadata files

Empty placeholders so the API has something to read. `/scan-permissions`, `/gather-resources`, etc. populate them later:

```bash
DIR="packages/config/src/projects/$0"
echo '{"version":"1.0","contracts":{}}' > "$DIR/functions.json"
echo '{"version":"1.0","tags":[]}'      > "$DIR/contract-tags.json"
echo '{"resources":[],"audits":[]}'     > "$DIR/resources.json"
```

Do **not** create `governance.json` — `/generate-governance` writes it.
Do **not** create `review-config.json` — `/generate-review` writes it.
Do **not** create `permission-overrides.json` — its writes happen via `/scan-permissions` and `/score-contract` through the API.

## 4. Register the project in `defidisco-config.json`

Without this, the project doesn't appear in the gallery / API filter and the frontend never loads it — even after `compile-review` writes the compiled file.

Append the slug to `defiProjects[]` in `packages/config/src/defidisco-config.json`:

```bash
python3 -c "
import json
path = 'packages/config/src/defidisco-config.json'
d = json.load(open(path))
slug = '$0'
if slug not in d['defiProjects']:
    d['defiProjects'].append(slug)
    json.dump(d, open(path, 'w'), indent=2)
    print(f'Added {slug}')
else:
    print(f'{slug} already present')"
```

Keep the file's existing JSON formatting (2-space indent).

---

## Bootstrap is done

At this point:
- `packages/config/src/projects/$0/config.jsonc` exists with `initialAddresses` + `maxDepth: 3`.
- The three stub metadata files exist.
- The slug is registered.

Hand control back to the main SKILL.md and proceed to Step 2 (Discovery + resources).
