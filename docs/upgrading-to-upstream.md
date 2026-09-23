# Upgrading the fork to a new upstream Helium release

Durable procedure for bumping this fork up to the current `imputnet/helium` release
and carrying only our thin custom layer on top. Policy: ride upstream, never hand-fix
core (see `CLAUDE.md`). Worked example: **149.0.7827.155 → 153.0.8010.52**.

Run everything on a networked machine. The `helium-chromium` submodule is upstream core —
its changes are authored by you personally (its own `CLAUDE.md` forbids AI edits).

## Why we bump instead of patching

Root cause (confirmed): our 149 `clone.py` fetches depot_tools at **HEAD**, so once
depot_tools rolls it no longer matches `depot_tools.patch`. Upstream already fixed this — the
153 `clone.py` pins depot_tools to the commit recorded in Chromium's own `DEPS` file:

```python
dt_commit = re.search(r"depot_tools\.git'...'@'...'([^']+)',",
                      (args.output / 'DEPS').read_text()).group(1)
```

A DEPS-pinned depot_tools always matches the patch, so we inherit the fix for free by syncing
upstream's `clone.py` during the bump. Do not re-pin or hand-edit depot_tools by hand.

## Our custom layer (the ONLY thing we carry)

Chromium (submodule `helium-chromium`, in `patches/`) — genuinely ours (absent from upstream):
- `helium/core/nested-tabs.patch`
- `helium/core/clean-context-menu.patch` *(check: may be redundant with upstream #1951)*
- `helium/core/mru-activation-on-close.patch`
- `helium/ui/improve-toast.patch` *(check: may be redundant with upstream #1806)*
- `helium/ui/thinner-infobar.patch`
- `helium/ui/restore-password-page-action.patch` *(password manager — the new-file part)*
- Experiments (keep OUT of the default series): `zen-mode.patch`, `zen-mode-wiring.patch`,
  `zen-caption-buttons.patch`, `anchor-tabs/`

**NOT ours — do not carry (verified against upstream/main 153):**
- `helium/core/add-zen-importer.patch` is an **upstream** patch — keep upstream's version.
- `helium/ui/layout/vertical.patch` height tweak (30->36) was a **core edit**. To keep height 36,
  add a separate overlay patch during refresh; do not edit `vertical.patch`.
- The password-manager **hunk-removals** from `brave/custom-importer.patch`,
  `helium/ui/remove-dead-profile-actions.patch`, `inox-patchset/modify-default-prefs.patch`
  are core edits. Re-verify on 153 whether PM needs them; if so, re-express as an overlay.

macOS (this repo, `patches/helium/macos/`):
- `open-shortcuts-settings-action.patch`
- `applescript-tab-select-command.patch`

**Drop** every past hand-fix to a core patch (they come corrected from upstream now), e.g.
"update fix-building-without-safebrowsing offset", "create parent dirs when unpacking".
Verify each against `upstream/main` before discarding.

## 1. Submodule: rebuild our layer on top of upstream 153

Cleaner than rebasing the interleaved history — start from upstream and re-add only our files:

```sh
cd helium-chromium
git remote add upstream https://github.com/imputnet/helium.git   # once
git fetch upstream
git show upstream/main:chromium_version.txt        # expect 153.0.8010.52

git checkout -b nested-tabs-153 upstream/main
git checkout nested-tabs -- \
  patches/helium/core/nested-tabs.patch \
  patches/helium/core/clean-context-menu.patch \
  patches/helium/core/mru-activation-on-close.patch \
  patches/helium/ui/improve-toast.patch \
  patches/helium/ui/thinner-infobar.patch \
  patches/helium/core/add-zen-importer.patch
# append these to the END of patches/series (after upstream patches)
# keep experiments in a separate series (e.g. series.experimental), not the default
```

This also brings upstream's current `clone.py` + `depot_tools.patch` — the build blocker.

## 2. Refresh OUR patches against Chromium 153

Follow `docs/building.md` -> "Updating for a new Chromium release":

```sh
source dev.sh
he presetup
cd build/src
quilt push -a --refresh      # refresh as applied; fix breaks ONLY in our patches
he unmerge
```

Rule: never resolve a conflict by editing an upstream/core patch. If a core patch
conflicts, the base is wrong (re-sync) — not the patch.

## 3. Zero-offset check

The `sanity` CI job fails on ANY offset/fuzz. Re-run `quilt refresh` until our patches
apply cleanly, then commit + push the submodule branch.

## 4. helium-macos side

```sh
cd ..                                   # helium-macos root
git fetch upstream                      # imputnet/helium-macos
# take upstream's macOS packaging + core macOS patches for 153;
# refresh ONLY our macOS patches (open-shortcuts-settings-action, applescript-tab-select-command)
# bump the submodule pointer to the new nested-tabs-153 commit
```

The Chromium version lives in the submodule's `chromium_version.txt`; this repo only
carries `revision.txt` (packaging revision) — bump it as `imputnet/helium-macos` does.

## 5. Push + build

Push the submodule branch, bump + commit the gitlink here, push this repo's branch, then:

```sh
gh workflow run build-dev.yml --ref <your-branch>
```

Iterate only on our layer if anything still drifts.

## 6. Experiments stay quarantined

`zen-*` / `anchor-tabs` live in a separate series and are enabled only when explicitly
testing, so a broken experiment can never block a release build.
