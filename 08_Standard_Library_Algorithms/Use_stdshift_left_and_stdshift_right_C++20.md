# Use std::shift_left and std::shift_right (C++20)

**Category:** Standard Library Algorithms  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/shift>  

---

## Topic Overview

`std::shift_left` and `std::shift_right` (C++20) move elements within a range by a specified number of positions. Unlike `std::rotate`, shifted-out elements are gone - there's no wrapping. The vacated positions at the opposite end are left in a valid-but-unspecified (moved-from) state.

### How They Work

Think of a shift as pushing elements in one direction. Elements that fall off the edge are lost, and the positions they vacated are "holes" you need to overwrite or resize away:

```text
shift_left(first, last, n):
  Before:  [A, B, C, D, E]
  After:   [C, D, E, ?, ?]    <- A,B lost; last 2 positions are moved-from
  Returns: iterator to new end (past the last shifted element)

shift_right(first, last, n):
  Before:  [A, B, C, D, E]
  After:   [?, ?, A, B, C]    <- D,E lost; first 2 positions are moved-from
  Returns: iterator to new begin
```

### shift vs rotate

| Feature | `shift_left/right` | `rotate` |
| --- | --- | --- |
| Data loss | Yes - shifted elements lost | No - cyclic permutation |
| Vacated positions | Moved-from state | Contains wrapped elements |
| Use case | Queue-like discard | Circular reorder |
| Complexity | O(n) | O(n) |

---

## Self-Assessment

### Q1: Use std::shift_left to implement a rotating buffer that discards the oldest entries

`shift_left` is the natural primitive for a fixed-size log buffer: when the buffer is full and a new entry arrives, shift everything left by one to drop the oldest, then write the new entry into the last position.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === Basic shift_left ===
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};

    std::cout << "Before shift_left(3): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    auto new_end = std::shift_left(v.begin(), v.end(), 3);
    // Elements 4,5,6,7,8 moved to front; last 3 positions are moved-from

    std::cout << "After:                ";
    for (auto it = v.begin(); it != new_end; ++it)
        std::cout << *it << " ";
    std::cout << "\n";
    // 4 5 6 7 8

    // === Log buffer: keep last N entries, discard oldest ===
    std::vector<std::string> log_buffer = {
        "event_1", "event_2", "event_3", "event_4", "event_5"
    };
    constexpr size_t MAX_LOG = 5;

    auto add_log = [&](const std::string& entry) {
        if (log_buffer.size() >= MAX_LOG) {
            // Shift left by 1 (discard oldest)
            auto new_end = std::shift_left(log_buffer.begin(), log_buffer.end(), 1);
            // Overwrite the last (moved-from) position
            *(new_end - 1) = entry;
            // Or: shift_left, resize, push_back
        } else {
            log_buffer.push_back(entry);
        }
    };

    add_log("event_6");
    add_log("event_7");

    std::cout << "\nLog buffer after adds:\n";
    for (auto& e : log_buffer) std::cout << "  " << e << "\n";
    // event_3, event_4, event_5, event_6, event_7

    return 0;
}
```

The key detail is that `shift_left` returns an iterator to the new logical end - so `*(new_end - 1)` is exactly the moved-from position that you want to overwrite with the incoming entry. Always use the returned iterator to find the boundary; don't guess based on arithmetic.

### Q2: Show the difference between shift and rotate for in-place element movement

The fundamental distinction is whether you want a lossless reorder (rotate) or a destructive eviction (shift). Both run in O(n), but they answer different questions.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

void print(const char* label, const std::vector<int>& v) {
    std::cout << label << ": ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
}

int main() {
    // === shift_left: elements "fall off" the left side ===
    std::vector<int> v1 = {1, 2, 3, 4, 5};
    auto new_end = std::shift_left(v1.begin(), v1.end(), 2);
    std::cout << "shift_left(2):\n";
    std::cout << "  Valid range: ";
    for (auto it = v1.begin(); it != new_end; ++it) std::cout << *it << " ";
    std::cout << "\n";  // 3 4 5
    // v1 is now [3, 4, 5, ?, ?] - last 2 are moved-from

    // === rotate: cyclic - nothing is lost ===
    std::vector<int> v2 = {1, 2, 3, 4, 5};
    std::rotate(v2.begin(), v2.begin() + 2, v2.end());
    print("  rotate(2)  ", v2);  // 3 4 5 1 2

    std::cout << "\n";

    // === shift_right: elements "fall off" the right side ===
    std::vector<int> v3 = {1, 2, 3, 4, 5};
    auto new_begin = std::shift_right(v3.begin(), v3.end(), 2);
    std::cout << "shift_right(2):\n";
    std::cout << "  Valid range: ";
    for (auto it = new_begin; it != v3.end(); ++it) std::cout << *it << " ";
    std::cout << "\n";  // 1 2 3
    // v3 is now [?, ?, 1, 2, 3] - first 2 are moved-from

    // === Summary ===
    std::cout << "\n=== Key Difference ===\n";
    std::cout << "shift:  destructive - shifted-out elements are lost\n";
    std::cout << "rotate: cyclic - all elements preserved, just reordered\n";

    // === Edge case: shift by 0 = no-op ===
    std::vector<int> v4 = {1, 2, 3};
    std::shift_left(v4.begin(), v4.end(), 0);
    print("shift(0)", v4);  // 1 2 3

    // === Edge case: shift by >= size = empty ===
    auto end5 = std::shift_left(v4.begin(), v4.end(), 10);
    std::cout << "shift(10) valid count: "
              << std::distance(v4.begin(), end5) << "\n";  // 0

    return 0;
}
```

After `shift_left(2)` on `{1,2,3,4,5}`, the vector physically holds `{3,4,5,?,?}` but the returned `new_end` tells you only the first three positions are valid. After `rotate` by 2 on the same input, the vector holds `{3,4,5,1,2}` - all five values are still there, just reordered.

### Q3: Demonstrate using shift_left as a building block for erasing elements from the front of a vector

Erasing from the front of a `vector` is O(n) because every subsequent element has to move. `shift_left` is exactly that move step, and it lets you control when the resize happens.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

// Erase N elements from the front of a vector efficiently
template <typename T>
void erase_front(std::vector<T>& v, size_t n) {
    if (n >= v.size()) {
        v.clear();
        return;
    }
    std::shift_left(v.begin(), v.end(), n);
    v.resize(v.size() - n);
}

int main() {
    // === erase_front using shift_left ===
    std::vector<int> v = {10, 20, 30, 40, 50, 60};

    std::cout << "Before: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    erase_front(v, 3);

    std::cout << "After erase_front(3): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 40 50 60

    // === Compare with v.erase(begin, begin+n) ===
    // v.erase already does shift_left internally,
    // but shift_left gives you control over when to resize

    // === Practical: process and discard batches ===
    std::vector<std::string> message_queue = {
        "msg1", "msg2", "msg3", "msg4", "msg5", "msg6"
    };

    // Process first 2 messages, then discard them
    std::cout << "\nProcessing: ";
    for (size_t i = 0; i < 2; ++i)
        std::cout << message_queue[i] << " ";
    std::cout << "\n";

    erase_front(message_queue, 2);

    std::cout << "Queue after: ";
    for (auto& m : message_queue) std::cout << m << " ";
    std::cout << "\n";
    // msg3 msg4 msg5 msg6

    // === shift_right + assign for prepending ===
    std::vector<int> data = {3, 4, 5};
    data.resize(data.size() + 2);  // make room
    std::shift_right(data.begin(), data.end(), 2);
    data[0] = 1;
    data[1] = 2;

    std::cout << "\nAfter prepend: ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    return 0;
}
```

The prepend pattern at the end is a nice application of `shift_right`: resize the vector to create space at the front, shift the existing elements right by N positions, then write the new elements into positions 0..N-1. This is exactly what `v.insert(v.begin(), ...)` does internally.

---

## Notes

- **C++20 required.** `<algorithm>` header.
- `shift_left` returns an iterator to the new logical end; `shift_right` returns an iterator to the new logical begin.
- **Vacated positions** are in a moved-from state - valid but unspecified. You should overwrite or resize.
- **n = 0** is a no-op; **n >= size** moves nothing (returns begin for shift_left, end for shift_right).
- `shift_left` requires forward iterators; `shift_right` requires bidirectional iterators.
- For queue-like FIFO behavior, prefer `std::deque` which has O(1) `pop_front`. Use `shift_left` on vectors only when deque isn't an option.
