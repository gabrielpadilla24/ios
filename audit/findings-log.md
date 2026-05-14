# Runtime Findings Log — Live Capture

Findings observed during runtime setup of the audit fork (free Apple Developer account, iPhone 17 Pro simulator, iOS 26.3.1).
These will be triaged into final report sections (§8 Memory, §9 ECn/Robustness, §13 Optimizations) after profiling completes.

- Audit commit: 5218b7f21bce14622b823e41e844844aeedd593b
- Date: 2026-05-13
- Environment: macOS, Xcode 26.5, free-tier Apple Developer Team U67Q3MTU2M
- Custom bundle ID: com.gabrielpadilla24.bwaudit.passwordmanager

---

## F-RT-01: Force-unwrap on App Group container URL with no fallback

- **File:** BitwardenShared/Core/Platform/Services/Stores/DataStore.swift:71
- **Trigger:** App cold-start when com.apple.security.application-groups entitlement is missing.
- **Symptom:** Fatal error "Unexpectedly found nil while unwrapping an Optional value". Crash occurs in DataStore.init before any UI is rendered.
- **Root cause:** FileManager.default.containerURL(forSecurityApplicationGroupIdentifier:)! force-unwraps a nullable return.
- **Workaround applied (local-only):** Modified ServiceContainer.swift:498 to pass storeType: .memory, bypassing the persisted-store branch.
- **Category:** Memory anti-pattern (force-unwrap) + Robustness.

---

## F-RT-02: Same force-unwrap pattern in AuthenticatorBridgeDataStore

- **File:** AuthenticatorBridgeKit/AuthenticatorBridgeDataStore.swift:103
- **Trigger:** Same as F-RT-01.
- **Symptom:** Same crash, different stack frame.
- **Workaround applied (local-only):** Modified ServiceContainer.swift:1097 to pass storeType: .memory.
- **Category:** Memory anti-pattern + Robustness.
- **Pattern observation:** Same anti-pattern replicated across modules, indicating systemic architectural issue rather than isolated bug.

---

## F-RT-03: Duplicate font file registration warnings at startup

- **Component:** Resource loading, BitwardenResources.framework.
- **Symptom:** OS console emits multiple "GSFont: file already registered" warnings for the DMSans typeface family (Bold, Regular, SemiBold, Italic, BoldItalic, SemiBoldItalic) at every app launch.
- **Hypothesis:** The same font file is either bundled in multiple frameworks (main app + BitwardenShared + BitwardenResources), or the registration happens multiple times during framework initialization.
- **Impact:** Not blocking, but indicates redundant resource loading and minor app startup overhead.
- **Category:** Resource loading anti-pattern.

---

## F-RT-04: Debug assertionFailure conflates programmer errors with operational errors

- **File:** BitwardenKit/Core/Platform/Utilities/OSLogErrorReporter.swift:40-43
- **Symptom:** When the app encounters any "unexpected" runtime error (e.g., keychain access denied with osStatusError(-34018) due to missing entitlement), Debug builds crash with assertion failure.
- **Pattern issue:** Conflates two distinct error categories: (a) genuine programmer errors that warrant crashing, and (b) operational errors caused by external conditions (entitlements, server state, network) that should degrade gracefully.
- **Category:** Error handling philosophy + Robustness.

---

## F-RT-05: Firebase Crashlytics initialization at startup with no graceful fallback

- **File:** Bitwarden/Application/Services/ErrorReporter/CrashlyticsErrorReporter.swift, invoked from AppDelegate.application(_:didFinishLaunchingWithOptions:).
- **Trigger:** Any build with a bundle identifier that does not match the GOOGLE_APP_ID in GoogleService-Info.plist, OR any build where the DEBUG preprocessor macro is undefined.
- **Symptom:** Uncaught NSException from Firebase Core: "Configuration fails. It may be caused by an invalid GOOGLE_APP_ID in GoogleService-Info.plist or set in the customized options." App terminates before reaching the first screen.
- **Impact:** Blocks all forks, audits, tutorials, and dev environments that use a custom bundle identifier from running the app.
- **Workaround applied (local-only):** Modified ErrorReporterFactory.swift to always return OSLogErrorReporter() regardless of build configuration.
- **Suggested upstream fix:** Wrap FIRApp.configure() in @try/@catch (Objective-C exception handler) or behind a runtime flag that disables Crashlytics when the plist is invalid or missing.
- **Category:** Coupling to production environment + Robustness + Architecture.

---

## Setup observations (non-blocking)

- Build warnings: 75 SwiftLint warnings on main app target (mostly trailing-comma, unused-closure-parameter, superfluous-disable-command rules). These reflect ongoing lint-debt in the upstream codebase.
- Vault sync from server: with 150 items, sync to fully-rendered vault list took approximately 5-10 seconds on first cold-start over WiFi. Subsequent re-launches with in-memory store start from scratch (no persistence) but re-sync is similar.
- 12 favorites confirmed visible in vault list, matching seed CSV.
