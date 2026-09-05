# Use views::iota, views::repeat, and other generating views

**Category:** Ranges Cpp20  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/iota_view>  

---

## Topic Overview

Generating views create elements **on the fly** without storing them. They are the building blocks for constructing ranges from scratch - the range equivalent of writing a `for (int i = 0; i < n; ++i)` loop, but composable with the rest of the ranges machinery.

### Generating Views Summary

| View | Standard | What it generates | Finite/Infinite |
| --- | --- | --- | --- |
| `views::iota(start)` | C++20 | `start, start+1, start+2, ...` | Infinite |
| `views::iota(start, end)` | C++20 | `start, start+1, ..., end-1` | Finite |
| `views::repeat(val)` | C++23 | `val, val, val, ...` | Infinite |
| `views::repeat(val, n)` | C++23 | `val` repeated `n` times | Finite |
| `views::empty<T>` | C++20 | Nothing | Finite (0 elements) |
| `views::single(val)` | C++20 | Just `val` | Finite (1 element) |

### iota in Detail

`iota` is the one you will use most often - it is essentially the range version of a counting loop:

```cpp
views::iota(0, 5)    ->  0, 1, 2, 3, 4
views::iota(0)       ->  0, 1, 2, 3, ...  (infinite)
views::iota('a', 'f') ->  'a', 'b', 'c', 'd', 'e'
```

`iota` works with any **weakly_incrementable** type: `int`, `char`, iterators, pointers, custom types with `operator++`.

### repeat in Detail (C++23)

```cpp
views::repeat(42)     ->  42, 42, 42, ...  (infinite)
views::repeat(42, 5)  ->  42, 42, 42, 42, 42
```

`repeat` stores a single copy of the value, yielding a reference to it on each dereference. That means the string `"hello"` in `repeat("hello", 100)` is stored exactly once - you don't pay for 100 copies.

### Infinite Ranges Safety

Infinite ranges must be **truncated** before consuming (with `take`, `take_while`, etc.), or iterated in a bounded loop. Passing an infinite range to `ranges::to<vector>()` would attempt infinite allocation. The compiler won't stop you, so this is a runtime trap to be aware of.

---

## Self-Assessment

### Q1: Use `views::iota(0, 100) | views::filter(is_prime)` to generate primes lazily

Here `iota` replaces the loop counter entirely. The integers are generated on demand, filtered through `is_prime`, and only the passing values flow through to the output - no array of 100 numbers is ever built.

```cpp
#include <cmath>
#include <iostream>
#include <ranges>

bool is_prime(int n) {
    if (n < 2) return false;
    if (n < 4) return true;
    if (n % 2 == 0 || n % 3 == 0) return false;
    for (int i = 5; i * i <= n; i += 6)
        if (n % i == 0 || n % (i + 2) == 0) return false;
    return true;
}

int main() {
    // Primes below 100
    auto primes_below_100 = std::views::iota(0, 100)
        | std::views::filter(is_prime);

    std::cout << "Primes < 100: ";
    for (int p : primes_below_100)
        std::cout << p << ' ';
    std::cout << '\n';

    // First 10 primes (from an infinite iota)
    auto first_10_primes = std::views::iota(2)
        | std::views::filter(is_prime)
        | std::views::take(10);

    std::cout << "First 10 primes: ";
    for (int p : first_10_primes)
        std::cout << p << ' ';
    std::cout << '\n';

    // Count primes in a range
    auto count = std::ranges::distance(
        std::views::iota(2, 1000) | std::views::filter(is_prime));
    std::cout << "Primes below 1000: " << count << '\n';
}
// Expected output:
// Primes < 100: 2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97
// First 10 primes: 2 3 5 7 11 13 17 19 23 29
// Primes below 1000: 168
```

The infinite `iota(2)` pipeline is a nice example of safe infinite-range usage: `take(10)` guarantees iteration stops, so the filter never runs forever.

- `views::iota(0, 100)` generates integers 0 through 99 lazily (no array).
- `views::filter(is_prime)` passes through only primes.
- `views::iota(2)` (no bound) generates an infinite sequence; `take(10)` truncates it.
- `ranges::distance` counts elements by iterating - works on any range.

### Q2: Combine `views::repeat` with `views::take` to create an infinite then truncated range

`repeat` is useful anywhere you need a constant sequence - default values, padding, filling gaps in a zip. The key thing to remember is that it physically stores only one copy of whatever value you give it.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    // Infinite repeat, truncated
    auto fives = std::views::repeat(5) | std::views::take(8);
    std::cout << "8 fives: ";
    for (int x : fives) std::cout << x << ' ';
    std::cout << '\n';

    // Bounded repeat (equivalent)
    auto bounded = std::views::repeat(42, 5);
    std::cout << "5 forty-twos: ";
    for (int x : bounded) std::cout << x << ' ';
    std::cout << '\n';

    // Practical: fill a vector with a default value
    auto defaults = std::views::repeat(std::string("N/A"), 4)
        | std::ranges::to<std::vector>();
    std::cout << "Defaults: ";
    for (const auto& s : defaults) std::cout << '"' << s << "\" ";
    std::cout << '\n';

    // Combine iota + repeat with zip for indexed defaults
    auto indexed = std::views::zip(
        std::views::iota(0),
        std::views::repeat(std::string("empty"))
    ) | std::views::take(5);

    std::cout << "Indexed: ";
    for (auto [i, val] : indexed)
        std::cout << '(' << i << "," << val << ") ";
    std::cout << '\n';
}
// Expected output:
// 8 fives: 5 5 5 5 5 5 5 5
// 5 forty-twos: 42 42 42 42 42
// Defaults: "N/A" "N/A" "N/A" "N/A"
// Indexed: (0,empty) (1,empty) (2,empty) (3,empty) (4,empty)
```

The `zip(iota(0), repeat("empty"))` pattern is a handy idiom when you need an infinite sequence of (index, constant) pairs - the `zip` truncates naturally when the finite side runs out, or you add `take` when both sides are infinite.

- `views::repeat(5)` generates an infinite stream of 5s; `take(8)` truncates to 8.
- `views::repeat(42, 5)` directly creates a bounded range of 5 elements (C++23).
- `repeat` stores a single copy - `views::repeat("N/A", 4)` doesn't duplicate the string 4 times internally.
- `zip(iota(0), repeat("empty"))` pairs each index with the same default value.

### Q3: Write a custom view generator using a coroutine (C++23)

When the generating logic gets complex - stateful sequences, conditional yields, anything that would require a custom iterator class - `std::generator` (C++23) gives you a clean way to express it using familiar coroutine syntax. The `co_yield` statement is the suspension point: execution pauses there and resumes when the caller asks for the next value.

```cpp
#include <generator>   // C++23: std::generator
#include <iostream>
#include <ranges>

// Fibonacci generator using C++23 std::generator
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        int next = a + b;
        a = b;
        b = next;
    }
}

// Custom range-generating coroutine
std::generator<int> squares_up_to(int limit) {
    for (int i = 1; i * i <= limit; ++i)
        co_yield i * i;
}

int main() {
    // Fibonacci: take first 12 numbers
    std::cout << "Fibonacci: ";
    int count = 0;
    for (int f : fibonacci()) {
        std::cout << f << ' ';
        if (++count == 12) break;
    }
    std::cout << '\n';

    // Perfect squares up to 100
    std::cout << "Squares <= 100: ";
    for (int s : squares_up_to(100))
        std::cout << s << ' ';
    std::cout << '\n';

    // Generators work with range algorithms
    // (std::generator models input_range)
}
// Expected output:
// Fibonacci: 0 1 1 2 3 5 8 13 21 34 55 89
// Squares <= 100: 1 4 9 16 25 36 49 64 81 100
```

The reason `std::generator` models only `input_range` (single-pass) is that coroutines have internal state - you cannot rewind or fork them. That means you can iterate a generator once with a range-for, but you cannot pass it to an algorithm that needs to iterate twice. If you need multi-pass behaviour, materialize it to a `vector` first.

- `std::generator<T>` (C++23, `<generator>`) is a coroutine-based range that yields values via `co_yield`.
- `fibonacci()` is an infinite generator - it suspends after each `co_yield` and resumes when the caller advances the iterator.
- `squares_up_to(100)` is a finite generator that stops when the condition fails.
- `std::generator` models `input_range` (single-pass), so it works with range-for and algorithms that require only input.
- Note: `<generator>` requires C++23 support (GCC 14+, Clang 18+, MSVC 17.10+).

---

## Notes

- `views::iota(0)` is random-access + infinite. `views::iota(0, 10)` is random-access + sized.
- `views::repeat(val)` is random-access + infinite. `views::repeat(val, n)` is random-access + sized.
- `views::empty<T>` is useful as a default/sentinel in generic code.
- `views::single(x)` creates a range with exactly one element - useful for wrapping a single value in range pipelines.
- For complex generation logic (state machines, I/O), prefer `std::generator` coroutines over building custom `view_interface` types.
