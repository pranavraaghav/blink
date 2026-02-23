# Personal Build — Detailed Changes

This documents every divergence from upstream `blinksh/blink` on the `personal` branch.

---

## 1. Entitlements Removed

**File:** `Blink/Blink.entitlements`

Removed (unsupported by personal Apple Developer teams):
- `com.apple.developer.associated-domains` — Associated Domains (webcredentials:blink.sh)
- `com.apple.developer.user-fonts` — Font installation (app-usage)
- `com.apple.developer.web-browser` — Default Web Browser capability
- `com.apple.developer.icloud-container-identifiers` — iCloud containers
- `com.apple.developer.icloud-services` — CloudKit / CloudDocuments
- `com.apple.developer.ubiquity-container-identifiers` — Ubiquity containers
- `com.apple.developer.ubiquity-kvstore-identifier` — KV store sync
- `aps-environment` — Push notifications
- `keychain-access-groups` — Keychain sharing

**File:** `BlinkFileProviderExtension/BlinkFileProviderExtension.entitlements`
- Removed `keychain-access-groups`

**Impact:** No iCloud sync, no push notifications, no keychain sharing, no custom fonts, no Associated Domains. Core SSH/Mosh functionality is unaffected.

---

## 2. Paywall Bypass

**File:** `Blink/SceneDelegate.swift` (line ~107)

```swift
// Before:
let doShowPaywall = !entitlements.hasActiveSubscriptions()

// After:
let doShowPaywall = !entitlements.hasActiveSubscriptions() && !FeatureFlags.noSubscriptionNag
```

`FeatureFlags.noSubscriptionNag` is already `true` for `.developer` builds (defined in `FeatureFlags.swift:37`). The flag existed but wasn't wired into the paywall check.

Without this, the intro screen CTA button spins forever waiting for StoreKit prices that never load (no IAP configured for personal bundle ID).

---

## 3. Xcode Project Changes

**File:** `Blink.xcodeproj/project.pbxproj`

- **DEVELOPMENT_TEAM** changed from `A2H2CL32AG` to `693L542U78` (personal team) across all targets (main app, BlinkSnippetsTests, etc.)
- **StoreKit.framework** removed from Frameworks (IAP not needed; the framework reference and link phase entry were both removed)
- **Capabilities disabled** in project: Keychain (`enabled = 0`), Push (`enabled = 0`), iCloud (`enabled = 0`)
- **Deleted** `Blink.xcscmblueprint` (stale Xcode source control metadata)

---

## 4. Xcode 16 Debug Dylib Fix

**File:** `developer_setup.xcconfig` (gitignored — local only)

```
ENABLE_DEBUG_DYLIB = NO
```

**Why this is critical:** Xcode 16+ debug builds compile all app code into a separate dynamic library, leaving the main executable as a thin stub. Blink's `ios_system` framework resolves built-in commands (`config`, `help`, `clear`, `history`, etc.) via `dlsym(RTLD_MAIN_ONLY, "config_main")`. `RTLD_MAIN_ONLY` only searches the main executable — which is now empty.

`ENABLE_DEBUG_DYLIB = NO` forces Xcode to put all code in the main executable, restoring `dlsym` lookups.

**Affected commands (all resolved via dlsym in ios_system):**
`config`, `help`, `clear`, `history`, `showkey`, `open`, `bench`, `device-info`, `build`, `whatsnew`, `openurl`, `xcall`, `geo`, `say`

**Unaffected commands (use native Swift/ObjC dispatch):**
`ssh`, `mosh` — these are instantiated directly as `BlinkSSH`/`BlinkMosh` classes from `MCPSession.m`, bypassing `dlsym` entirely.

---

## 5. Developer Setup (xcconfig)

**File:** `developer_setup.xcconfig` (gitignored)

Full contents for reference:
```
TEAM_ID = 693L542U78
BUNDLE_ID = com.pranavraaghav.blinkshell
GROUP_ID  = com.pranavraaghav.blink
CLOUD_ID = com.pranavraaghav.blinkshell
KEYCHAIN_ID1 = sh.blink

SWIFT_ACTIVE_COMPILATION_CONDITIONS[config=Debug]   = BLINK_PUBLISHING_OPTION_DEVELOPER
SWIFT_ACTIVE_COMPILATION_CONDITIONS[config=Release] = BLINK_PUBLISHING_OPTION_TESTFLIGHT

BLINK_MIGRATION_SCHEME = blinkv15
WHATS_NEW_URL = http:/$()/localhost/whats-new
CONVERSION_OPPORTUNITY_URL = http:/$()/localhost/conversionOpportunity
BLINK_APP_FONT = JetBrains Mono

ENABLE_DEBUG_DYLIB = NO
```

---

## Architecture Notes

### How ios_system resolves commands

`ios_system` is a **prebuilt xcframework** (downloaded from GitHub releases, not built from source). It maintains a command dictionary loaded from `Resources/blinkCommandsDictionary.plist`. Each entry maps a command name to:

1. Library name (`"MAIN"` for main binary, `"SELF"` for ios_system, or a dylib path)
2. Function name (e.g., `"config_main"`)

At execution time (`ios_system.m` ~line 2888):
```objc
if ([libraryName isEqualToString: @"MAIN"]) handle = RTLD_MAIN_ONLY;
function = dlsym(handle, functionName.UTF8String);
```

This is why `ENABLE_DEBUG_DYLIB = NO` is essential — it's the only way to make `RTLD_MAIN_ONLY` find these symbols without rebuilding `ios_system` from source.

### Prebuilt dependencies

All xcframeworks come from `xcfs/Package.swift` as binary targets downloaded from GitHub releases:
- `ios_system` v3.0.3 from `holzschu/ios_system`
- `mosh` v1.4.0 from `blinksh/mosh-apple`
- `LibSSH`, `OpenSSH`, `openssl`, `libssh2` from blinksh forks

---

## Potential Upstream Conflict Points

When rebasing onto upstream updates, watch for conflicts in:

| File | Why |
|------|-----|
| `project.pbxproj` | Team IDs, StoreKit removal, capability toggles |
| `Blink.entitlements` | Upstream may add new entitlements we need to remove |
| `SceneDelegate.swift` | Paywall logic may change |
| `FeatureFlags.swift` | Flag definitions may change |
