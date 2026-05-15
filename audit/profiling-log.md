# Bitwarden iOS — Profiling Log (App Report 4)

**Repository:** bitwarden/ios (audit fork: gabrielpadilla24/ios)
**Branch audited:** main
**Audit branch:** audit/app-report-4

**Profiling host:** macOS [version], Apple Silicon
**Xcode version:** 26.5 (repo expects 26.2 — compatibility warning acknowledged)
**Build configuration:** Release
**Profiling target:** iPhone 17 Pro Simulator, iOS 26.3.1 (per `.test-simulator-device-name`)

**Authors:** Gabriel Padilla, Pablo Galindo, Juan Pablo Rivera
**Audit start date:** May 13, 2026
**Limitation:** Profiling conducted on iOS simulators (free Apple Developer account constraints regarding multi-target signing and entitlements). Reframed as cross-device fragmentation analysis (extra requirement #6).

**Document organization:** Each scenario (S1-S4) is analyzed across the four rubric dimensions: GPU rendering analysis, Overdrawing analysis, Memory management, and Threading. Memory management is further decomposed into the four sub-questions from the rubric (M-i through M-iv). Threading is decomposed into three sub-questions (T-i through T-iii).

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

# Scenario S1 — Cold start → Sign in → Vault list

## S1 — Methodology

Three independent cold-start runs were executed per instrument, with the simulator restored to clean home-screen state between runs (app terminated via `xcrun simctl terminate`, 10s idle wait, no simulator reboot). The same SQLite-backed vault (~150 items, 80 KB main DB + 314 KB WAL) was preserved across all runs to ensure measurements reflect a realistic warm-vault cold-start, not a first-install scenario.

**Instruments used for S1:**
- App Launch (cold start latency, dyld activity, thread state) — completed
- Allocations + Leaks (memory footprint, leak detection, allocation patterns) — completed
- Color Blended Layers debug overlay (overdrawing visualization) — completed
- Animation Hitches template (Display + Time Profiler + Thread State Trace + Thermal State + Hangs sub-instruments; the Hitches sub-instrument is not supported on Simulator and was removed) — completed
- Metal System Trace (GPU consumption analysis attempt) — failed, see "GPU consumption — scope limitation" subsection
- Energy Log (power consumption analysis) — not attempted, physical-device-only, see "Power consumption — scope limitation" subsection

**Raw screenshots:** `audit/profiling/screenshots/s1_app_launch/` (12 PNGs, 3 runs), `audit/profiling/screenshots/s1_allocations/` (12 PNGs, 3 runs), `audit/profiling/screenshots/s1_overdrawing/` (4 PNGs), and `audit/profiling/screenshots/s1_threading/` (18 PNGs, 3 runs). Total: 46 PNGs documenting S1.

---

## S1 — GPU rendering analysis

*Status: COMPLETE — data from Animation Hitches instrument (Display sub-instrument) across 3 runs of 30s each. Note: the Hitches sub-instrument itself is not supported on the iOS Simulator (Xcode emits "Hitches is not supported on this platform"), so quantitative hitch counts at the OS level cannot be reported from this environment. Display, Time Profiler, Thread State Trace, Thermal State, and Hangs sub-instruments all work.*

### Frame rate metrics

The Display instrument captures Average Frame Time over the cold-start window. Across the 3 runs, the timeline shows a consistent pattern: a tall spike near t≈0:01–0:02s (initial render of the splash + login + vault transition), followed by a sustained moderate-frequency bar pattern through the 30s window with no sustained periods of dropped frames. VSync alignment is visible in the Display 1 track (the red tick pattern under "VSync") and remains regular throughout.

Surface composition during cold start is visible in the Display 1 surface track:
- Runs 1 & 2: Surface 7 (SimRenderServer) appears first, transitions to Surface 7 alone, then to Surface 8.
- Run 3: Surface 7 → Surface 9 (SimRenderServer) → Surface 8 → Surface 7 → Surface 8 chain. The longer Surface 9 segment in Run 3 aligns temporally with the cluster of hangs (see Threading section), suggesting the render server held a single surface composition longer because the app was unresponsive.

### Problems and strengths

**Strengths:**
- Thermal state remains `Nominal` for all 30 seconds across all 3 runs (no throttling).
- VSync cadence is regular; no extended stalls in the display pipeline visible.
- Average Frame Time bars taper down in amplitude after the initial ~2s cold-start render, indicating steady-state rendering is well within budget.

**Problems:**
- The cluster of hangs detected by the Hangs instrument (1 in Run 1, 3 in Run 2, 5 in Run 3 — see Threading T-iii) does coincide temporally with surface composition events in the Display track, meaning the user perceives the unresponsiveness during the visual transition from splash to vault. The render pipeline itself is healthy; the work being done on the main thread blocks frame presentation.
- The Hitches sub-instrument is unavailable on Simulator, so finer-grained metrics (hitch ratio, hitch duration percentile, frame deadlines missed) cannot be quantified without a physical device. This is a free-tier / Simulator limitation, not an app defect.

### GPU consumption — scope limitation

The rubric (item 2) requires GPU consumption analysis. The appropriate Instruments templates for measuring GPU utilization percentage, GPU memory pressure, and per-frame GPU commit cost on iOS are Metal System Trace and the now-deprecated Core Animation instrument (removed in Xcode 26).

**Metal System Trace was attempted** on the iPhone 17 Pro simulator (iOS 26.3.1) with the same target configuration used for the other S1 instruments. Instruments returned **"Failed to load configuration options for iPhone 17 Pro (26.3.1). Error during device communication."** for the Metal Application sub-instrument, and **"No Recording Options"** for the GPU, Display, Metal Resource Events, and Thermal State sub-instruments. The Apple Silicon iOS Simulator does not expose the host Metal stack as a profilable GPU target for iOS apps; Apple's documentation indicates these instruments require a physical iOS device.

**What is reported instead, from the Animation Hitches template's Display sub-instrument:** Average Frame Time per run, VSync alignment, and surface composition (Surface 7 → 8 → 9 chain). These cover the *rendering time* question (rubric item 3.a) but do not give GPU utilization percentages.

Free-tier Apple Developer provisioning (Personal Team, 7-day certificate) prevents practical physical-device deployment at audit timescale (would require re-signing every 7 days for the 3-week audit window). Documented as scope limitation, consistent with the same limitation that applies to power consumption analysis below.

### Power consumption — scope limitation

The rubric (item 2) also requires power consumption analysis. The Energy Log instrument is the appropriate tool but is **physical-device-only** — it requires hardware energy counters not available in the iOS Simulator.

The same free-tier provisioning limitation that blocks Metal System Trace also blocks Energy Log usage. As an indirect proxy, the Animation Hitches template's Thermal State sub-instrument reports `Nominal` across all 30 seconds of every S1 run (3 runs), indicating that whatever the actual power draw is, it is below the threshold where the OS begins thermal throttling. Time Profiler additionally reports total CPU residency (Bitwarden 14.62-40.45 s of CPU time across 30 s wall-clock per run); high-quality power estimation would require correlating this with per-core frequency states, which Energy Log provides and Simulator does not.

Documented as scope limitation alongside GPU consumption. The two limitations are recorded together because they share the same underlying cause (free-tier provisioning blocking physical-device deployment).

---
---

## S1 — Overdrawing analysis

### Method
Simulator → Debug → Color Blended Layers toggle enabled. Four key S1 surfaces captured: master-password lock screen, Vault list (scrolled to top), View Login item detail, and Generator tab. Convention: pixels rendered in red are opaque single-layer (one composition pass, optimal); pixels in green are blended (alpha-composited from multiple layers — overdrawing). Higher green saturation indicates more layers stacked per pixel.

### Captured surfaces

| Screenshot | Surface | File |
|------------|---------|------|
| Lock screen | `Verify master password` view | `s1_overdrawing_lock.png` |
| Vault list (top) | `My Vault` favorites list | `s1_overdrawing_vault_top.png` |
| Item detail | View Login (Airbnb item) | `s1_overdrawing_vault_item.png` |
| Generator tab | Password generator view | `s1_overdrawing_tabbar.png` |

All screenshots in `audit/profiling/screenshots/s1_overdrawing/`.

### Observations by surface

**Lock screen (`s1_overdrawing_lock.png`):**
- Background of the entire screen renders green, including the large empty area below the `Unlock` button. SwiftUI containers are not marked `isOpaque`, so the screen background composes alpha with the layer behind it on every frame.
- Title text (`Verify master password`), avatar circle (`GA`), and dots-menu icon all on green background.
- Opaque (red) components: master-password input field, the info box (`Your vault is locked...`), and the Unlock button itself.
- Net composition: ~70% of pixels are blended single-overlay; ~30% are correctly opaque.

**Vault list scrolled to top (`s1_overdrawing_vault_top.png`):**
- Top section (`My vault` header, search field, `FAVORITES (12)` section label) renders green.
- Each row of the list is rendered as light pink/red (single-overlay with row tint), which is acceptable.
- Tab bar (`My vault`, `Send`, `Generator`, `Settings`) renders strong red on the selected tab and pink on inactive tabs, indicating multi-layer composition on the tab bar specifically — the capsule background, blur, icon, and selected-state highlight stack on top of each other.
- App icons (Airbnb, Amazon, Apple Music, etc.) display in natural colors → single-layer rendering for the icon assets themselves.

**Item detail view (`s1_overdrawing_vault_item.png`):**
- Most aggressive overdrawing observed of the four surfaces. Background is uniformly green; status-bar zone shows a brown/maroon tint, indicating the GPU is compositing more than two layers stacked at the top of the screen.
- All cards (`LOGIN CREDENTIALS`, `AUTOFILL OPTIONS`, `ADDITIONAL OPTIONS`) render as light pink/red over green background → two-layer composition (card overlay + screen background blend).
- Only the floating pencil-edit FAB and individual buttons (star, copy, eye) are correctly opaque.
- Status-bar zone color shift is consistent with the modal being presented on top of the vault list without the underlying view being torn down — the GPU composes Vault list + modal status overlay + ItemDetail content on every frame.

**Generator tab (`s1_overdrawing_tabbar.png`):**
- Background green throughout.
- Card containers (Password/Passphrase/Username selector, generated-password field, length/charset section) render light pink/red over green.
- Interactive controls (Copy button in saturated red, toggles, +/− buttons) correctly opaque.
- Tab bar pattern identical to Vault list: selected tab strong red, inactive pink.
- The lower portion of the screen (around `Avoid ambiguous characters` and the tab bar) shows accumulated red intensity, suggesting additional layers in that region (likely the tab bar's blur/material background composing over the scroll content).

### Problems

| Problem | Surface(s) | Likely cause | Performance implication |
|---------|------------|---------------|--------------------------|
| Screen-wide transparent backgrounds | All four | SwiftUI containers default to non-opaque (`isOpaque = false`); developers did not annotate top-level backgrounds as opaque | GPU compositing on every frame even when content is static; battery and thermal cost in idle and amplified cost during animations |
| Tab bar multi-layer composition | Vault list, Generator (and likely all main tabs) | Tab bar uses material blur + capsule background + selection highlight + icon, all stacked | Per-frame compositing cost concentrated in a 15% screen region; aggravated during tab transitions |
| Item detail modal stacks over Vault list | `s1_overdrawing_vault_item.png` | Item detail presented as modal without tearing down the underlying vault list; both layers remain in the compositor tree | Highest per-frame cost of all S1 surfaces; visible only as battery/thermal drain, not user-visible glitch (until Core Animation hitches surface in S2/S3 measurements) |
| Header zone alpha stacking | Lock, Vault list, Generator | Status bar overlay + screen header background not consolidated | Small per-frame cost but consistent across all screens |

### Strengths

- Interactive elements (buttons, input fields, toggles, icons, FAB) are consistently rendered as opaque single-layer surfaces — these are the touch targets and have correct rendering attributes.
- App-icon assets (Airbnb logo, Amazon Prime logo, etc.) render as opaque single-layer, indicating image assets are properly pre-rendered without unnecessary alpha channels.
- The Unlock button on the lock screen and the Copy button on the Generator render in saturated red, indicating fully opaque rendering for the primary CTAs.
- No shadows, complex gradient overlays, or custom blur effects layered on top of content were observed; the overdrawing comes from default container behavior, not from intentional visual effects.

---

## S1 — Memory management

### M-i — Memory leaks: which, where?

Three Leaks-instrument snapshots per run (every 10 seconds), with the first snapshot at ~0:10s and subsequent at ~0:20s.

| Run | Leaked allocations | Total bytes leaked | Snapshot pattern | Primary responsible library |
|-----|--------------------|--------------------|--------------------|------------------------------|
| 1   | 18                 | ~1.45 KB           | First snapshot clean (green), second leaks detected (red) | `BitwardenSdk_4641...` (Rust SDK via UniFFI) |
| 2   | ~19 (multiple grouping; some counts=2-4) | ~2.0 KB | Both snapshots show leaks (both red) | `BitwardenSdk_4641...` (Rust SDK via UniFFI) |
| 3   | 18                 | ~1.45 KB           | Both snapshots show leaks (both red) | `BitwardenSdk_4641...` (Rust SDK via UniFFI) |

**Leak categorization by source:**

| Source | Per-run count (approx) | Sizes observed | Stack trace signature |
|--------|------------------------|----------------|------------------------|
| Bitwarden Rust SDK via UniFFI FFI boundary | 17-18 | 64 bytes (most), 128 bytes (some) | `alloc::alloc::exchange_malloc` → `alloc::sync::Arc<T>::new` → `uniffi_core::ffi::rustfuture::future` → `ffi_bitwarden_uniffi_rust_future_po...` → `@nonobjc ffi_bitwarden_uniffi_rust...` → `swift::runJobInEstablishedExecutor` |
| CoreData internal | 1 | 16 bytes | `+[_NSMemoryStorePredicateRemapper defaultRemapper]` |

**Interpretation:** All Bitwarden-attributed leaks originate from the Rust SDK boundary, specifically from `Arc<T>` allocations associated with `RustFuture` instances exposed to Swift via UniFFI. The sizes (64 / 128 bytes) are consistent with Arc reference-count headers and small FFI payload structs. When Swift abandons a future before Rust-side completion (cancellation, error path, race), the Arc reference held by Rust does not get decremented, leaving the heap allocation unreachable. The pattern reproduces deterministically across runs (sizes, stack frames, responsible library all identical), indicating structural rather than stochastic origin.

The single CoreData leak (`_NSMemoryStorePredicateRemapper`) is in Apple framework code, not Bitwarden code. It is reported as observed framework behavior; Approach 2 (sandbox `Documents/` directory bypass for free-tier signing) exercises CoreData more visibly than the upstream App Group container configuration, but the leak attribution is to CoreData internals.

**Zero leaks detected in pure Swift Bitwarden code** (no retain-cycle leaks in Swift View / ViewModel / Coordinator code paths).

**Screenshots:** `s1_allocations_run{1,2,3}_leaks.png`

### M-ii — RAM consumption across the scenario

#### Cold-start memory growth pattern

All three runs exhibit a consistent shape: rapid growth during the first ~12-15 seconds (cold start phase, vault decryption and rendering), followed by a plateau in the remaining ~15-18 seconds (idle steady-state with vault rendered).

#### Per-run memory footprint at end of 30-second window

| Run | Heap & Anonymous VM Persistent | Heap Allocations Persistent | Anonymous VM Persistent | Total Bytes (cumulative churn) | # Persistent allocations | # Transient allocations |
|-----|-------------------------------|------------------------------|---------------------------|-----|-----|-----|
| 1   | 59.68 MiB                     | 23.34 MiB                    | 36.34 MiB                 | 285.46 MiB | 161,141 | 985,941 |
| 2   | 59.82 MiB                     | 23.44 MiB                    | 36.38 MiB                 | 291.96 MiB | 162,126 | 1,045,420 |
| 3   | 60.94 MiB                     | 23.91 MiB                    | 37.03 MiB                 | 293.17 MiB | 163,821 | 1,004,642 |
| **Mean ± SD** | **60.15 MiB ± 0.69** | **23.56 MiB ± 0.30** | **36.58 MiB ± 0.39** | **290.20 MiB ± 4.10** | **162,363 ± 1,355** | **1,012,001 ± 30,322** |
| **Coefficient of variation** | 1.15% | 1.28% | 1.06% | 1.41% | 0.83% | 3.00% |

**Interpretation:**
- Coefficient of variation under 1.5% on all four primary memory metrics → measurements are highly reproducible, findings are structural rather than observational artifacts.
- Total cumulative allocation (~290 MiB) vs persistent footprint (~60 MiB) → ratio ~5:1, indicating ~230 MiB of memory is allocated and freed during the 30-second cold-start window. High churn but consistent with cold-start expectations.
- Heap-real footprint (~23.5 MiB) is small compared to the total Heap+VM number (~60 MiB). The bulk of the footprint is Anonymous VM, dominated by thread stacks (see M-iv) and CoreAnimation surface buffers, not Swift heap objects.

#### Top resident-memory categories at run end (Run 1 representative, Run 2/3 within ±5%)

| Category | Persistent Bytes | # Persistent | Notes |
|----------|------------------|--------------|-------|
| VM: Stack | 24.16 MiB | 14 | Thread stacks (see M-iv for thread origin breakdown) |
| VM: CoreServices | 7.64 MiB | 1 | Apple framework |
| Malloc 16.00 KiB | 3.11 MiB | 199 | Generic heap blocks |
| VM: CoreUI image data | 1.38 MiB | 5 | UI image cache |
| CFString (store) | 1.17 MiB | 6,204 | String storage backing |
| Malloc 8.00 KiB | 824 KiB | 103 | Generic heap blocks |
| VM: SQLite page cache | 768 KiB | 6 | Vault DB page cache |
| Malloc 32 Bytes | 766.56 KiB | 24,530 | High count → many small allocations |
| Malloc 2.00 KiB | 736 KiB | 368 | Generic heap blocks |
| VM: CoreAnimation | 688 KiB | 15 | Render surface buffers |
| UnknownObjectType | 661.30 KiB | 7,457 | Custom Swift type without resolved symbol |
| CFString (immutable) | 610.28 KiB | 13,142 | Immutable strings |
| VM: _SwiftUILayerDelegate | 544 KiB | 26 | SwiftUI layer infrastructure |
| VM: ImageIO_PNG_Data | 400 KiB | 5 | PNG decode buffers |

**Observations on memory-consumer composition:**
- VM: SQLite page cache holds at 768 KiB consistently, indicating bounded DB cache (good: not unbounded growth).
- VM: CoreAnimation 688 KiB for a single-screen app at idle is moderate, consistent with one tab bar + nav stack + list view rendered.
- Two high-count small-allocation categories worth noting: Malloc 32 Bytes (24,530 allocs / 766 KiB) and CFString (immutable) (13,142 allocs / 610 KiB). High allocation counts at small sizes can be a marker of either intensive string processing or many short-lived data structures during cold start. Material for cross-correlation with the Call Tree (see M-iv).

#### Memory under different use scenarios

S1 (cold start) provides the baseline. RAM behavior under S2 (scroll/search), S3 (edit), and S4 (navigation cycles) is documented in the corresponding scenario sections of this log.

**Screenshots:** `s1_allocations_run{1,2,3}_alltracks.png` (timeline), `s1_allocations_run{1,2,3}_summary.png` (Statistics panel).

### M-iii — Libraries available for leak management in iOS apps

Brief survey of tooling applicable to memory and leak management in this codebase. iOS does not provide a third-party "memory library" ecosystem comparable to Java/Android (where libraries like LeakCanary exist) because the platform exposes native Apple-provided tooling. The relevant landscape:

| Tool / Framework | Type | Use case | Applicability to Bitwarden iOS |
|------------------|------|----------|--------------------------------|
| **Instruments Leaks** | Apple, Xcode-bundled | Cycle detection and `malloc`-tracked leak reporting during a profiling session | Used in this audit (S1 / S2 / S3 / S4). Primary leak-detection tool. |
| **Instruments Allocations** | Apple, Xcode-bundled | Heap profile, allocation lifetime tracking, retain-cycle root analysis | Used in this audit. Complementary to Leaks for understanding allocation patterns. |
| **MallocStackLogging** | Apple, system-level env var | Records full backtrace for every allocation; enables `malloc_history` post-mortem analysis | Available as `MallocStackLogging=1` env var in scheme; not used in this audit (overhead too high for 3-run reproducibility). |
| **NSZombie / Zombies instrument** | Apple, Xcode-bundled | Detects over-release of Objective-C objects (use-after-free) | Disabled in this audit (Swift codebase, ARC-managed; would add noise without finding Swift bugs). |
| **MetricKit** (`MXMetricPayload`) | Apple, public framework | On-device collection of memory and performance metrics from real users in production | Not part of this profiling audit, but a candidate optimization for Bitwarden to ship telemetry without sacrificing user privacy. Noted for the optimization proposals section. |
| **os_signpost** | Apple, public framework | Custom code annotations visible in Instruments timelines | Could be added by Bitwarden to make specific phases (decryption, vault load) measurable as Points of Interest. Not yet present in the codebase per static review. |
| **swift_dump_objc_object** / Memory Graph Debugger | Apple, Xcode-bundled | Visual graph of all live objects and their retain relationships at a paused breakpoint | Useful for retain-cycle hunting at a specific moment; not used in this audit (Leaks instrument already covers the cycle-detection use case). |
| **Address Sanitizer (ASan)** | LLVM, opt-in compile flag | Detects use-after-free, heap-buffer-overflow, double-free at runtime | Not enabled in this audit (significant runtime overhead, incompatible with profiling realism). Worth running once outside profiling for code-correctness validation. |

**Note on Rust SDK side:** The `BitwardenSdk` Rust crate observed in this audit could leverage Rust-side tooling (Miri, Valgrind on Linux/CI, the `tracing` crate for instrumented allocations) for leak detection at the FFI boundary independently of iOS Instruments. The leaks detected in S1 M-i originate from Rust-side `Arc<T>` allocations; a Rust-side investigation using these tools could complement what Instruments observes from the Swift side.

### M-iv — Allocation patterns, GC, heap dumps

#### Garbage collection

**iOS does not have a garbage collector.** Swift and Objective-C use Automatic Reference Counting (ARC) — a deterministic, compile-time-inserted reference counting mechanism. The compiler inserts `retain` / `release` calls at object boundaries; an object is deallocated immediately when its retain count drops to zero. There is no background GC thread, no stop-the-world pauses, and no GC frequency to measure.

The corresponding question for this codebase is therefore: **how often are objects allocated and deallocated**, and **are there deep allocation patterns or retain cycles** that indicate suboptimal memory management. This is addressed below.

#### Allocation patterns observed during S1 cold start

Top allocation sites by cumulative bytes, from Allocations Call Tree (Invert Call Tree + Hide System Libraries enabled). Counts are calls during the 30-second window, reproduced across all three runs within ±2%:

| Symbol | Library | Bytes Used (Run 1) | Call Count | Pattern observation |
|--------|---------|---------------------|------------|----------------------|
| `std::sys::thread::unix::Thread::new` | BitwardenSdk (Rust) | 22.52 MB (37.7%) | 11 | Native thread spawn; each thread reserves ~2 MB stack VM. Eager `rayon` thread-pool initialization. |
| `main` | Bitwarden | 18.67 MB (31.3%) | 103,585 | Aggregate of all Swift heap allocations attributable to main. Baseline. |
| `FontConvertible.registerIfNeeded()` (inlined) | BitwardenResources | 1.58 MB | 11,612 | **Deterministic count across runs (11,612 / 11,613 / 11,612).** Suspected over-invocation pattern — likely registered per-view instead of once at launch. |
| `closure #1 in PositionObservingView.body.getter` | BitwardenShared | 1.21 MB | 4,270 | SwiftUI body re-evaluation. ~142 evaluations/second sustained — possible missing `Equatable` conformance allowing parent re-renders to cascade. |
| `closure #1 in FetchedResultsSubscription.init(...)` | BitwardenKit | 808.81 KB | 1,790 | CoreData fetched-results subscriptions; ratio to vault size (150 items) is ~12:1, suggesting multiple subscriptions per item. |
| `UINavigationController.replace<A>(_:animated:)` | BitwardenKit | 314.42 KB | 1,878 | Navigation stack mutations during cold start; high count for a path with no user navigation yet. |
| `DataStore.init(errorReporter:storeType:)` | BitwardenShared | 247.03 KB | 898 | DataStore re-instantiation. **Approach-2-related caveat: this audit modified `DataStore.swift` for sandbox Documents bypass; re-instantiation count must be validated against upstream before being reported as a finding (see findings-log.md notes).** |
| `AuthenticatorBridgeDataStore.init(errorReporter:groupIdentifier:storeType:)` | AuthenticatorBridgeKit | 216.22 KB | 424 | Same pattern as DataStore; same Approach-2 caveat. |
| `BitwardenTabBarController.setNavigators<A>(_:)` | BitwardenShared | 210.95 KB | 1,424 | Tab bar navigator setup; high call count under inspection. |
| `RootViewController.childViewController.didset` | BitwardenKit | 125.38 KB | 1,225 | View-controller hierarchy mutations during launch. |
| `static UI.applyDefaultAppearances()` | BitwardenKit | 150.16 KB | 1,019 | Appearance proxy configuration; called once per UIKit object instance? |
| `specialized SceneDelegate.scene(_:willConnectTo:options:)` | Bitwarden | 113.56 KB | 950 | SceneDelegate initialization. |
| `__swift_instantiateConcreteTypeFromMangledNameV2` | Bitwarden | 270.08 KB | 56 | Swift runtime type instantiation; expected during cold start. |

#### Heap dumps

Instruments captured snapshots are functionally equivalent to heap dumps and are visible in the `summary.png` and `calltree.png` screenshots. No standalone `vmmap` or `malloc_history` post-mortem dump was generated for this scenario, as the Instruments deferred-mode traces (saved as `.trace` files; not committed to repo due to size) provide superset information.

#### Deep allocation patterns of note

1. **Rust thread-pool eager initialization** (22.52 MB / 37% of footprint): the Bitwarden Rust SDK spawns ~11 native threads at startup via `rayon`, each reserving the default ~2 MB stack. This is the largest single contributor to total VM footprint during S1. Possible micro-optimization: lazy thread-pool initialization or pool size tuning for mobile target.

2. **High-count small-allocation patterns**: `Malloc 32 Bytes` (24,530 / 766 KiB) and `CFString (immutable)` (13,142 / 610 KiB) both indicate intensive small-object churn during cold start. Cross-referenced with the Call Tree, this aligns with `FontConvertible.registerIfNeeded` (11,612 calls) and `PositionObservingView.body.getter` (4,270 calls) — strongly suggesting these two paths drive the churn.

3. **Determinism**: most call counts vary by less than 1% across runs (e.g., `FontConvertible.registerIfNeeded`: 11,612 / 11,613 / 11,612). This level of determinism indicates these are not race-driven; they are structural code paths called the same number of times every cold start.

**Screenshots:** `s1_allocations_run{1,2,3}_calltree.png`.

---

## S1 — Threading

*Status: COMPLETE — Animation Hitches template with Time Profiler, Thread State Trace, Thermal State, and Hangs sub-instruments across 3 runs of 30s each. Screenshots: `audit/profiling/screenshots/s1_threading/` (18 PNGs total, 6 per run).*

### Cross-run summary

| Metric | Run 1 | Run 2 | Run 3 | Notes |
|---|---|---|---|---|
| Duration | 30.620 s | 30.620 s | 30.585 s | Stop-after-30s honored |
| Bitwarden PID | 70538 | 79206 | 76119 | Fresh PID per run (terminate + relaunch) |
| Bitwarden CPU total (Weight) | 40.45 s | 14.62 s | 18.76 s | Cumulative across all Bitwarden threads |
| Bitwarden context switches | 18,727 | 7,661 | 7,640 | |
| All-process context switches | 23,880 | 12,754 | 12,854 | |
| Thread-state events (Bitwarden) | 16,907 | 9,502 | 8,990 | Running + Runnable + Blocked + Wait + Idle + Interrupted + Preempted |
| Bitwarden total thread residency | 4.39 min | 30.56 s | 1.95 min | Sum across all threads; Run 1 had concurrent thread work |
| Hangs count | 1 | 3 | 5 | Bitwarden process only |
| Max hang duration | 489.10 ms | 520.68 ms | 630.88 ms | Threshold for "Hang" classification is 500 ms |
| Thermal state | Nominal | Nominal | Nominal | No throttling in any run |

**Note on variance:** Unlike the Allocations runs (coefficient of variation < 1.5%), Threading metrics show much higher run-to-run variance (CPU total CV ≈ 50%). The first run, executed immediately after a `xcrun simctl terminate` of the app, carries a heavier cost from simulator-level cache cold state (filesystem snapshots, daemon warmup, WindowServer composition state). Subsequent runs benefit from warm caches even though the app process itself is fresh. This is a property of the measurement environment, not the app — and it is a methodological finding worth carrying into S2-S4 (see F-RT-12).

### T-i — Where and how are threads created? Async/await usage observed

From the Allocations Call Tree (S1 M-iv) and Time Profiler Heaviest Stack Trace (Run 1: 40.45 s aggregate weight on `Bitwarden (70538)` → `main`), the following thread-creation evidence is captured:

| Source | Mechanism | Evidence |
|--------|-----------|----------|
| Bitwarden Rust SDK (`rayon` thread pool) | `std::sys::thread::unix::Thread::new` (POSIX pthread spawn from Rust) | 11 native threads observed in Allocations |
| GCD / Dispatch workers | `_dispatch_worker_thread2`, `_dispatch_root_queue_drain` | Visible in stack traces |
| Swift Concurrency | `swift::runJobInEstablishedExecutor`, `swift_job_runImpl` | Visible in stack traces |
| pthread workqueue | `start_wqthread`, `_pthread_wqthread` | Visible in leak stack traces |
| CoreData publishers | `DataStore.cipherPublisher`, `DataStore.fetchAllOrganizations` | Time Profiler Run 1: top Bitwarden self-weight symbols |
| SwiftUI body re-evaluation | `closure #1 in PositionObservingView.body.getter`, `closure #1 in SearchableVaultListView.search.getter` | Time Profiler Run 1: 35 ms self-weight on body.getter alone |

Async/await is used extensively. The Heaviest Stack Trace consistently shows `Bitwarden → main` as root for the bulk of CPU, with parallel work on Rust threads via UniFFI's `RustFuture` exposed to Swift. Across the 3 runs, between 7,640 and 18,727 context switches occurred inside the Bitwarden process in 30 s — i.e., roughly 254–624 context switches per second — indicating heavy concurrent activity throughout the cold-start window.

### T-ii — Possible locks on main thread

The Heaviest Stack Trace by Weight in Time Profiler points to `main` (Bitwarden) as the heaviest single symbol in all 3 runs: **135 ms (Run 1), 85 ms (Run 2), 150 ms (Run 3)** of self-weight on `main`. This is significant because work that lands on `main` directly blocks frame presentation.

Concrete suspicions of main-thread work during cold start (Bitwarden self-weight, system libraries hidden, call tree inverted):

| Symbol | Self-weight (representative) | Run | Concern |
|---|---|---|---|
| `sha2::sha256::compress256` (Rust SDK, BitwardenSdk_PackageProduct) | 35 ms | 1 | Crypto compression on a thread that participates in cold start. See F-RT-09. |
| `AccelerateCrypto_SHA256_compress` (com.apple.kec.corecrypto) | 15 ms | 2 | Apple's accelerated SHA-256 also active concurrently. See F-RT-09. |
| `closure #1 in PositionObservingView.body.getter` (BitwardenShared) | 35 ms | 1 | SwiftUI body re-evaluation; consistent with M-iv observation of 4,270 body evaluations during S1. |
| `DataStore.cipherPublisher(userId:)` (BitwardenShared) | 10 ms | 1 | CoreData publisher emitting on main; aligns with M-i observation of `_NSMemoryStorePredicateRemapper` leaks. |
| `FontConvertible.register()` (BitwardenResources, INLINED) | 10 ms | 1 | Aligns with the 11,612 over-invocation finding (F-RT-03). |
| `static UI.applyDefaultAppearances()` (BitwardenKit) | 10 ms | 3 | UIAppearance configuration on main during scene setup. |
| `RootViewController.childViewController.didset` (BitwardenKit) | 15 ms | 3 | View-controller hierarchy mutation on main. |
| `specialized SceneDelegate.scene(_:willConnectTo:options:)` (Bitwarden) | 10 ms | 3 | Scene attachment work on main. |
| `ObservableObject.objectWillChange.getter` (BitwardenKit) | 5 ms | 2 | Each `objectWillChange` triggers downstream view invalidation; multiple emissions per second observed. |

The 489.10 ms microhang (Run 1) and the larger 520.68 ms and 630.88 ms hangs (Runs 2-3) are direct evidence of main-thread blocking exceeding the 500 ms "Hang" threshold. These align temporally with the splash-to-vault transition (t≈0:08-0:12 across runs, visible in the alltracks screenshots).

### T-iii — How multithreading affects performance

**Positive contributions of multithreading:**
- Heavy SDK and crypto work runs on Rust-spawned threads (rayon pool, 11 native threads, ~22.5 MB stack VM as documented in M-iv). Without this offloading, cold start would be substantially worse.
- Thermal state is `Nominal` for all 30 s in every run, meaning the simulator never throttles. Per-core utilization stays below sustained thresholds.
- Context-switch density (254-624/s inside Bitwarden) demonstrates the scheduler is actively distributing work across threads; the app is not single-threaded-bound.

**Negative contributions / observed bottlenecks:**
- **Progressive hang degradation across consecutive runs without simulator reset**: 1 → 3 → 5 hangs as Runs 1 → 2 → 3 progressed. The app process was terminated and freshly launched between each run, but the simulator (and macOS host caches, WindowServer state, etc.) was not reset. This makes the hang count a property of the test environment as much as of the app. See F-RT-08 for the methodological consequence.
- **Two crypto pipelines visible in cold start**: both `AccelerateCrypto_SHA256_compress` (Apple's accelerated path) and `sha2::sha256::compress256` (Rust SDK pure-software path) appear in Bitwarden self-weight across runs. This suggests duplicated cryptographic work on cold start — once via the SDK during vault unlock/decrypt, once via Apple's framework presumably for keychain/biometric verification. See F-RT-09.
- **SwiftUI observable churn during render**: `Store.state.setter`, `ObservableObject.objectWillChange.getter`, `PositionObservingView.body.getter`, and `SearchableVaultListView.search.getter` are all present in top self-weight across runs. Combined with the M-iv finding of 4,270 body re-evaluations in 30 s (~142/s) for `PositionObservingView` alone, this indicates the view tree is being invalidated more frequently than the visible rendering demands. See F-RT-10.
- **Scene/Navigation setup accumulates ~30-40 ms of main-thread work in Run 3**: `SceneDelegate.scene(_:willConnectTo:options:)` 10 ms + `SceneDelegate.buildSplashWindow` 5 ms + `BitwardenTabBarController.setNavigators` 5 ms + `ViewLoggingNavigationController.viewDidLoad` 5 ms + `RootViewController.childViewController.didset` 15 ms. None of these are individually large, but they are all on main and they all happen sequentially. See F-RT-11.

### Cold-start latency reference (from App Launch instrument)

| Run | Cold-start time |
|-----|------------------|
| 1   | (per s1_app_launch screenshots) |
| 2   | (per s1_app_launch screenshots) |
| 3   | (per s1_app_launch screenshots) |
| **Mean ± SD** | **15.67s ± 1.15s** |

**Screenshots:** `audit/profiling/screenshots/s1_app_launch/` (12 PNGs across 3 runs), `audit/profiling/screenshots/s1_threading/` (18 PNGs across 3 runs).

---
# Scenario S2 — Vault scroll + live search

## S2 — Methodology

Three independent runs were executed for Allocations + Leaks (45 seconds each, deferred mode, leak checks every 10s) and one run for the Animation Hitches template (45 seconds, Time Profiler + Thread State Trace + Thermal State + Hangs; the Hitches sub-instrument was deleted from the template since it is not supported on the iOS Simulator on Apple Silicon — same limitation documented in S1). The simulator was not reset between runs (consistent with F-RT-08 methodological note). The vault was preloaded with ~150 items via manual Sync now (per F-RT-06 workaround).

The S2 script (see "Scenario scripts" section above) replaces the idle phase of S1 with active interaction: slow scroll top→bottom, fast flick bottom→top, tap search field, type "test" character by character, clear search, slow scroll bottom→top, idle. Total active interaction time: ~30 seconds within the 45-second recording window. The first ~15 seconds are consumed by cold start + master-password unlock + vault load, providing comparable baseline to S1.

**Instruments used for S2:**
- Allocations + Leaks (3 runs × 45 s) — completed
- Animation Hitches template, Hitches sub-instrument removed (1 run × 45 s) — completed
- Color Blended Layers debug overlay (3 surfaces captured during the active scroll/search interaction) — completed
- Metal System Trace: not re-attempted (same Apple Silicon Simulator limitation documented in S1 applies)
- Energy Log: not attempted (same physical-device-only limitation documented in S1 applies)

**Raw screenshots:** `audit/profiling/screenshots/s2_allocations/` (12 PNGs, 3 runs × 4 views), `audit/profiling/screenshots/s2_threading/` (5 PNGs, 1 run, Hitches excluded per Simulator limitation), `audit/profiling/screenshots/s2_overdrawing/` (3 PNGs, 3 surfaces). Total: 20 PNGs documenting S2.

---

## S2 — GPU rendering analysis

*Status: COMPLETE — data from Animation Hitches template's Display sub-instrument (1 run, 45.577 s). Same Simulator limitations apply as in S1.*

### Frame rate metrics

The Display sub-instrument captured Average Frame Time across the 45.577 s window. Two distinct phases are visible in the timeline:

- **Phase 1 (t ≈ 0–14 s, cold start)**: low Average Frame Time bars with occasional spikes during splash-to-vault transition. Two tall spikes near t≈4 s and t≈12 s coincide with surface composition events (Surface 7 → Surface 8 transition; see Display 1 track).
- **Phase 2 (t ≈ 14–45 s, active interaction)**: higher density of Average Frame Time bars throughout, with a particularly tall spike near t≈15 s (start of slow scroll). Bars remain present but moderate during the scroll/search interaction window (t≈15–35 s), then taper during idle (t≈35–45 s).

Surface composition during S2 follows the pattern: Surface 7 (initial) → Surface 8 (post-cold-start, brief) → Surface 7 (mid) → Surface 8 (brief) → Surface 9 (sustained, t≈12–32 s) → Surface 8/7 alternation (t≈32–45 s). The sustained Surface 9 segment overlaps temporally with the cluster of hangs detected during the search interaction (see Threading T-iii), consistent with S1 observation: when the render server holds a single surface composition for an extended interval, the main thread is doing heavy work that delays present.

VSync alignment (red tick pattern in Display 1 track) remains regular throughout, indicating the display pipeline cadence itself is not compromised; the issue is upstream main-thread work that prevents frames from being committed in time.

### Problems and strengths

**Strengths:**
- Thermal state remains `Nominal` for all 45.577 seconds (no throttling during active interaction; same as S1).
- VSync cadence regular throughout; no extended display-pipeline stalls.
- Average Frame Time bars during pure idle (t≈35–45 s) drop to near-baseline, indicating steady-state rendering remains well within budget when no user input arrives.

**Problems:**
- The eight hangs detected during S2 (see Threading T-iii) cluster precisely in the active-interaction window (t≈14–25 s), and the largest hang (1.68 s — sub-second user-perceptible) occurs during the search-typing phase. This is empirical evidence that the search pipeline does not isolate enough work off the main thread.
- The Hitches sub-instrument remains unavailable on Simulator. The same scope limitation documented in S1 applies.

### GPU consumption — scope limitation

Same as S1. Not re-tested. The Animation Hitches template's Display + Time Profiler + Thread State Trace sub-instruments cover the rendering-time question (rubric item 3.a) for S2 but do not give GPU utilization percentages.

### Power consumption — scope limitation

Same as S1. Not re-tested. Thermal State reports `Nominal` for all 45.577 s of the S2 run.

---

## S2 — Overdrawing analysis

### Method
Simulator → Debug → Color Blended Layers toggle enabled during the active S2 interaction. Three surfaces captured at distinct interaction states: vault scrolled mid-position with modal navigation drill-in, search field active with matching results, and search field active with no-match empty state.

### Captured surfaces

| Screenshot | Surface | File |
|------------|---------|------|
| Vault drilled-into Logins category (mid-scroll) | Logins list with vault stack underneath | `s2_overdrawing_vault_scrolled.png` |
| Search active, results filtered ("Test") | Vault search with 7 matching items | `s2_overdrawing_search_results.png` |
| Search active, no matches ("xyz123") | Empty-state placeholder | `s2_overdrawing_search_empty.png` |

All screenshots in `audit/profiling/screenshots/s2_overdrawing/`.

### Observations by surface

**Vault drilled-into Logins category, mid-scroll (`s2_overdrawing_vault_scrolled.png`):**
- **Most severe overdrawing observed in the entire audit so far.** Nearly the full screen renders saturated red — not the light pink/red of acceptable two-layer composition observed in S1, but a continuous saturated red across header, search field, list rows, dividers, and tab bar.
- Only two thin vertical margin strips at the left and right edges remain green (likely outside the modal's scrollable content area).
- Header "Logins" + back button: opaque red background → 3+ layers stacked.
- Search field placeholder "Search": light pink/red, slightly less saturated than the body content but still overdrawing.
- List items (Best Buy through Costco Visa visible): each row's background is intense red, with icons and text rendering an even deeper red on top → 4 layers in the row regions.
- FAB "+" floating button: saturated red, correctly opaque.
- Tab bar at bottom: My Vault tab in deepest red (correct opaque selection state), but the rest of the tab bar continues the overdrawing pattern.
- **Inferred composition stack**: vault root list (Layer 1) + modal "Logins" drill-in container (Layer 2) + scroll content background (Layer 3) + row backgrounds (Layer 4) + row content (Layer 5). The compositor renders all five on every frame.

**Search active, results filtered (`s2_overdrawing_search_results.png`):**
- Background between rows: uniform green → not overdrawing in empty inter-row space.
- Search field with "Test" text: orange background (2–3 layers stacked) → search overlay composes with vault root.
- Each result row (Adidas, Audit Test Identity, Best Buy, Cloudflare, Google Cloud, Tumblr, Yahoo Mail):
  - App icon container: orange/red (2–3 layers) — the icon's circular container blends with row + screen background.
  - Item title (in red, opaque): correct.
  - Username/email (red/orange, 2 layers): name is opaque but field background blends.
  - "..." overflow menu: red opaque (correct).
- Divider lines between rows: tint of green → lines drawn with alpha over background.
- Tab bar identical to S1 pattern: selected tab strong red, inactive pink.

**Search active, no matches (`s2_overdrawing_search_empty.png`):**
- Background **100% green** — worst overdrawing baseline observed for empty space (S1 had similar but less saturated). Every pixel of the empty placeholder area composites with the vault list root behind it.
- Status bar opaque (correct).
- Search field with "xyz123": orange field background (2–3 layers).
- Clear button (X inside field) and close button (X outside field): green container with red icon → icon-on-translucent-button composition.
- **Magnifying glass empty-state icon: orange** → the icon's square container is 3-layer composed (icon glyph + container + screen background).
- "There are no items that match the search" label: orange/red text on green background → text rendered correctly but the area surrounding it (the entire screen) is unnecessarily composited.
- Tab bar same as Search results screen.

### Problems

| Problem | Surface(s) | Likely cause | Performance implication |
|---------|------------|---------------|--------------------------|
| Modal drill-in stacks over vault root | `s2_overdrawing_vault_scrolled.png` | "Logins" category presented as modal navigation on top of vault list without tearing down the underlying view; both layers remain in compositor tree, multiplied by row content layers | Per-frame compositing cost is highest of all surfaces audited; each scroll tick recomposites 4-5 layers across full screen |
| Search overlay does not opaque-fill background | Both search screens | Search results / empty state rendered as overlay on vault list, not as a screen replacement | GPU continues compositing vault list rows behind the search results even when no vault content is visible to the user |
| Empty-state placeholder still renders over vault root | `s2_overdrawing_search_empty.png` | No `.background(Color.systemBackground).ignoresSafeArea()` or equivalent opaque fill on the empty state | When user types a non-matching query, GPU spends time compositing layers that are 100% obscured by the empty-state view |
| App-icon row containers blend instead of being opaque | `s2_overdrawing_search_results.png` | Icon containers use rounded-rect shape with implicit alpha background instead of opaque fill | Repeated cost per row, accumulates with list length; in a 150-item vault during scroll this is a measurable repeated cost |
| Tab bar overdrawing pattern unchanged from S1 | All three S2 surfaces | Material blur + capsule + selection highlight + icon stacking persists across all screens | ~15% of screen area continuously overdrawing in every scenario; aggravated during tab transitions |

### Strengths

- Interactive controls (FAB, search field input area, clear/close buttons, tab bar icons) consistently render as opaque single-layer surfaces — touch targets have correct rendering attributes (same as S1).
- App-icon assets (Cloudflare, Google, Tumblr, Yahoo) render their bitmaps opaquely; only the surrounding container blends (consistent with S1 finding).
- Divider lines between search result rows are rendered with alpha (intentional design choice to maintain visual continuity with row content); this is acceptable use of blending for a thin pixel band, not a per-frame cost concern.
- Search field text input itself (the typed characters) renders opaque red over the field background — text legibility is preserved despite overdrawing of the field's container.
- During the empty-state screen, the magnifying-glass icon and label are positioned and rendered correctly even on top of the overdrawing background; user perception is not impacted, only GPU efficiency.

---

## S2 — Memory management

### M-i — Memory leaks: which, where?

Three Leaks-instrument snapshots per run (every 10 seconds within the 45-second window).

| Run | Leaked allocations (approx) | Total bytes leaked (approx) | Snapshot pattern | Primary responsible library |
|-----|------------------------------|------------------------------|--------------------|------------------------------|
| 1   | 17                          | ~1.39 KB                     | First snapshot leaks detected (green observed momentarily then red), second & third snapshots red | `BitwardenSdk_4641...` (Rust SDK via UniFFI) + 1× CoreData |
| 2   | 17                          | ~1.40 KB                     | All three snapshots red | `BitwardenSdk_4641...` (Rust SDK via UniFFI) |
| 3   | 18                          | ~1.45 KB                     | All three snapshots red | `BitwardenSdk_4641...` (Rust SDK via UniFFI) + 1× CoreData |

**Leak categorization by source (consistent across S1 and S2):**

| Source | Per-run count (approx) | Sizes observed | Stack trace signature |
|--------|------------------------|----------------|------------------------|
| Bitwarden Rust SDK via UniFFI FFI boundary | 16-18 | 64 bytes (most), 128 bytes (some) | `alloc::alloc::exchange_malloc` → `alloc::sync::Arc<T>::new` → `uniffi_core::ffi::rustfuture::future` → `ffi_bitwarden_uniffi_rust_future_po...` → `@nonobjc ffi_bitwarden_uniffi_rust...` → `swift::runJobInEstablishedExecutor` |
| CoreData internal | 0–1 per run | 16 bytes | `+[_NSMemoryStorePredicateRemapper defaultInstance]` |

**Interpretation:** F-RT-09 is **deterministically reproduced in S2** with identical stack-trace signature to S1 (Arc<T> via UniFFI rust_future via Swift executor job). The volume of leaks (16–18 per run) is comparable to S1 (17–19 per run) despite the addition of search and scroll interaction — meaning the Arc leak pattern is **driven by SDK initialization** (during the cold-start phase of the 45 s window), not by per-interaction work. The CoreData `_NSMemoryStorePredicateRemapper` leak appears intermittently across runs (visible in runs 1 and 3, not visible in run 2's captured screenshot — may be outside viewport).

**Zero new Swift-side leaks detected during the S2 interaction phase**: no scroll-induced or search-induced retain cycles observed. The vault list, search field, and item-detail navigation paths do not leak Swift objects under the S2 script.

**Screenshots:** `s2_allocations_run{1,2,3}_leaks.png`.

### M-ii — RAM consumption across the scenario

#### Active-interaction memory growth pattern

All three S2 runs share a consistent shape: rapid growth during the cold-start phase (t≈0–14 s, same shape as S1), brief plateau (t≈14–15 s, idle just before scroll), then continued sustained growth during the scroll+search interaction window (t≈15–35 s), with a final plateau during idle (t≈35–45 s). Unlike S1, the heap does not fully plateau by run end — it continues climbing at a reduced rate, indicating allocations during the interaction outpace deallocations.

#### Per-run memory footprint at end of 45-second window

| Run | All Heap Persistent | # Persistent | # Transient | Total Bytes (cumulative) | # Total |
|-----|----------------------|--------------|--------------|---------------------------|---------|
| 1   | 32.79 MiB           | 241,315      | 3,418,118    | 532.17 MiB                | 3,659,433 |
| 2   | 34.00 MiB           | 251,530      | 3,819,408    | 586.35 MiB                | 4,070,938 |
| 3   | 33.78 MiB           | 249,936      | 3,845,041    | 591.46 MiB                | 4,094,977 |
| **Mean ± SD** | **33.52 MiB ± 0.65** | **247,594 ± 5,520** | **3,694,189 ± 238,749** | **569.99 MiB ± 32.71** | **3,941,783 ± 244,884** |
| **Coefficient of variation** | 1.93% | 2.23% | 6.46% | 5.74% | 6.21% |

**Interpretation:**
- All Heap Persistent CV ≈ 1.93% across S2 runs: reproducibility is high, consistent with S1 (CV ≈ 1.15%). Memory measurements remain the most stable signal in the audit.
- Transient allocation count CV (6.46%) is higher than S1 (3.00%) because scroll velocity and search-typing cadence introduce small per-run variation in the number of intermediate allocations created during cell recycling, search-result diffing, and text-field state updates. This is expected and does not undermine the structural conclusions.
- The all-heap persistent footprint at the end of S2 (~33.5 MiB) is **lower** than S1's heap-allocations-persistent footprint (~23.5 MiB heap only, or ~60 MiB heap + anonymous VM). The S2 measurement reports a different metric grouping (heap only, not heap+anonymous VM), so a direct apples-to-apples comparison would require re-reading the same Statistics filter; the order of magnitude is consistent.
- **Total cumulative bytes (~570 MiB) vs persistent (~33.5 MiB) → ratio ~17:1**. Compared to S1's 5:1 ratio, S2 generates substantially more transient allocation churn per persistent byte retained. This is the expected signature of an interactive scenario: many short-lived objects (gesture recognizers, scroll-state snapshots, text-edit deltas, search-filter intermediates) are created and freed within milliseconds.

#### Top resident-memory categories at run end (Run 3 representative)

| Category | Persistent Bytes | # Persistent | Notes |
|----------|------------------|--------------|-------|
| All Heap Allocations | 33.78 MiB | 249,936 | Aggregate (filter row) |
| Malloc 16.00 KiB | 3.27 MiB | 209 | Generic heap blocks (same category dominant in S1) |
| UnknownObjectType | 1.80 MiB | 19,445 | Custom Swift types without resolved symbol; +290% from S1 (7,457). Likely scroll-related vault row models. |
| CFString (store) | 1.17 MiB | 6,290 | String storage backing |
| Malloc 32 Bytes | 913.69 KiB | 29,238 | High count → many small allocations, consistent with S1 pattern |
| Malloc 8.00 KiB | 904.00 KiB | 113 | Generic heap |
| Malloc 2.00 KiB | 814.00 KiB | 407 | Generic heap |
| Malloc 128 Bytes | 789.62 KiB | 6,317 | Generic heap |
| _DictionaryStorage<Obj...> | 741.75 KiB | 681 | Swift dictionary storage; +30% from S1 |
| CFString (immutable) | 719.19 KiB | 15,185 | Immutable strings |
| Malloc 80 Bytes | 612.50 KiB | 7,840 | Generic heap |
| Malloc 256 Bytes | 608.00 KiB | 2,432 | Generic heap |
| CFData | 589.22 KiB | 620 | CFData buffers |
| Swift.__StringStorage | 575.64 KiB | 3,914 | Swift String storage |
| Malloc 64 Bytes | 353.56 KiB | 5,657 | Generic heap |

**Observations on memory-consumer composition (S2 vs S1):**
- **UnknownObjectType count grew from 7,457 (S1) to 19,445 (S2) — a 161% increase** for the same vault size. Strongly suggests row cell view-models or search-filter intermediates are being created per row during scroll without being deallocated.
- **_DictionaryStorage** allocations grew from S1; likely indicative of search-state dictionaries or scroll-position observers.
- Generic Malloc bins (16 KiB, 8 KiB, 2 KiB) sizes consistent with S1 — backing storage for SwiftUI/UIKit infrastructure.
- No category showed unexpected explosive growth → no clear "leak by accumulation" pattern in any one allocator class.

#### Memory under different use scenarios

S2 vs S1 differential: ~10 MiB additional growth during scroll+search interaction (33.5 MiB final vs 23.5 MiB heap-only S1 final), with disproportionate growth concentrated in UnknownObjectType (+161%). This is the cost of interactive UI work. S3 and S4 will measure save-flow and navigation-cycle memory respectively.

**Screenshots:** `s2_allocations_run{1,2,3}_alltracks.png`, `s2_allocations_run{1,2,3}_summary.png`.

### M-iii — Libraries for leak management

Cross-reference to S1 M-iii. The library landscape (Instruments Leaks/Allocations, MetricKit, os_signpost, MallocStackLogging, Memory Graph Debugger, ASan, Rust-side Miri/tracing) does not vary by scenario. Same recommendations apply.

### M-iv — Allocation patterns, GC, heap dumps

#### Garbage collection

Same as S1: iOS does not have a GC. ARC is deterministic. The relevant question is how often objects allocate and deallocate, and whether interactive scenarios surface retain cycles. None observed in S2 Swift code paths.

#### Allocation patterns observed during S2

Top allocation sites by cumulative bytes, from Allocations Call Tree (Run 3 representative; Runs 1-2 within ±5% on top symbols):

| Symbol | Library | Bytes Used (Run 3) | Call Count (Run 3) | Cross-run consistency |
|--------|---------|---------------------|---------------------|------------------------|
| `main` | Bitwarden | 22.78 MB (71.9%) | 181,284 | ±1.9% across 3 runs |
| `FontConvertible.registerIfNeeded()` (inlined) | BitwardenResources | 1.58 MB | **11,609** | **Identical across 3 runs (0% variance)** |
| `closure #1 in PositionObservingView.body.getter` | BitwardenShared | 1.23 MB | 4,208 | 4,137 / 4,208 / 4,204 across runs (±0.9%) |
| `closure #1 in FetchedResultsSubscription.init(...)` | BitwardenKit | 563.66 KB | 1,880 | ~consistent |
| `UINavigationController.replace<A>(_:animated:)` | BitwardenKit | 322.30 KB | 1,855 | ~consistent |
| `specialized _ContiguousArrayBuffer._consumeAndCreateNew(...)` | BitwardenShared | 323.92 KB | 18 | Array growth during scroll |
| `CipherDetailsResponseModel.init(from:)` | BitwardenShared | 230.34 KB | 1,354 | ~consistent |
| `RootViewController.childViewController.didset` | BitwardenKit | 187.48 KB | 1,431 | ~consistent |
| `BitwardenTabBarController.setNavigators<A>(_:)` | BitwardenShared | 174.84 KB | 1,375 | ~consistent |
| `@nonobjc UIImage.__allocating_init(named:in:compatibleWith:)` (inlined) | BitwardenResources | 157.50 KB | 1,750 | UIImage instantiation for vault row icons |
| `specialized FontConvertible.register()` | BitwardenResources | 155.58 KB | 85 | ~consistent |
| `static UI.applyDefaultAppearances()` | BitwardenKit | 149.00 KB | 1,016 | ~consistent |
| `alloc::raw_vec::RawVecInner<...>::try_allocate_in::...` | BitwardenSdk | 140.00 KB | 98 | Rust vector allocations |
| `@nonobjc NSManagedObjectModel.init(contentsOf:)` | BitwardenShared | 125.73 KB | 343 | CoreData model loads |
| `TabCoordinator.start()` | BitwardenShared | 114.95 KB | 978 | ~consistent |
| `DataStore.init(errorReporter:storeType:)` | BitwardenShared | 114.41 KB | 891 | Approach-2 caveat from S1 still applies |
| `_RNvCs5QKde7ScR4H_7__rustc14__rust_realloc` | BitwardenSdk | 104.88 KB | 837 | Rust reallocations |

**Key cross-run determinism finding (reproduction of F-RT-03):**
- `FontConvertible.registerIfNeeded()`: **11,609 calls across all 3 S2 runs (CV = 0%)**. Identical to S1 (11,612 / 11,613 / 11,612). The over-invocation pattern is fully structural, fully deterministic, and reproduces across scenarios. The count is not affected by user interaction (scroll, search) because the registrations happen at view-load time, before interaction begins.

**Key cross-run consistency finding (reproduction of F-RT-10):**
- `closure #1 in PositionObservingView.body.getter`: 4,137 / 4,208 / 4,204 calls across S2 runs (mean 4,183, CV 0.85%). Mean is **lower than S1** (4,270), which is surprising for an interactive scenario. Hypothesis: the scroll interaction in S2 short-circuits some of the during-cold-start body re-evaluation cascade because the SwiftUI render path receives different state-change signals during active scroll versus pure idle. Worth flagging as observation; does not change the conclusion that the over-evaluation pattern (~93 evals/s) is structural.

**Heap dumps:** Functionally equivalent to S1's; Instruments deferred traces (.trace files, not committed) provide superset information.

#### Deep allocation patterns of note (S2-specific)

1. **UnknownObjectType growth (+161% vs S1)**: 19,445 persistent allocations classified under UnknownObjectType in S2 vs 7,457 in S1. Suggests scroll/search creates Swift types (likely row view-models or filter intermediates) that Instruments cannot symbolicate. Recommendation: enable Swift type symbolication or inspect via Memory Graph Debugger to identify the class.

2. **CipherDetailsResponseModel instantiation rate (1,354 in S2)**: This count is unexpectedly high for an interactive scenario that does not drill into item details — suggests the search filter or row rendering may be (re-)decoding cipher response models from CoreData each frame instead of using cached decoded values.

3. **Rust SDK allocations (`alloc::raw_vec::RawVecInner` 98 calls, 140 KB)**: Rust-side vector allocations during S2 reflect the SDK doing search-related decryption work. Less than 0.5% of total persistent footprint — Rust SDK memory is not a concern.

4. **Determinism preserved**: even with the interactive workload, top symbols' call counts vary by less than 2% across runs. The audit's measurement methodology continues to surface structural patterns rather than noise.

**Screenshots:** `s2_allocations_run{1,2,3}_calltree.png`.

---

## S2 — Threading

*Status: COMPLETE — Animation Hitches template with Time Profiler + Thread State Trace + Thermal State + Hangs sub-instruments (Hitches removed per Simulator limitation). 1 run × 45.577 s. Screenshots: `audit/profiling/screenshots/s2_threading/` (5 PNGs).*

### Run summary

| Metric | S2 Run 1 | S1 Run 3 (worst) | Δ vs S1 worst |
|---|---|---|---|
| Duration | 45.577 s | 30.585 s | — (different windows) |
| Bitwarden PID | 29462 | 76119 | — |
| Bitwarden CPU total (Weight) | 1.04 s | 18.76 s | Different time windows; per-second rate is comparable |
| Hangs count | **8** | 5 | **+60%** |
| Min hang duration | 181.73 ms | (not separately reported) | — |
| Avg hang duration | **562.40 ms** | (not separately reported) | — |
| Std Dev hang duration | 467.49 ms | 219.24 ms | +113% (more variable) |
| Max hang duration | **1.68 s** | 630.88 ms | **+167%** (sub-second user-perceptible) |
| Hang range | 1.50 s | (not reported) | — |
| Thermal state | Nominal | Nominal | unchanged |
| Thread state transitions (all states) | 30,045 | 8,990 | +234% (mostly accounted for by 45 s vs 30 s + interactive load) |

### T-i — Where and how are threads created? Async/await usage observed

S2 confirms the threading sources observed in S1 (rayon thread pool spawn, GCD workers, Swift Concurrency executor, pthread workqueue, CoreData publishers, SwiftUI body re-evaluation). Heaviest Stack Trace from S2 Time Profiler additionally surfaces:

| Source (S2-specific evidence) | Mechanism | Weight |
|---|---|---|
| `main` (Bitwarden) | App main thread | 782 ms (top consumer) |
| `__CFRunLoopRun` (CoreFoundation) | Main RunLoop event dispatch | 765 ms |
| `__CFRunLoopDoSour...` (CoreFoundation) | RunLoop source handling | 330 ms |
| `_UIUpdateSequenceR...` (UIKitCore) | UIKit display-update sequence | 310 ms |
| `_setupUpdateSeque...` (UIKitCore) | Scene/Navigation setup (confirms F-RT-11) | 210 ms |
| `ViewGraph.updateOu...` (SwiftUICore) | SwiftUI view-graph update | 103 ms |
| `closure #1 in Position...` (BitwardenShared) | SwiftUI PositionObservingView body (confirms F-RT-10) | 46 ms |

The 765 ms on `__CFRunLoopRun` is expected and not a defect — it represents the RunLoop blocking waiting for events. The 310 ms on `_UIUpdateSequenceR...` and 210 ms on `_setupUpdateSeque...` together indicate ~520 ms of UIKit / SwiftUI display-update work concentrated on main during the 45 s window, which directly explains the hang pattern.

### T-ii — Possible locks on main thread

The Time Profiler Call Tree (inverted, Hide System Libraries) surfaces several main-thread-attributed Bitwarden symbols during S2:

| Symbol | Self-weight | Concern |
|---|---|---|
| `main` (Bitwarden) | 692.00 ms (66.7%) | Aggregate of all main-thread Bitwarden work; consistent with S1 |
| `closure #1 in PositionObservingView.body.getter` (BitwardenShared) | 44.00 ms (4.2%) | Reproduces F-RT-10 in S2 |
| `_$LT$tracing_oslog..logger..OsLogger$u20$as$u20$tracing_subscriber..layer..Lay` (BitwardenSdk) | 16.00 ms (1.5%) | Rust SDK logging subscriber on main thread |
| `std::sys::pal::unix::sync::mutex::Mutex::lock` (BitwardenSdk) | 11.00 ms (1.1%) | **Mutex lock observed on main thread** (new finding F-RT-13) |
| `FontConvertible.register()` (BitwardenResources, inlined) | 7.00 ms (0.7%) | Reproduces F-RT-03 in S2 |
| `__swift_instantiateConcreteTypeFromMangledNameV2` (BitwardenShared / Bitwarden) | 6.00 / 5.00 ms | Swift runtime type instantiation |
| `__swift_instantiateConcreteTypeFromMangledNameAbstractV2` (BitwardenShared) | 5.00 ms | Same family |
| `specialized SceneDelegate.scene(_:willConnectTo:options:)` (Bitwarden) | 4.00 ms | Reproduces F-RT-11 in S2 |
| `rayon_core::registry::WorkerThread::wait_until_cold` (BitwardenSdk) | 4.00 ms | **Rayon worker waiting (new finding F-RT-13)** |
| `static UI.applyDefaultAppearances()` (BitwardenKit) | 2.00 ms | Same as S1 |
| `@nonobjc AVCaptureMetadataOutput.init()` (BitwardenShared, inlined) | 2.00 ms | Camera output init (likely Authenticator-related preload) |
| `Store.state.getter` (BitwardenKit) | 2.00 ms | Store state access on main; minor |
| `ShakeWindow.init(windowScene:onShakeDetected:)` (BitwardenKit) | 2.00 ms | Shake-to-lock feature overhead |
| `protocol witness for ObservableObject.objectWillChange.getter in conformance Store<A, B, C>` (BitwardenKit) | 2.00 ms | Reproduces F-RT-10 ObservableObject pattern |
| `core::ops::function::FnOnce::call_once::h251a9b2e0ee5dfb2` (BitwardenSdk) | 2.00 ms | Rust closure invocation on main |

**New finding (F-RT-13):** Two distinct Rust-side symbols appear in main-thread self-weight that warrant separate tracking from F-RT-09 (Arc leaks):
- `std::sys::pal::unix::sync::mutex::Mutex::lock` (11 ms) — main thread is acquiring a Rust-side mutex synchronously.
- `rayon_core::registry::WorkerThread::wait_until_cold` (4 ms) — main thread is waiting on the rayon worker pool.

These are not memory leaks; they are synchronization primitives. Combined, they indicate the Swift-side calling code is making blocking calls into the Rust SDK that wait for Rust-side worker availability. See F-RT-13.

### T-iii — How multithreading affects performance

**Hang count and severity in S2 vs S1:**

| Metric | S1 Run 1 | S1 Run 2 | S1 Run 3 | S2 Run 1 |
|---|---|---|---|---|
| Hangs | 1 | 3 | 5 | **8** |
| Max hang duration (ms) | 489.10 | 520.68 | 630.88 | **1,680** |

S2 reproduces F-RT-08 (progressive hang degradation) and intensifies it: 8 hangs is +60% over S1's worst run, and the max hang duration of 1.68 s is sub-second user-perceptible — i.e., the user would notice a stall during the search-typing interaction. Average hang duration (562 ms) is roughly the boundary at which hangs cease being "microhangs" and become "perceptible delays" per Apple's classification. Std Dev (467 ms) is roughly 83% of mean, indicating high variability — some hangs are short (181 ms minimum), others are sub-second.

The Hangs track timeline shows:
- 1 hang (orange, "Hang" label) near t ≈ 3 s — cold start cluster, same pattern as S1.
- 7 hangs (blue rectangles) clustered in t ≈ 14–25 s window — this is precisely the search-interaction phase (type "test" → results filter → clear → scroll). The hangs occur during user input handling.

**Thread State Trace (S2 new data):**

| State | Count | Duration | Avg | Max |
|---|---|---|---|---|
| Blocked | 10,162 | 3,719.04 s | 21.96 s | 45.58 s |
| Running | 9,705 | 4.51 s | 464.42 µs | 473.88 ms |
| Runnable | 5,182 | 7.89 s | 1.52 ms | 7.87 s |
| Interrupted | 3,480 | 5.71 ms | 1.64 µs | 11.38 µs |
| Preempted | 946 | 41.70 ms | 44.08 µs | 1.09 ms |
| Unknown | 560 | 417.69 min | 44.75 s | 45.58 s |
| Idle | 10 | 7.60 min | 45.58 s | 45.58 s |

**Interpretation:**
- 30,045 state transitions in 45.58 s ≈ 659 transitions/s — high context-switching density, consistent with concurrent work across many threads.
- **Blocked dominates the state mix** (10,162 transitions / 3,719 s cumulative across all threads). Threads spend most time waiting for I/O, locks, or async completion. This aligns with the F-RT-09 (Arc leaks via UniFFI futures) and F-RT-13 (mutex contention) findings: many threads are blocked on synchronization primitives.
- **Running max 473.88 ms** — a single thread ran continuously for nearly half a second without yielding. When this happens on the main thread, it directly produces a hang.
- **Runnable max 7.87 s** — a thread waited up to 7.87 s to be scheduled. This is consistent with priority inversion or worker starvation (threads ready to run but scheduler choosing others). Combined with the rayon `wait_until_cold` observation, suggests the Rust SDK thread pool is sometimes starved of CPU.

**Thermal:** 100% Nominal for 45.577 s, no throttling. Same caveat as S1 (Simulator does not implement real thermal modeling).

**Net conclusion for S2 Threading:**
- Multithreading benefits are preserved (Rust SDK work is off main, crypto/decryption is parallelized).
- But the boundary between Swift main thread and Rust SDK is a measurable bottleneck: mutex locks and worker waits on main account for 15 ms of self-weight, and they correlate temporally with the 8 hangs in the interaction window.
- F-RT-08 (progressive hang degradation), F-RT-09 (Rust SDK Arc leaks), F-RT-10 (SwiftUI observable churn), F-RT-11 (Scene/Navigation main-thread setup), and F-RT-12 (high run-to-run variance) all reproduce in S2 with the same signatures as S1.
- **F-RT-13 is new in S2**: Rust SDK mutex contention + rayon worker wait observable on main thread.

**Screenshots:** `audit/profiling/screenshots/s2_threading/` (5 PNGs).

---

# Scenario S3 — Open item → Edit → Save

## S3 — Methodology
*Status: PENDING. Network instrument may be added to capture sync round-trip.*

## S3 — GPU rendering analysis
*Status: PENDING*

## S3 — Overdrawing analysis
*Status: PENDING*

## S3 — Memory management
### M-i — Leaks
*Status: PENDING*

### M-ii — RAM consumption
*Status: PENDING. Focus: does memory grow during edit→save cycle, and does it return to baseline?*

### M-iii — Libraries for leak management
*Cross-reference to S1 M-iii.*

### M-iv — Allocation patterns, GC, heap dumps
*Status: PENDING*

## S3 — Threading
### T-i — Thread creation, async usage
*Status: PENDING*

### T-ii — Main-thread locks
*Status: PENDING — save path likely involves CoreData write + sync; both candidates for off-main work.*

### T-iii — Multithreading performance impact
*Status: PENDING*

---

# Scenario S4 — Navigation cycle + Password generation + Copy

## S4 — Methodology
*Status: PENDING. Particular focus on whether allocated memory returns to baseline after each navigation cycle (memory leak detection across repeated cycles).*

## S4 — GPU rendering analysis
*Status: PENDING*

## S4 — Overdrawing analysis
*Status: PENDING*

## S4 — Memory management
### M-i — Leaks
*Status: PENDING. S4 is the primary scenario for leak detection across coordinator/processor lifecycles.*

### M-ii — RAM consumption
*Status: PENDING. Key question: does each navigation cycle (Generator → Vault → Settings → Vault) return memory to a stable baseline, or does memory grow monotonically?*

### M-iii — Libraries for leak management
*Cross-reference to S1 M-iii.*

### M-iv — Allocation patterns, GC, heap dumps
*Status: PENDING. Focus: deinit patterns, weak references in coordinators (carry-over from Report 3 finding #6 in the table below).*

## S4 — Threading
### T-i — Thread creation, async usage
*Status: PENDING*

### T-ii — Main-thread locks
*Status: PENDING. Password generation is a candidate for crypto on background thread.*

### T-iii — Multithreading performance impact
*Status: PENDING*

---

## Fragmentation runs

| Simulator | iOS | S1 cold start (ms) | S2 fps avg | S3 latency (ms) | S4 leaks | Notes |
|-----------|-----|--------------------|------------|-----------------|----------|-------|
| iPhone 17 Pro | 26.3.1 | ~15,670 (S1 mean from App Launch) | | | | Reference (repo official sim) |
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
