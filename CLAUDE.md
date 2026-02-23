# Blink Shell — Personal Fork

Forked from [blinksh/blink](https://github.com/blinksh/blink) for personal development builds.

## Branch Structure

- `raw` — tracks upstream `blinksh/blink:raw` (do not commit here)
- `personal` — our changes rebased on top of `raw`

## Key Divergences from Upstream

See [docs/personal-build.md](docs/personal-build.md) for full details.

**Summary of changes:**
1. **Entitlements stripped** — removed capabilities unsupported by personal dev teams
2. **Paywall bypassed** — `noSubscriptionNag` flag used in `SceneDelegate.swift`
3. **Xcode 16 debug dylib fix** — `ENABLE_DEBUG_DYLIB = NO` in xcconfig (critical for built-in commands)
4. **Signing reconfigured** — personal team ID across all targets, StoreKit removed

## Build Requirements

`developer_setup.xcconfig` is gitignored. It must contain:
```
TEAM_ID = 693L542U78
BUNDLE_ID = com.pranavraaghav.blinkshell
GROUP_ID = com.pranavraaghav.blink
CLOUD_ID = com.pranavraaghav.blinkshell
ENABLE_DEBUG_DYLIB = NO
```

## Updating from Upstream

```bash
git fetch upstream
git rebase upstream/raw
git push --force-with-lease
```

Conflicts are most likely in `project.pbxproj` and `Blink.entitlements`.

## Conventions

- Keep `personal` branch as a minimal diff from upstream — don't accumulate unnecessary changes
- When adding features, update [docs/personal-build.md](docs/personal-build.md) with what changed and why
