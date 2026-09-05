# Understand and use address-space sanitizer for security auditing

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

AddressSanitizer (ASAN) detects memory errors at runtime by inserting checks around every memory access. What makes it especially valuable for security work is that the bugs it finds - heap overflows, use-after-free, stack overflows - are not just correctness issues. They map directly to well-known vulnerability classes that attackers exploit in the wild. Finding one with ASAN before a fuzzer or a real attacker does is a significant win.

| Bug Class | CWE | ASAN Detects? |
| --- | --- | --- |
| Heap buffer overflow | CWE-122 | Yes |
| Stack buffer overflow | CWE-121 | Yes |
| Use-after-free | CWE-416 | Yes |
| Use-after-return | CWE-562 | Yes (with flag) |
| Memory leaks | CWE-401 | Yes (LeakSanitizer) |
| Double-free | CWE-415 | Yes |

The compile-and-run recipe is straightforward:

```cpp
Compile:  clang++ -fsanitize=address -fno-omit-frame-pointer -g prog.cpp
Run:      ASAN_OPTIONS=detect_leaks=1 ./a.out
```

ASAN runs at roughly 2x the normal execution speed - dramatically faster than Valgrind's 20x overhead - which makes it practical to use in CI and during development.

---

## Self-Assessment

### Q1: Find a heap buffer overflow with ASAN

The tricky thing about a heap buffer overflow is that it often does not crash immediately. The out-of-bounds write lands in memory that happens to be unused at that moment, so the program appears to work. ASAN catches it at the moment the bad write happens, before any damage has a chance to propagate:

```cpp
#include <iostream>
#include <cstdlib>

// This function has an off-by-one bug
void fill_buffer(int* buf, int size) {
    for (int i = 0; i <= size; ++i) {  // BUG: should be i < size
        buf[i] = i * 10;
    }
}

int main() {
    const int N = 10;
    int* data = new int[N];

    fill_buffer(data, N);  // writes data[10] - heap overflow!

    // Print contents
    for (int i = 0; i < N; ++i)
        std::cout << data[i] << ' ';
    std::cout << '\n';

    delete[] data;
    return 0;
}
// Compile normally: g++ -o prog prog.cpp
//   -> may silently "work" (undefined behavior!)
//
// Compile with ASAN: clang++ -fsanitize=address -g -o prog prog.cpp
// ASAN output (approximate):
//   ==12345==ERROR: AddressSanitizer: heap-buffer-overflow
//   WRITE of size 4 at 0x... thread T0
//       #0 0x... in fill_buffer(int*, int) prog.cpp:6
//       #1 0x... in main prog.cpp:12
//   0x... is located 0 bytes to the right of 40-byte region
//   allocated by thread T0 here:
//       #0 0x... in operator new[](unsigned long)
//       #1 0x... in main prog.cpp:11
```

ASAN's report tells you exactly what happened: a 4-byte write landed 0 bytes past the end of a 40-byte region, and it shows you both where the bad write occurred and where the allocation was made. That is everything you need to find and fix the bug.

The **fix**: change `i <= size` to `i < size` in the loop.

### Q2: Detect memory leaks with LeakSanitizer

LeakSanitizer (LSan) is bundled with ASAN and runs automatically at program exit when you set `ASAN_OPTIONS=detect_leaks=1`. It reports every heap allocation that was never freed. Here is a realistic example where a factory function forgets to clean up:

```cpp
#include <iostream>
#include <string>
#include <vector>

struct Connection {
    std::string host;
    int port;
    Connection(std::string h, int p) : host(std::move(h)), port(p) {
        std::cout << "Connected to " << host << ':' << port << '\n';
    }
    ~Connection() {
        std::cout << "Disconnected from " << host << ':' << port << '\n';
    }
};

void handle_request() {
    // BUG: raw new without delete - leak!
    Connection* conn = new Connection("db.example.com", 5432);
    conn->host;  // use it
    // forgot: delete conn;
}

int main() {
    for (int i = 0; i < 3; ++i)
        handle_request();  // leaks 3 connections

    std::cout << "Server shutting down\n";
    return 0;
}
// Compile: clang++ -fsanitize=address -g -o prog prog.cpp
// Run:     ASAN_OPTIONS=detect_leaks=1 ./prog
//
// Normal output:
//   Connected to db.example.com:5432
//   Connected to db.example.com:5432
//   Connected to db.example.com:5432
//   Server shutting down
//
// LSAN report at exit:
//   ==12345==ERROR: LeakSanitizer: detected memory leaks
//   Direct leak of 120 byte(s) in 3 object(s) allocated from:
//       #0 0x... in operator new(unsigned long)
//       #1 0x... in handle_request() prog.cpp:19
//   SUMMARY: AddressSanitizer: 120 byte(s) leaked in 3 allocation(s).
```

Notice that the destructors never run - the objects are leaked, so their `~Connection()` is never called. LSan pinpoints the allocation site, making the fix obvious.

**Fix**: use `std::unique_ptr<Connection>` instead of raw `new`.

### Q3: ASAN's shadow memory model

Understanding how ASAN works internally helps you reason about its overhead and its limitations. The core idea is that ASAN maintains a parallel block of memory - the shadow memory - that tracks whether each byte in your program is accessible or "poisoned."

ASAN maps every 8 bytes of application memory to 1 byte of **shadow memory**:

```cpp
Application memory:          Shadow memory:
┌────────────────────┐      ┌──────────┐
│ 8 bytes app data   │ -->  │ 1 byte   │
│ 8 bytes app data   │ -->  │ 1 byte   │
│ 8 bytes app data   │ -->  │ 1 byte   │
└────────────────────┘      └──────────┘

Shadow byte values:
  0x00 = all 8 bytes accessible
  0x01..0x07 = first N bytes accessible, rest poisoned
  0xfa = heap left redzone
  0xfb = heap right redzone
  0xfd = freed heap memory
  0xf1 = stack left redzone
  0xf5 = stack partial redzone
```

When you allocate an array of 10 integers (40 bytes), ASAN surrounds it with "redzone" regions and poisons them with the special 0xfa/0xfb marker bytes. If you write one element past the end, the shadow lookup immediately returns a non-zero value and ASAN reports the error before any real damage occurs.

**Memory overhead breakdown:**

- Shadow memory: 1/8 of virtual address space (~12.5%)
- Redzones: extra bytes around every allocation (~50-100%)
- Quarantine: freed memory kept poisoned to detect use-after-free
- **Total: ~2-3x memory overhead**

Before every memory access, ASAN executes roughly this check - the compiler inserts it at compile time:

```cpp
// Before every memory access:
byte* shadow = (addr >> 3) + SHADOW_OFFSET;
if (*shadow != 0) {
    // Detailed check for partial accessibility
    if (*shadow <= (addr & 7) + access_size - 1)
        report_error();  // invalid access!
}
```

The quarantine is worth understanding separately. When you call `delete`, ASAN does not immediately return the memory to the allocator. Instead, it keeps the memory in a "quarantine" zone, poisoned with 0xfd. If any subsequent code tries to read or write that memory, ASAN will catch it as a use-after-free. Without quarantine, the allocator could hand that memory to a new allocation, and the bug would go undetected.

**Performance impact:** ~2x slowdown (much faster than Valgrind's ~20x).

---

## Notes

- ASAN is supported by GCC (4.8+), Clang (3.1+), and MSVC (/fsanitize=address).
- Use `-fno-omit-frame-pointer` for readable stack traces.
- ASAN cannot detect uninitialized reads - use MSan for that.
- For security auditing, combine ASAN with fuzzing (libFuzzer, AFL).
- Set `ASAN_OPTIONS=detect_stack_use_after_return=1` for use-after-return detection.
