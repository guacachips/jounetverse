# Agent Guide — Jounetverse Client Modpack

This guide is for LLMs managing the Jounetverse client modpack.
The modpack is distributed as an `.mrpack` file and installed via Prism Launcher.

**GitHub repo:** `github.com/guacachips/jounetverse`
**Based on:** CobbleVerse 1.7.2 (Modrinth distribution)
**Minecraft:** 1.21.1 (Fabric)

For big-picture context see `../AGENTS.md`.
This file contains **no server credentials** — those are in `../server/AGENTS.md`.

---

## Repo Structure

```
client-modpack/
├── modrinth.index.json    ← mod list: URLs, hashes, env flags, file sizes
├── overrides/             ← files bundled directly into the modpack
│   ├── config/            ← pre-configured mod settings for clients
│   └── mods/              ← JARs that aren't on Modrinth (bundled directly)
└── README.md
```

---

## modrinth.index.json Structure

The file contains a `files` array. Each entry looks like:

```json
{
  "path": "mods/modname-version.jar",
  "hashes": {
    "sha512": "<sha512 hash>",
    "sha1": "<sha1 hash>"
  },
  "env": {
    "client": "required",
    "server": "optional"
  },
  "downloads": [
    "https://cdn.modrinth.com/data/<PROJECT_ID>/versions/<VERSION_ID>/<filename>.jar"
  ],
  "fileSize": 123456
}
```

### `env` flag meanings

| Value | Meaning |
|-------|---------|
| `required` | Must be present on this side |
| `optional` | Recommended but not required |
| `unsupported` | Must NOT be present on this side |

An entry with `"client": "required", "server": "unsupported"` is a pure client mod.
An entry with both `"client": "required", "server": "required"` must be on both sides.

### Top-level fields

```json
{
  "formatVersion": 1,
  "game": "minecraft",
  "versionId": "1.0.5",
  "name": "Jounetverse",
  "files": [...],
  "dependencies": {
    "minecraft": "1.21.1",
    "fabric-loader": "0.16.10"
  }
}
```

Bump `versionId` every time you add, remove, or update a mod entry.

---

## Versioning Convention

- **Patch bump** (e.g. `1.0.4` → `1.0.5`): adding/removing/updating individual mods, config tweaks
- **Minor bump** (e.g. `1.0.x` → `1.1.0`): larger changes — new modpack theme, many mods at once, Minecraft/Fabric version bump

---

## How to Add a Mod

### 1. Find the mod version on Modrinth

```bash
# Get version info (replace VERSION_ID with the actual Modrinth version ID)
curl -s "https://api.modrinth.com/v2/version/VERSION_ID" | jq '{id, name, files, game_versions, loaders}'
```

Or fetch by project slug + game version:
```bash
curl -s "https://api.modrinth.com/v2/project/PROJECT_SLUG/version?game_versions=[\"1.21.1\"]&loaders=[\"fabric\"]" | jq '.[0]'
```

### 2. Extract the required fields from the API response

From the version object:
- `files[0].url` → the download URL
- `files[0].hashes.sha512` → sha512 hash
- `files[0].hashes.sha1` → sha1 hash
- `files[0].size` → fileSize
- `files[0].filename` → use as `path`: `"mods/<filename>"`

Check the project page or `files[0].primary` for the primary file if multiple files are listed.

### 3. Determine env flags

Check the Modrinth project page "Environments" section, or:
```bash
curl -s "https://api.modrinth.com/v2/project/PROJECT_SLUG" | jq '{client_side, server_side}'
```

Map Modrinth project-level env to modrinth.index.json env:
- `client_side: required` → `"client": "required"`
- `client_side: optional` → `"client": "optional"`
- `client_side: unsupported` → `"client": "unsupported"`
- `server_side: required` → `"server": "required"`
- etc.

### 4. Add the entry to modrinth.index.json

Add to the `files` array (maintain alphabetical order by `path` if possible):

```json
{
  "path": "mods/modname-1.2.3+mc1.21.1.jar",
  "hashes": {
    "sha512": "<sha512>",
    "sha1": "<sha1>"
  },
  "env": {
    "client": "required",
    "server": "optional"
  },
  "downloads": [
    "https://cdn.modrinth.com/data/<PROJECT_ID>/versions/<VERSION_ID>/modname-1.2.3+mc1.21.1.jar"
  ],
  "fileSize": 123456
}
```

### 5. Bump versionId

Increment the patch version in the top-level `versionId` field.

### 6. Check if the mod also needs to go on the server

See the decision matrix in `../AGENTS.md`. If `server: required` or `server: optional`, also upload the JAR to the live server via SFTP (see `../server/AGENTS.md`).

---

## How to Remove a Mod

1. Delete the mod's entry from the `files` array in `modrinth.index.json`
2. Bump `versionId`
3. If the mod was also on the server, remove it there too (see `../server/AGENTS.md`)

---

## How to Build the mrpack

From the `client-modpack/` directory:

```bash
zip -r ../jounetverse-X.X.X.mrpack modrinth.index.json overrides/
```

Replace `X.X.X` with the current `versionId`.

The resulting `.mrpack` file lands in the parent directory (`jounetverse/`).

---

## How to Release on GitHub

After building the mrpack:

```bash
gh release create vX.X.X ../jounetverse-X.X.X.mrpack \
  --repo guacachips/jounetverse \
  --title "Jounetverse vX.X.X" \
  --notes "Brief description of what changed in this release."
```

Players download the `.mrpack` from the GitHub releases page and import it into Prism Launcher.

---

## Git Workflow

```bash
# Stage and commit changes
git add modrinth.index.json overrides/ AGENTS.md
git commit -m "Brief description of change"
git push
```

Always push after changes so the repo stays up to date.

---

## overrides/ Structure

Files in `overrides/` are extracted directly into the Minecraft instance directory on install.
Common uses:

- `overrides/config/` — pre-configured mod settings (e.g. keybinds, mod options)
- `overrides/mods/` — JARs not available on Modrinth (bundled directly, not listed in `files`)

If you add a file to `overrides/`, it does not need a `files` entry. If you add a mod JAR to
`overrides/mods/`, it will not be verified by hash — so prefer Modrinth-hosted entries where possible.
