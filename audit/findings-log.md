# Runtime Findings Log — Live Capture

Findings observed during runtime setup of the audit fork (free Apple Developer account, iPhone 17 Pro simulator, iOS 26.3.1).
These will be triaged into final report sections (§8 Memory, §9 ECn/Robustness, §13 Optimizations) after profiling completes.

- Audit commit: 5218b7f21bce14622b823e41e844844aeedd593b
- Date: 2026-05-13 (initial setup) / 2026-05-14 (Approach 2 persistence-fidelity refinement)
- Environment: macOS, Xcode 26.5, free-tier Apple Developer Team U67Q3MTU2M
- Custom bundle ID: com.gabrielpadilla24.bwaudit.passwordmanager

---

## F-RT-01: Force-unwrap on App Group container URL with no fallback

- **File:** BitwardenShared/Core/Platform/Services/Stores/DataStore.swift:71
- **Trigger:** App cold-start when com.apple.security.application-groups entitlement is missing.
- **Symptom:** Fatal error "Unexpectedly found nil while unwrapping an Optional value". Crash occurs in DataStore.init before any UI is rendered.
- **Root cause:** FileManager.default.containerURL(forSecurityApplicationGroupIdentifier:)! force-unwraps a nullable return.
- **Workaround applied (local-only):** Replaced the App Group container URL lookup with FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first! in the .persisted branch. Real disk-backed CoreData is preserved; only the container location changes from shared App Group to per-process sandbox.
- **Suggested upstream fix:** Replace force-unwrap with graceful fallback to per-process applicationSupportDirectory, or fail with a user-facing diagnostic that names the missing entitlement.
- **Category:** Memory anti-pattern (force-unwrap) + Robustness.

---

## F-RT-02: Same force-unwrap pattern in AuthenticatorBridgeDataStore

- **File:** AuthenticatorBridgeKit/AuthenticatorBridgeDataStore.swift:103
- **Trigger:** Same as F-RT-01.
- **Symptom:** Same crash, different stack frame.
- **Workaround applied (local-only):** Same approach as F-RT-01: replaced the App Group container URL lookup with FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first!. The groupIdentifier parameter is no longer consulted in the audit build, but its signature is preserved to minimize the diff from upstream. Also modified AuthenticatorShared/Core/Platform/Services/Stores/DataStore.swift (~line 67) analogously for the Authenticator's DataStore.
- **Pattern observation:** Same anti-pattern replicated across modules, indicating systemic architectural assumption that App Groups are always available, rather than isolated bug. A third occurrence in BitwardenKit/Core/Platform/Extensions/FileManager+Extensions.swift:9 (flightRecorderLogURL helper) uses optional chaining rather than force-unwrap; while not crash-inducing, it returns nil silently and disables the flight recorder feature without notifying the user.
- **Suggested upstream fix:** Centralize App Group container access through a helper that gracefully falls back to per-process directories. A linting rule flagging containerURL(forSecurityApplicationGroupIdentifier:)! would prevent recurrence.
- **Category:** Memory anti-pattern + Robustness.

---

## F-RT-03: Duplicate font file registration warnings at startup

- **Component:** Resource loading, BitwardenResources.framework.
- **Symptom:** OS console emits multiple "GSFont: file already registered" warnings for the DMSans typeface family (Bold, Regular, SemiBold, Italic, BoldItalic, SemiBoldItalic) at every app launch.
- **Hypothesis:** The same font file is either bundled in multiple frameworks (main app + BitwardenShared + BitwardenResources), or the registration happens multiple times during framework initialization.
- **Impact:** Not blocking, but indicates redundant resource loading and minor app startup overhead.
- **Suggested upstream fix:** Centralize font registration in one framework, or guard the registration call with a per-process flag.
- **Category:** Resource loading anti-pattern.

---

## F-RT-04: Debug assertionFailure conflates programmer errors with operational errors

- **File:** BitwardenKit/Core/Platform/Utilities/OSLogErrorReporter.swift:40-43
- **Symptom:** When the app encounters any "unexpected" runtime error (e.g., keychain access denied with osStatusError(-34018) due to missing entitlement, see F-RT-07), Debug builds crash with assertion failure.
- **Pattern issue:** Conflates two distinct error categories: (a) genuine programmer errors that warrant crashing, and (b) operational errors caused by external conditions (entitlements, server state, network) that should degrade gracefully.
- **Suggested upstream fix:** Introduce an ErrorClassifier that distinguishes programmer errors (assert in Debug) from operational errors (log only, never assert). At minimum, exempt keychain and entitlement errors from the assertion path.
- **Category:** Error handling philosophy + Robustness.

---

## F-RT-05: Firebase Crashlytics initialization at startup with no graceful fallback

- **File:** Bitwarden/Application/Services/ErrorReporter/CrashlyticsErrorReporter.swift, invoked from AppDelegate via ErrorReporterFactory.makeDefaultErrorReporter().
- **Trigger:** Build with bundle identifier not matching GOOGLE_APP_ID in GoogleService-Info.plist, or where the DEBUG macro is undefined.
- **Symptom:** Uncaught NSException from Firebase Core: "Configuration fails. It may be caused by an invalid GOOGLE_APP_ID in GoogleService-Info.plist or set in the customized options." App terminates before first screen renders.
- **Impact:** Blocks all forks, audits, and educational uses that change the bundle identifier.
- **Workaround applied (local-only):** Modified ErrorReporterFactory.swift to always return OSLogErrorReporter() regardless of build configuration, bypassing the Firebase initialization path.
- **Suggested upstream fix:** Wrap FIRApp.configure() in an Objective-C exception handler, and gate Crashlytics behind a runtime check that validates the plist matches the current bundle identifier. Alternatively, gate behind an explicit build flag rather than #if DEBUG.
- **Category:** Coupling to production environment + Robustness + Architecture.

---

## F-RT-06: Initial sync does not trigger automatically on first cold start after login

- **Component:** Vault synchronization, observed across SyncService and VaultRepository initialization paths.
- **Trigger:** First cold start of the app after a clean simulator state (no prior SQLite file in the sandbox), with the user successfully completing the master-password sign-in flow.
- **Symptom:** App authenticates successfully (OAuth token obtained, crypto state initialized, OS confirms session validity), but the vault list renders empty with the placeholder "Save and protect your data". Server holds 150 items for the account. Network logs show an initial GET /accounts/revision-date returning 200 OK, but the subsequent GET /sync is not issued automatically. The vault remains empty until the user manually invokes Settings > Other > Sync now, at which point all 150 items are fetched, decrypted, and persisted in approximately 5-15 seconds.
- **Hypothesis:** Sync trigger logic compares local revision date in CoreData against server's revision date. On a fresh install with empty SQLite store, the local revision date may default to a value (0, nil, or current-time) that the comparison logic interprets as "already up to date", suppressing the full sync. Manual Sync now bypasses this comparison and forces an unconditional sync.
- **Impact:** A new install or fresh-keychain user experiences an empty vault on first cold start and must discover the manual Sync now option to populate it. Poor first-run experience, particularly impactful for users restoring on a new device.
- **Workaround in audit run protocol:** For scenarios S2-S4, ensure the vault is populated via one manual Sync now before profiling. Scenario S1 measures both the cold-start path (without items) and the manual-sync latency separately. This is the only finding for which NO source modification is applied; the workaround is procedural.
- **Suggested upstream fix:** On first cold start with empty CoreData store, unconditionally trigger a full sync as part of post-login flow, independent of revision-date comparison. Alternatively, treat a nil or zero local revision date as a sentinel that forces a full sync.
- **Category:** First-run experience + Synchronization logic.
- **Priority:** HIGH for production users (affects any new-device install), unlike F-RT-01 through F-RT-05/F-RT-07 which are latent for production users.

---

## F-RT-07: Keychain access group entitlement required to read auth tokens

- **Files:**
  - BitwardenKit/Core/Platform/Extensions/Bundle+Extensions.swift:67 (keychainAccessGroup computed property)
  - BitwardenKit/Core/Platform/Services/KeychainServiceFacade.swift (lines 178, 190, 216, 314) for keychain queries
  - BitwardenShared/Core/Platform/Services/MigrationService.swift:153 for legacy-item migration path
- **Trigger:** Any keychain read or write that includes kSecAttrAccessGroup as a query attribute, when the binary is not signed with a matching keychain-access-groups entitlement.
- **Symptom:** Repeated "Error: BitwardenKit.KeychainServiceError.osStatusError(-34018)" entries in runtime log (errSecMissingEntitlement). The user is technically authenticated (OAuth token in memory), but subsequent reads of the persisted token from the keychain fail, so the app cannot make authenticated requests after the foreground session expires. In Debug builds, the assertionFailure from F-RT-04 fires and the app crashes outright.
- **Root cause:** Keychain Sharing entitlements are not issuable under free-tier signing. The code unconditionally includes the access-group attribute in every keychain query, so iOS rejects each query with -34018.
- **Workaround applied (local-only):**
  - Bundle+Extensions.swift:67: keychainAccessGroup modified to return an empty string.
  - KeychainServiceFacade.swift: four occurrences of kSecAttrAccessGroup in keychain queries commented out, so the default access group (the app's own bundle ID) is used. The default access group is always available regardless of entitlements and is functionally sufficient for an app that does not need cross-process keychain sharing.
  - MigrationService.swift:153: kSecAttrAccessGroup: Bundle.main.keychainAccessGroup entry removed from the attributesToUpdate dictionary; dictionary now initialized as [:] and populated only by the actual migration logic.
- **Suggested upstream fix:** Introduce a build-time switch or runtime detection that omits kSecAttrAccessGroup from keychain queries when the entitlement is not present. This would allow the app to fall back to the default keychain access group (its own bundle ID) for forks and educational builds, without compromising production behavior. A more architectural fix would centralize the access-group decision in a single helper that the four query sites consult.
- **Category:** Entitlement-coupled persistence + Robustness.

---

## Cross-cutting observations

These seven findings, taken together, paint a consistent picture: Bitwarden iOS is well-architected for its intended production deployment but is brittle when the environment deviates from that deployment. The brittleness concentrates in four places:

1. The persistence layer's reliance on App Groups (F-RT-01, F-RT-02)
2. The keychain layer's reliance on Keychain Sharing entitlements (F-RT-07)
3. The error-reporting layer's coupling to Firebase (F-RT-04, F-RT-05)
4. First-run synchronization logic (F-RT-06) — the only finding that would affect production users on a new-device install

Resource loading hygiene (F-RT-03) is a smaller separate concern.

A modest set of upstream changes (graceful fallbacks for App Group and Keychain access group lookups, runtime-validated Crashlytics initialization, classified error handling, and an unconditional first-sync after empty-store login) would substantially improve the codebase's accessibility to external contributors and security researchers, and would also improve the first-install experience for production users, all without affecting the steady-state production behavior.
