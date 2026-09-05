# Use erase-remove idiom and the erase_if free functions (C++20)

**Category:** Standard Library Algorithms  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/container/vector/erase2>  

---

## Topic Overview

### The Problem

`std::remove` / `std::remove_if` do **not** actually remove elements from a container. They move unwanted elements to the end and return an iterator to the new logical end. The container's size remains unchanged. You must call `.erase()` to actually shrink the container.

This is a frequent source of confusion because the function is named `remove`, yet nothing is removed from the container - it is purely an algorithm operating on iterators that knows nothing about the container itself.

### Classic Erase-Remove Idiom (pre-C++20)

The diagram below shows what each step does to the internal state of the vector:

```cpp
Before remove_if (remove odds):  [1, 2, 3, 4, 5, 6]
After  remove_if:                [2, 4, 6, ?, ?, ?]  <- "?" = unspecified values
                                          ^
                                     returned iterator (new logical end)
After .erase(new_end, old_end):  [2, 4, 6]  <- container actually shrunk
```

```cpp
// The classic two-step idiom:
v.erase(std::remove_if(v.begin(), v.end(), pred), v.end());
```

### C++20 Free Functions: `std::erase` and `std::erase_if`

C++20 adds free-standing `std::erase` and `std::erase_if` for all standard containers, which collapse the two-step idiom into a single call:

| Container | `std::erase(container, value)` | `std::erase_if(container, pred)` |
| --- | --- | --- |
| `vector` | Yes | Yes |
| `deque` | Yes | Yes |
| `list` | Yes | Yes |
| `string` | Yes | Yes |
| `set/map` | - | Yes |
| `unordered_set/map` | - | Yes |

Both return the number of elements erased (`size_t`).

---

## Self-Assessment

### Q1: Write the classic erase(remove_if(...), end()) pattern and explain why both calls are needed

The reason two separate calls are needed comes down to a fundamental design principle: algorithms operate on iterator ranges and know nothing about the container, while containers know nothing about algorithms. Each does only its half of the job.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === Classic erase-remove idiom ===
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Step 1: remove_if moves "kept" elements to the front
    // Returns iterator to the new logical end
    auto new_end = std::remove_if(v.begin(), v.end(),
                                  [](int x) { return x % 2 != 0; });

    std::cout << "After remove_if (size unchanged): " << v.size() << "\n";
    // Size is still 10! The last 5 positions have unspecified values.

    std::cout << "Elements before new_end: ";
    for (auto it = v.begin(); it != new_end; ++it)
        std::cout << *it << " ";
    std::cout << "\n";
    // Output: 2 4 6 8 10

    // Step 2: erase from new_end to actual end -> shrinks the vector
    v.erase(new_end, v.end());

    std::cout << "After erase (size shrunk): " << v.size() << "\n";
    // Size is now 5
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: 2 4 6 8 10

    // === Combined in one line (the idiom) ===
    std::vector<std::string> words = {"hello", "", "world", "", "!", ""};
    words.erase(
        std::remove_if(words.begin(), words.end(),
                       [](const std::string& s) { return s.empty(); }),
        words.end()
    );

    for (const auto& w : words) std::cout << "[" << w << "] ";
    std::cout << "\n";
    // Output: [hello] [world] [!]

    return 0;
}
```

Why both calls are needed: **`std::remove_if`** is an algorithm - it operates on iterator ranges and knows nothing about the container. It can only overwrite elements by moving the "kept" ones forward. It cannot resize the container. **`container.erase()`** is a container member function - only the container itself can deallocate storage and update its size. If you skip the `erase()` call, the vector still has its original size with moved-from values sitting in the tail positions.

### Q2: Replace the erase-remove idiom with std::erase_if (C++20) and compare readability

The C++20 versions remove an entire category of bugs - you cannot forget the second step because there is only one step.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <list>
#include <map>
#include <set>

int main() {
    // === C++20 std::erase_if — one-liner replacement ===
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Old way:
    // v.erase(std::remove_if(v.begin(), v.end(), [](int x) { return x % 2 != 0; }), v.end());

    // C++20 way:
    auto count = std::erase_if(v, [](int x) { return x % 2 != 0; });
    std::cout << "Removed " << count << " odd numbers\n";  // Removed 5
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";  // 2 4 6 8 10

    // === std::erase for value-based removal ===
    std::vector<int> v2 = {1, 3, 3, 7, 3, 5};
    auto removed = std::erase(v2, 3);
    std::cout << "Erased " << removed << " threes\n";  // Erased 3
    // v2 = {1, 7, 5}

    // === Works on strings ===
    std::string s = "Hello, World!";
    std::erase_if(s, [](char c) { return !std::isalpha(c); });
    std::cout << "Letters only: " << s << "\n";  // HelloWorld

    // === Works on associative containers too ===
    std::map<std::string, int> scores = {
        {"Alice", 95}, {"Bob", 42}, {"Charlie", 78}, {"Dave", 33}
    };
    std::erase_if(scores, [](const auto& pair) { return pair.second < 50; });
    std::cout << "Passing students:\n";
    for (const auto& [name, score] : scores)
        std::cout << "  " << name << ": " << score << "\n";
    // Alice: 95, Charlie: 78

    // === Works on list (no erase-remove needed — list has remove_if member) ===
    std::list<int> lst = {1, 2, 3, 4, 5};
    std::erase_if(lst, [](int x) { return x < 3; });
    for (int x : lst) std::cout << x << " ";
    std::cout << "\n";  // 3 4 5

    // === Readability comparison ===
    // OLD (3 concerns mixed together):
    //   v.erase(std::remove_if(v.begin(), v.end(), pred), v.end());
    //
    // NEW (one intent, clearly stated):
    //   std::erase_if(v, pred);
    //
    // The C++20 version is:
    //   - Shorter
    //   - No begin/end boilerplate
    //   - No "forgot the erase" bug
    //   - Works uniformly across ALL container types
    //   - Returns count of removed elements (useful!)

    return 0;
}
```

Notice that `std::erase_if` works uniformly on vectors, maps, sets, lists, and strings - the same call regardless of container type. The old idiom required knowing which variant to use for each container (and lists had a separate `remove_if` member). The returned count of removed elements is also a bonus you did not get from the old idiom.

### Q3: Show a bug where only remove_if is called without the erase step

This is one of the most common bugs when people first encounter these algorithms. The vector looks wrong but does not crash - which makes it harder to debug.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6};

    std::cout << "Before: size = " << v.size() << ", elements: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Before: size = 6, elements: 1 2 3 4 5 6

    // === BUG: forgot the erase step! ===
    std::remove_if(v.begin(), v.end(),
                   [](int x) { return x % 2 != 0; });
    // Return value discarded!

    std::cout << "After remove_if ONLY: size = " << v.size() << ", elements: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // After remove_if ONLY: size = 6, elements: 2 4 6 4 5 6
    //                                                 ^^^^^^^ leftover garbage!
    // The vector still has 6 elements.
    // Positions 0-2 have the "kept" values (2, 4, 6).
    // Positions 3-5 have unspecified values (moved-from, implementation-defined).

    // === Another common bug: discarding the return value ===
    std::vector<int> v2 = {10, 20, 30, 40, 50};
    // This compiles but does NOTHING visible:
    std::remove(v2.begin(), v2.end(), 30);  // [[nodiscard]] in C++20!
    std::cout << "v2 after remove without erase: ";
    for (int x : v2) std::cout << x << " ";
    std::cout << "\n";
    // Output: 10 20 40 50 50  <- still size 5, last element duplicated

    // === Fix: always use the two-step idiom or C++20 std::erase ===
    std::vector<int> v3 = {10, 20, 30, 40, 50};
    // Option 1: Classic idiom (correct)
    v3.erase(std::remove(v3.begin(), v3.end(), 30), v3.end());
    // Option 2: C++20 (best)
    // std::erase(v3, 30);

    std::cout << "v3 fixed: ";
    for (int x : v3) std::cout << x << " ";
    std::cout << "\n";
    // Output: 10 20 40 50

    return 0;
}
```

Here is how the bug manifests: `std::remove_if` shuffles wanted elements to the front but does not change the container's size. The "tail" contains moved-from objects with unspecified values. If you iterate the full container afterward, you process those garbage values as if they were real data. In C++20, `std::remove` and `std::remove_if` are marked `[[nodiscard]]`, so modern compilers will warn if you discard the return value - that warning is a lifesaver.

---

## Notes

- **Always prefer `std::erase` / `std::erase_if` (C++20)** - they are correct by construction and work with all standard containers.
- **Return value:** `std::erase_if` returns `size_t` (the count of erased elements), unlike the classic idiom.
- **For `std::list` and `std::forward_list`:** The member functions `remove()` / `remove_if()` already erase in one step (O(n), no shifting). `std::erase_if` calls them internally.
- **Performance:** `std::erase_if` on vector is O(n) - same as the erase-remove idiom. No magic.
- **For associative containers:** `std::erase_if` is the only convenient way to conditionally erase - the old approach required a manual loop with careful iterator advancement.
- **`std::erase` (by value)** is available for sequence containers and strings, but NOT for associative containers (use `container.erase(key)` instead).
