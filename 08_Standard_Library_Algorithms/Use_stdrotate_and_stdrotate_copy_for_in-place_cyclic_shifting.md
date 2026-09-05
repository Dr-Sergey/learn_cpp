# Use std::rotate and std::rotate_copy for in-place cyclic shifting

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/rotate>  

---

## Topic Overview

`std::rotate` performs an in-place cyclic permutation that moves a "middle" element to the front position. It's one of the most powerful and underappreciated standard algorithms - it looks like a small utility but turns up in a surprising number of real problems.

### How It Works

The three-iterator interface is the key thing to internalize. You give it a range and say "I want this element to be at the front":

```text
rotate(first, middle, last)

Before: [first ... middle-1 | middle ... last-1]
After:  [middle ... last-1  | first ... middle-1]
        ^returned iterator points here
```

```text
Example: rotate by 2  (left-rotate)
Before: [1, 2, 3, 4, 5]
         f     m        l
After:  [3, 4, 5, 1, 2]
                  ^ returned iterator
```

The returned iterator is easy to overlook but genuinely useful - it marks the "seam" in the result, pointing to where the original `first` element ended up.

### Signatures

```cpp
#include <algorithm>

// In-place rotation
auto new_middle = std::rotate(first, middle, last);

// Non-modifying: writes to output
auto out = std::rotate_copy(first, middle, last, dest);
```

### Complexity

| Algorithm | Time | Space | Swaps |
| --- | --- | --- | --- |
| `std::rotate` | O(n) | O(1) | At most n |
| Naive shift (K times) | O(n x K) | O(1) | n x K |
| Copy-based | O(n) | O(n) | 0 (copies) |

---

## Self-Assessment

### Q1: Use std::rotate to left-rotate a vector by K positions efficiently

The key idea is that left-rotating by K means you want the element currently at position K to move to position 0 - which is exactly what you express as `middle = begin + K`.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7};
    int K = 3;

    std::cout << "Before: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Left-rotate by K: element at position K moves to position 0
    std::rotate(v.begin(), v.begin() + K, v.end());

    std::cout << "After rotate left by " << K << ": ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 4 5 6 7 1 2 3

    // === Right-rotate by K ===
    v = {1, 2, 3, 4, 5, 6, 7};
    // Right-rotate is left-rotate by (n - K)
    std::rotate(v.begin(), v.end() - K, v.end());

    std::cout << "After rotate right by " << K << ": ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 5 6 7 1 2 3 4

    // === Handle K > n with modulo ===
    v = {1, 2, 3, 4, 5};
    K = 12;  // K > size
    int effective_k = K % static_cast<int>(v.size());
    if (effective_k > 0)
        std::rotate(v.begin(), v.begin() + effective_k, v.end());

    std::cout << "After rotate left by " << K << " (mod 5 = " << effective_k << "): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 3 4 5 1 2

    // === rotate_copy: non-modifying ===
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> dst(src.size());
    std::rotate_copy(src.begin(), src.begin() + 2, src.end(), dst.begin());

    std::cout << "\nSource:      ";
    for (int x : src) std::cout << x << " ";  // unchanged: 1 2 3 4 5
    std::cout << "\nRotate copy: ";
    for (int x : dst) std::cout << x << " ";  // 3 4 5 1 2
    std::cout << "\n";

    return 0;
}
```

The modulo trick for K > n is worth remembering - rotating by 12 positions in a 5-element range is identical to rotating by 2 positions, so always reduce first.

### Q2: Implement a circular buffer drain using rotate to bring the oldest element to the front

A ring buffer stores elements by wrapping around, so the physical layout in memory gets out of chronological order once the buffer fills. `std::rotate` is the standard way to fix that in a single O(n) pass.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

// A simple ring buffer that uses rotate to linearize for output
class RingBuffer {
    std::vector<std::string> buf;
    size_t head = 0;  // next write position
    size_t count = 0;

public:
    explicit RingBuffer(size_t capacity) : buf(capacity) {}

    void push(std::string val) {
        buf[head] = std::move(val);
        head = (head + 1) % buf.size();
        if (count < buf.size()) ++count;
    }

    // Drain: return all elements in chronological order (oldest first)
    std::vector<std::string> drain() {
        // Current layout: [...new... | ...old...]
        //                  0    head   ...     count
        // We need to rotate so oldest comes first

        std::vector<std::string> result(buf.begin(), buf.begin() + count);

        if (count == buf.size()) {
            // Buffer is full: head points to oldest element
            std::rotate(result.begin(), result.begin() + head, result.end());
        }
        // If not full, elements are already in order [0..count)

        return result;
    }

    void print() {
        auto items = drain();
        for (auto& s : items) std::cout << s << " ";
        std::cout << "\n";
    }
};

int main() {
    RingBuffer rb(5);

    rb.push("A"); rb.push("B"); rb.push("C");
    std::cout << "After 3 pushes: ";
    rb.print();  // A B C

    rb.push("D"); rb.push("E"); rb.push("F"); rb.push("G");
    std::cout << "After 7 pushes (cap=5): ";
    rb.print();  // C D E F G  (oldest 2 overwritten)

    // === Another use: move an element to the front ===
    std::vector<int> v = {1, 2, 3, 4, 5};
    auto it = std::find(v.begin(), v.end(), 4);
    std::rotate(v.begin(), it, it + 1);  // move 4 to front
    std::cout << "\nMove 4 to front: ";
    for (int x : v) std::cout << x << " ";  // 4 1 2 3 5
    std::cout << "\n";

    // === Insert at position using rotate ===
    // 1. push_back new element
    // 2. rotate it from end to desired position
    std::vector<int> nums = {10, 20, 30, 40};
    nums.push_back(25);  // append
    // Now rotate 25 from end to between 20 and 30
    std::rotate(nums.begin() + 2, nums.end() - 1, nums.end());
    std::cout << "Insert 25: ";
    for (int x : nums) std::cout << x << " ";  // 10 20 25 30 40
    std::cout << "\n";

    return 0;
}
```

The "insert at position" trick at the end is a handy idiom: `push_back` to grow the vector and ensure space, then rotate the new element from the tail to wherever it belongs. This avoids the cost of a mid-vector insert.

### Q3: Show the O(n) complexity of std::rotate vs an O(n x K) naive shift

The naive approach shifts elements one position at a time, K times - that's O(n*K) total moves. `std::rotate` uses the "three reverses" trick to get the same result in O(n) regardless of K.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <numeric>

// Naive left-shift by 1 position
void left_shift_one(std::vector<int>& v) {
    if (v.empty()) return;
    int first = v[0];
    for (size_t i = 0; i + 1 < v.size(); ++i)
        v[i] = v[i + 1];
    v.back() = first;
}

// Naive left-rotate by K: call shift-one K times -> O(n*K)
void naive_rotate(std::vector<int>& v, int k) {
    for (int i = 0; i < k; ++i)
        left_shift_one(v);
}

int main() {
    constexpr int N = 100'000;
    constexpr int K = 30'000;

    // === Benchmark naive O(n*K) ===
    std::vector<int> v1(N);
    std::iota(v1.begin(), v1.end(), 0);

    auto t1 = std::chrono::high_resolution_clock::now();
    naive_rotate(v1, K);
    auto t2 = std::chrono::high_resolution_clock::now();
    auto naive_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // === Benchmark std::rotate O(n) ===
    std::vector<int> v2(N);
    std::iota(v2.begin(), v2.end(), 0);

    t1 = std::chrono::high_resolution_clock::now();
    std::rotate(v2.begin(), v2.begin() + K, v2.end());
    t2 = std::chrono::high_resolution_clock::now();
    auto rotate_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    std::cout << "N=" << N << ", K=" << K << "\n";
    std::cout << "Naive:       " << naive_ms << " ms\n";
    std::cout << "std::rotate: " << rotate_ms << " ms\n";

    // Verify both produce the same result
    bool match = (v1 == v2);
    std::cout << "Results match: " << std::boolalpha << match << "\n";

    // === How std::rotate achieves O(n) ===
    // The "three reverses" algorithm:
    //   reverse(first, middle);
    //   reverse(middle, last);
    //   reverse(first, last);
    // Each reverse is O(n), so total is O(n) - not O(n*K)!
    //
    // Example: [1,2,3|4,5]
    // reverse first: [3,2,1|4,5]
    // reverse second: [3,2,1|5,4]
    // reverse all: [4,5,1,2,3] -- correct

    return 0;
}
```

The three-reverses insight is genuinely elegant: each reverse is O(n), you do three of them, and the constant factor is small enough that the total beats 30,000 single-element shifts by a large margin.

---

## Notes

- **The returned iterator** from `rotate(first, mid, last)` points to the new position of the element that was previously at `first`. This is useful for subsequent operations.
- **Three common patterns with rotate:**
  1. **Left/right rotation** of arrays
  2. **Move element to front/back** - `rotate(begin, target, target+1)` moves `*target` to front
  3. **Insert at position** - `push_back()` then `rotate(pos, end-1, end)`
- `std::rotate` works with **forward iterators** (not just random access), thanks to different internal algorithms for different iterator categories.
- **The "three reverses" trick** is the classic O(n) implementation for random-access iterators.
