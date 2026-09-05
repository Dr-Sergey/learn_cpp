# Implement SO and DLL Versioning Strategies

**Category:** ABI and Binary Compatibility  
**Standard:** C++11 and later (platform-specific linking)  
**Reference:** https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html  

---

## Topic Overview

Shared library versioning is the mechanism that allows multiple incompatible versions of a library to coexist on a system, and ensures that applications link against the correct version. On Linux, this is handled through the **soname** convention and symbolic links. On Windows, DLL versioning uses a combination of file naming, embedded version resources, and SxS (Side-by-Side) assemblies. Getting versioning right prevents "DLL hell" and "dependency hell" - two of the most notorious deployment problems in software engineering.

The Linux soname convention uses a three-part version number embedded in the library filename. Here is how the parts map to the actual files and symlinks on disk:

```cpp
// libmylib.so.2.4.1
//   |         | | |
//   |         | | +-- patch: bug fixes, fully compatible
//   |         | +---- minor: new features, backward compatible
//   |         +------ major: ABI-breaking changes
//   |
//   +-- "real name" on disk
//
// Symbolic links:
//   libmylib.so       -> libmylib.so.2.4.1  (linker name, used by -lmylib)
//   libmylib.so.2     -> libmylib.so.2.4.1  (soname, embedded in ELF)
```

| Version Component | When to Bump | ABI Impact                              |
| --- | :---: | --- |
| Major (2.x.x)    | ABI break    | Old binaries CANNOT use new library     |
| Minor (x.4.x)    | New symbols  | Old binaries CAN use new (subset)       |
| Patch (x.x.1)    | Bug fix only | Fully interchangeable                   |

On Windows, there is no equivalent of soname - DLL resolution uses the **search path** (application directory, system directories, PATH). This means version conflicts are resolved by which DLL is found first, not by embedded metadata. The modern solution is SxS manifests or simply shipping DLLs alongside the application. Version information is embedded in the PE resource section via `.rc` files.

---

## Self-Assessment

### Q1: Build a shared library with proper soname versioning using CMake and verify the runtime linking behavior

CMake handles soname automatically once you set the `VERSION` and `SOVERSION` properties on a library target. Here is a complete setup that produces the correct three-symlink structure:

```cpp
// === CMakeLists.txt ===
/*
cmake_minimum_required(VERSION 3.16)
project(mylib VERSION 2.4.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# Create shared library with proper versioning
add_library(mylib SHARED
    src/mylib.cpp
)

# VERSION  = full version (used for filename: libmylib.so.2.4.1)
# SOVERSION = major version (used for soname: libmylib.so.2)
set_target_properties(mylib PROPERTIES
    VERSION   ${PROJECT_VERSION}        # 2.4.1
    SOVERSION ${PROJECT_VERSION_MAJOR}  # 2
    # On macOS, this controls the install_name:
    # MACOSX_RPATH TRUE
)

# Export only public symbols
target_compile_options(mylib PRIVATE -fvisibility=hidden)

# Install with proper symlinks
install(TARGETS mylib
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin  # DLLs on Windows go to bin
)
*/

// === src/mylib.h ===
#pragma once

#ifdef _WIN32
    #ifdef MYLIB_EXPORTS
        #define MYLIB_API __declspec(dllexport)
    #else
        #define MYLIB_API __declspec(dllimport)
    #endif
#else
    #define MYLIB_API __attribute__((visibility("default")))
#endif

namespace mylib {

MYLIB_API int version_major();
MYLIB_API int version_minor();
MYLIB_API int version_patch();
MYLIB_API const char* version_string();

MYLIB_API int compute(int a, int b);

// Added in 2.1.0:
MYLIB_API int compute_extended(int a, int b, int mode);

// Added in 2.4.0:
MYLIB_API void set_precision(int digits);

}

// === src/mylib.cpp ===
// #include "mylib.h"
#include <cstdio>

namespace mylib {

MYLIB_API int version_major() { return 2; }
MYLIB_API int version_minor() { return 4; }
MYLIB_API int version_patch() { return 1; }
MYLIB_API const char* version_string() { return "2.4.1"; }

MYLIB_API int compute(int a, int b) { return a + b; }
MYLIB_API int compute_extended(int a, int b, int mode) {
    switch (mode) {
        case 0: return a + b;
        case 1: return a * b;
        default: return a - b;
    }
}
MYLIB_API void set_precision(int digits) {
    std::printf("Precision set to %d\n", digits);
}

}
```

After building, use these shell commands to verify that the soname is correctly embedded and that the symlinks are in place:

```bash
# After building:
$ ls -la build/
# libmylib.so -> libmylib.so.2        (linker name symlink)
# libmylib.so.2 -> libmylib.so.2.4.1  (soname symlink)
# libmylib.so.2.4.1                    (actual library file)

# Check embedded soname:
$ readelf -d build/libmylib.so.2.4.1 | grep SONAME
#  (SONAME)   Library soname: [libmylib.so.2]

# Check what an application links against:
$ readelf -d myapp | grep NEEDED
#  (NEEDED)   Shared library: [libmylib.so.2]
# Note: it records the SONAME, not the full filename

# At runtime, ld.so searches for libmylib.so.2
# which is a symlink to whichever 2.x.y is installed
```

Notice that the application records `libmylib.so.2` (the soname), not `libmylib.so.2.4.1`. This is the whole point: you can replace `libmylib.so.2.4.1` with `libmylib.so.2.5.0` and update the `libmylib.so.2` symlink, and all existing binaries automatically use the new version without relinking.

### Q2: Implement Linux symbol versioning with `.symver` to maintain multiple function implementations in a single shared library

Sometimes you ship a new version of a library where a function's semantics change in a way that is not safe for old binaries - for example, returning a heap-allocated string that the caller must free. You cannot silently update the function because old callers will not free it. Symbol versioning lets you keep both implementations in the same `.so` and the linker wires each binary to the right one:

```cpp
#include <cstdio>
#include <cstring>

// === Version 1.0: Original implementation ===
extern "C" {

// v1 implementation: basic string processing
char* process_string_v1(const char* input) {
    static char buf[256];
    std::snprintf(buf, sizeof(buf), "[v1] %s", input);
    return buf;
}

// v2 implementation: added length validation + heap allocation
char* process_string_v2(const char* input) {
    if (!input) return nullptr;
    std::size_t len = std::strlen(input);
    // In real code, caller must free() this
    char* result = static_cast<char*>(std::malloc(len + 8));
    if (!result) return nullptr;
    std::snprintf(result, len + 8, "[v2] %s", input);
    return result;
}

// GNU symbol versioning directives
// @ = compat version (old binaries use this)
// @@ = default version (new binaries use this)

// Old binaries linked against MYLIB_1.0 get process_string_v1
__asm__(".symver process_string_v1, process_string@MYLIB_1.0");

// New binaries linked against MYLIB_2.0 get process_string_v2
__asm__(".symver process_string_v2, process_string@@MYLIB_2.0");

}

// === Version map file: mylib_versions.map ===
/*
MYLIB_1.0 {
    global:
        process_string;
        init_library;
    local:
        *;
};

MYLIB_2.0 {
    global:
        process_string;
        configure_library;
} MYLIB_1.0;
*/

// Build: gcc -shared -Wl,--version-script=mylib_versions.map \
//        -o libmylib.so.2.0.0 mylib.c

// Result: nm -D libmylib.so shows:
//   T process_string@@MYLIB_2.0
//   T process_string@MYLIB_1.0
//
// An app linked against the old .so calls process_string@MYLIB_1.0
// An app linked against the new .so calls process_string@@MYLIB_2.0
// Both versions coexist in the same shared library!
```

The `@@` double-at marks the default version, meaning any new binary linking against this library gets `process_string_v2`. The single `@` marks the compatibility version: an old binary that originally linked against the MYLIB_1.0 version of the symbol continues to call `process_string_v1` forever. This is exactly how glibc has maintained backward compatibility since the 1990s.

### Q3: Implement Windows DLL versioning with version resources, delay-load, and runtime version checks

Windows does not have a soname, so version management is done differently. You embed version metadata into the DLL via a `.rc` resource file, and consumer code reads it back at runtime using the `version.lib` API. Delay-loading lets you check the version before committing to using the DLL:

```cpp
// === mylib.rc (Windows Resource File) ===
/*
#include <winver.h>

VS_VERSION_INFO VERSIONINFO
FILEVERSION    2,4,1,0
PRODUCTVERSION 2,4,1,0
FILEFLAGSMASK  VS_FFI_FILEFLAGSMASK
FILEFLAGS      0
FILEOS         VOS_NT_WINDOWS32
FILETYPE       VFT_DLL
FILESUBTYPE    VFT2_UNKNOWN
BEGIN
    BLOCK "StringFileInfo"
    BEGIN
        BLOCK "040904B0"
        BEGIN
            VALUE "FileDescription",  "MyLib Shared Library"
            VALUE "FileVersion",      "2.4.1.0"
            VALUE "ProductName",      "MyLib"
            VALUE "ProductVersion",   "2.4.1.0"
            VALUE "InternalName",     "mylib"
            VALUE "OriginalFilename", "mylib.dll"
            VALUE "CompanyName",      "MyOrg"
            VALUE "LegalCopyright",   "Copyright (C) 2026"
        END
    END
    BLOCK "VarFileInfo"
    BEGIN
        VALUE "Translation", 0x409, 1200
    END
END
*/

// === Runtime version checking (consumer side) ===
#ifdef _WIN32
#include <windows.h>
#include <cstdio>
#include <memory>

#pragma comment(lib, "version.lib")

struct DllVersion {
    int major, minor, patch, build;
};

bool get_dll_version(const char* dll_path, DllVersion& ver) {
    DWORD handle = 0;
    DWORD size = GetFileVersionInfoSizeA(dll_path, &handle);
    if (size == 0) return false;

    auto buffer = std::make_unique<char[]>(size);
    if (!GetFileVersionInfoA(dll_path, handle, size, buffer.get()))
        return false;

    VS_FIXEDFILEINFO* file_info = nullptr;
    UINT len = 0;
    if (!VerQueryValueA(buffer.get(), "\\",
                        reinterpret_cast<void**>(&file_info), &len))
        return false;

    ver.major = HIWORD(file_info->dwFileVersionMS);
    ver.minor = LOWORD(file_info->dwFileVersionMS);
    ver.patch = HIWORD(file_info->dwFileVersionLS);
    ver.build = LOWORD(file_info->dwFileVersionLS);
    return true;
}

// Delay-load pattern for optional DLL features
// Link with: /DELAYLOAD:mylib_ext.dll
// #include <delayimp.h>

bool try_load_extension() {
    // Check if DLL exists and has compatible version
    DllVersion ver{};
    if (!get_dll_version("mylib_ext.dll", ver)) {
        std::printf("Extension DLL not found\n");
        return false;
    }

    if (ver.major != 2) {
        std::printf("Incompatible extension: v%d.%d.%d (need v2.x.x)\n",
                    ver.major, ver.minor, ver.patch);
        return false;
    }

    std::printf("Loaded extension v%d.%d.%d\n",
                ver.major, ver.minor, ver.patch);
    return true;
}

int main() {
    DllVersion ver{};
    if (get_dll_version("mylib.dll", ver)) {
        std::printf("mylib.dll version: %d.%d.%d.%d\n",
                    ver.major, ver.minor, ver.patch, ver.build);
    }
    try_load_extension();
    return 0;
}
#endif
```

The delay-load approach is particularly useful for optional features: link with `/DELAYLOAD:mylib_ext.dll`, check the version before the first call, and gracefully skip functionality if the DLL is absent or incompatible. Without delay-load, a missing DLL causes immediate startup failure even if you never needed it.

---

## Notes

- The **soname** (e.g., `libfoo.so.2`) is embedded in the ELF library and recorded by applications at link time - the dynamic linker resolves it at runtime via symlinks.
- Bump the major version (soname) only for ABI-breaking changes; keep minor/patch for backward-compatible additions and fixes.
- Use `ldconfig` after installing a new `.so` to update the shared library cache and create soname symlinks.
- On macOS, the equivalent of soname is **install_name** - set via `-install_name @rpath/libfoo.2.dylib` and managed with `install_name_tool`.
- Windows has no soname equivalent - DLL resolution depends on search order (application dir -> system dir -> PATH), making "DLL hell" a real problem.
- `.symver` directives allow multiple implementations of the same function in one `.so`, selected by the version tag recorded at link time - this is how glibc maintains backward compatibility for decades.
- Always embed version resources in Windows DLLs (`.rc` file) - this enables runtime version checking and is visible in file properties.
