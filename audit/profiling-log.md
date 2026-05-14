# Bitwarden iOS — Profiling Log (App Report 4)

**Repository:** bitwarden/ios (audit fork: gabrielpadilla24/ios)
**Branch audited:** main @ commit [PLACEHOLDER — actualizar tras primer build]
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
- Core Animation (GPU rendering, frame rate, hitches) — pending
- Color Blended Layers toggle (overdrawing visualization) — pending
- Time Profiler (CPU profile, main thread analysis) — pending

**Raw screenshots:** `audit/profiling/screenshots/s1_app_launch/` (12 PNGs, 3 runs) and `audit/profiling/screenshots/s1_allocations/` (12 PNGs, 3 runs).

---

## S1 — GPU rendering analysis

*Status: PENDING — to be completed via Core Animation instrument*

### Frame rate metrics

| Run | fps avg | fps min | Hitches | Notes |
|-----|---------|---------|---------|-------|
| 1   |         |         |         |       |
| 2   |         |         |         |       |
| 3   |         |         |         |       |
| **Mean ± SD** |  |  |  |  |

### Problems and strengths
*To be analyzed after Core Animation runs.*

---

## S1 — Overdrawing analysis

*Status: PENDING — to be completed via Color Blended Layers simulator toggle*

### Method
Enable Simulator → Debug → Color Blended Layers, navigate to Vault list, capture screenshot. Areas rendered as red indicate opaque single-layer rendering (optimal); areas in green indicate blended layers (overdrawing — multiple semi-transparent surfaces stacked).

### Observations
*To be captured.*

### Problems and strengths
*To be analyzed.*

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

*Status: PARTIALLY DOCUMENTED — partial data from App Launch and Allocations; full Time Profiler analysis pending.*

### T-i — Where and how are threads created? Async/await usage observed

From the Allocations Call Tree (S1 M-iv), the following thread-creation evidence is captured:

| Source | Count observed | Mechanism |
|--------|----------------|-----------|
| Bitwarden Rust SDK (`rayon` thread pool) | 11 native threads | `std::sys::thread::unix::Thread::new` (POSIX pthread spawn from Rust) |
| GCD / Dispatch workers | observed in stack traces | `_dispatch_worker_thread2`, `_dispatch_root_queue_drain` |
| Swift Concurrency | observed in stack traces | `swift::runJobInEstablishedExecutor`, `swift_job_runImpl` |
| pthread workqueue | observed in leak stack traces | `start_wqthread`, `_pthread_wqthread` |

Async/await is used extensively in the codebase. From the Allocations Call Tree, calls to `_$LT$async_compat...` and Swift Concurrency executor frames are observed across the FFI boundary, confirming async/await coordinates with the Rust SDK's `RustFuture` exposed via UniFFI.

### T-ii — Possible locks on main thread

*Status: PENDING — to be completed via Time Profiler instrument*

Time Profiler will provide:
- Percentage of CPU time on main thread during the 30s window
- Stack traces of any blocking operations on main thread
- Identification of potentially-blocking calls (CoreData synchronous fetches, file I/O, crypto operations)

The cold-start latency of 15.67s ± 1.15s (from App Launch instrument) is long enough that some portion of that time on main thread is expected, but the breakdown of *what* is on main vs background requires Time Profiler.

### T-iii — How multithreading affects performance

*Status: PENDING — to be completed via Time Profiler instrument, combined with cross-referenced Allocations data.*

Preliminary observations from existing data:
- Heavy crypto and SDK work appears to run off main (Rust threads via UniFFI), which is positive for UI responsiveness.
- The 22.5 MB of stack VM dedicated to Rust threads is a memory cost paid for keeping crypto off main thread — a tradeoff worth quantifying in the final analysis.

### Cold-start latency reference (from App Launch instrument)

| Run | Cold-start time |
|-----|------------------|
| 1   | (per s1_app_launch screenshots) |
| 2   | (per s1_app_launch screenshots) |
| 3   | (per s1_app_launch screenshots) |
| **Mean ± SD** | **15.67s ± 1.15s** |

**Screenshots:** `audit/profiling/screenshots/s1_app_launch/` (12 PNGs across 3 runs).

---

# Scenario S2 — Vault scroll + live search

## S2 — Methodology
*Status: PENDING*

## S2 — GPU rendering analysis
*Status: PENDING*

### Frame rate metrics
| Run | fps avg | fps min | Hitches | Notes |
|-----|---------|---------|---------|-------|
| 1   |         |         |         |       |
| 2   |         |         |         |       |
| 3   |         |         |         |       |
| **Mean ± SD** |  |  |  |  |

### Problems and strengths
*To be analyzed.*

## S2 — Overdrawing analysis
*Status: PENDING*

## S2 — Memory management
### M-i — Leaks
*Status: PENDING*

### M-ii — RAM consumption
| Run | Peak (MB) | Final (MB) | Δ from baseline | # Persistent allocs | Notes |
|-----|-----------|------------|------------------|----------------------|-------|
| 1   |           |            |                  |                      |       |
| 2   |           |            |                  |                      |       |
| 3   |           |            |                  |                      |       |
| **Mean ± SD** |  |  |  |  |  |

### M-iii — Libraries for leak management
*Cross-reference to S1 M-iii; library landscape does not vary by scenario.*

### M-iv — Allocation patterns, GC, heap dumps
*Status: PENDING*

## S2 — Threading
### T-i — Thread creation, async usage
*Status: PENDING*

### T-ii — Main-thread locks
*Status: PENDING — particularly relevant for S2 given fast typing in search (~150ms cadence).*

### T-iii — Multithreading performance impact
*Status: PENDING*

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
