# NanaZip.UI.Classic concurrency risk assessment

This report documents concrete concurrency risks found in `NanaZip.UI.Classic` and proposes targeted fixes.

## 1) Data race in lazy initialization of `NtQueryInformationFile`

- **Location:** `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/FSFolder.cpp` (lines ~465-478)
- **Current pattern:**
  - Uses mutable globals:
    - `static Func_NtQueryInformationFile f_NtQueryInformationFile;`
    - `static bool g_NtQueryInformationFile_WasRequested = false;`
  - `ReadChangeTime()` performs unchecked read/write of these globals.
- **Why this is risky:**
  - If multiple threads call `ReadChangeTime()` concurrently, writes to `g_NtQueryInformationFile_WasRequested` and `f_NtQueryInformationFile` are unsynchronized.
  - This is undefined behavior in C++ (data race), and can produce inconsistent function pointer state.
- **Proposed fix:**
  - Replace the ad-hoc boolean guard with one-time initialization:
    - `std::once_flag` + `std::call_once`, or
    - function-local static initialization that returns the cached function pointer.
  - Example direction:
    - `static Func_NtQueryInformationFile GetNtQueryInformationFileFunc();`
    - Do all `GetModuleHandleW/GetProcAddress` in that helper, and return an immutable value.

## 2) Data race in lazy loading of `Psapi.dll` handle

- **Location:** `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/PanelItemOpen.cpp` (lines ~187, ~227-233)
- **Current pattern:**
  - Global mutable state:
    - `static HMODULE g_Psapi_dll_module;`
  - `My_GetProcessFileName()` checks and writes it without synchronization.
- **Why this is risky:**
  - `My_GetProcessFileName()` can be reached from worker threads created by `Thread_Create()` in the same file (lines ~1937-1941).
  - Concurrent `if (!g_Psapi_dll_module) g_Psapi_dll_module = LoadLibraryW(...)` is a classic race and can result in duplicate loads or torn visibility.
- **Proposed fix:**
  - Use one-time thread-safe initialization for module loading (`std::call_once`), or use an immutable function-scope static wrapper.
  - Store and read the handle through an atomic or fully initialize-before-publish pattern.
  - Optional hardening: use `GetModuleHandleW(L"Psapi.dll")` first, then `LoadLibraryW` only if needed.

## 3) Potential UI-thread deadlock during close flow

- **Locations:**
  - `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/MyLoadMenu.cpp` (line ~708)
  - `NanaZip.UI.Classic/SevenZip/CPP/7zip/UI/FileManager/PanelItemOpen.cpp` (line ~1391)
- **Current pattern:**
  - UI close command calls `g_ExitEventLauncher.Exit(false)` (non-hard exit waits for worker completion).
  - Worker thread path may call `SendMessage(...)` back to UI thread (`kOpenItemChanged`).
- **Why this is risky:**
  - `SendMessage` is synchronous; if UI thread is blocked waiting for worker shutdown, worker can block waiting for UI message handling.
  - This creates a lockstep deadlock risk under unlucky timing.
- **Proposed fix:**
  - Replace worker-side `SendMessage` with `PostMessage` (async) + explicit ownership/lifetime handling for payload.
  - Or, in shutdown path, avoid indefinite waits when worker->UI synchronous calls are possible (bounded waits + cancel path).
  - Preferred long-term: centralize worker/UI marshaling through a non-blocking message queue.

## Suggested implementation order

1. Fix lazy initialization races (`FSFolder.cpp`, `PanelItemOpen.cpp`) via `std::call_once` helpers.
2. Eliminate synchronous worker->UI call during close-sensitive path (`SendMessage` -> async marshaling).
3. Add stress validation:
   - Repeated open/close of archives while closing app.
   - Parallel folder/property reads across both panels.
   - Run under ThreadSanitizer-equivalent tooling where available, or Win32 race-focused stress harness.
