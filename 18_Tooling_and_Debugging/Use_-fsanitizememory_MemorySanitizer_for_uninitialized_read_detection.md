# Use -fsanitize=memory (MemorySanitizer) for uninitialized read detection

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/MemorySanitizer.html>  

---

## Topic Overview

MemorySanitizer (MSan) detects reads of **uninitialized memory** - a class of bug that is notoriously hard to find any other way. The problem with uninitialized reads is that the values you get are not random garbage; they are whatever happened to be in memory at that address before your allocation. On a developer's machine, the memory might always happen to hold the value your code expects, so the bug hides completely. On a different machine, in a different build, or under a different load pattern, you get a crash or - worse - a security vulnerability.

MSan tracks every bit of memory, marking it as initialized or uninitialized. When your code branches on an uninitialized value, MSan fires immediately and tells you exactly where the uninitialized memory was allocated.

```cpp
Compile:  clang++ -fsanitize=memory -fsanitize-memory-track-origins -fno-omit-frame-pointer -g prog.cpp
Run:      ./a.out
```

| Feature | MSan | Valgrind/Memcheck |
| --- | --- | --- |
| Overhead | ~3x | ~20x |
| Instrumentation | Compile-time | Binary (no recompile) |
| All libs instrumented? | Required | Not required |
| Platform | Clang on Linux only | Any platform |
| False positives | Minimal (if all libs instrumented) | Some |

The trade-off is clear: MSan is much faster than Valgrind and gives more precise origin information, but it requires you to rebuild your dependencies with MSan instrumentation - which is the main reason people sometimes reach for Valgrind instead despite its higher overhead.

---

## Self-Assessment

### Q1: Trigger a MemorySanitizer report

Here is a minimal example that demonstrates two common patterns MSan catches. The first is a heap allocation that is read before being initialized; the second is a stack struct with no constructor:

```cpp
#include <iostream>
#include <cstdlib>

int main() {
    // Allocate heap memory - contents are UNINITIALIZED
    int* data = new int[5];

    // BUG: reading uninitialized value
    if (data[2] > 0) {          // MSan catches this!
        std::cout << "positive\n";
    } else {
        std::cout << "non-positive\n";
    }

    // Another common pattern: uninitialized struct field
    struct Config {
        int timeout;
        bool verbose;
        // oops: no constructor, fields are garbage
    };

    Config cfg;           // uninitialized
    if (cfg.verbose) {    // MSan catches this!
        std::cout << "verbose mode\n";
    }

    delete[] data;
    return 0;
}
// Compile: clang++ -fsanitize=memory -fsanitize-memory-track-origins -g prog.cpp
// MSan output (approximate):
//   ==12345==WARNING: MemorySanitizer: use-of-uninitialized-value
//       #0 0x... in main prog.cpp:9
//   Uninitialized value was created by a heap allocation
//       #0 0x... in operator new[](unsigned long)
//       #1 0x... in main prog.cpp:6
//
// With -fsanitize-memory-track-origins, MSan shows WHERE the
// uninitialized memory was allocated (invaluable for debugging).
```

The `-fsanitize-memory-track-origins` flag is worth the extra overhead it adds - without it MSan tells you where the bad read happened, but with it MSan also tells you where the uninitialized memory was originally created. That second piece of information is often what you need to identify and fix the root cause.

### Q2: Why MSan requires all libraries to be instrumented

This is the requirement that catches people off guard and makes MSan harder to set up than ASAN. The reason is fundamental to how MSan works.

MSan tracks initialization at **compile time** by inserting shadow checks into every function. When a function in an uninstrumented library writes to memory, it does so without updating the shadow bytes. From MSan's perspective, the destination memory still looks uninitialized, even though the library filled it with valid data. This produces false positives:

```cpp
Instrumented code:              Uninstrumented libc:
┌─────────────────────┐        ┌─────────────────────┐
│ int x;              │        │ memcpy(dst, src, n)  │
│ shadow[&x] = UNINIT │        │ // no shadow update! │
│                     │        │ // MSan thinks dst   │
│ memcpy(&y, &x, 4)  │-->     │ // is still uninit   │
│                     │        │ // even if src was   │
│ if (y > 0) ...      │        │ // fully initialized │
│ // FALSE POSITIVE!  │        └─────────────────────┘
└─────────────────────┘

Result: false positives (thinks data is uninit when it isn't)
OR false negatives (uninstrumented code writes without updating shadow)
```

The fix is to build libc++ with MSan instrumentation yourself and link against it. This is genuinely work, but it is a one-time setup:

```bash
cmake -DLLVM_USE_SANITIZER=Memory \
      -DLLVM_ENABLE_PROJECTS="libcxx;libcxxabi" \
      ../llvm
```

- Link with `-stdlib=libc++` (not libstdc++)
- Third-party libraries must also be rebuilt with MSan
- This is the main reason MSan is harder to set up than ASAN

### Q3: MSan vs Valgrind/Memcheck comparison

This example makes it easy to compare the two tools on the same bug. The uninitialized elements in both the heap array and the stack array will be read and their values used in arithmetic:

```cpp
#include <iostream>
#include <vector>

int compute(int* arr, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i)
        sum += arr[i];  // BUG if arr has uninitialized elements
    return sum;
}

int main() {
    // Case 1: Heap uninitialized
    int* heap = new int[4];  // not zeroed!
    heap[0] = 10;
    heap[1] = 20;
    // heap[2] and heap[3] are uninitialized

    int result = compute(heap, 4);  // reads uninit values!
    std::cout << "Sum: " << result << '\n';
    delete[] heap;

    // Case 2: Stack uninitialized
    int stack_arr[3];
    stack_arr[0] = 5;
    // stack_arr[1] and stack_arr[2] are uninitialized

    int sum2 = compute(stack_arr, 3);
    std::cout << "Stack sum: " << sum2 << '\n';

    return 0;
}
// MSan (compile-time instrumentation):
//   clang++ -fsanitize=memory -fsanitize-memory-track-origins -g prog.cpp
//   Catches BOTH cases immediately with exact origin info.
//   ~3x overhead.
//
// Valgrind (binary instrumentation):
//   g++ -g prog.cpp && valgrind --track-origins=yes ./a.out
//   Also catches both, but:
//   - ~20x overhead (much slower!)
//   - No recompilation needed (works on any binary)
//   - May miss some cases in optimized code
//   - Available on more platforms
//   - Reports: "Conditional jump depends on uninitialised value(s)"
```

The practical takeaway: if you already have a working MSan-instrumented build, use it - you get faster feedback and better origin information. If setting that up is not practical for your project at the moment, Valgrind is still a solid option that requires no changes to your build system.

---

## Notes

- MSan is **Clang-only** - GCC does not support it.
- MSan is **Linux-only** - not available on macOS or Windows.
- Use `-fsanitize-memory-track-origins=2` for full origin tracking (more overhead).
- MSan is **incompatible** with ASAN and TSAN - run in separate CI jobs.
- For quick checks without MSan setup, Valgrind is simpler (no recompilation needed).
- Always use `-fno-omit-frame-pointer` for readable stack traces.
