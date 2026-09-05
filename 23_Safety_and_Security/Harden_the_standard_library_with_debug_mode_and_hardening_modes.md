# Harden the standard library with debug mode and hardening modes

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://libcxx.llvm.org/Hardening.html>  

---

## Topic Overview

This topic focuses on iterator validity checks and comparing hardening modes - complementing #558 (production hardening + perf analysis) and #650 (debug builds + OOB demos).

Iterator invalidation is one of the most insidious categories of C++ bugs. The problem is that an iterator is just a pointer to memory that was valid at the moment you obtained it. If the container reallocates or you erase elements, that pointer is now dangling - but it looks exactly like a valid pointer. In release builds, using it reads or writes arbitrary memory and the program may continue running for a long time before anything observable goes wrong. In debug mode, the containers track which iterators they have handed out and immediately detect any use after invalidation.

```cpp
Iterator invalidation bugs (silent in release, caught in debug):

  vector<int> v = {1, 2, 3};
  auto it = v.begin();
  v.push_back(4);     // may reallocate -> iterators invalidated!
  *it;                 // Release: UB (dangling pointer)
                       // Debug: assertion failure!

  map<int,int> m = {{1,10}, {2,20}};
  for (auto it = m.begin(); it != m.end(); ++it)
      if (it->first == 1)
          m.erase(it);  // Invalidates it! ++it is UB!
  // Release: sometimes works, sometimes crashes
  // Debug: immediate assertion failure
```

---

## Self-Assessment

### Q1: _GLIBCXX_DEBUG for iterator validity

This example demonstrates three different invalidation scenarios: reallocation from `push_back`, element shifting from `erase`, and the correct erase-in-loop pattern. The correct version using the return value of `erase` is worth studying carefully - it is the standard-safe pattern that avoids the bug entirely.

```cpp
// Compile: g++ -std=c++20 -D_GLIBCXX_DEBUG -g iter_debug.cpp
#include <iostream>
#include <vector>
#include <map>

int main() {
    // === Iterator invalidation after push_back ===
    {
        std::vector<int> v = {1, 2, 3};
        auto it = v.begin();
        std::cout << "Before push_back: *it = " << *it << '\n';

        v.push_back(4);  // may reallocate

        // With _GLIBCXX_DEBUG: next line aborts!
        // "attempt to dereference a singular iterator"
        // std::cout << *it;  // Uncomment to test

        std::cout << "push_back may invalidate all iterators\n";
    }

    // === Iterator invalidation after erase ===
    {
        std::vector<int> v = {1, 2, 3, 4, 5};
        auto it = v.begin() + 2;  // points to 3
        v.erase(v.begin());  // elements shift left

        // it is now invalid (pointed to index 2, but elements shifted)
        // std::cout << *it;  // _GLIBCXX_DEBUG catches this!

        std::cout << "erase invalidates iterators at/after erased pos\n";
    }

    // === Safe erase pattern ===
    {
        std::map<int, std::string> m = {{1,"a"}, {2,"b"}, {3,"c"}};

        // WRONG: erase invalidates it, then ++it is UB
        // for (auto it = m.begin(); it != m.end(); ++it)
        //     if (it->first == 2) m.erase(it);  // BUG!

        // CORRECT: erase returns next valid iterator
        for (auto it = m.begin(); it != m.end(); ) {
            if (it->first == 2)
                it = m.erase(it);  // returns next iterator
            else
                ++it;
        }

        for (auto& [k, v] : m)
            std::cout << k << ":" << v << " ";
        std::cout << '\n';
    }

    std::cout << "\nIterator invalidation rules:\n";
    std::cout << "  vector push_back: all iterators if realloc\n";
    std::cout << "  vector insert/erase: at/after position\n";
    std::cout << "  deque insert/erase: all iterators\n";
    std::cout << "  map/set erase: only erased element\n";
    std::cout << "  unordered_map rehash: all iterators\n";
}
```

The "WRONG" erase pattern in the map example is a bug that often goes undetected in release builds because `std::map` iterators are not invalidated by erasing other elements - only the erased element's iterator becomes invalid. The code happens to work most of the time because the map node was erased before `++it` ran, but the behavior is technically undefined and the debug mode correctly flags it.

### Q2: libc++ hardening mode preconditions

The libc++ hardening system gives you four distinct levels rather than a single on/off switch. This is useful because you can match the level to the build type - more thorough checking during development, lighter checking in production, no checking only for measured hot paths.

```cpp
// Compile with libc++:
// clang++ -std=c++20 -stdlib=libc++ \
//   -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_EXTENSIVE precond.cpp

#include <iostream>
#include <vector>
#include <optional>
#include <string_view>

int main() {
    // libc++ precondition checks per hardening level:

    // FAST (production-safe, ~1-5% overhead):
    //   - Container operator[] bounds
    //   - front()/back() on empty
    //   - optional dereference without value
    //   - span bounds

    // EXTENSIVE (testing, ~10-30% overhead):
    //   + Algorithm preconditions (sorted input for binary_search)
    //   + Regex category checks
    //   + String null-pointer checks

    // DEBUG (development, ~50-200% overhead):
    //   + Full iterator tracking (invalidation detection)
    //   + Move-from-a-moved-object detection
    //   + Detailed diagnostics

    std::vector<int> v = {1, 2, 3};
    std::optional<int> opt;

    // FAST catches: v[10], opt.value() without value
    // EXTENSIVE catches: binary_search on unsorted range
    // DEBUG catches: using iterator after push_back

    std::cout << "Hardening mode comparison:\n";
    std::cout << "  FAST:      best for production deployments\n";
    std::cout << "  EXTENSIVE: best for CI/staging\n";
    std::cout << "  DEBUG:     best for local development\n";
    std::cout << "  NONE:      only for benchmarks (rare)\n";
}
```

"Move-from-a-moved-object detection" in the DEBUG level is worth calling out. After you move from an object, the standard says that object is in a "valid but unspecified state." Using it may or may not crash. The DEBUG level tracks moved-from objects and aborts if you try to use them as if they were intact - catching a whole category of use-after-move bugs that are otherwise invisible.

### Q3: OOB caught in debug, silent in release

This example captures what real production bugs often look like: an off-by-one loop error that "works" in most cases because the adjacent heap memory happens to contain benign values, and only fails mysteriously under specific conditions.

```cpp
#include <iostream>
#include <vector>
#include <string>

int main() {
    // This code has a bug that only manifests in certain conditions
    std::vector<int> data = {100, 200, 300};

    // Off-by-one error: loop should be i < data.size()
    for (size_t i = 0; i <= data.size(); ++i) {
        // When i == 3 (data.size()):
        //   Release: reads whatever is after the vector's buffer
        //            (often 0 or garbage, may "work" for years)
        //   Debug:   ABORT! index 3 out of range for size 3

        if (i < data.size())  // Safe guard for demo
            std::cout << "data[" << i << "] = " << data[i] << '\n';
    }

    // Another common bug: empty string access
    std::string name;
    // char first = name[0];   // Release: reads '\0' or garbage
                               // Debug: ABORT!

    // Why this matters:
    std::cout << "\nIn release mode, OOB reads often 'work':\n";
    std::cout << "  - Reads adjacent heap memory\n";
    std::cout << "  - Returns garbage that happens to be valid\n";
    std::cout << "  - Bug goes undetected for years\n";
    std::cout << "  - Then exploited: Heartbleed read 64KB past buffer\n";
    std::cout << "\nWith debug/hardened mode:\n";
    std::cout << "  - Caught immediately during development\n";
    std::cout << "  - Clear error message with location\n";
    std::cout << "  - Bug fixed before it reaches production\n";
}
```

The Heartbleed bug is the canonical example here: a missing bounds check in a TLS heartbeat handler let attackers read up to 64 KB of the server's heap memory per request, including private keys, session tokens, and passwords. The fix was one added bounds check. The lesson is that "my tests never showed a problem" is not the same as "the code is correct" - hardened mode turns those silent heap reads into immediate, loud failures.

---

## Notes

- Complementary to #558 (production flags + perf) and #650 (debug build + demos).
- `_GLIBCXX_DEBUG` adds a wrapper around every container - ABI-incompatible with release builds.
- MSVC `/sdl` (Security Development Lifecycle) enables some checks similarly.
- GCC 14+: `-D_GLIBCXX_ASSERTIONS` is enabled by default in some distros.
- Always enable at least `_GLIBCXX_ASSERTIONS` or `FAST` hardening in production.
