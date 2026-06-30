# TORICO fork — CLI npm distribution (`@torico` scope)

This repository (`torico-tokyo/screenpipe`) is a **fork** of upstream screenpipe.
Its sole shipping artifact is the **patched CLI binary**, distributed on npm under
the **`@torico`** scope. The fork never ships the desktop app.

If you are an AI agent or maintainer changing anything under `packages/cli/`,
`.github/workflows/release-cli.yml`, or the record/a11y code, read this first.

## Why the fork exists

The consuming app `torico-worktime-aggregation` (a Windows desktop work-hours
tracker) launches screenpipe for recording. Windows UIA / accessibility-tree
walking is expensive (upstream issue #4350, no off switch upstream). This fork
adds record flags to bound or disable that walk:

- `--disable-a11y-tree` — stop the UIA/a11y tree walk entirely (focus logging kept)
- `--tree-max-elements <N>` — cap the number of elements walked (default 10000)
- `--tree-max-depth <N>` — cap walk depth (default 0 = unlimited)

The app installs the patched binary via `npx -y @torico/screenpipe@<ver> record ...`.
Upstream's own `screenpipe` / `@screenpipe/cli-*` packages are **patch-free** and
must not be used for this.

## Published packages (npm)

| package | role |
|---|---|
| `@torico/screenpipe` | CLI wrapper (the `npx` entry; bin command stays `screenpipe`) |
| `@torico/cli-win32-x64` | Windows binary (the platform that matters) |
| `@torico/cli-linux-x64` | Linux binary (builds cleanly, published too) |
| `@torico/cli-darwin-arm64` / `-darwin-x64` | macOS binaries — **currently NOT published** (see below) |

- The npm scope **`@torico`** (npm org `torico`) is unrelated to the GitHub host
  org `torico-tokyo`. Don't confuse the two when search-replacing.
- The wrapper's `optionalDependencies` lists **all four** platform packages even
  though only some are published. Unpublished ones 404 on install but are
  `optional`, so they are skipped (install succeeds). Keeping all four means a
  later macOS publish needs no wrapper edit.
- First release: **`v0.4.25-torico.2`** (tracks upstream CLI `0.4.25`).

## How to cut a release

1. Ensure the npm `torico` org exists and the repo has the `NPM_TOKEN` secret
   (publish-scoped, e.g. a granular token for `@torico/*`). A brand-new npm
   package cannot bootstrap via OIDC/trusted-publishing — the first publish must
   use a token.
2. Tag `vX.Y.Z-torico.N` and push it. `release-cli.yml` builds, then `publish-npm`
   publishes whatever platform binaries built successfully + the wrapper.
   `--access public` is required for scoped public packages (already set).
3. The published npm `version` is derived from the tag (`v` stripped).

## CI design notes (release-cli.yml) — do not regress these

The fork diverges from upstream's release workflow in three deliberate ways:

1. **`publish-npm` only `needs: [build-windows]`**, and the Prepare / Publish /
   Verify steps guard each platform on the presence of its build artifact /
   binary. A platform whose build failed/was skipped is simply absent instead of
   blocking the publish of the platforms that did build. Never publish a platform
   package without its binary.
2. **Sentry debug-symbol upload steps are `continue-on-error: true`.** The fork
   has no `SENTRY_AUTH_TOKEN` (upstream targets the `mediar` Sentry org), so
   sentry-cli returns 401 and would otherwise exit 1 and kill the build. The
   upload is optional telemetry — never let it fail the release. (Same spirit as
   the SSL.com signing step that no-ops without `ESIGNER_*` secrets.)
3. **`e2e-test.yml` (desktop Tauri app E2E, ~3h of runner time/run) is disabled
   to `workflow_dispatch` only.** It's irrelevant to a CLI-only fork. Re-enable
   `push`/`pull_request` only if the fork ever starts touching the desktop app.

## macOS is currently broken (infra, not our code)

`build-macos` fails on the `macos-latest` runner (now macOS 26 / Xcode 26.5) with
`ld: library 'clang_rt.osx' not found` — a runner/toolchain regression, not a
source issue. That's why `@torico/cli-darwin-*` are unpublished. The resilient
publish design above means: once the runner is fixed (or the macOS jobs pin a
working Xcode via `sudo xcode-select -s /Applications/Xcode_<ver>.app`), a re-tag
publishes the macOS packages with no further changes. `test-cli-npm-e2e.yml`
builds on macOS too, so it is red for the same reason — it is **not** a required
check and does not block merges.

## When renaming the scope or platform packages

The change-list is bigger than it looks. Grep the scope string across **all** of
`packages/cli` first. Beyond the obvious surface (each `package.json` `name` +
`optionalDependencies`, `screenpipe/lib/index.js` `PLATFORMS` map, the
`release-cli.yml` Verify step) you MUST also sync:

- **npm-e2e harness**: `npm-e2e/lib/stage.ts` (`PACKAGES` / `WRAPPER`),
  `npm-e2e/tests/record.ts` (npx target), `npm-e2e/lib/registry.ts` (verdaccio
  scope), and the root `packages/cli/package.json`. The wrapper's
  `optionalDependencies` must resolve the staged platform package names or the
  e2e smoke test fails.
- **`screenpipe/scripts/postinstall.sh` `remove_quarantine`**: a scoped wrapper
  lives at `node_modules/@torico/screenpipe`, so `dirname(PKG_DIR)` is the scope
  dir, not `node_modules`. It resolves sibling platform packages from the scope
  dir using **unscoped leaf names** (`$SCOPE_DIR/cli-darwin-arm64`), which is
  scope-name-agnostic. (Runtime `getBinaryPath()` in `index.js` also strips the
  macOS quarantine xattr, so a bug here is non-fatal but still wrong.)

Upstream PostHog install telemetry (`cli_install_npm`) was stripped from
`postinstall.js` / `postinstall.sh` for internal distribution — do not re-add it.

## Hand-off

Confirmed package names / version / flag spec are reported back to the requesting
app at `torico-worktime-aggregation`:
`_issues/20260630-lightweight-mode-screenpipe-fork/ISSUE.md`.
