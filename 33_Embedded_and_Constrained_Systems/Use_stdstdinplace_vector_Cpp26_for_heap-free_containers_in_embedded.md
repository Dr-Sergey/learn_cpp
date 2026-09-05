# Use std::inplace_vector (C++26) for heap-free containers in embedded

**Category:** Embedded and Constrained Systems  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/inplace_vector>  

---

## Topic Overview

### The Problem: `std::vector` Allocates on the Heap

In embedded systems with no heap (`-fno-exceptions`, no `malloc`), `std::vector` is unusable. Developers historically worked around this with:

- Fixed-size `std::array` (no dynamic size - you waste capacity or overflow silently)
- C-style arrays with a separate `count` variable (error-prone, no iterator support)
- Third-party containers like `etl::vector` from the Embedded Template Library

C++26 introduces **`std::inplace_vector<T, N>`** - a standard, heap-free, fixed-capacity vector that stores all elements inline. It is exactly what the embedded world has needed for years, and it is now in the standard.

### What Is `std::inplace_vector`

You use it just like `std::vector`, but the capacity is a compile-time template parameter and the storage lives right inside the object - no heap involved:

```cpp
#include <inplace_vector>

// Stores up to 10 ints - entirely on the stack (or in static memory)
std::inplace_vector<int, 10> v;

v.push_back(1);        // OK
v.push_back(2);        // OK
v.size();              // 2
v.capacity();          // 10 (always, compile-time constant)
v.push_back(42);       // OK
// v is currently {1, 2, 42}

v.try_push_back(99);   // Returns pointer to inserted element, or nullptr if full
```

### Key Properties

Here is how `std::inplace_vector` compares to the alternatives you may already be using:

| Property | `std::vector<T>` | `std::array<T,N>` | `std::inplace_vector<T,N>` |
| --- | --- | --- | --- |
| Capacity | Dynamic (heap) | Fixed = N | Fixed = N |
| Size | Dynamic | Fixed = N | Dynamic (0..N) |
| Heap allocation | Yes | No | No |
| `push_back` | Yes | No | Yes |
| `constexpr` | Partial (C++20) | Full | Full |
| Trivially copyable | No | If T is | If T is |
| `sizeof` | 3 pointers | N x sizeof(T) | N x sizeof(T) + size_type |

The trivially copyable property is worth calling out: if `T` is trivially copyable, so is the whole `inplace_vector`, which means you can `memcpy` it, place it in shared memory, or pass it to DMA - things you cannot do with `std::vector`.

### Basic Usage

Here is a realistic embedded example - a sensor reading buffer that supports push, clear, iteration, and standard algorithms without any heap involvement:

```cpp
#include <inplace_vector>
#include <algorithm>
#include <cstdint>

// Sensor reading buffer - fixed capacity, no heap
struct SensorReading {
    uint32_t timestamp;
    int16_t  value;
    uint8_t  channel;
};

class SensorBuffer {
    std::inplace_vector<SensorReading, 64> readings_;

public:
    bool add(SensorReading r) {
        return readings_.try_push_back(r) != nullptr;
    }

    void clear() { readings_.clear(); }

    [[nodiscard]] size_t count() const { return readings_.size(); }
    [[nodiscard]] bool full() const { return readings_.size() == readings_.capacity(); }

    // Average value for a specific channel
    [[nodiscard]] float average(uint8_t channel) const {
        int32_t sum = 0;
        int32_t count = 0;
        for (const auto& r : readings_) {
            if (r.channel == channel) {
                sum += r.value;
                ++count;
            }
        }
        return count > 0 ? static_cast<float>(sum) / count : 0.0f;
    }

    // Works with standard algorithms
    void sort_by_time() {
        std::sort(readings_.begin(), readings_.end(),
                  [](const auto& a, const auto& b) {
                      return a.timestamp < b.timestamp;
                  });
    }
};
```

Because `inplace_vector` provides standard begin/end iterators and contiguous storage, every `<algorithm>` function works on it without any changes.

### `try_push_back` vs `push_back`

This distinction matters a lot in embedded code. `push_back` throws `std::bad_alloc` when the vector is full - which calls `std::terminate()` under `-fno-exceptions`. `try_push_back` returns a pointer instead, letting you handle the full-buffer case gracefully:

```cpp
std::inplace_vector<int, 4> v = {1, 2, 3, 4};

// push_back on a full vector: throws std::bad_alloc (or terminates with -fno-exceptions)
// v.push_back(5);  // BAD in embedded!

// try_push_back: returns nullptr on failure - no exception
auto* ptr = v.try_push_back(5);
if (!ptr) {
    // Handle full buffer gracefully
    v.erase(v.begin());  // Remove oldest
    v.try_push_back(5);  // Now it fits
}

// try_emplace_back for in-place construction
struct Event {
    uint32_t id;
    const char* name;
};

std::inplace_vector<Event, 16> events;
auto* e = events.try_emplace_back(42, "sensor_fault");
if (e) {
    // Successfully inserted
}
```

The rule of thumb: in embedded code, always use `try_push_back` and `try_emplace_back`. Reserve the throwing variants for hosted tests where exceptions are enabled and you want to detect logic errors.

### Constexpr Usage

`std::inplace_vector` is fully `constexpr`, which means you can build compile-time lookup tables with it. This is impossible with `std::vector` (even in C++20, the result cannot be a `constexpr` variable) and awkward with `std::array` (you need to know the exact size up front):

```cpp
constexpr auto make_primes() {
    std::inplace_vector<int, 20> primes;
    for (int n = 2; primes.size() < 20; ++n) {
        bool is_prime = true;
        for (const auto& p : primes) {
            if (p * p > n) break;
            if (n % p == 0) { is_prime = false; break; }
        }
        if (is_prime) primes.try_push_back(n);
    }
    return primes;
}

// Computed entirely at compile time!
constexpr auto primes = make_primes();
static_assert(primes.size() == 20);
static_assert(primes[0] == 2);
static_assert(primes[19] == 71);
```

This kind of compile-time computation is ideal for CRC tables, sin/cos lookup tables, character classification tables, and similar constant data that was previously generated by external scripts.

### Replacing C-Style Patterns

If you have existing code using raw arrays with a manual count, here is how the migration looks:

```cpp
// BEFORE: C-style fixed buffer (error-prone)
struct CommandQueue_C {
    Command buffer[16];
    int count = 0;

    bool push(Command c) {
        if (count >= 16) return false;
        buffer[count++] = c;
        return true;
    }

    // Easy to forget bounds checking, no iterators, no algorithms...
};

// AFTER: std::inplace_vector (type-safe, STL-compatible)
struct CommandQueue {
    std::inplace_vector<Command, 16> commands;

    bool push(Command c) {
        return commands.try_push_back(c) != nullptr;
    }

    // Get range, sort, find, erase - all work out of the box
    auto pending() const { return std::span(commands); }
};
```

The after version is shorter, has no manual bounds checking to forget, and works with any standard algorithm.

### Using with PMR or Custom Allocators

`std::inplace_vector` does **not** use an allocator - it is always inline. But you can use it as the backing buffer for a PMR monotonic allocator, giving you a stack-allocated arena for other PMR containers:

```cpp
#include <memory_resource>

// Use inplace_vector as the buffer for a monotonic allocator
std::inplace_vector<std::byte, 4096> buffer;
std::pmr::monotonic_buffer_resource resource(
    buffer.data(), buffer.capacity());

std::pmr::vector<int> dynamic_vec(&resource);
// Allocations come from the inplace_vector's storage - still no heap!
```

### Size and Layout

```cpp
// sizeof includes storage for N elements + a size counter
static_assert(sizeof(std::inplace_vector<int, 10>) ==
              10 * sizeof(int) + sizeof(std::size_t));  // Typically

// Trivially copyable if T is trivially copyable
static_assert(std::is_trivially_copyable_v<std::inplace_vector<int, 10>>);

// Can be memcpy'd, placed in shared memory, DMA buffers, etc.
```

---

## Self-Assessment

### Q1: When should you use `try_push_back` instead of `push_back`

**Always in embedded code compiled with `-fno-exceptions`**. `push_back` throws `std::bad_alloc` when full, which causes `std::terminate()` without exception support. `try_push_back` returns a pointer (or `nullptr` on failure) - allowing graceful handling:

```cpp
auto* p = buffer.try_push_back(value);
if (!p) {
    // Strategy: drop oldest, log overflow, return error...
    handle_overflow();
}
```

### Q2: How does `std::inplace_vector` compare to `etl::vector`

Both are fixed-capacity, heap-free vectors. Key differences:

- `std::inplace_vector` is **standardized** (C++26) - portable across compilers without third-party dependencies.
- ETL vector is available **today** for C++11 and up - you do not have to wait for C++26 compiler support.
- `std::inplace_vector` is **fully `constexpr`** - ETL is not.
- `std::inplace_vector` is **trivially copyable** when `T` is, enabling `memcpy` and `bit_cast`.
- ETL has a richer ecosystem (pools, queues, FSMs, etc.) beyond just the vector.

For new projects targeting C++26 compilers, prefer the standard. For projects needing C++14/17 compatibility, ETL remains the best option.

### Q3: Show a compile-time lookup table built with `std::inplace_vector`

```cpp
constexpr auto build_ascii_table() {
    std::inplace_vector<char, 128> printable;
    for (char c = 32; c < 127; ++c) {
        printable.try_push_back(c);
    }
    return printable;
}

constexpr auto printable_ascii = build_ascii_table();
static_assert(printable_ascii.size() == 95);
static_assert(printable_ascii[0] == ' ');
static_assert(printable_ascii[94] == '~');
```

---

## Notes

- Available in GCC 15+ and Clang 19+ with `-std=c++26`.
- `std::inplace_vector` is contiguous - works with `std::span` and C APIs expecting pointers.
- Capacity `N` is a compile-time constant - you cannot change it at runtime.
- For embedded, prefer `try_push_back` / `try_emplace_back` over throwing variants.
- If you need a circular buffer pattern, wrap `std::inplace_vector` or use `etl::circular_buffer`.
