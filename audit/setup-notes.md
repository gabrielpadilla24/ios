# Audit Setup Notes

Working notes from the setup phase of the audit fork. Captures every local-only modification, the rationale, and how to revert if needed.

This document reflects **Approach 2** of the setup: the persistence-fidelity refinement that replaces the original `.memory` workaround for F-RT-01/F-RT-02 with a sandbox `Documents/` directory bypass, preserving real disk-backed CoreData. Additionally captures the keychain access-group bypass for F-RT-07.

## Environment

- macOS host: Apple Silicon Mac
- Xcode: 26.5
- Reference simulator: iPhone 17 Pro, iOS 26.3.1
- Apple Developer Team: U67Q3MTU2M (free Personal Team)
- Apple ID: gabrielpadillab03@gmail.com
- Custom bundle ID: com.gabrielpadilla24.bwaudit.passwordmanager
- Audit branch: audit/app-report-4
- Audited commit: 5218b7f21bce14622b823e41e844844aeedd593b
- Date: 2026-05-13 (initial setup) / 2026-05-14 (Approach 2 refinement)

## Local-only modifications NOT committed

All of the modifications below are required to run the app under free-tier signing while preserving real disk-backed persistence for the audit. None of them is committed to the audit branch. They are documented here, in audit/findings-log.md, and in the report itself (§17.4).

---

### M1. Configs/Local-bwpm.xcconfig (gitignored)

Path: Configs/Local-bwpm.xcconfig

Contents:
    DEVELOPMENT_TEAM = U67Q3MTU2M
    ORGANIZATION_IDENTIFIER = com.gabrielpadilla24.bwaudit
    BASE_BUNDLE_ID = $(ORGANIZATION_IDENTIFIER).passwordmanager
    SHARED_APP_GROUP_IDENTIFIER = group.$(ORGANIZATION_IDENTIFIER)
    CODE_SIGN_STYLE = Automatic

This file is excluded by .gitignore (pattern: Configs/Local-*.xcconfig).

---

### M2. Bitwarden.entitlements replaced with empty dict

Path: Bitwarden/Application/Support/Bitwarden.entitlements

Original backed up at: Bitwarden/Application/Support/Bitwarden.entitlements.original
(also gitignored, pattern: Bitwarden/Application/Support/Bitwarden.entitlements.original)

Capabilities disabled (not supported under free-tier signing):
- App Groups
- Associated Domains
- iCloud
- Keychain Sharing
- NFC Tag Reading
- Push Notifications
- AutoFill Credential Provider

To revert:
    cp Bitwarden/Application/Support/Bitwarden.entitlements.original \
       Bitwarden/Application/Support/Bitwarden.entitlements

---

## Persistence-layer bypass (Approach 2) — F-RT-01, F-RT-02

The original setup used `storeType: .memory` for both DataStore initializers (M3, M4 of the previous version). This was replaced in Approach 2 with a Documents-directory bypass that preserves real disk-backed CoreData. The `.persisted` branch of each DataStore is now functional; only the container location differs from production (per-process sandbox instead of shared App Group).

### M3. BitwardenShared DataStore — sandbox Documents directory

Path: BitwardenShared/Core/Platform/Services/Stores/DataStore.swift

Change in the `.persisted` branch of the switch (~line 71):
    BEFORE:
        let storeURL = FileManager.default
            .containerURL(forSecurityApplicationGroupIdentifier: Bundle.main.groupIdentifier)!
            .appendingPathComponent("Bitwarden.sqlite")
    AFTER:
        // AUDIT: bypass App Group container (free-tier signing), use sandbox Documents directory
        let storeURL = FileManager.default
            .urls(for: .documentDirectory, in: .userDomainMask)
            .first!
            .appendingPathComponent("Bitwarden.sqlite")

Reason: the App Group container URL returns nil under free-tier signing. Real disk-backed CoreData is preserved; only the container location changes. See finding F-RT-01.

NOTE: Also reverted the original `.memory` workaround in ServiceContainer.swift:498. The constructor call is now back to its upstream form:
    let dataStore = DataStore(errorReporter: errorReporter)

### M4. AuthenticatorShared DataStore — sandbox Documents directory

Path: AuthenticatorShared/Core/Platform/Services/Stores/DataStore.swift

Same change as M3 in the `.persisted` branch (~line 67), but with "Authenticator.sqlite" filename instead of "Bitwarden.sqlite".

### M5. AuthenticatorBridgeDataStore — sandbox Documents directory

Path: AuthenticatorBridgeKit/AuthenticatorBridgeDataStore.swift

Same change as M3 in the `.persisted` branch (~line 102). The `groupIdentifier` parameter is no longer consulted in the audit build, but its signature is preserved to minimize the diff from upstream.

NOTE: Also reverted the original `.memory` workaround in ServiceContainer.swift:1097. The constructor call is now back to its upstream form:
    let authenticatorDataStore = AuthenticatorBridgeDataStore(
        errorReporter: errorReporter,
        groupIdentifier: Bundle.main.sharedAppGroupIdentifier,
    )

### M6. FileManager+Extensions flight-recorder URL — sandbox Documents directory

Path: BitwardenKit/Core/Platform/Extensions/FileManager+Extensions.swift

Change at line 9 (flightRecorderLogURL function):
    BEFORE:
        containerURL(forSecurityApplicationGroupIdentifier: Bundle.main.groupIdentifier)?
            .appendingPathComponent("FlightRecorderLogs", isDirectory: true)
    AFTER:
        // AUDIT: bypass App Group container (free-tier signing), use sandbox Documents directory
        urls(for: .documentDirectory, in: .userDomainMask)
            .first?
            .appendingPathComponent("FlightRecorderLogs", isDirectory: true)

Reason: prevents the flight recorder feature from silently disabling itself under free-tier signing.

---

## Keychain access-group bypass — F-RT-07

The keychain queries in the codebase unconditionally include `kSecAttrAccessGroup` as a query attribute. Under free-tier signing, keychain-access-groups entitlements are not issuable, so iOS rejects every query with `errSecMissingEntitlement (-34018)`. The bypass below removes the access-group attribute, falling back to the default access group (the app's own bundle ID) which is always available.

### M7. Bundle+Extensions keychainAccessGroup returns empty

Path: BitwardenKit/Core/Platform/Extensions/Bundle+Extensions.swift

Change at line 67:
    BEFORE:
        public var keychainAccessGroup: String {
            infoDictionary?["BitwardenKeychainAccessGroup"] as? String ?? appIdentifier
        }
    AFTER:
        public var keychainAccessGroup: String {
            // AUDIT: empty string disables kSecAttrAccessGroup in keychain queries (free-tier signing)
            ""
        }

### M8. KeychainServiceFacade — 4 kSecAttrAccessGroup lines commented

Path: BitwardenKit/Core/Platform/Services/KeychainServiceFacade.swift

Lines 178, 190, 216, 314 (one occurrence each in different keychain query dictionaries):
    BEFORE:
        kSecAttrAccessGroup as String: appSecAttrAccessGroup,
    AFTER:
        // AUDIT: kSecAttrAccessGroup as String: appSecAttrAccessGroup,

Backup: BitwardenKit/Core/Platform/Services/KeychainServiceFacade.swift.original (gitignored).

### M9. MigrationService dictionary fix

Path: BitwardenShared/Core/Platform/Services/MigrationService.swift

Change at line 153:
    BEFORE:
        var attributesToUpdate: [CFString: Any] = [
            kSecAttrAccessGroup: Bundle.main.keychainAccessGroup,
        ]
    AFTER:
        // AUDIT: kSecAttrAccessGroup removed for free-tier signing
        var attributesToUpdate: [CFString: Any] = [:]

Reason: empty dictionary literal `[:]` required to satisfy Swift's type inference after removing the only initial key.

---

## Crash-reporting bypass — F-RT-05

### M10. ErrorReporterFactory.swift -- unconditional OSLogErrorReporter

Path: Bitwarden/Application/Services/ErrorReporter/ErrorReporterFactory.swift

The full file was replaced. The new content unconditionally returns OSLogErrorReporter regardless of build configuration. The original used #if DEBUG / #else with CrashlyticsErrorReporter.shared in the else branch.

Reason: Firebase Crashlytics initialization crashes with NSException when the bundle identifier does not match the GOOGLE_APP_ID in GoogleService-Info.plist. See finding F-RT-05.

---

## Scheme modifications

### M11. Xcode scheme -- removed Watch and Extensions

Path: Bitwarden.xcodeproj/xcshareddata/xcschemes/Bitwarden.xcscheme

Modified via Xcode UI (not by hand-editing the XML). Changes:

a) Target Dependencies of the Bitwarden target (Build Phases tab):
   Removed: BitwardenWatchApp, BitwardenWatchWidgetExtension, BitwardenActionExtension,
            BitwardenAutoFillExtension, BitwardenNotificationExtension, BitwardenShareExtension
   Kept:    BitwardenShared, AuthenticatorBridgeKit, BitwardenKit, BitwardenResources, Networking

b) Embed Watch Content build phase: deleted entirely.

c) Embed Foundation Extensions build phase: 4 items removed
   (ActionExtension, AutoFillExtension, NotificationExtension, ShareExtension).

d) Build options in the scheme: unchecked "Find Implicit Dependencies".

Reason: prevents Xcode from requesting download of watchOS 26.5 SDK and from
trying to compile extensions that cannot be signed under free-tier.

---

## Procedural workaround — F-RT-06

The first cold start after a clean simulator state requires a manual sync to populate the vault from the server. This is a workflow step, NOT a source modification.

Procedure: after sign-in completes and the vault renders empty with "Save and protect your data", tap:
    Settings > Other > Sync now

The full sync takes approximately 5-15 seconds and persists 150 items to disk. Subsequent cold starts will load from disk normally; the manual sync is only required on first install.

For audit profiling:
- Scenario S1 measures both the cold-start path (without items) and the manual-sync latency separately.
- Scenarios S2, S3, S4 assume the vault is already populated; manual sync must be done before profiling them.

---

## What these modifications do NOT change

The modifications target only four entitlement boundaries (App Groups, Keychain access groups, Firebase, build flags) and the schemes. The following layers remain unchanged from upstream:

- The CoreData stack itself: NSPersistentContainer, NSPersistentStoreDescription, managed object model. Disk I/O is genuine.
- The authentication flow, OAuth token handling, and crypto initialization. Sign-in produces a real bearer token, used for real API requests.
- The vault synchronization logic. The GET /sync response is processed identically; items are decrypted and inserted into CoreData as in production.
- The view layer: processors, coordinators, store. UI behavior is identical to production.
- The networking stack, error handling, and retry logic.

---

## Workflow rules

1. Never use `git add -A` or `git add .` in this repository.
   The Bitwarden.entitlements modification (and the 8 other source modifications) would be staged accidentally.
   Always use specific paths: `git add path/to/specific/file`.

2. When pulling from upstream:
   - Save current local-only changes (git stash).
   - Pull.
   - Restore: cp Bitwarden.entitlements.original onto Bitwarden.entitlements if needed,
     then re-apply the source modifications M3 through M10.

3. Verify before each profiling session (sanity checks):

   a) git status should show as modified:
      - Bitwarden/Application/Support/Bitwarden.entitlements
      - BitwardenShared/Core/Platform/Services/Stores/DataStore.swift
      - AuthenticatorShared/Core/Platform/Services/Stores/DataStore.swift
      - AuthenticatorBridgeKit/AuthenticatorBridgeDataStore.swift
      - BitwardenKit/Core/Platform/Extensions/FileManager+Extensions.swift
      - BitwardenKit/Core/Platform/Extensions/Bundle+Extensions.swift
      - BitwardenKit/Core/Platform/Services/KeychainServiceFacade.swift
      - BitwardenShared/Core/Platform/Services/MigrationService.swift
      - Bitwarden/Application/Services/ErrorReporter/ErrorReporterFactory.swift

   b) cat Bitwarden/Application/Support/Bitwarden.entitlements
      should show empty <dict></dict>.

   c) grep -n "documentDirectory" BitwardenShared/Core/Platform/Services/Stores/DataStore.swift
      should show the bypass at ~line 71.

   d) grep -n "containerURL(forSecurityApplicationGroupIdentifier" --include="*.swift" .
      should return empty (no remaining App Group container references in code).

## Build commands

Clean build:
    Cmd+Shift+K in Xcode, or:
    xcodebuild clean -workspace Bitwarden.xcworkspace -scheme Bitwarden

Build for Debug (Cmd+R):
    Uses scheme Run action, buildConfiguration = Debug.

Build for Release (Cmd+I or Profile in Instruments):
    Uses scheme Profile action, buildConfiguration = Release.

## Test account

vault.bitwarden.com test account, populated with 150 items via CSV import.
Items: 110 logins, 20 secure notes, 12 cards, 8 identities.
Favorites: ~12 items.
TOTP seeds: 28 items.

CSV file in branch: audit/bitwarden_audit_seed.csv

## Recovery from broken state

If the app stops launching after a code change:

1. git status -- verify expected files are modified (see Workflow rules 3a above).
2. Verify the Documents-directory bypass is in place: grep -n "documentDirectory" in the 4 affected files.
3. Verify the keychain bypass: grep -n "AUDIT: kSecAttrAccessGroup" in KeychainServiceFacade.swift.
4. Verify ErrorReporterFactory.swift still returns OSLogErrorReporter().
5. Cmd+Shift+K (Clean), Cmd+B (Build), Cmd+R (Run).

If Xcode requests watchOS 26.5 download again:
- Edit Scheme, Build tab, uncheck Find Implicit Dependencies, Close.
- Check Build Phases of Bitwarden target: ensure no "Embed Watch Content" phase exists.

## Revert all local modifications (clean state for paid Apple Developer)

To revert to a state suitable for paid Apple Developer signing:

1. Restore entitlements:
       cp Bitwarden/Application/Support/Bitwarden.entitlements.original \
          Bitwarden/Application/Support/Bitwarden.entitlements

2. Restore KeychainServiceFacade:
       cp BitwardenKit/Core/Platform/Services/KeychainServiceFacade.swift.original \
          BitwardenKit/Core/Platform/Services/KeychainServiceFacade.swift

3. Revert source modifications via git:
       git checkout HEAD -- \
         BitwardenShared/Core/Platform/Services/Stores/DataStore.swift \
         AuthenticatorShared/Core/Platform/Services/Stores/DataStore.swift \
         AuthenticatorBridgeKit/AuthenticatorBridgeDataStore.swift \
         BitwardenKit/Core/Platform/Extensions/FileManager+Extensions.swift \
         BitwardenKit/Core/Platform/Extensions/Bundle+Extensions.swift \
         BitwardenShared/Core/Platform/Services/MigrationService.swift \
         Bitwarden/Application/Services/ErrorReporter/ErrorReporterFactory.swift

4. Restore the original scheme via the Xcode UI (re-add removed extensions).

5. Configure paid signing in Configs/Local-bwpm.xcconfig.
