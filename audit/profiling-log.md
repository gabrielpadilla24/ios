# Bitwarden iOS — Profiling Log (App Report 4)

**Repository:** bitwarden/ios (audit fork: gabrielpadilla24/ios)
**Branch audited:** main @ commit [PLACEHOLDER — actualizar tras primer build]
**Audit branch:** audit/app-report-4

**Profiling host:** macOS [version], Apple Silicon
**Xcode version:** 26.5 (repo expects 26.2 — compatibility warning acknowledged)
**Build configuration:** Release
**Profiling target:** iPhone 17 Pro Simulator, iOS 26.2 (per `.test-simulator-device-name`)

**Authors:** Gabriel Padilla, Pablo Galindo, Juan Pablo Rivera
**Audit start date:** May 13, 2026
**Limitation:** Profiling conducted on iOS simulators (free Apple Developer account constraints regarding multi-target signing and entitlements). Reframed as cross-device fragmentation analysis (extra requirement #6).

---

## Screens inventory

| # | Screen | Type | Target | Key actions | Heavy components |
|---|--------|------|--------|-------------|------------------|
| 1 | Sign In / Lock | Auth | Bitwarden (main) | Email/password, biometric unlock | Crypto operations |
| 2 | Vault list | List view | Bitwarden | Scroll, search, item tap | UITableView, image loading |
| 3 | Item detail | Detail | Bitwarden | View, edit, copy | Form fields, sync |
| 4 | Password generator | Tool | Bitwarden | Generate, copy, save | Random generation |
| 5 | Send | Feature | Bitwarden | Create text/file send | File picker |
| 6 | Settings | Hierarchical | Bitwarden | Multiple subscreens | List navigation |
| 7 | AutoFill extension | Extension | BitwardenAutoFillExtension | Suggest credentials | Out of scope (signing) |
| 8 | Authenticator | Separate app | Authenticator | TOTP codes | Camera, QR scan |

---

## Scenario scripts

### S1 — Cold start → Sign in → Vault list
1. Quit app from simulator app switcher.
2. Wait 10 seconds.
3. Tap Bitwarden icon.
4. On Welcome screen, enter memorized credentials (test account).
5. Tap Sign In.
6. Wait until Vault list is fully rendered with all items visible.
7. Remain idle for 5 seconds.

### S2 — Vault scroll + live search
1. From Vault list, slow scroll top → bottom.
2. Fast flick back to top.
3. Tap search field, type "test" character by character (~150ms per keystroke).
4. Clear search.
5. Scroll bottom → top one more time.
6. Remain idle for 3 seconds.

### S3 — Open item → Edit → Save
1. From Vault list, tap a Login item.
2. Wait for detail view to load.
3. Tap Edit.
4. Modify password field (delete and re-type).
5. Tap Save.
6. Wait for save confirmation and return to list.
7. Remain idle for 3 seconds.

### S4 — Navigation cycle + Password generation + Copy
1. Tap Generator tab.
2. Generate password (tap Regenerate 3 times).
3. Copy generated password to clipboard.
4. Navigate to Settings.
5. Return to Vault.
6. Tap Generator → Vault → Settings → Vault (full cycle ×3).
7. Remain idle for 3 seconds.

---

## Scenario S1 — Cold start → Sign in → Vault list

### Time Profiler

| Run | TTFF (ms) | TTI (ms) | Main thread % | Threads | Notes |
|-----|-----------|----------|---------------|---------|-------|
| 1   |           |          |               |         |       |
| 2   |           |          |               |         |       |
| 3   |           |          |               |         |       |
| **Mean ± SD** |  |  |  |  |  |

### SwiftUI Instrument

| Run | View body calls | Slowest view | Notes |
|-----|-----------------|--------------|-------|
| 1   |                 |              |       |
| 2   |                 |              |       |
| 3   |                 |              |       |

### Core Animation

| Run | fps avg | fps min | Hitches | Notes |
|-----|---------|---------|---------|-------|
| 1   |         |         |         |       |
| 2   |         |         |         |       |
| 3   |         |         |         |       |

### Allocations + Leaks

| Run | Peak RAM (MB) | Final RAM (MB) | Leaks | Notes |
|-----|---------------|----------------|-------|-------|
| 1   |               |                |       |       |
| 2   |               |                |       |       |
| 3   |               |                |       |       |

---

## Scenario S2 — Vault scroll + live search

[Mismas tablas]

---

## Scenario S3 — Open item → Edit → Save

[Mismas tablas, agregar Network instrument para capturar sync]

---

## Scenario S4 — Password gen + Navigation cycle

[Mismas tablas, foco en Allocations: ¿la memoria vuelve a baseline?]

---

## Fragmentation runs

| Simulator | iOS | S1 cold start (ms) | S2 fps avg | S3 latency (ms) | S4 leaks | Notes |
|-----------|-----|--------------------|------------|-----------------|----------|-------|
| iPhone 17 Pro | 26.2 | | | | | Reference (repo official sim) |
| iPhone SE 3 | 26.0 | | | | | Low-end |
| iPhone 16 | 26.0 | | | | | Mid-range |

---

## Carry-over validation from App Report 3

Map of patterns/anti-patterns identified statically in Report 3 → empirical validation in this profiling.

| # | Source (Report 3) | File:lines | Pattern type | Validation method (Report 4) | Validated? |
|---|-------------------|------------|--------------|------------------------------|------------|
| 1 | Good ECn: Retry on 401 | `Networking/.../HTTPService.swift:152-161` | Eventual connectivity | Network instrument under NLC 3G | Pending |
| 2 | Good ECn: Skip sync gracefully | `BitwardenShared/.../SyncService.swift:326-343` | Eventual connectivity | NLC 100% Loss | Pending |
| 3 | Anti-pattern: No offline queue | (project-wide) | Eventual connectivity | Behavioral observation under NLC | Pending |
| 4 | Cache: NSCache for images | `BitwardenWatchApp/.../ImageView.swift:39-108` | Caching | Allocations during scroll | Pending |
| 5 | Cache: CoreData vault | `BitwardenShared/.../CipherDataStore.swift:118-124` | Caching | Cold start latency on warm vs cold start | Pending |
| 6 | Memory: weak delegates | `BitwardenShared/.../VaultCoordinator.swift` | Memory mgmt | Leaks instrument under S4 | Pending |
| 7 | Memory: [weak self] in Timer | `BitwardenShared/.../AppProcessor.swift:572-574` | Memory mgmt | Allocations during 5-min wait | Pending |
| 8 | Memory: deinit Task cancel | `BitwardenShared/.../SearchProcessorMediator.swift:63-67` | Memory mgmt | Threading during search exit | Pending |
| 9 | Anti-pattern: Strong capture + force-unwrap | `BitwardenWatchApp/.../ImageView.swift:88-94` | Memory bug | Static evidence + propose fix | N/A (out of scope: watchOS) |
| 10 | Threading: async/await + backgroundContext | `CipherDataStore.swift:118-124` | Threading | Main thread % during vault fetch | Pending |
| 11 | Threading: DispatchQueue QoS for Camera | `CameraService.swift:150-157` | Threading | (Out of scope: requires camera entitlement) | N/A |
| 12 | Threading: Task cancellation for search | `SearchProcessorMediator.swift:71-89` | Threading | Time Profiler during fast typing in search | Pending |
| 13 | Anti-pattern: Image decoding on main thread | `BitwardenWatchApp/.../ImageView.swift:88-95` | Threading bug | Static evidence + propose fix | N/A (out of scope: watchOS) |

---

## Bug log (issues opened in upstream GitHub)

| # | Severity | Title | Scenario | File/Line | URL | Status |
|---|----------|-------|----------|-----------|-----|--------|
| 1 |          |       |          |           |     | open   |

---

## Optimization implementation tracker

| # | Optimization | File | Status | Before metric | After metric | Δ | PR URL |
|---|--------------|------|--------|---------------|--------------|---|--------|
| 1 |              |      | TODO   |               |              |   |        |