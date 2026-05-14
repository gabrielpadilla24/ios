# Audit Setup Notes

Working notes from the setup phase of the audit fork. Captures every local-only modification, the rationale, and how to revert if needed.

## Environment

- macOS host: Apple Silicon Mac
- Xcode: 26.5
- Reference simulator: iPhone 17 Pro, iOS 26.3.1
- Apple Developer Team: U67Q3MTU2M (free Personal Team)
- Apple ID: gabrielpadillab03@gmail.com
- Custom bundle ID: com.gabrielpadilla24.bwaudit.passwordmanager
- Audit branch: audit/app-report-4
- Audited commit: 5218b7f21bce14622b823e41e844844aeedd593b
- Date: 2026-05-13

## Local-only modifications NOT committed

All of the modifications below are required to run the app under free-tier signing.
None of them is committed to the audit branch.
They are documented here, in audit/findings-log.md, and in the report itself.

### M1. Configs/Local-bwpm.xcconfig (gitignored)

Path: Configs/Local-bwpm.xcconfig

Contents:
    DEVELOPMENT_TEAM = U67Q3MTU2M
    ORGANIZATION_IDENTIFIER = com.gabrielpadilla24.bwaudit
    BASE_BUNDLE_ID = $(ORGANIZATION_IDENTIFIER).passwordmanager
    SHARED_APP_GROUP_IDENTIFIER = group.$(ORGANIZATION_IDENTIFIER)
    CODE_SIGN_STYLE = Automatic

This file is excluded by .gitignore (pattern: Configs/Local-*.xcconfig).

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

### M3. ServiceContainer.swift line 498 -- DataStore storeType .memory

Path: BitwardenShared/Core/Platform/Services/ServiceContainer.swift

Change at line 498 (after our intervention):
    BEFORE: let dataStore = DataStore(errorReporter: errorReporter)
    AFTER:  let dataStore = DataStore(errorReporter: errorReporter, storeType: .memory) // AUDIT: workaround for free Apple Developer (no App Groups)

Reason: bypasses force-unwrap on FileManager.default.containerURL(forSecurityApplicationGroupIdentifier:) which returns nil under free-tier signing. See finding F-RT-01.

### M4. ServiceContainer.swift line 1097 -- AuthenticatorBridgeDataStore .memory

Path: BitwardenShared/Core/Platform/Services/ServiceContainer.swift

Change at line 1097:
    BEFORE: storeType: .persisted,
    AFTER:  storeType: .memory, // AUDIT: workaround for free Apple Developer (no App Groups)

Reason: same as M3, applied to AuthenticatorBridgeDataStore. See finding F-RT-02.

### M5. ErrorReporterFactory.swift -- unconditional OSLogErrorReporter

Path: Bitwarden/Application/Services/ErrorReporter/ErrorReporterFactory.swift

The full file was replaced. The new content unconditionally returns OSLogErrorReporter regardless of build configuration. The original used #if DEBUG / #else with CrashlyticsErrorReporter.shared in the else branch.

Reason: Firebase Crashlytics initialization crashes with NSException when the bundle identifier does not match the GOOGLE_APP_ID in GoogleService-Info.plist. See finding F-RT-05.

### M6. Xcode scheme modifications -- removed Watch and Extensions

Path: Bitwarden.xcodeproj/xcshareddata/xcschemes/Bitwarden.xcscheme

Modified via Xcode UI (not by hand-editing the XML). Changes:

a) Target Dependencies of the Bitwarden target (Build Phases tab):
   Removed: BitwardenWatchApp, BitwardenActionExtension, BitwardenAutoFillExtension,
            BitwardenNotificationExtension, BitwardenShareExtension
   Kept:    BitwardenShared, AuthenticatorBridgeKit, BitwardenKit, BitwardenResources, Networking

b) Embed Watch Content build phase: deleted entirely.

c) Embed Foundation Extensions build phase: 4 items removed
   (ActionExtension, AutoFillExtension, NotificationExtension, ShareExtension).

d) Build options in the scheme: unchecked "Find Implicit Dependencies".

Reason: prevents Xcode from requesting download of watchOS 26.5 SDK and from
trying to compile extensions that cannot be signed under free-tier.

## Workflow rules

1. Never use `git add -A` or `git add .` in this repository.
   The Bitwarden.entitlements modification would be staged accidentally.
   Always use specific paths: `git add path/to/specific/file`.

2. When pulling from upstream:
   - Save current local-only changes (git stash).
   - Pull.
   - Restore: cp Bitwarden.entitlements.original onto Bitwarden.entitlements if needed,
     then re-apply the source modifications (M3, M4, M5).

3. Verify before each profiling session:
   - git status should show: Bitwarden.entitlements, ServiceContainer.swift,
     ErrorReporterFactory.swift as modified.
   - cat Bitwarden/Application/Support/Bitwarden.entitlements
     should show empty <dict></dict>.
   - sed -n '498p' BitwardenShared/Core/Platform/Services/ServiceContainer.swift
     should show storeType: .memory.

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

1. git status -- verify expected files are modified.
2. Check that the .memory workarounds are still in ServiceContainer.swift.
3. Check that ErrorReporterFactory.swift still returns OSLogErrorReporter().
4. Cmd+Shift+K (Clean), Cmd+B (Build), Cmd+R (Run).

If Xcode requests watchOS 26.5 download again:
- Edit Scheme, Build tab, uncheck Find Implicit Dependencies, Close.
- Check Build Phases of Bitwarden target: ensure no "Embed Watch Content" phase exists.
