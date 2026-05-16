# Atom — 2026 Revival Orientation

**Author:** Pax (frontend lane)
**Date:** 2026-05-16
**Repo:** `keldessouky/atom`, fork of archived `atom/atom`
**HEAD:** `1c3bd35` — Dec 2022, final pre-sunset upstream merge
**Status:** read-only orientation pass. No builds attempted, no installs run.

---

## 1. Repository structure

| Path | What it is |
|---|---|
| `src/` | Atom core — 122 files of editor runtime. Main process under `src/main-process/`; renderer / API surfaces at root. Still mixed JS + CoffeeScript (~12 `.coffee` files, e.g. `context-menu-manager.coffee`, `layer-decoration.coffee`, `keymap-extensions.coffee`). |
| `packages/` | 28 bundled packages shipped with Atom. Mostly themes (`one-dark-*`, `atom-light-*`, etc.) and a handful of features (`autoflow`, `dalek`, `welcome`, `dev-live-reload`, `update-package-dependencies`). **Most "real" packages are not here** — they're pulled as `git+https://github.com/atom/...#sha` deps in `package.json` (tabs, tree-view, find-and-replace, github, settings-view, …). |
| `script/` | Build & test orchestration. Entrypoints `script/build`, `script/bootstrap`, `script/test`, `script/lint`. Heavy lifting in `script/lib/` (transpile, package, sign, notarize, snapshot). Has its own `package.json` with build-time deps (`electron-packager`, `electron-link`, `electron-mksnapshot`, `legal-eagle`, lint stack). |
| `script/vsts/` | **Azure Pipelines** YAML (`release-branch-build.yml`, `pull-requests.yml`, `nightly-release.yml`, `lint.yml`) + per-OS templates in `script/vsts/platforms/{macos,linux,windows}.yml`. This is the *only* CI infra — there's no GitHub Actions build. |
| `apm/` | Wrapper that installs `atom-package-manager@2.6.2` as a nested dep so the build can resolve `apm`. Single-purpose. |
| `vendor/` | Tiny — just `jasmine.js` and `jasmine-jquery.js`. |
| `spec/` | 86 Jasmine test files for core. |
| `benchmarks/` | Perf scripts run via `script/test --core-benchmark`. |
| `resources/mac/` | `atom-Info.plist`, `helper-Info.plist`, `entitlements.plist`, `file.icns`. **No** `electron-builder` config — packaging is custom. |
| `dot-atom/`, `keymaps/`, `menus/`, `static/`, `exports/` | Default user dotfiles, OS keymaps, menu definitions, bundled static assets (less / images), and `exports/` (the public API surface mapped into `require('atom')`). |
| `docs/` | Mostly RFCs and contribution guides. `docs/build-instructions/` mirrors what used to live on flight-manual.atom.io (now offline). |
| `atom.sh` | Launcher shim for macOS/Linux. |
| `Dockerfile` | Linux build env (for the Azure Pipelines Linux job). |

## 2. `package.json` essentials

- **`name`:** `atom`  **`version`:** `1.65.0-dev`
- **`electronVersion`:** `11.5.0` — *6 majors behind current Electron 38.x as of 2026*
- **`engines.node`:** *not declared.* The only Node check is `script/lib/verify-machine-requirements.js`, which gates on `majorVersion >= 11 || (10 && minor >= 12)`. **A v25+ Node will pass this check numerically but break native module compilation under node-gyp 5.x.**
- **`devDependencies`:** *none declared.* Everything (chai, sinon, mocha, eslint configs, postcss tooling) is in `dependencies` because Atom is bundled, not published.
- **`scripts`:** only `preinstall: node -e 'process.exit(0)'` (no-op, blocks npm's default install lifecycle) and `test: node script/test`. **There is no `npm start` and no `npm run build`** — `script/build` is invoked directly.
- **Dependency shape:** ~75% of bundled packages are `git+https://github.com/atom/<pkg>.git#<sha>` refs. Final pre-archive commit (`Remove dependancies`, sic) flipped many deps to git URLs deliberately — upstream anticipated registry packages being yanked. **`tree-sitter` already points at a community fork** (`DeeDeeG/node-tree-sitter`), a pre-existing patch.

### `script/package.json` build-deps (the ones that matter for revival)

- `electron-packager ^15.1.0` (renamed to `@electron/packager` in 2022; current major is 18)
- `electron-chromedriver ^11.0.0` (matches Electron 11; needs to track Electron upgrade)
- `electron-mksnapshot ^11.0.1`, `electron-link ^0.6.0` (snapshot pipeline — Atom-specific)
- `eslint ^5.16.0`, `prettier ^1.17.0`, `stylelint ^9.0.0` (lint stack from 2019-era)
- `coffeelint 1.15.7`, `donna 1.0.16`, `joanna 0.0.10`, `tello 1.2.0` (Atom's own doc / lint tooling)
- `npm ^6.14.16` pinned (the build invokes a vendored npm to keep node-gyp 5.x behavior)
- `postinstall: node ./redownload-electron-bins.js`

## 3. The build dance

### Bootstrap (`script/bootstrap`)
1. `verifyMachineRequirements` — checks Node ≥ 10.12, Python 2.6/2.7/≥3.5 on PATH.
2. `dependenciesFingerprint.isOutdated()` → maybe `cleanDependencies()`.
3. `installScriptDependencies` (npm install inside `script/`).
4. `installApm` (npm install inside `apm/`).
5. `runApmInstall` at repo root — uses the vendored apm to install all `dependencies` from `package.json`. **This is where native modules get rebuilt against Electron 11 ABI** (via apm's electron-rebuild path).

### Build (`script/build`)
1. `checkChromedriverVersion` → `cleanOutputDirectory` → `copyAssets`.
2. Transpile pass: `transpilePackagesWithCustomTranspilerPaths` → Babel → CoffeeScript → CSON → PEG.js.
3. `generateModuleCache` → `prebuildLessCache` → `generateMetadata` → `generateAPIDocs`.
4. `dumpSymbols` (breakpad — for crash reports).
5. `packageApplication` (electron-packager 15 produces `out/Atom.app`).
6. `generateStartupSnapshot` (V8 snapshot via `electron-mksnapshot` + `electron-link`).
7. Platform branch:
   - **macOS:** if `--code-sign`, run `codeSignOnMac` then `notarizeOnMac`. Otherwise `--test-sign` for ad-hoc, otherwise skip.
   - Windows: `codeSignOnWindows`, optional Squirrel installer.
   - Linux: `.deb` / `.rpm` packagers.
8. Optional `compressArtifacts`, optional `installApplication`.

### What a Mac binary build actually needs

**Tooling:**
- Node 10.12–14ish (the safe corridor — anything ≥15 risks node-gyp 5.x incompatibilities; **v25 is well outside this corridor**)
- Python 2.7 or 3.5+ (3.11+ may break some legacy packages)
- Xcode CLT for headers / linker
- `apm` (the vendored install handles this)

**Code signing (`script/lib/code-sign-on-mac.js`):**
- Apple Developer ID Application cert (`.p12`)
- Identity is **hardcoded** as `Developer ID Application: GitHub` (line 137 of `code-sign-on-mac.js`) — *will need to change this string* to point at Karim's identity, e.g. `Developer ID Application: Karim Eldessouky (TEAMID)`.
- Env vars expected: `ATOM_MAC_CODE_SIGNING_CERT_DOWNLOAD_URL` *or* `ATOM_MAC_CODE_SIGNING_CERT_PATH`, plus `ATOM_MAC_CODE_SIGNING_CERT_PASSWORD`, `ATOM_MAC_CODE_SIGNING_KEYCHAIN`, `ATOM_MAC_CODE_SIGNING_KEYCHAIN_PASSWORD`.
- Uses `electron-osx-sign` 0.5.0 with `hardenedRuntime: true`.

**Entitlements (`resources/mac/entitlements.plist`):**
```
com.apple.security.cs.allow-unsigned-executable-memory  true
com.apple.security.cs.disable-library-validation        true
```
Minimal — would need review against current Apple requirements (e.g. JIT entitlement for V8, allow-jit, etc.).

**Notarization (`script/lib/notarize-on-mac.js`):**
- Uses `electron-notarize` 1.0.0 with `AC_USER` / `AC_PASSWORD` env vars — **this is the legacy `altool` flow**.
- Apple deprecated `altool` notarization on **November 1, 2023**. Only `notarytool` is accepted now.
- `@electron/notarize` (renamed from `electron-notarize`) supports `notarytool` from v1.2.2+ and is required.

## 4. CI state

- `.github/` contains only **issue-triage automation**: `lock.yml`, `move.yml`, `no-response.yml`, `stale.yml`. These are settings for third-party GitHub Apps (probot-style), not Actions workflows.
- **There is no `.github/workflows/` directory at all.** The repo has never had GitHub Actions CI.
- All build/test/release CI lived on **Azure DevOps** at `dev.azure.com/github/Atom`. The README badge points at a definition (id=32) that's now inaccessible to outsiders.
- `script/vsts/release-branch-build.yml` shows the prod pipeline assumed: secrets for `GITHUB_TOKEN`, `ATOM_RELEASES_S3_KEY/SECRET/BUCKET`, `ATOM_RELEASES_AZURE_CONN_STRING`, `PACKAGE_CLOUD_API_KEY`, plus mac/windows code-signing creds. None of this is reusable as-is.

**Implication:** any revival needs **CI built from scratch**. GitHub Actions is the obvious target; the `script/vsts/platforms/*.yml` files are useful as a reference for the *steps*, but the YAML schema, agent images, and artifact upload logic all have to be rewritten.

## 5. Official build docs

- `README.md`'s "Building" section links to `flight-manual.atom.io` — **the site is offline** since the sunset.
- `docs/build-instructions/` *does* exist in the repo (likely a mirror of the flight manual's build pages). Worth reading before any actual build attempt — not loaded into this orientation pass to keep scope tight.
- `CONTRIBUTING.md` (48KB) is contribution-process guidance, not build steps; mostly stale references to atom/atom workflow.

## 6. Sanity check results

- **Local Node:** `v25.9.0` at `/opt/homebrew/bin/node`. **Well outside the build's working corridor** (Node 10–14 era). The version check in `verify-machine-requirements.js` will let it through, but native-module compilation under node-gyp 5.x will explode on a modern Node.
- **`npm install` not attempted** — per instructions. The expected failure mode: native module postinstalls (`@atom/nsfw`, `@atom/watcher`, `superstring`, `git-utils`, `pathwatcher`, `fs-admin`, `nslog`, etc.) try to build against Electron 11's V8 ABI using node-gyp 5.x, hitting C++17 issues with current Xcode and/or Python 3.12+ removing `distutils`.
- **No git submodules.** `git config --file .gitmodules` is absent; nothing to init.
- **Origin = `keldessouky/atom`**, **upstream = `atom/atom`** (the archived repo). Worktree branch is `claude/zealous-ardinghelli-fe460a` based on `master` at `1c3bd35`.

## 7. Modernization gap inventory

Ordered by how hard each one bites.

| # | Gap | Severity | Notes |
|---|---|---|---|
| 1 | **Electron 11 → 38+** (six majors, ~5 years) | 🔴 Foundational | Bundled Node 12 → 22, V8 ABI changes, removal of remote module, contextIsolation defaults flipped, deprecation of `nodeIntegration: true`. Atom uses all of the above. Cascading changes across `electron-packager` (→ `@electron/packager` 18), `electron-chromedriver` 11 → 38, `electron-mksnapshot` 11 → 38, `electron-link` 0.6 → latest. |
| 2 | **Native modules broken** | 🔴 Blocks build | `@atom/nsfw`, `@atom/watcher`, `superstring`, `git-utils`, `pathwatcher`, `fs-admin`, `nslog`, `tree-sitter` all need rebuilds (or forks) for current Node + Electron + macOS SDK. Several were last published 2020-2021. |
| 3 | **Notarization (`altool` → `notarytool`)** | 🔴 Blocks distribution on macOS | `electron-notarize` 1.0.0 with `AC_USER`/`AC_PASSWORD` no longer works — Apple turned off `altool` notarization Nov 2023. Need `@electron/notarize` with `notarytool` + an app-specific password or App Store Connect API key. |
| 4 | **Code-signing identity hardcoded as "GitHub"** | 🟡 One-line fix | `script/lib/code-sign-on-mac.js:137`. Plus entitlements may need additions (e.g. `com.apple.security.cs.allow-jit`, `allow-dyld-environment-variables` depending on Electron version). |
| 5 | **node-gyp 5 → 11** | 🟡 Painful but mechanical | Driven by upgrading the vendored npm (currently 6.14) to a current one. Affects how Python is discovered and how C++ standards are set. |
| 6 | **Python 2.7 assumption** | 🟡 macOS Sequoia / Tahoe drops it | `verify-machine-requirements.js` accepts 2.6 / 2.7 / ≥3.5. Modern macOS ships 3.x only. Should be pinned to ≥3.10. |
| 7 | **All CI rebuilt from zero** | 🟡 Big but contained | Azure Pipelines → GitHub Actions; signing secrets → repo secrets; S3/Azure release upload → new release infra (or just GitHub Releases). |
| 8 | **CoffeeScript still in core** | 🟢 Optional | 12 `.coffee` files in `src/`. Transpiled at build time via `coffee-script/register`. Can stay, but every contributor learning the codebase will trip over it. |
| 9 | **Babel 5.8.38** | 🟢 Cosmetic | Bundled as a *runtime* dep for `babel.js` shim and `language-typescript` transpile path. Replace with current `@babel/core` if any of that code is touched. |
| 10 | **`apm` registry** | 🟢 Aspirational | The `apm` CLI talks to `atom.io/api`, which is gone. Without it, package install/discovery from Settings View dies. Either stub it, point at a new registry, or just live with manual `git clone` into `~/.atom/packages`. |

## 8. Candidate first sprints

Five plausible directions, each scoped to roughly one focused week. Pick one — don't fan out.

### Sprint A — "Walking skeleton on Electron 11"
**Goal:** Get a locally-built, ad-hoc-signed `Atom.app` running on current macOS. No notarization, no distribution.
**Steps:**
1. Set up a pinned Node 14 + Python 3.11 toolchain via asdf/nvm in a `.tool-versions` file.
2. `script/bootstrap`. Triage native module failures one at a time. Likely patches: fork `superstring` / `nsfw` / `git-utils` for current Xcode; freeze on Electron 11.5 ABI.
3. `script/build` (no `--code-sign`). Use `--test-sign` to make Gatekeeper tolerable.
4. Document every workaround in `docs/build-2026.md`.
**Why first:** establishes whether the codebase can boot at all before committing to an Electron upgrade. Smallest blast radius.
**Risk:** macOS Sequoia + Xcode 16 may simply refuse to build Electron 11's V8 — could hit a wall.

### Sprint B — "Electron upgrade spike (11 → LTS)"
**Goal:** Single-commit-per-step bump to current Electron LTS (or the latest Atom-compatible-feeling version).
**Steps:**
1. Bump `electronVersion`, `electron-packager` → `@electron/packager`, `electron-chromedriver`, `electron-mksnapshot` together.
2. Replace `electron-osx-sign` with `@electron/osx-sign`, `electron-notarize` with `@electron/notarize` using `notarytool`.
3. Fix `nodeIntegration` / `contextIsolation` / `remote` regressions in `src/main-process/` and `src/atom-environment.js`.
4. Ignore packages — expect breakage and disable them for now.
**Why:** every other piece of work compounds on this. Doing it first means the rest of the codebase only gets touched once.
**Risk:** Atom's `remote` and renderer-IPC patterns are pervasive. The blast radius is the whole UI layer.

### Sprint C — "Macing it stick — signing + notarization for Karim's identity"
**Goal:** End-to-end Developer ID signed + notarized DMG/zip, downloadable, opens without Gatekeeper warning on a clean Mac.
**Steps:**
1. Generalize `script/lib/code-sign-on-mac.js` to read identity from env (`ATOM_MAC_CODE_SIGNING_IDENTITY`).
2. Swap notarize.js to `@electron/notarize` + notarytool with `APPLE_ID` / `APPLE_APP_SPECIFIC_PASSWORD` / `APPLE_TEAM_ID` (or API key).
3. Audit entitlements.plist for missing keys (likely needs `com.apple.security.cs.allow-jit` for V8 on current Electron).
4. Build a one-shot script that takes a fresh checkout and produces a signed `.dmg` end-to-end.
**Why:** answers the practical "can Karim actually ship this" question without first solving every other problem.
**Risk:** depends on Sprint A or B succeeding; signing a broken build is pointless.

### Sprint D — "GitHub Actions CI for one platform"
**Goal:** `.github/workflows/build-macos.yml` that runs `script/bootstrap && script/build` on a `macos-latest` runner and uploads the artifact.
**Steps:**
1. Translate `script/vsts/platforms/macos.yml` step-by-step to GHA syntax.
2. Move Azure secret names → GHA secret names; document required secrets in `docs/ci-secrets.md`.
3. Skip code-signing initially (PR builds shouldn't sign anyway). Add a separate `release.yml` later that runs on tag push with signing secrets.
4. Use the GHA artifact upload, not S3.
**Why:** unblocks every future change. Right now any modification needs a manual local build to verify.
**Risk:** macOS runners are slow and expensive; an Atom build is ~15-30 min historically. Keep an eye on minute usage.

### Sprint E — "Triage the package dependency graph"
**Goal:** Audit which of the ~50 `git+https://github.com/atom/...` deps still resolve and which point at SHAs in repos that may have been archived without preserving refs.
**Steps:**
1. Script that walks `package.json` deps, attempts a shallow clone of each `git+https` ref, records pass/fail.
2. For each failure, decide: fork into `keldessouky/`, drop the package, or replace with a community successor (e.g. Pulsar's forks of these same packages).
3. Produce a `docs/dependency-status-2026.md` table.
**Why:** the supply-chain risk is invisible until you try to install. Better to know up front whether the deps are even reachable.
**Risk:** mostly research-shaped — won't produce running code by itself. Best paired with Sprint A.

### Recommendation

**A → C → D**, in that order, if the goal is "Karim has a working personal Atom build he can run nightly." Sprint B (the Electron upgrade) is the right *long-term* investment but it's a multi-week thing, not one week, and it's dramatically easier to do *after* you've reproduced the existing build at all. Sprint E is useful background work to run in parallel.

If the goal is different — say, "fork as a public successor project" — then E becomes load-bearing earlier and B becomes unavoidable.

## 9. Open questions for Karim

1. **What's the actual goal?** Personal nightly build, public Pulsar-style fork, or a learning exercise on the codebase itself? Different goals → different sprint orderings.
2. **Apple Developer membership?** Required for signing + notarization. (Pulsar — the community Atom successor — sidesteps this by shipping unsigned and asking users to `xattr -d com.apple.quarantine`.)
3. **Branch hygiene:** the origin has ~25 release branches mirrored from upstream (1.5, 1.22, 1.33, …, 1.62). Keep them around as history, or prune to just `master`?
4. **Awareness of Pulsar:** `pulsar-edit/pulsar` is a community fork that's been actively shipping since early 2023. Worth deciding whether to converge with their patches (most of the modernization work has been done there) or do this from scratch as a learning project.
