# Write self-testing code using constexpr and static_assert as documentation

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/static_assert>  

---

## Topic Overview

`static_assert` combined with `constexpr` functions gives you **compile-time tests** that are impossible to skip. They run on every build, fail with clear error messages, and double as executable documentation - a specification that the compiler enforces for you. Unlike a comment saying "this function returns 120 for input 5," a `static_assert` actually verifies it.

```cpp
constexpr int factorial(int n) { return n <= 1 ? 1 : n * factorial(n-1); }
static_assert(factorial(5) == 120);  // test + documentation in one line
```

The really nice part is that `static_assert` has zero runtime cost. It exists only at compile time. If it passes, it disappears.

---

## Self-Assessment

### Q1: Enforce struct sizes with `static_assert`

This is one of the most practical uses: locking down the layout of a struct that has to match an external format. If anyone adds a field or removes `#pragma pack`, the build fails immediately - you don't discover the problem at runtime when the protocol starts misbehaving.

```cpp
#include <cstdint>
#include <iostream>

// Network protocol header - must match wire format exactly!
#pragma pack(push, 1)
struct NetworkHeader {
    uint8_t  version;      // 1 byte
    uint8_t  type;         // 1 byte
    uint16_t length;       // 2 bytes
    uint32_t sequence;     // 4 bytes
    uint32_t source_addr;  // 4 bytes
    uint32_t dest_addr;    // 4 bytes
    uint32_t checksum;     // 4 bytes
};  // total: 20 bytes
#pragma pack(pop)

// Compile-time enforcement - if someone adds a field, this FAILS
static_assert(sizeof(NetworkHeader) == 20,
    "NetworkHeader must be exactly 20 bytes for protocol compliance");

// More struct invariants:
static_assert(alignof(NetworkHeader) == 1,
    "NetworkHeader must be byte-aligned (packed)");

static_assert(offsetof(NetworkHeader, checksum) == 16,
    "checksum must be at offset 16");

// Type size assumptions for cross-platform code:
static_assert(sizeof(int) >= 4, "int must be at least 32 bits");
static_assert(sizeof(void*) == 8, "64-bit platform required");
static_assert(sizeof(double) == 8, "IEEE 754 double expected");

int main() {
    NetworkHeader h{1, 0x02, 100, 42, 0xC0A80001, 0xC0A80002, 0};
    std::cout << "Header size: " << sizeof(h) << " bytes\n";
    std::cout << "Version: " << static_cast<int>(h.version) << '\n';
}
// Expected output:
// Header size: 20 bytes
// Version: 1
```

The platform assumptions at the bottom (`sizeof(void*) == 8`, etc.) are especially useful in cross-platform code. They fail loudly on a platform that doesn't match your assumptions, instead of silently producing wrong results.

### Q2: Document algorithm invariants with `constexpr` + `static_assert`

The idea here is to treat your `static_assert` lines as an executable specification. Each one says: "for these inputs, the function produces this output." That's more useful than a comment because it can never go out of date - if you change the algorithm and break an invariant, the build tells you.

```cpp
#include <array>
#include <iostream>
#include <cstddef>

// constexpr algorithms - testable at compile time!
constexpr int gcd(int a, int b) {
    while (b != 0) {
        int t = b;
        b = a % b;
        a = t;
    }
    return a;
}

constexpr bool is_prime(int n) {
    if (n < 2) return false;
    for (int i = 2; i * i <= n; ++i)
        if (n % i == 0) return false;
    return true;
}

constexpr int fibonacci(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}

// Executable specifications - these ARE the tests AND the docs!
static_assert(gcd(12, 8) == 4);
static_assert(gcd(17, 13) == 1);   // coprime
static_assert(gcd(100, 0) == 100); // edge case

static_assert(is_prime(2));
static_assert(is_prime(17));
static_assert(!is_prime(1));
static_assert(!is_prime(15));

static_assert(fibonacci(0) == 0);
static_assert(fibonacci(1) == 1);
static_assert(fibonacci(10) == 55);
static_assert(fibonacci(20) == 6765);

// Compile-time lookup table validation
constexpr std::array<int, 5> PRIMES = {2, 3, 5, 7, 11};
static_assert(PRIMES.size() == 5);
static_assert(PRIMES[0] == 2);
static_assert(PRIMES[4] == 11);

int main() {
    std::cout << "gcd(12,8) = " << gcd(12, 8) << '\n';
    std::cout << "fib(10)   = " << fibonacci(10) << '\n';
    std::cout << "All compile-time tests passed!\n";
}
// Expected output:
// gcd(12,8) = 4
// fib(10)   = 55
// All compile-time tests passed!
```

The edge cases are particularly valuable here: `gcd(100, 0) == 100` and `!is_prime(1)` document behavior that's easy to get wrong and easy to forget to test in a separate unit test suite.

### Q3: `static_assert` as executable comments that never go stale

Regular comments become lies over time - someone writes "Widget is always 8 bytes," then later adds a field, and nobody updates the comment. A `static_assert` on `sizeof(Widget)` can't become a lie: if `Widget` changes size, the build fails and you have to update it consciously.

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// Regular comments can become stale:
// "Widget is always 8 bytes"  <-- maybe it was once, but is it still?

// static_assert can't go stale - compiler checks it every build!
struct Widget {
    int id;
    float weight;
};
static_assert(std::is_trivially_copyable_v<Widget>,
    "Widget must be trivially copyable for memcpy into GPU buffer");
static_assert(std::is_standard_layout_v<Widget>,
    "Widget must be standard layout for C interop");

// Document template requirements:
template<typename T>
class Cache {
    static_assert(std::is_default_constructible_v<T>,
        "Cache<T> requires T to be default-constructible");
    static_assert(std::is_copy_assignable_v<T>,
        "Cache<T> requires T to be copy-assignable");

    std::vector<T> data_;
public:
    void add(const T& item) { data_.push_back(item); }
    size_t size() const { return data_.size(); }
};

// Document invariants about platform/configuration:
static_assert(__cplusplus >= 201703L, "C++17 or later required");
static_assert(sizeof(size_t) == 8, "64-bit platform required");

int main() {
    Cache<int> ic;
    ic.add(42);
    std::cout << "Cache size: " << ic.size() << '\n';

    // Cache<std::unique_ptr<int>> uc;  // FAILS: not copy-assignable!

    std::cout << "All static assertions verified at compile time.\n";
}
// Expected output:
// Cache size: 1
// All static assertions verified at compile time.
```

The `Cache` template assertions are especially useful: they give whoever instantiates `Cache<T>` an immediate, readable error if their type doesn't satisfy the requirements - instead of a wall of cryptic template substitution failures deeper in the implementation.

---

## Notes

- `static_assert` requires a constant expression, so the code must be `constexpr`.
- `static_assert` with no message is allowed since C++17.
- Use for: struct sizes, type traits, algorithm correctness, platform assumptions.
- Unlike runtime tests, `static_assert` has zero runtime cost.
- Combine with `if constexpr` for conditional compile-time logic.
