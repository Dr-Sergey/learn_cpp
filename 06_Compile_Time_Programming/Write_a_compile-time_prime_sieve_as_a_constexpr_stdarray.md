# Write a Compile-Time Prime Sieve as a `constexpr std::array`

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/consteval>  

---

## Topic Overview

### Compile-Time Sieve of Eratosthenes

The Sieve of Eratosthenes is a classic algorithm that marks off multiples to find all primes up to some limit N. Normally you would run it at startup and store the result in a runtime array. But since it is just a fixed computation on compile-time constants, you can also express it as a `consteval` or `constexpr` function that returns a `std::array<bool, N>`. When you do that, the compiler runs the entire sieve during compilation. Your binary ends up with a pre-computed lookup table in `.rodata` and absolutely zero work happens at startup.

The beauty of this approach is that primality testing becomes an O(1) array index at runtime - just `sieve[n]`. No math, no trial division, nothing. The compiler already did all the work.

### Key Idea

The structure is simple: write a function that fills a `std::array<bool, N>` using the standard sieve algorithm, return it, and store the result in a `constexpr` variable. The compiler does the rest.

```cpp
consteval auto sieve() {
    std::array<bool, 1000> is_prime{};
    // ... fill using Sieve of Eratosthenes ...
    return is_prime;
}
constexpr auto primes = sieve(); // computed at compile time
static_assert(primes[2]);        // 2 is prime
static_assert(!primes[4]);       // 4 is not prime
```

### Why Compile-Time

The trade-offs depend on your use case. For embedded work the benefits of the compile-time version are especially significant.

| Aspect | Runtime Sieve | Compile-Time Sieve |
| --- | --- | --- |
| Startup cost | O(N log log N) | Zero |
| Binary size | Code + data | Data only (table in .rodata) |
| Verification | Runtime tests | `static_assert` |
| Embedded | RAM + CPU at boot | Flash/ROM, no CPU |

---

## Self-Assessment

### Q1: Implement the Sieve of Eratosthenes as a `consteval` function returning `std::array<bool, N>`

The `consteval` keyword guarantees that the function is called only during compilation. If you accidentally call it from a runtime context, the compiler will reject the code. For a precomputed table like this that is exactly the safety guarantee you want - if someone refactors the code and the table suddenly becomes runtime-computed, you get a compile error immediately rather than a silent performance regression.

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// === Sieve of Eratosthenes at compile time ===
template<std::size_t N>
consteval auto make_prime_sieve() {
    std::array<bool, N> is_prime{};

    // Initialize: 0 and 1 are not prime
    for (std::size_t i = 2; i < N; ++i)
        is_prime[i] = true;

    // Sieve
    for (std::size_t i = 2; i * i < N; ++i) {
        if (is_prime[i]) {
            for (std::size_t j = i * i; j < N; ++j)
                if (j % i == 0)
                    is_prime[j] = false;
        }
    }

    return is_prime;
}

// Compile-time computation - this table lives in .rodata
constexpr auto sieve = make_prime_sieve<1000>();

// Compile-time verification
static_assert(!sieve[0]);   // 0 is not prime
static_assert(!sieve[1]);   // 1 is not prime
static_assert(sieve[2]);    // 2 is prime
static_assert(sieve[3]);    // 3 is prime
static_assert(!sieve[4]);   // 4 is not prime
static_assert(sieve[5]);    // 5 is prime
static_assert(sieve[7]);    // 7 is prime
static_assert(!sieve[9]);   // 9 is not prime
static_assert(sieve[97]);   // 97 is prime
static_assert(sieve[997]);  // 997 is prime
static_assert(!sieve[998]); // 998 is not prime

// === Count primes at compile time ===
template<std::size_t N>
consteval std::size_t count_primes(const std::array<bool, N>& s) {
    std::size_t count = 0;
    for (std::size_t i = 0; i < N; ++i)
        if (s[i]) ++count;
    return count;
}

constexpr auto num_primes = count_primes(sieve);
static_assert(num_primes == 168);  // pi(1000) = 168

int main() {
    std::cout << "=== Compile-Time Prime Sieve ===\n";
    std::cout << "Number of primes below 1000: " << num_primes << "\n";

    std::cout << "\nPrimes below 100:\n";
    for (std::size_t i = 2; i < 100; ++i) {
        if (sieve[i]) std::cout << i << " ";
    }
    std::cout << "\n";

    std::cout << "\nLargest primes below 1000:\n";
    int count = 0;
    for (int i = 999; i >= 2 && count < 10; --i) {
        if (sieve[i]) {
            std::cout << i << " ";
            ++count;
        }
    }
    std::cout << "\n";

    return 0;
}
```

Notice that `count_primes` is also `consteval`, and `num_primes` is verified with `static_assert(num_primes == 168)`. The prime-counting function pi(1000) = 168 is a known mathematical fact, so this assertion is both a correctness test and a demonstration that the entire sieve result is available as a compile-time constant. If you have a bug in the sieve, this `static_assert` catches it before your program ever runs.

**Expected output:**

```text
=== Compile-Time Prime Sieve ===
Number of primes below 1000: 168

Primes below 100:
2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97

Largest primes below 1000:
997 991 983 977 971 967 953 947 941 937
```

### Q2: Use the sieve in a `constexpr` context to check primality without runtime cost

Once you have a compile-time sieve, you can build more compile-time computations on top of it. Here we add a `consteval` Goldbach checker and an nth-prime finder, both of which are fully verified at compile time. The Goldbach verification is particularly striking - the compiler literally proves a number theory statement during compilation.

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// Reuse the sieve from Q1
template<std::size_t N>
consteval auto make_prime_sieve() {
    std::array<bool, N> is_prime{};
    for (std::size_t i = 2; i < N; ++i) is_prime[i] = true;
    for (std::size_t i = 2; i * i < N; ++i)
        if (is_prime[i])
            for (std::size_t j = i * i; j < N; j += i)
                is_prime[j] = false;
    return is_prime;
}

constexpr auto sieve = make_prime_sieve<10000>();

// === constexpr primality check - O(1) lookup ===
constexpr bool is_prime(std::size_t n) {
    return n < sieve.size() && sieve[n];
}

// === Compile-time Goldbach verification ===
// Every even number > 2 is the sum of two primes
consteval bool verify_goldbach(std::size_t limit) {
    for (std::size_t n = 4; n <= limit; n += 2) {
        bool found = false;
        for (std::size_t p = 2; p <= n / 2; ++p) {
            if (is_prime(p) && is_prime(n - p)) {
                found = true;
                break;
            }
        }
        if (!found) return false;
    }
    return true;
}

// Verified at compile time - Goldbach holds up to 1000
static_assert(verify_goldbach(1000));

// === Compile-time nth prime ===
consteval std::size_t nth_prime(std::size_t n) {
    std::size_t count = 0;
    for (std::size_t i = 2; i < sieve.size(); ++i) {
        if (sieve[i]) {
            ++count;
            if (count == n) return i;
        }
    }
    return 0; // not found within sieve range
}

static_assert(nth_prime(1) == 2);
static_assert(nth_prime(2) == 3);
static_assert(nth_prime(10) == 29);
static_assert(nth_prime(100) == 541);
static_assert(nth_prime(1000) == 7919);

int main() {
    std::cout << "=== Compile-Time Primality Check ===\n";
    for (int n : {1, 2, 3, 4, 5, 13, 42, 97, 100, 997, 7919}) {
        std::cout << n << " is " << (is_prime(n) ? "prime" : "not prime") << "\n";
    }

    std::cout << "\n=== Nth Prime (all computed at compile time) ===\n";
    std::cout << "1st prime:    " << nth_prime(1) << "\n";
    std::cout << "10th prime:   " << nth_prime(10) << "\n";
    std::cout << "100th prime:  " << nth_prime(100) << "\n";
    std::cout << "1000th prime: " << nth_prime(1000) << "\n";

    std::cout << "\nGoldbach conjecture verified up to 1000 at compile time!\n";

    return 0;
}
```

The `static_assert(verify_goldbach(1000))` line is particularly interesting - it means the compiler itself verified a number theory statement for all even numbers up to 1000. If the sieve or the checker had a bug, the code would not compile. You get mathematical correctness as a compile-time guarantee.

**Expected output:**

```text
=== Compile-Time Primality Check ===
1 is not prime
2 is prime
3 is prime
4 is not prime
5 is prime
13 is prime
42 is not prime
97 is prime
100 is not prime
997 is prime
7919 is prime

=== Nth Prime (all computed at compile time) ===
1st prime:    2
10th prime:   29
100th prime:  541
1000th prime: 7919

Goldbach conjecture verified up to 1000 at compile time!
```

### Q3: Compare compile-time vs runtime sieve for initialization cost in embedded systems

To make the performance benefit concrete, this example builds the same sieve both ways and measures the runtime cost. On embedded platforms the difference matters even more because boot time directly affects product responsiveness, and a CPU that is busy running a sieve at boot time is not doing anything useful for the user.

```cpp
#include <iostream>
#include <array>
#include <chrono>
#include <cstddef>

// === Compile-time sieve (in .rodata / flash) ===
template<std::size_t N>
consteval auto make_prime_sieve() {
    std::array<bool, N> is_prime{};
    for (std::size_t i = 2; i < N; ++i) is_prime[i] = true;
    for (std::size_t i = 2; i * i < N; ++i)
        if (is_prime[i])
            for (std::size_t j = i * i; j < N; j += i)
                is_prime[j] = false;
    return is_prime;
}

constexpr auto ct_sieve = make_prime_sieve<100000>();

// === Runtime sieve (computed at startup) ===
auto make_runtime_sieve() {
    std::array<bool, 100000> is_prime{};
    for (std::size_t i = 2; i < 100000; ++i) is_prime[i] = true;
    for (std::size_t i = 2; i * i < 100000; ++i)
        if (is_prime[i])
            for (std::size_t j = i * i; j < 100000; j += i)
                is_prime[j] = false;
    return is_prime;
}

int main() {
    // === Measure runtime sieve initialization ===
    auto start = std::chrono::high_resolution_clock::now();
    auto rt_sieve = make_runtime_sieve();
    auto end = std::chrono::high_resolution_clock::now();

    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();

    std::cout << "=== Compile-Time vs Runtime Sieve Comparison ===\n\n";

    std::cout << "Sieve size: 100,000 entries\n";
    std::cout << "Runtime sieve initialization: " << us << " microseconds\n";
    std::cout << "Compile-time sieve initialization: 0 microseconds (in binary)\n";

    // Verify both produce same results
    bool match = true;
    for (std::size_t i = 0; i < 100000; ++i) {
        if (ct_sieve[i] != rt_sieve[i]) { match = false; break; }
    }
    std::cout << "Results match: " << (match ? "yes" : "NO!") << "\n";

    std::cout << "\n=== Embedded Systems Comparison ===\n";
    std::cout << "+----------------------+------------------+------------------+\n";
    std::cout << "| Aspect               | Runtime Sieve    | Compile-Time     |\n";
    std::cout << "+----------------------+------------------+------------------+\n";
    std::cout << "| Startup time         | O(N log log N)   | 0                |\n";
    std::cout << "| RAM usage            | N bytes (stack)  | 0 (read-only)    |\n";
    std::cout << "| Flash/ROM            | code only        | code + N bytes   |\n";
    std::cout << "| CPU at boot          | Required         | Not required     |\n";
    std::cout << "| Correctness          | Runtime check    | static_assert    |\n";
    std::cout << "| Binary size          | Smaller          | +N bytes         |\n";
    std::cout << "+----------------------+------------------+------------------+\n";

    std::cout << "\n=== When to choose which ===\n";
    std::cout << "Compile-time: Known N, deterministic boot, embedded, safety-critical\n";
    std::cout << "Runtime: N determined at runtime, very large N, memory-constrained flash\n";

    return 0;
}
```

The key trade-off is flash space versus boot time. The compile-time sieve burns `N` bytes in your read-only binary image, but costs zero CPU cycles at startup. Whether that is the right trade depends on your constraints - for safety-critical systems where deterministic boot time matters, compile-time tables are often the better choice.

**Expected output (timing varies):**

```text
=== Compile-Time vs Runtime Sieve Comparison ===

Sieve size: 100,000 entries
Runtime sieve initialization: 142 microseconds
Compile-time sieve initialization: 0 microseconds (in binary)
Results match: yes

=== Embedded Systems Comparison ===
+----------------------+------------------+------------------+
| Aspect               | Runtime Sieve    | Compile-Time     |
+----------------------+------------------+------------------+
| Startup time         | O(N log log N)   | 0                |
| RAM usage            | N bytes (stack)  | 0 (read-only)    |
| Flash/ROM            | code only        | code + N bytes   |
| CPU at boot          | Required         | Not required     |
| Correctness          | Runtime check    | static_assert    |
| Binary size          | Smaller          | +N bytes         |
+----------------------+------------------+------------------+

=== When to choose which ===
Compile-time: Known N, deterministic boot, embedded, safety-critical
Runtime: N determined at runtime, very large N, memory-constrained flash
```

---

## Notes

- `consteval` functions **must** be evaluated at compile time - use for guaranteed compile-time sieves.
- `constexpr` functions **can** run at either compile or runtime - use when dual behavior is desired.
- The sieve table ends up in `.rodata` (read-only data segment) - no startup initialization.
- For embedded: compile-time tables trade flash space for zero boot time and deterministic startup.
- `static_assert` lets you verify properties (prime count, specific values) at compile time.
- The sieve in Q1 uses `j % i == 0` for clarity; Q2/Q3 use `j += i` for efficiency - both work at compile time.
