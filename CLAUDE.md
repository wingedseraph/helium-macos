# Working on helium-macos

Guidelines for any agent (or human) making changes here. Read before touching anything.

## Prime directives

1. **No fragile solutions.** Do not pin \`depot_tools\`, hand-fix drifted upstream patches, add fuzz to \`git apply\`, or otherwise patch around upstream breakage. If upstream build infra drifts, the fix is to sync to a newer upstream release that already carries the fix — never a local workaround.
2. **We do not edit core.** We layer our changes on top like plugins. Core = everything from upstream: the \`helium-chromium\` submodule and the upstream parts of this repo.
3. **Upstream is the source of truth.** Track the latest imputnet releases and re-apply/refresh our thin layer on top.

## Repo layout

- **helium-macos** — our fork of \`imputnet/helium-macos\` (macOS packaging + our macOS patches). This is where we work. \`upstream\` = \`imputnet/helium-macos\`.
- **helium-chromium/** (submodule) — fork of \`imputnet/helium\`: the buildkit (\`clone.py\`, \`depot_tools.patch\`, \`devutils/\`) and the full Chromium patch set. **Treat as upstream core.** Its own \`CLAUDE.md\` forbids AI-authored edits — respect that. Any change there is authored by you personally, not the agent.

## What "layer like a plugin" means

- Our additions are **new, namespaced patch files** appended to the series. We never modify upstream patch files: \`ungoogled-chromium/*\`, \`inox-patchset/*\`, \`brave/*\`, \`bromite/*\`, \`iridium-browser/*\`, \`debian/*\`, \`upstream-fixes/*\`, and upstream \`helium/*\`.
- Our macOS layer lives in \`patches/helium/macos/\` (e.g. \`open-shortcuts-settings-action\`, \`applescript-tab-select-command\`). Add ours; do not rewrite upstream ones.
- **Experimental / want-to-try patches** (e.g. \`zen-mode*\`, \`anchor-tabs\`) stay **out of the default build series** so a broken experiment can never block a build. Enable them only when explicitly testing.

## Updating to a new upstream release (the durable path)

1. \`git fetch upstream\` and merge/rebase onto the new imputnet release — in this repo, and (by you) in the \`helium-chromium\` fork. This pulls the refreshed \`clone.py\`, \`depot_tools.patch\`, and core patches matching the new Chromium, so infra breakage (e.g. depot_tools) resolves itself.
2. Refresh **only our patches** against the new source via the quilt flow in \`docs/building.md\` ("Updating for a new Chromium release"): \`he presetup\` → \`quilt push -a --refresh\` → fix only our layered patches → \`he unmerge\`.
3. Never resolve a refresh conflict by editing a core patch. Adjust only our layer.

## Build & environment

- Local macOS is too old for Xcode 26 → **no local builds.** Author and refresh patches locally; build in GitHub Actions (\`build-dev.yml\`). See \`docs/building.md\`.

## Zero-offset rule

- The \`sanity\` check fails on any patch applying with offset or fuzz. After a refresh, our patches must apply at **zero offset** — re-run \`quilt refresh\` / \`he unmerge\` until clean before committing.
