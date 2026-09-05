# Use Valgrind/Helgrind for memory and thread error detection

**Category:** Tooling and Debugging  
**Standard:** C++11  
**Reference:** <https://valgrind.org>  

---

## Topic Overview

Valgrind is a dynamic analysis framework that runs your binary on a software-emulated CPU. Because it interprets every instruction rather than running native code, it can observe every memory access, every allocation, and every synchronization event - and that is what gives it its extraordinary power to detect subtle bugs. The trade-off is that it is slow, typically 20 to 50 times slower than native execution, but for thorough correctness checking that is often an acceptable cost.

Valgrind is not a single tool; it is a framework with several analysis plugins:

| Tool | Detects |
| --- | --- |
| **Memcheck** | Memory leaks, use-after-free, uninitialized reads, buffer overflows |
| **Helgrind** | Data races, lock order violations, misuse of pthreads |
| **DRD** | Data races (alternative to Helgrind) |
| **Massif** | Heap memory profiling over time |
| **Callgrind** | Call graph profiling |

Memcheck is the default and the one you will use most often. Helgrind and DRD are the tools for multi-threaded code.

---

## Self-Assessment

### Q1: Memcheck - detect uninitialized memory use

Reading from an uninitialized variable is undefined behavior in C++. The tricky part is that the bug often does not crash immediately - the garbage value flows through the program and causes a wrong conditional branch, a corrupted output, or a crash somewhere completely unrelated to the original mistake. Memcheck catches this at the source by tracking which bytes are "defined" and reporting the moment an uninitialized value influences control flow.

```cpp
// uninit.cpp — compile: g++ -g -O0 -std=c++20 uninit.cpp -o uninit
#include <iostream>

int compute(int x) {
    int result;                // uninitialized!
    if (x > 10)
        result = x * 2;
    // Bug: if x <= 10, result is uninitialized
    return result;
}

int main() {
    int val = compute(5);      // uses uninitialized 'result'
    if (val > 0)               // conditional jump on uninitialized value
        std::cout << val << '\n';
}
```

Run this under Memcheck and you get a precise report:

```bash
$ valgrind --tool=memcheck --track-origins=yes ./uninit

# Output:
# ==12345== Conditional jump or move depends on uninitialised value(s)
# ==12345==    at 0x401234: main (uninit.cpp:12)
# ==12345==  Uninitialised value was created by a stack allocation
# ==12345==    at 0x401210: compute(int) (uninit.cpp:4)
#
# ==12345== Use of uninitialised value of size 4
# ==12345==    at 0x401234: compute(int) (uninit.cpp:8)

# Interpreting:
# - "Conditional jump" = branching on uninitialized data
# - "created by stack allocation" = local variable on stack
# - --track-origins=yes shows WHERE the uninit value came from

# Common Memcheck options:
$ valgrind --leak-check=full --show-leak-kinds=all ./myapp
$ valgrind --track-origins=yes --expensive-definedness-checks=yes ./myapp
```

The `--track-origins=yes` flag is the most useful addition because it tells you not just where the uninitialized value was used, but exactly where in the code the uninitialized storage was created. That source location is usually all you need to fix the bug.

### Q2: Helgrind - detect lock ordering violation

A lock ordering violation is a potential deadlock waiting to happen. If thread 1 always acquires lock A then lock B, but thread 2 acquires B then A, there is a window where both threads can hold one lock and wait forever for the other. In testing you may never hit this window, but in production under load it can happen. Helgrind detects the inconsistent ordering even before a deadlock actually occurs.

```cpp
// lock_order.cpp — compile: g++ -g -O0 -std=c++20 -pthread lock_order.cpp -o lock_order
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mutex_a;
std::mutex mutex_b;
int shared_data = 0;

void thread1() {
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Lock A first
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Then lock B
    shared_data += 1;
}

void thread2() {
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Lock B first!
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Then lock A!
    // DEADLOCK RISK: opposite lock order from thread1!
    shared_data += 2;
}

int main() {
    std::thread t1(thread1);
    std::thread t2(thread2);
    t1.join();
    t2.join();
    std::cout << shared_data << '\n';
}
```

Helgrind catches this even if the two threads never actually deadlock in the particular run:

```bash
$ valgrind --tool=helgrind ./lock_order

# Output:
# ==12345== Thread #3: lock order violation
# ==12345==
# ==12345== Observed lock ordering:
# ==12345==   First observed acquisition order:
# ==12345==     Lock at 0x... (mutex_a) acquired at 0x401234 (lock_order.cpp:10)
# ==12345==     Lock at 0x... (mutex_b) acquired at 0x401245 (lock_order.cpp:11)
# ==12345==
# ==12345==   Conflicting acquisition order:
# ==12345==     Lock at 0x... (mutex_b) acquired at 0x401267 (lock_order.cpp:16)
# ==12345==     Lock at 0x... (mutex_a) acquired at 0x401278 (lock_order.cpp:17)
# ==12345==
# ==12345== This could result in a DEADLOCK

# Fix: always acquire locks in the same order, or use std::scoped_lock
void thread1_fixed() {
    std::scoped_lock lock(mutex_a, mutex_b);  // deadlock-free
    shared_data += 1;
}
void thread2_fixed() {
    std::scoped_lock lock(mutex_a, mutex_b);  // same order guaranteed
    shared_data += 2;
}
```

`std::scoped_lock` (C++17) is the idiomatic fix because it uses a deadlock-avoidance algorithm when locking multiple mutexes. You never need to remember the ordering yourself.

### Q3: Valgrind overhead vs sanitizers

Valgrind is thorough, but it is not the only tool in the toolbox. AddressSanitizer, ThreadSanitizer, and MemorySanitizer are compiler-based tools that instrument your code at compile time rather than interpreting it at runtime. They are much faster - a practical choice for everyday development and CI runs - but they require recompilation and cannot inspect code you do not have the source for.

| Aspect | Valgrind/Memcheck | ASan | TSan | MSan |
| --- | --- | --- | --- | --- |
| Slowdown | 20-50x | 2x | 5-15x | 3x |
| Memory overhead | 2-3x | 2-3x | 5-10x | 2-3x |
| Recompilation needed | No | Yes | Yes | Yes |
| Works on stripped binaries | Yes | No | No | No |
| Linux support | Excellent | Excellent | Excellent | Clang only |
| macOS support | Partial | Excellent | Excellent | No |
| Windows support | No | MSVC partial | No | No |
| Detects uninitialized reads | Yes | No | No | Yes (MSan) |

The right mental model is to use both, but at different points in the workflow:

```cpp
Valgrind:                           Sanitizers:
// GOOD: No recompilation available       // GOOD: Fast enough for CI
// GOOD: Third-party binaries             // GOOD: Daily development
// GOOD: Complete memory model check      // GOOD: Integration tests
// GOOD: Thorough but slow analysis       // GOOD: Fuzz testing (needs speed)
// BAD: Too slow for large test suites    // BAD: Require recompilation
// BAD: Not available on Windows          // BAD: Can't check third-party binaries
```

A practical CI strategy layers both approaches:

```bash
# Recommended CI strategy:
# Fast (every commit): sanitizers
$ g++ -fsanitize=address,undefined -g ./tests.cpp -o tests && ./tests

# Thorough (nightly): Valgrind
$ valgrind --leak-check=full --error-exitcode=1 ./tests

# Thread checking:
$ g++ -fsanitize=thread -g -O1 ./tests.cpp -o tests_tsan && ./tests_tsan  # fast
$ valgrind --tool=helgrind ./tests                                         # thorough
```

Run sanitizer-instrumented tests on every commit because they are fast enough to fit in your normal build pipeline. Reserve Valgrind for nightly or pre-release runs where you have time for its thorough but slow analysis.

---

## Notes

- Compile with `-g -O0` for best Valgrind accuracy - debug symbols give you source line numbers, and disabling optimization prevents the compiler from eliminating variables that Valgrind needs to track.
- `--error-exitcode=1` makes Valgrind return a non-zero exit code on errors, which is what you need for CI to flag failures automatically.
- Valgrind does not support AVX-512 or some newer SSE instructions; if your code uses them, you may need to disable those at compile time for Valgrind runs.
- The `VALGRIND_DO_LEAK_CHECK` macro lets you trigger a programmatic leak check at a specific point in your code, which is useful for test teardown.
- Helgrind detects lock order violations that may never deadlock in practice but are still latent bugs - treat every Helgrind finding as a real bug, not a false alarm.
