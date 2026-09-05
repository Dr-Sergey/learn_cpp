# Use AddressSanitizer, MemorySanitizer, and ThreadSanitizer effectively

**Category:** Tooling and Debugging  
**Standard:** C++11  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

ASAN, MSan, and TSan form a complementary toolkit - each one catches a different category of runtime bug that the others cannot. Knowing which sanitizer to reach for is as important as knowing how to use them.

| Sanitizer | Catches | Flag | Overhead |
| --- | --- | --- | --- |
| ASAN | Memory errors (overflow, use-after-free, leaks) | `-fsanitize=address` | ~2x |
| MSAN | Uninitialized reads | `-fsanitize=memory` | ~3x |
| TSAN | Data races, deadlocks | `-fsanitize=thread` | ~5-15x |

The three sanitizers cannot all run at the same time, which is why you set them up as separate CI jobs. UBSAN is the exception - it is lightweight enough to combine with any of the three:

```cpp
Compatibility matrix:
          ASAN   MSAN   TSAN   UBSAN
  ASAN     -      No     No     Yes
  MSAN     No     -      No     Yes
  TSAN     No     No     -      Yes
  UBSAN    Yes    Yes    Yes    -

  Yes = combinable    No = mutually exclusive
```

In practice, "ASAN + UBSAN" is the most useful everyday combination for catching both memory errors and undefined behavior in a single run.

---

## Self-Assessment

### Q1: Trigger a heap-use-after-free with ASAN

Use-after-free is exactly the kind of bug that behaves silently in normal builds. After `delete`, the memory is returned to the allocator, which might immediately hand it to a new allocation. Your dangling pointer now reads from memory that belongs to something else. Crashes are not guaranteed - and when they do happen, they show up far from the actual bug. ASAN solves this by keeping freed memory poisoned in a quarantine zone:

```cpp
#include <iostream>
#include <vector>

int main() {
    int* ptr = new int(42);
    std::cout << "Value: " << *ptr << '\n';

    delete ptr;  // Free the memory

    // BUG: use-after-free!
    std::cout << "After free: " << *ptr << '\n';  // ASAN catches this!

    return 0;
}
// Compile: clang++ -fsanitize=address -fno-omit-frame-pointer -g -o uaf uaf.cpp
// ASAN report:
//   ==PID==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
//   READ of size 4 at 0x... thread T0
//       #0 0x... in main uaf.cpp:10
//   0x... is located 0 bytes inside of 4-byte region
//   freed by thread T0 here:
//       #0 0x... in operator delete(void*)
//       #1 0x... in main uaf.cpp:8
//   previously allocated by thread T0 here:
//       #0 0x... in operator new(unsigned long)
//       #1 0x... in main uaf.cpp:5
```

ASAN's report is unusually informative: it tells you where the bad read happened, where the memory was freed, and where it was originally allocated. You have all three pieces of information you need to fix the bug.

### Q2: Detect a data race with TSAN

A `std::vector` is not thread-safe. Calling `push_back` in one thread while calling `back` in another thread is a data race, even if it "usually works" in testing. The underlying implementation reads and writes internal pointers and size fields that are not protected:

```cpp
#include <iostream>
#include <thread>
#include <vector>

std::vector<int> shared_data;

void writer() {
    for (int i = 0; i < 1000; ++i)
        shared_data.push_back(i);  // DATA RACE: concurrent modification
}

void reader() {
    for (int i = 0; i < 1000; ++i) {
        if (!shared_data.empty())
            [[maybe_unused]] auto val = shared_data.back();  // DATA RACE
    }
}

int main() {
    std::thread t1(writer);
    std::thread t2(reader);
    t1.join();
    t2.join();
    std::cout << "Size: " << shared_data.size() << '\n';
}
// Compile: clang++ -fsanitize=thread -g -O1 -o race race.cpp -lpthread
// TSAN report:
//   WARNING: ThreadSanitizer: data race
//     Write of size 8 at 0x... by thread T1:
//       #0 std::vector<int>::push_back
//     Previous read of size 8 at 0x... by thread T2:
//       #0 std::vector<int>::back
//
// Fix: protect shared_data with std::mutex or use lock-free container
```

TSan gives you both sides of the race - the writing thread and the reading thread - along with stack traces for both. That is exactly the information you need to reason about what synchronization is missing.

### Q3: Why sanitizers can't be combined arbitrarily

The reason sanitizers cannot be combined is not a configuration issue - it is a fundamental conflict in how they are implemented. Each one needs to install itself as the sole owner of a critical piece of the runtime: the memory allocator and the shadow memory address space.

Here is what each sanitizer actually does at the implementation level:

```cpp
ASAN: Shadow memory 1:8 ratio

  - Poisons redzones around allocations
  - Intercepts malloc/free with custom allocator
  - Uses ~2x memory

MSAN: Shadow memory 1:1 ratio (bit-level tracking)

  - Tracks initialization status of every bit
  - Requires all code to be instrumented
  - Uses ~3x memory

TSAN: Shadow memory 4:1 ratio (4 shadow words per 8 bytes)

  - Stores access history per memory location
  - Intercepts all synchronization primitives
  - Uses ~5-10x memory

Conflict: They all need to intercept malloc/free
and use the SAME shadow memory address space differently.
```

The shadow memory address ranges these sanitizers use overlap in the virtual address space. ASAN's 1:8 scheme is physically incompatible with TSAN's 4:1 scheme - they cannot both be mapped at the same time. This is why the conflict is absolute, not something you can work around with flags.

The supported combinations tell the whole story:

| Combination | Works? | Notes |
| --- | --- | --- |
| ASAN + UBSAN | Yes | Most useful combo |
| MSAN + UBSAN | Yes | Clang-only |
| TSAN + UBSAN | Yes | Different instrumentation layers |
| ASAN + TSAN | No | Both replace allocator |
| ASAN + MSAN | No | Shadow memory conflict |
| TSAN + MSAN | No | Shadow memory conflict |

The practical consequence for your CI pipeline is to split sanitizers into separate jobs. Three parallel jobs run faster than one serial one, so the total CI time stays manageable:

```yaml
jobs:
  sanitize:
    strategy:
      matrix:
        config:
          - { name: "ASAN+UBSAN", flags: "-fsanitize=address,undefined" }
          - { name: "TSAN",       flags: "-fsanitize=thread" }
          - { name: "MSAN",       flags: "-fsanitize=memory" }
```

---

## Notes

- Always add `-fno-omit-frame-pointer -g` for readable stack traces.
- MSAN is Clang-only and requires all libraries to be instrumented.
- Use `halt_on_error=1` in sanitizer options for CI.
- UBSAN is lightweight (~5% overhead) and safe to combine with any sanitizer.
- Consider running ASAN+UBSAN in debug builds during development.
