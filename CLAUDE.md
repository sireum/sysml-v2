# SysML v2 (SysIDE fork) — Extension Dev Guide

This repo is Sireum's fork of SysIDE (Langium-based SysML v2/KerML tooling), packaged as
the **`sireum.sysml-v2`** VS Code extension. It is one of two SysML-related extensions used
in Sireum's CodeIVE (VSCodium) distribution:

| Extension | Repo | Provides | Built with |
|---|---|---|---|
| `sireum.sysml-v2` (SysIDE fork) | **this repo** | TextMate grammar + Langium TS language server | pnpm / esbuild / vsce |
| `sireum.vscode-extension` | `sireum/vscode-extension` (cloned at `$SIREUM_HOME/vscode-extension`) | TextMate grammar + client that launches the Java `sysml-lsp-server.jar` over stdio | npm / esbuild / vsce |

Both contribute a grammar for `scopeName: source.sysml`, so **only one can be enabled at a
time for SysML highlighting** (see Testing). They are pinned independently in kekinian
`versions.properties` and installed by `sireum setup vscode` (see `runtime/.../Init.scala`).

---

## What to install

- **Node** (CI uses 22 for this repo; 20 for the vscode-extension). 
- **pnpm** — this repo only: `npm install -g pnpm` (or `corepack enable && corepack prepare pnpm@10 --activate`). CI pins pnpm 10.
- **vsce** — packaging tool, NOT a package.json dependency: `npm install -g @vscode/vsce` (the repos invoke `vsce`/`npx @vscode/vsce` from PATH).
- **Sireum** (`$SIREUM_HOME`) — for `sireum setup vscode` and the kekinian build.

The `sireum.vscode-extension` repo additionally only needs `npm` + `vsce` (its `tsc`/`eslint`/
`esbuild` come via `npm install` as devDeps); see that repo's `CLAUDE.md`/`readme-dev.md`.

---

## SysML grammar — source of truth (important)

The grammar lives at:

```
packages/syside-languageserver/syntaxes/sysml.tmLanguage.json   <-- EDIT THIS
```

- It is **hand-patched and committed** (e.g. the GUMBO keyword rules). Edit it directly.
- ⚠️ **Never run `pnpm run grammar:generate` (or `langium generate`).** `langium-config.json`
  declares this file as a generated TextMate output (`textMate.out: syntaxes/sysml.tmLanguage.json`
  from `src/grammar/SysML.langium`). Regenerating **overwrites all hand patches**, wiping the
  GUMBO customizations. The normal `pnpm run build` does NOT regenerate it.
- `packages/syside-vscode/syntaxes/` is **gitignored** (a build artifact) — never edit there.

### How the grammar reaches the vsix
1. languageserver `prebuild`: `cp -R ./syntaxes ../syside-vscode/syntaxes` (copies your edit in).
2. vscode `scripts/prepublish.mjs`: re-copies the *other* syntaxes (kerml…) but is filtered to
   **preserve `sysml.tmLanguage.json`**, so your version is what ships.

### GUMBO keywords
The canonical keyword list is the GUMBO grammar in `sireum/hamr-sysml-parser`
(`.../parser/GUMBO.tokens`). Add only GUMBO keywords that are **not already** base SysML
keywords elsewhere in the grammar (e.g. `after`, `abstract`, `def`, `state` are already SysML
keywords — leave them in their existing rules). Keep this grammar in sync with the identical
GUMBO rules in `sireum/vscode-extension`'s `syntaxes/sysml.tmLanguage.json`.

Current GUMBO additions (mirrored in both grammars):
- `keyword.other.gumbo.sysml` rule: `invariants inv integration initialize compute compute_cases
  composition components ports schema label split sequence infoflow handle guarantee functions
  mut invariant reads modifies`
- separate `keyword.control.import.sysml` block: `before` (colors like `at`/`after`)
- separate `keyword.declaration.type.sysml` block: `property` (colors like type-def keywords)
- `storage.modifier.annotation.sysml`: `@strictpure @pure @spec`

---

## Build (this repo)

```bash
npm install -g pnpm                  # once
cd <this-repo>
pnpm install                         # large; also runs `prepare` -> `pnpm run build`
pnpm run build                       # prebuild (copies syntaxes) -> tsc -> esbuild
```

### Package the vsix
```bash
cd packages/syside-vscode
npx @vscode/vsce package --no-dependencies --allow-unused-files-pattern
mv *.vsix sireum-sysml-v2.vsix       # CI naming; internal version is 0.9.1
```

**pnpm 11 gotcha:** `vsce` runs the `vscode:prepublish` script via pnpm, and pnpm 11's
"verify-deps-before-run" spawns a nested `pnpm install` that exits non-zero on un-approved
native build scripts (`ERR_PNPM_IGNORED_BUILDS` for swc/esbuild/keytar). Work around with:
```bash
pnpm config set verify-deps-before-run false
```
CI avoids this (pnpm 10 + `--frozen-lockfile`). The CI also bundles the SysML standard library
(`packages/syside-languageserver/scripts/clone-sysml-release.mjs` → `packages/syside-vscode/sysml.library`);
that step is only needed to match the released artifact, not for a highlighting test.

---

## Test (local, in CodeIVE)

CodeIVE is the portable VSCodium under `$SIREUM_HOME/bin/<os>/vscodium/`. Its extensions live in:
```
$SIREUM_HOME/bin/<os>/vscodium/codium-portable-data/extensions/
```

Because both SysML extensions register `source.sysml`, **enable exactly one**:
to test THIS extension's grammar, enable `sireum.sysml-v2` and disable `sireum.vscode-extension`.

Deploy the freshly built vsix over the installed dir (the vsix's internal version is `0.9.1`,
matching the installed `sireum.sysml-v2-0.9.1` dir). Overlay rather than wipe, to preserve the
bundled `sysml.library`:
```bash
EXT="$SIREUM_HOME/bin/mac/vscodium/codium-portable-data/extensions"
TARGET="$EXT/sireum.sysml-v2-0.9.1"
TMP=$(mktemp -d)
unzip -q packages/syside-vscode/sireum-sysml-v2.vsix "extension/*" -d "$TMP"
cp -R "$TMP/extension/." "$TARGET/"     # overwrites files, keeps sysml.library
rm -rf "$TMP"
```
Then in CodeIVE run **Developer: Reload Window** (grammar-only change, same version → no full
restart) and open a `.sysml` file. Use **Developer: Inspect Editor Tokens and Scopes** to check
scopes (e.g. `keyword.other.gumbo.sysml`).

Quick artifact check without the editor:
```bash
unzip -p packages/syside-vscode/sireum-sysml-v2.vsix extension/syntaxes/sysml.tmLanguage.json | grep composition
```

---

## Deploy / release

1. Commit the grammar edit on **`main`** (this fork's default branch).
2. Push with **`[release]`** in the commit message (or run the workflow manually). `.github/workflows/release.yml`
   triggers on `main` + `[release]`, builds, bundles `sysml.library`, packages `sireum-sysml-v2.vsix`,
   and publishes a GitHub release tagged **`0.9.1-<UTCdate>.<shorthash>`** with `make_latest`.
   The workflow does NOT commit anything back.
3. Adopt in kekinian: bump `org.sireum.version.vscodium.extension.syside` in `versions.properties`
   to that tag, then commit. `sireum setup vscode` installs it from
   `https://github.com/sireum/sysml-v2/releases/download/<tag>/sireum-sysml-v2.vsix`.

---

## Companion: `sireum.vscode-extension` (summary)

Full details in that repo's `CLAUDE.md` / `readme-dev.md`. Key parallels:
- Install: `npm install` (devDeps) + global `vsce`.
- Build/package: `bin/build.cmd package` → `sireum-vscode-extension.vsix`, version `4.<date>.<hash>`.
  `build.cmd` substitutes the `0.0.0` / `v0.0.0` placeholders in `package.json` — keep those
  placeholders committed, or local builds emit a stale version.
- Release: push to `master` with `[release]` (its CI) → tag `4.<date>.<hash>`.
- Adopt in kekinian: bump `org.sireum.version.vscodium.extension`.
- The Java LSP server it launches is `sysml-lsp-server.jar`, pinned by
  `org.sireum.version.sysml-v2-lsp` and built from `sireum/sysml-v2-lsp` (which pins the OMG
  Pilot Implementation branch via its own `versions.properties` `org.omg.sysml.version`).

## kekinian version pins (all three SysML pieces)

```
org.sireum.version.sysml-v2-lsp            # the Java LSP jar (sireum/sysml-v2-lsp release)
org.sireum.version.vscodium.extension      # sireum.vscode-extension release (4.<date>.<hash>)
org.sireum.version.vscodium.extension.syside  # this fork's release (0.9.1-<date>.<hash>)
```
`init` reads `$SIREUM_HOME/versions.properties` at runtime, so editing the file is enough — no
rebuild needed for `sireum setup vscode` to pick up new pins.
