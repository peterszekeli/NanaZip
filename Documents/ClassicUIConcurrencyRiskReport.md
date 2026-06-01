# NanaZip Classic UI Concurrency Risk Review

This report reviews concurrency-sensitive code paths in the classic UI implementation and proposes low-risk fixes.

## Scope

- `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/PanelItemOpen.cpp`
- `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/Panel.h`
- `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/FSFolder.cpp`

## Risk 1: UI-thread deadlock via synchronous cross-thread messaging

**Where**

- `PanelItemOpen.cpp:1388-1392` (`SendMessage(tpi->Window, kOpenItemChanged, ...)` from worker thread)
- `PanelItemOpen.cpp:1207-1233` (`CExitEventLauncher::Exit()` waits for worker threads)

**Why this is risky**

If the UI thread enters `Exit()` and waits on worker thread handles while a worker thread issues synchronous `SendMessage()` to the same UI thread, both threads can wait on each other.

**Proposed fix**

1. Replace worker-thread `SendMessage()` with `PostMessage()` (or `SendMessageTimeout()` with timeout and fallback).
2. If synchronous result is required, use an event/future-style completion object instead of blocking the UI thread.
3. In shutdown paths, avoid indefinite waits from the UI thread when workers can call back into UI message handling.

## Risk 2: Shared global launcher state is not synchronized

**Where**

- `Panel.h:934-937` (`_needExit`, `_threads`, `_numActiveThreads` in global `g_ExitEventLauncher`)
- `PanelItemOpen.cpp:1207-1233` (mutating shared fields in `Exit()`)
- `PanelItemOpen.cpp:1940-1941` (adding threads and incrementing counters from open-item flow)

**Why this is risky**

The global state is read/written without a lock or atomic operations. Multiple callers invoking shutdown/open flows can race and corrupt lifecycle bookkeeping (double-close handle, stale count, missed thread joins).

**Proposed fix**

1. Add a critical section/mutex inside `CExitEventLauncher` and guard all accesses to `_threads`, `_numActiveThreads`, and `_needExit`.
2. Replace `_numActiveThreads` with a value derived from the protected thread container, or make it atomic and still protect handle ownership transitions with a lock.
3. Consolidate thread registration/join logic into methods on `CExitEventLauncher` so callers do not mutate internals directly.

## Risk 3: Non-thread-safe lazy initialization for `NtQueryInformationFile`

**Where**

- `FSFolder.cpp:465-479` (`static Func_NtQueryInformationFile f_NtQueryInformationFile;`
  and `static bool g_NtQueryInformationFile_WasRequested`)

**Why this is risky**

The one-time initialization check uses plain globals with no synchronization. Concurrent calls to `ReadChangeTime()` can race during first access, causing undefined behavior and partially-initialized state visibility.

**Proposed fix**

1. Use thread-safe one-time initialization (`std::once_flag` + `std::call_once`, or equivalent Win32 one-time init primitive).
2. Keep function pointer write inside the one-time init section and treat it as immutable afterward.

## Risk 4: Process wait set truncation may cause cleanup races

**Where**

- `PanelItemOpen.cpp:1280-1286` (loop breaks when `handles.Size() > 60`)
- `PanelItemOpen.cpp:1421-1430` (temp file/folder deletion after wait logic)

**Why this is risky**

The wait set intentionally truncates tracked handles. If many child processes exist, some may remain unobserved while cleanup proceeds, creating races between still-running processes and temp file/folder deletion.

**Proposed fix**

1. Process handles in stable chunks until all `NeedWait` entries are observed complete.
2. Delay delete/repack operations until all tracked child processes are confirmed done.
3. Emit diagnostic logging when handle truncation occurs to aid field diagnosis.

## Suggested implementation order

1. Fix deadlock risk (Risk 1) first.
2. Add synchronization in `CExitEventLauncher` (Risk 2).
3. Harden one-time init in `FSFolder.cpp` (Risk 3).
4. Improve wait-set chunking and cleanup sequencing (Risk 4).
