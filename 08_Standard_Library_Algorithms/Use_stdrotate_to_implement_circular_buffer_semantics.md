# Use std::rotate to implement circular buffer semantics

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/rotate>  

---

## Topic Overview

This file focuses on **using `std::rotate` as a building block for circular buffer patterns** - linearizing ring buffers, understanding the returned iterator, and implementing sliding windows.

### The Circular Buffer Problem

A circular (ring) buffer writes to positions `head % capacity`. When you need to read elements in chronological order, the physical layout doesn't match the logical order. `std::rotate` fixes this in O(n), turning the wrapped-around mess into a clean contiguous sequence.

```text
Physical: [new_3, new_4 | old_1, old_2]
                   ^head
After rotate:
Logical:  [old_1, old_2, new_3, new_4]  <- chronological order
```

### The Return Value

The return value of `std::rotate` is easy to overlook, but it carries real information:

```cpp
auto new_pos = std::rotate(first, middle, last);
// new_pos points to where *first ended up (i.e., first + (last - middle))
```

This returned iterator is the "seam" between the two halves after rotation. It tells you exactly where the original left part begins in the new layout - which is useful for chaining additional operations on just one half.

---

## Self-Assessment

### Q1: Use std::rotate to bring the logical beginning of a circular buffer to physical position 0

When a ring buffer wraps around, `head` points to the oldest element - making it the natural `middle` argument for `std::rotate`. After rotation, position 0 holds the oldest entry and the buffer is ready to read sequentially.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

// Simulate a circular buffer with a fixed-size vector
class CircularLog {
    std::vector<int> buffer;
    size_t head = 0;       // next write position
    size_t count = 0;

public:
    explicit CircularLog(size_t cap) : buffer(cap, 0) {}

    void push(int val) {
        buffer[head] = val;
        head = (head + 1) % buffer.size();
        if (count < buffer.size()) ++count;
    }

    // Linearize: re-arrange so buffer[0] is the oldest entry
    void linearize() {
        if (count < buffer.size()) return;  // not wrapped yet
        // head points to oldest element (it was overwritten last cycle ago)
        std::rotate(buffer.begin(), buffer.begin() + head, buffer.end());
        head = 0;
    }

    void print() const {
        std::cout << "[";
        for (size_t i = 0; i < count; ++i) {
            if (i > 0) std::cout << ", ";
            // Logical index: oldest first
            size_t idx = (count < buffer.size())
                         ? i
                         : (head + i) % buffer.size();
            std::cout << buffer[idx];
        }
        std::cout << "]\n";
    }

    void print_raw() const {
        std::cout << "Raw: ";
        for (size_t i = 0; i < count; ++i)
            std::cout << buffer[i] << " ";
        std::cout << "(head=" << head << ")\n";
    }
};

int main() {
    CircularLog log(5);

    // Fill beyond capacity
    for (int i = 1; i <= 8; ++i) log.push(i);

    std::cout << "Before linearize:\n";
    log.print_raw();   // Raw: 6 7 8 4 5 (head=3)
    log.print();       // Logical: [4, 5, 6, 7, 8]

    log.linearize();
    std::cout << "\nAfter linearize:\n";
    log.print_raw();   // Raw: 4 5 6 7 8 (head=0)
    log.print();       // Logical: [4, 5, 6, 7, 8]

    return 0;
}
```

After `linearize()` the logical view and the physical layout agree: both start at the oldest element. This makes the buffer safe to pass to any algorithm that expects a plain contiguous range.

### Q2: Show that rotate returns an iterator to the new position of the element that was at begin()

The formula is always `new_pos = first + (last - middle)`. Knowing this lets you chain operations on the two resulting halves without any searching.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {10, 20, 30, 40, 50};

    std::cout << "Before: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // rotate left by 2: move position [2] to front
    auto new_begin_pos = std::rotate(v.begin(), v.begin() + 2, v.end());

    std::cout << "After:  ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 30 40 50 10 20

    // The returned iterator points to where the original begin() element (10) ended up
    std::cout << "*new_begin_pos = " << *new_begin_pos << "\n";  // 10
    std::cout << "Index: " << std::distance(v.begin(), new_begin_pos) << "\n";  // 3
    // new_begin_pos == v.begin() + (v.size() - 2) == v.begin() + 3

    // === Using the returned iterator for chaining ===
    // Example: rotate, then sort only the "old" part
    v = {5, 3, 1, 4, 2};
    auto seam = std::rotate(v.begin(), v.begin() + 2, v.end());
    // v = [1 4 2 | 5 3]   seam points to 5
    std::sort(seam, v.end());
    // v = [1 4 2 | 3 5]   sorted the tail
    std::cout << "\nAfter rotate+sort tail: ";
    for (int x : v) std::cout << x << " ";  // 1 4 2 3 5
    std::cout << "\n";

    // === Formula: new_begin_pos = first + (last - middle) ===
    // This is always true for the return value

    return 0;
}
```

The chaining example makes the seam iterator concrete: after rotating `{5,3,1,4,2}` so that `1` moves to the front, the returned iterator points exactly at the start of the old prefix `{5,3}` - now sitting at the tail. You can then sort just that part without scanning the whole vector.

### Q3: Implement a sliding window by combining rotate with erase and push_back

When a new value arrives and the window is full, the oldest value needs to leave. `std::rotate` left by 1 moves the oldest element to the back, and then you simply overwrite it - all in O(n) with no extra allocation.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

// Sliding window of fixed size over a stream of values
class SlidingWindow {
    std::vector<double> window;
    size_t capacity;

public:
    explicit SlidingWindow(size_t cap) : capacity(cap) {}

    void add(double val) {
        if (window.size() < capacity) {
            window.push_back(val);
        } else {
            // Rotate left by 1 (discard oldest), replace last with new
            std::rotate(window.begin(), window.begin() + 1, window.end());
            window.back() = val;
        }
    }

    double average() const {
        if (window.empty()) return 0.0;
        double sum = std::accumulate(window.begin(), window.end(), 0.0);
        return sum / window.size();
    }

    void print() const {
        std::cout << "[";
        for (size_t i = 0; i < window.size(); ++i) {
            if (i > 0) std::cout << ", ";
            std::cout << window[i];
        }
        std::cout << "] avg=" << average() << "\n";
    }
};

int main() {
    SlidingWindow sw(4);

    // Stream of sensor readings
    double readings[] = {10, 20, 30, 40, 50, 60, 70};

    for (double r : readings) {
        sw.add(r);
        std::cout << "Added " << r << ": ";
        sw.print();
    }
    // Added 10: [10] avg=10
    // Added 20: [10, 20] avg=15
    // Added 30: [10, 20, 30] avg=20
    // Added 40: [10, 20, 30, 40] avg=25
    // Added 50: [20, 30, 40, 50] avg=35  <- 10 dropped
    // Added 60: [30, 40, 50, 60] avg=45
    // Added 70: [40, 50, 60, 70] avg=55

    // === Alternative approach: erase front + push_back ===
    // This is O(n) due to erase, same as rotate approach,
    // but rotate avoids the overhead of moving elements down
    // and then growing from push_back.

    // For a deque, pop_front()/push_back() is O(1) - often better
    // For a vector, rotate is elegant and allocates nothing extra

    return 0;
}
```

The comparison note is worth emphasizing: `erase` at the front also costs O(n) because it shifts every remaining element, so `rotate` is not worse - and it avoids any size change on the vector. If you need true O(1) window updates, `std::deque` with `pop_front`/`push_back` is the right tool instead.

---

## Notes

- **Return value of `rotate`:** Always `first + (last - middle)` - the position where the original first element ends up. This is the "seam" dividing the two rotated halves.
- **Linearizing a ring buffer** is `rotate`'s most practical application - it converts a wrapped circular layout into a contiguous chronological sequence.
- For **sliding windows** on vectors, `rotate` left by 1 + overwrite back is O(n). For O(1) sliding windows, prefer `std::deque` with `pop_front`/`push_back`.
- `rotate` is stable within each half - the relative order of elements in each partition is preserved.
