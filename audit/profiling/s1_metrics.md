# S1 — Cold Start Raw Profiling Data

Working data captured during App Launch profiling of scenario S1.
The curated analysis appears in main.tex §7. This file is the raw evidence trail.

- Trace file: s1_applaunch.trace (2.5 GB, not committed, deleted after extraction)
- Instruments template: App Launch (Time Profiler + Thread State Trace + dyld Activity)
- Date: 2026-05-13, 22:43-23:02
- Audited commit: 5218b7f21bce14622b823e41e844844aeedd593b
- Build: Release (Profile action)
- Simulator: iPhone 17 Pro device UUID 218345E8-81CD-4F0E-B4ED-DD7981D632D0

## Run timings

| Run | PID | Duration | Email entry |
|-----|-----|----------|-------------|
| #0 (sync, discarded) | - | 33.0s | Yes (first cold start, populated SQLite) |
| #1 official | 65800 | 15.0s | Yes |
| #2 official | 66226 | 15.0s | No (pre-filled) |
| #3 official | 66714 | 17.0s | No (pre-filled) |

Mean of runs 1-3: 15.67 s, SD 1.15 s, CV 7.4%.

## dyld phase timings (Run #3)

| Phase | Duration |
|-------|----------|
| Launch Executable umbrella | 16.59 s |
| Apply Fixups | 158.12 ms |
| libSystem.B.dylib static init | 200.09 ms |
| Objc image init Bitwarden | 172.73 ms |
| Network framework C++ static inits | ~10 ms cumulative across 200+ symbols |
| TextInputUI + AutoFillUI + AssistantServices dlopens | ~1.8 s cumulative |
| libcmark-gfm dlopen | 7.60 ms |
| Frameworks ~unrelated to password mgmt (PencilKit, ProofReader, CoreNLP, RawCamera, CMPhoto, WebKitLegacy) | ~20 ms cumulative |

## libSystem.B.dylib static init variability across runs

Run #1: 34.93 ms; Run #2: 92.76 ms; Run #3: 200.09 ms. SD 84 ms, CV 77%.

## Thread state breakdown (Run #3, all threads)

| State | Count | Total duration |
|-------|-------|-----------------|
| Running | 37,523 | 4.85 s |
| Blocked | 17,318 | 1277.34 s |
| Runnable | 13,932 | 44.14 ms |
| Interrupted | 13,800 | 13.27 ms |
| Preempted | 9,417 | 33.46 s |
| Idle | 1,488 | 4.85 min |
| Total | 93,959 | - |

## Time Profiler top symbols (Run #3, 1.85 min CPU total)

1. qos_class_main (libsystem_pthread): 1.46 min / 79.0%
2. thread_start (libsystem_pthread): 11.54 s / 10.4%
3. start (dyld): 9.13 s / 8.2%
4. start_wqthread: 2.18 s / 2.0%
5. completeTaskWithClosure (libswift_Concurrency): 398.97 ms / 0.4%

Release-symbol-stripped binary; Bitwarden-internal frames are aggregated under qos_class_main.

## F-RT-06 status

Not reproduced in any of the 4 runs. Vault loaded automatically with all 150 items in every run. Downgraded from deterministic bug to intermittent issue in §9.

## Screenshots (committed in audit/profiling/screenshots/)

s1_run1_threadstate.png, s1_run1_timeprofiler.png, s1_run2_dyld_overview.png, s1_run2_dyld_tasks.png, s1_run2_timeprofiler.png, s1_run3_dyld_overview.png, s1_run3_dyld_tasks.png, s1_run3_dyld_tasks_late.png, s1_run3_timeprofiler.png, s1_alltracks_run1.png, s1_alltracks_run2.png, s1_alltracks_run3.png.

## Instrument status (S1)

App Launch: complete. Allocations + Leaks: pending. Network: pending.
