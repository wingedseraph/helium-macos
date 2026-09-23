# Working on helium-macos

Guidelines for any agent (or human) making changes here. Read before touching anything.

## Prime directives

1. **No fragile solutions.** Do not pin `depot_tools`, hand-fix drifted upstream patches, add fuzz to `git apply`, or otherwise patch around upstream breakage. If upstream build infra drifts, sync to a newer upstream release that carries the fix — never a local workaround.
2. **We do not edit core.** We layer our changes on top like plugins. Core = everything from upstream: the `helium-chromium` submodule and the upstream parts of this repo.
3. **Upstream is the source of truth.** Track the latest imputnet releases and re-apply/refresh our thin layer on top.

## Repo layout

- **helium-macos** — our fork of `imputnet/helium-macos`. `upstream` = `imputnet/helium-macos`.
- **helium-chromium/** (submodule) — fork of `imputnet/helium`: the buildkit (`clone.py`, `depot_tools.patch`, `devutils/`) and the full Chromium patch set. **Treat as upstream core.** Its own `CLAUDE.md` forbids AI-authored edits — respect that.

## The custom layer — EXACTLY these four, nothing else

Our fork carries only the following on top of upstream. Do **NOT** add anything else. In particular these were removed and must not come back: `nested-tabs`, `mru-activation-on-close`, `clean-context-menu`, `improve-toast`, `thinner-infobar`, `disable-immersive-fullscreen`, `disable-crashpad-handler` (all superseded by upstream, obsolete, or unwanted).

1. **Restore passwords** — two coordinated changes: (a) DROP upstream `helium/hop/disable-password-manager.patch` from the series (it forces `kPasswordManagerEnabled=false` via mandatory HOP policy and kills the manager entirely); (b) our own `helium/ui/restore-password-page-action.patch`, which re-adds the `kActionShowPasswordsBubbleOrPage` action (removed by helium `remove-dead-toolbar-actions`) so `tab_features` re-creates the omnibox key icon. Both are required — the UI patch alone shows an icon with nothing behind it.
2. **tab height 36** — overlay that raises vertical tab height to 36. Do NOT edit upstream `vertical.patch`; express it as our own separate patch on top.
3. **`visual-tab-switcher`** — vendored from `imputnet/helium` PR #2539 (Ctrl+Tab thumbnail/list overlay).
4. **`address-bar-in-sidebar`** — vendored from `imputnet/helium` PR #2367 (address bar in the vertical sidebar). Needs the companion `download-bubble-state-accessor` fix noted in that PR (`HasOpenOrPendingBubble()` is undefined otherwise).

Rules for the layer:
- New, namespaced patch files appended to `series`. Never modify upstream patch files (`ungoogled-chromium/*`, `inox-patchset/*`, `brave/*`, `bromite/*`, `iridium-browser/*`, `debian/*`, `upstream-fixes/*`, upstream `helium/*`).
- Vendored PR patches (#2539, #2367) are pinned to a PR head commit; on each bump, **re-vendor from the PR** (or its newer head) rather than hand-fixing.
- Verify every patch applies at **zero offset** (`git apply --check`) before committing.

## Updating to a new upstream release

1. `git fetch upstream`; rebuild our layer on the new imputnet release (this repo + the `helium-chromium` fork). This pulls the refreshed `clone.py`, `depot_tools.patch`, and core patches, so infra breakage (e.g. depot_tools) resolves itself.
2. Refresh **only our layer** against the new source (`docs/building.md` → "Updating for a new Chromium release"): `he presetup` → `quilt push -a --refresh` → fix only our patches → `he unmerge`.
3. Re-vendor #2539 / #2367 from their current PR heads. Never resolve a conflict by editing a core patch.

## Build & environment

- Local macOS is too old for Xcode 26 → **no local builds.** Author/refresh patches locally; build in GitHub Actions (`build-dev.yml`). See `docs/building.md`.

## Zero-offset rule

- The `sanity` check fails on any patch applying with offset or fuzz. Our patches must apply at zero offset before committing.
