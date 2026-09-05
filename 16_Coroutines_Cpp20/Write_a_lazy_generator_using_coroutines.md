# Write a lazy generator using coroutines

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

A **lazy generator** produces values one at a time using `co_yield`, only computing the next value when requested. C++23 adds `std::generator<T>` in `<generator>`, but you can implement one from scratch in C++20.

The "lazy" part is important: a lazy generator never runs ahead of the caller. It suspends after every value and waits to be asked for the next one. This makes infinite sequences perfectly practical - you just stop asking when you have enough values.

### Generator vs Eager Collection

Here is the memory model difference between filling a vector upfront and using a lazy generator:

```cpp
Eager (vector):  [0] [1] [2] [3] [4] ... [999999]  <- all in memory
                  allocated upfront, O(N) memory

Lazy (generator): -- 0 -- 1 -- 2 -- 3 -- ...  <- one value at a time
                  computed on demand, O(1) memory
```

For large or infinite sequences, the generator wins decisively. You only ever hold one value at a time.

### std::generator (C++23)

C++23 gives you a ready-made generator type so you do not have to write the boilerplate yourself. If your compiler supports it, this is the cleanest option:

```cpp
#include <generator>

std::generator<int> iota(int start = 0) {
    while (true)
        co_yield start++;
}

for (int x : iota() | std::views::take(5))
    std::cout << x << ' ';  // 0 1 2 3 4
```

The rest of this file shows how to build the same machinery in C++20 so you understand what is happening under the hood.

---

## Self-Assessment

### Q1: Implement a Fibonacci generator using `co_yield` that produces values on demand

The generator here is a full C++20 implementation that also supports range-based `for` loops. The `iterator` struct is what makes `for (long long f : fibonacci())` work - it delegates to the coroutine handle under the hood. Each `operator++` resumes the coroutine, which runs until the next `co_yield` and then suspends again.

```cpp
#include <coroutine>
#include <iostream>
#include <utility>

// Custom Generator<T> type for C++20
template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;

        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        std::suspend_always yield_value(T value) {
            current_value = std::move(value);
            return {};  // suspend after each yield
        }

        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

private:
    std::coroutine_handle<promise_type> handle_;

    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}

public:
    Generator(Generator&& o) noexcept : handle_(std::exchange(o.handle_, {})) {}
    ~Generator() { if (handle_) handle_.destroy(); }

    Generator(const Generator&) = delete;
    Generator& operator=(const Generator&) = delete;

    // Iterator interface for range-for
    struct sentinel {};
    struct iterator {
        std::coroutine_handle<promise_type> handle;

        iterator& operator++() {
            handle.resume();
            return *this;
        }

        T operator*() const { return handle.promise().current_value; }

        bool operator==(sentinel) const { return handle.done(); }
    };

    iterator begin() {
        handle_.resume();  // run to first co_yield
        return {handle_};
    }

    sentinel end() { return {}; }
};

// Fibonacci generator - infinite sequence
Generator<long long> fibonacci() {
    long long a = 0, b = 1;
    while (true) {
        co_yield a;
        long long next = a + b;
        a = b;
        b = next;
    }
}

// Primes generator using trial division
Generator<int> primes() {
    co_yield 2;
    for (int n = 3; ; n += 2) {
        bool is_prime = true;
        for (int d = 3; d * d <= n; d += 2) {
            if (n % d == 0) { is_prime = false; break; }
        }
        if (is_prime) co_yield n;
    }
}

int main() {
    std::cout << "Fibonacci: ";
    int count = 0;
    for (long long f : fibonacci()) {
        std::cout << f << ' ';
        if (++count >= 12) break;
    }
    std::cout << '\n';

    std::cout << "Primes:    ";
    count = 0;
    for (int p : primes()) {
        std::cout << p << ' ';
        if (++count >= 10) break;
    }
    std::cout << '\n';
}
// Expected output:
// Fibonacci: 0 1 1 2 3 5 8 13 21 34 55 89
// Primes:    2 3 5 7 11 13 17 19 23 29
```

Both `fibonacci()` and `primes()` are infinite loops - they never stop yielding values on their own. The `break` in the range-for loop exits early, and the generator's RAII destructor cleans up the coroutine frame cleanly at that point. You never have to worry about the "rest" of the sequence leaking.

### Q2: Explain how the coroutine frame is allocated on the heap and when it can be elided

When you call a coroutine function, the compiler allocates a frame on the heap to store everything the coroutine needs to resume: its local variables, its current suspension point, the promise object, and the original parameters. Here is what that looks like for a Fibonacci generator:

```cpp
Generator<int> gen = fibonacci();
    │
    ├─ Compiler calls operator new(frame_size)
    │   Frame contains:
    │     • promise_type object (with current_value)
    │     • Local variables (a, b, next)
    │     • Suspension point index (which co_yield we're at)
    │     • Parameters (none for fibonacci)
    │
    ├─ get_return_object() creates Generator with handle
    ├─ initial_suspend() = suspend_always -> paused
    │
    ├─ [resumed] body runs to first co_yield -> paused
    ├─ [resumed] body runs to second co_yield -> paused
    │   ...
    ├─ [destroyed] operator delete(frame)
```

The heap allocation is the normal case. However, the compiler is allowed to elide it under a rule called **HALO (Heap Allocation eLision Optimization)** when it can prove the frame's lifetime is fully contained within the calling scope. This matters for performance-sensitive code:

```cpp
// HALO eligible: generator consumed and destroyed in same scope
int sum_first_10() {
    int total = 0;
    int count = 0;
    for (int f : fibonacci()) {  // generator created here
        total += f;
        if (++count >= 10) break;
    }                             // generator destroyed here
    return total;                 // frame never escaped
}

// HALO NOT eligible: generator returned (escapes scope)
Generator<int> make_fib() {
    return fibonacci();  // handle escapes to caller
}
```

The reason HALO cannot kick in for `make_fib` is that the compiler cannot see when the returned generator will be destroyed - it could be stored anywhere. When the frame is used and destroyed within the same function, the compiler has full visibility and can put the frame on the stack instead of the heap.

### Q3: Compare a coroutine generator with a manual iterator class for the same sequence

This is a useful comparison because it shows exactly what the compiler is doing for you when you write a generator. The squares sequence is simple enough to implement both ways in a short space:

**Coroutine generator (concise):**

```cpp
Generator<int> squares(int n) {
    for (int i = 0; i < n; ++i)
        co_yield i * i;
}
// 5 lines of logic
```

**Manual iterator class (verbose):**

```cpp
class SquareIterator {
    int current_ = 0;
    int limit_;

public:
    using value_type = int;
    using difference_type = std::ptrdiff_t;

    explicit SquareIterator(int n) : limit_(n) {}

    struct sentinel {};

    int operator*() const { return current_ * current_; }

    SquareIterator& operator++() {
        ++current_;
        return *this;
    }

    bool operator==(sentinel) const { return current_ >= limit_; }
    bool operator!=(sentinel s) const { return !(*this == s); }
};

class SquareRange {
    int n_;
public:
    explicit SquareRange(int n) : n_(n) {}
    SquareIterator begin() const { return SquareIterator(n_); }
    SquareIterator::sentinel end() const { return {}; }
};
// ~25 lines of boilerplate
```

Both produce identical output. The manual iterator stores its state as member variables and advances it in `operator++`. The coroutine generator stores its state implicitly in the coroutine frame and advances by resuming to the next `co_yield`. The coroutine version lets you write the algorithm as a loop - which is usually how you naturally think about it - rather than inverting it into an iterator state machine.

```cpp
#include <iostream>

int main() {
    // Both produce the same output:
    std::cout << "Generator: ";
    for (int x : squares(6))
        std::cout << x << ' ';
    std::cout << '\n';

    std::cout << "Iterator:  ";
    for (int x : SquareRange(6))
        std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Generator: 0 1 4 9 16 25
// Iterator:  0 1 4 9 16 25
```

| Aspect | Coroutine Generator | Manual Iterator |
| --- | --- | --- |
| Lines of code | ~5 | ~25 |
| Readability | Natural (sequential) | Complex (state machine) |
| State management | Automatic (compiler) | Manual (member variables) |
| Performance | Slight overhead (frame, resume) | Zero overhead |
| Composability | Works with range-for | Works with range-for |
| Infinite sequences | Easy (just don't stop) | Easy (no sentinel check) |
| C++23 `std::generator` | Drop-in replacement | Still manual |

---

## Notes

- C++23's `std::generator<T>` provides a ready-made generator type - no need to write your own.
- `std::generator<T>` supports `std::ranges::range`, so it works with all range adaptors.
- For recursive generators, `std::generator<T>` supports `co_yield std::ranges::elements_of(sub_gen)`.
- Generator coroutines are always **lazy** (initial_suspend = suspend_always) and **pull-based** (caller drives iteration).
- Break from a range-for loop properly cleans up the generator via RAII.
