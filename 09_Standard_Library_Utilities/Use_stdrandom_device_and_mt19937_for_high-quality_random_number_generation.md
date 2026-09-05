# Use std::random_device and mt19937 for high-quality random number generation

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/random>  

---

## Topic Overview

C++ provides a modern random number facility in `<random>` that separates **engines** (sources of random bits) from **distributions** (shaping those bits into useful ranges). This two-layer design is the key idea: the engine produces raw numbers, and the distribution maps them onto whatever range you actually need. It replaces the deeply flawed `rand()` / `srand()` from C, which you'll want to stop using as soon as you understand why.

### Architecture

Here's the mental model - an engine feeds raw bits into a distribution, and you get useful numbers out the other side:

```cpp
┌─────────────────────┐     ┌──────────────────────────┐
│  Random Engine       │────>│  Distribution             │
│  (generates raw bits)│     │  (shapes bits into range) │
└─────────────────────┘     └──────────────────────────┘

Engines:                      Distributions:
  std::random_device          std::uniform_int_distribution<int>
  std::mt19937                std::uniform_real_distribution<double>
  std::mt19937_64             std::normal_distribution<double>
  std::minstd_rand            std::bernoulli_distribution
                              std::discrete_distribution<int>
                              std::poisson_distribution<int>
```

### Key Components

The table below shows what each piece does. The short version: `random_device` gives you a good seed, `mt19937` does the actual generation, and the distributions shape the output.

| Component | Purpose | Notes |
| --- | --- | --- |
| `std::random_device` | True random (hardware/OS entropy) | Slow, used for seeding only |
| `std::mt19937` | Mersenne Twister 32-bit PRNG | Fast, 2^19937-1 period, 624×32-bit state |
| `std::mt19937_64` | Mersenne Twister 64-bit | For 64-bit systems/distributions |
| `uniform_int_distribution<T>` | Uniform integers in [a, b] | Both endpoints inclusive |
| `uniform_real_distribution<T>` | Uniform reals in [a, b) | Half-open interval |
| `normal_distribution<T>` | Gaussian (bell curve) | Mean μ, std deviation σ |

### Why NOT rand()

If the table feels like a lot, the takeaway is this: `rand()` is broken in several independent ways, and `<random>` fixes all of them.

| Problem | `rand()` | `<random>` |
| --- | --- | --- |
| Range | `[0, RAND_MAX]` - often only 32767 | Arbitrary range via distributions |
| Distribution | `rand() % n` introduces **modulo bias** | Mathematically correct distributions |
| Thread safety | Global state - data race in threads | Per-instance engines - thread-safe if separate |
| Seed quality | `srand(time(0))` - poor entropy | `std::random_device` - OS entropy pool |
| Period | Implementation-defined, often short | 2^19937-1 for mt19937 |
| Reproducibility | Platform-dependent sequences | Same seed -> same sequence (guaranteed) |

### Core Usage Pattern

The pattern is always the same four steps: create a `random_device` for seeding, seed an `mt19937` engine, create whatever distributions you need, and then call them. Notice that you create distributions once and reuse them - only the engine holds state.

```cpp
#include <random>
#include <iostream>

int main() {
    // Step 1: Create a random_device for seeding (true randomness)
    std::random_device rd;

    // Step 2: Seed a Mersenne Twister engine
    std::mt19937 gen(rd());

    // Step 3: Create distributions
    std::uniform_int_distribution<int> dice(1, 6);       // [1, 6] inclusive
    std::uniform_real_distribution<double> prob(0.0, 1.0); // [0.0, 1.0)
    std::normal_distribution<double> height(170.0, 10.0);  // mean=170, σ=10

    // Step 4: Generate numbers
    std::cout << "Dice: " << dice(gen) << "\n";
    std::cout << "Prob: " << prob(gen) << "\n";
    std::cout << "Height: " << height(gen) << " cm\n";

    // Roll dice 10 times
    for (int i = 0; i < 10; ++i)
        std::cout << dice(gen) << " ";
    std::cout << "\n";
    // Example output: 3 6 1 4 2 5 6 3 1 4
}
```

Each call to `dice(gen)` advances the engine and maps its output to [1, 6]. The distribution object itself is stateless for uniform distributions - only `gen` changes over time.

### Proper Full-State Seeding

Here's a subtlety that trips people up: `mt19937` has 624 × 32-bit words of internal state, but seeding it with a single `rd()` call only initializes one of those words. The rest get derived from that one value in a predictable way, which means two sequences seeded with different single values can still share a lot of statistical structure. For simulations that genuinely need high quality, fill the whole state:

```cpp
#include <random>
#include <array>
#include <algorithm>
#include <functional>
#include <iostream>

int main() {
    // mt19937 has 624 × 32-bit state words.
    // A single rd() seed only fills ONE word — the rest are derived.
    // For high-quality seeding, fill the entire state:
    std::random_device rd;
    std::array<std::mt19937::result_type, std::mt19937::state_size> seed_data;
    std::generate(seed_data.begin(), seed_data.end(), std::ref(rd));
    std::seed_seq seq(seed_data.begin(), seed_data.end());
    std::mt19937 gen(seq);

    std::uniform_int_distribution<int> dist(1, 100);
    for (int i = 0; i < 5; ++i)
        std::cout << dist(gen) << " ";
}
```

For most applications, seeding with a single `rd()` is fine. For Monte Carlo simulations or cryptographic-adjacent work, use the full-state approach above.

---

## Self-Assessment

### Q1: Seed a std::mt19937 with std::random_device and draw uniformly distributed integers

**Answer:**

```cpp
#include <random>
#include <iostream>
#include <map>

int main() {
    // Seed mt19937 from hardware random
    std::random_device rd;
    std::mt19937 gen(rd());

    // Uniform integer distribution [1, 10] (inclusive both ends)
    std::uniform_int_distribution<int> dist(1, 10);

    // Draw 10 random integers
    std::cout << "Random integers [1,10]: ";
    for (int i = 0; i < 10; ++i)
        std::cout << dist(gen) << " ";
    std::cout << "\n";
    // Example: 7 3 9 1 5 10 2 8 4 6

    // Verify distribution is uniform by sampling 100,000 times
    std::map<int, int> histogram;
    for (int i = 0; i < 100'000; ++i)
        ++histogram[dist(gen)];

    std::cout << "Distribution (100k samples):\n";
    for (auto [value, count] : histogram)
        std::cout << value << ": " << count << "\n";
    // Each bucket should be ~10,000 ± a few hundred
    // 1: 9987
    // 2: 10054
    // ...
    // 10: 10012
}
```

`std::random_device rd` provides a non-deterministic seed from the OS entropy pool. `std::mt19937 gen(rd())` creates a Mersenne Twister PRNG seeded with that value. `std::uniform_int_distribution<int> dist(1, 10)` shapes the engine's output into uniform integers in [1, 10]. Calling `dist(gen)` advances the engine and returns a properly distributed value - no modulo bias.

### Q2: Explain why rand() is not suitable for simulation or security applications

`rand()` has multiple fundamental problems:

**1. Modulo bias:**

```cpp
// rand() returns [0, RAND_MAX]. If RAND_MAX = 32767:
int roll = rand() % 6 + 1;  // NOT uniform!
// 32768 / 6 = 5461 remainder 2
// Values 1-2 appear 5462 times per cycle
// Values 3-6 appear 5461 times per cycle
// Bias: 0.018% — matters in simulations with millions of samples
```

**2. Short period:**

```cpp
// RAND_MAX is only guaranteed to be >= 32767
// Many implementations use a linear congruential generator with period 2^32
// mt19937 period: 2^19937 - 1 ≈ 10^6001  (incomparably larger)
```

**3. Global mutable state:**

```cpp
// rand() uses a single global seed — data race in multithreaded code!
// Thread 1: srand(42); int a = rand();
// Thread 2: srand(99); int b = rand();
// Results are interleaved and unpredictable (undefined behavior)
```

**4. Poor entropy from time-based seeding:**

```cpp
// srand(time(0)) has ~1 second resolution
// Two programs started in the same second get identical sequences
// An attacker can predict sequences by knowing the start time
```

**5. Not suitable for security:**

```cpp
// The internal state of rand() can be reconstructed from output
// For cryptographic use, use OS facilities:
// - Linux: /dev/urandom or getrandom()
// - Windows: BCryptGenRandom()
// - C++: std::random_device (may use hardware RNG)
// NOTE: std::mt19937 is also NOT cryptographically secure!
```

### Q3: Use std::uniform_real_distribution to generate floats in [0.0, 1.0)

**Answer:**

```cpp
#include <random>
#include <iostream>
#include <iomanip>
#include <cmath>

int main() {
    std::random_device rd;
    std::mt19937 gen(rd());

    // uniform_real_distribution<double> in [0.0, 1.0)
    // Note: 0.0 is included, 1.0 is EXCLUDED
    std::uniform_real_distribution<double> unit(0.0, 1.0);

    std::cout << std::fixed << std::setprecision(6);
    std::cout << "5 random doubles in [0, 1):\n";
    for (int i = 0; i < 5; ++i)
        std::cout << "  " << unit(gen) << "\n";
    // Example:
    //   0.394383
    //   0.783099
    //   0.198710
    //   0.911647
    //   0.520032

    // Practical use: Monte Carlo estimation of π
    // Drop random points in [0,1)×[0,1). Count those inside unit circle.
    int inside = 0;
    constexpr int N = 1'000'000;
    for (int i = 0; i < N; ++i) {
        double x = unit(gen);
        double y = unit(gen);
        if (x * x + y * y <= 1.0)
            ++inside;
    }
    double pi_estimate = 4.0 * inside / N;
    std::cout << "π ≈ " << pi_estimate << "\n";
    // Output: π ≈ 3.141??? (close to 3.14159...)

    // Other useful distributions:
    std::uniform_real_distribution<float> angle(0.0f, 360.0f);
    std::normal_distribution<double> noise(0.0, 0.1); // Gaussian noise
    std::bernoulli_distribution coin(0.5);              // true/false 50/50

    std::cout << "Random angle: " << angle(gen) << "°\n";
    std::cout << "Gaussian noise: " << noise(gen) << "\n";
    std::cout << "Coin flip: " << (coin(gen) ? "heads" : "tails") << "\n";
}
```

`uniform_real_distribution<double>(0.0, 1.0)` generates values in the half-open interval [0.0, 1.0) - `0.0` can be returned, but `1.0` cannot. This is the standard convention matching most mathematical uses. The distribution object is stateless (for uniform) and can be reused; the engine (`gen`) maintains the state.

---

## Notes

- **`std::random_device` on MinGW:** Some MinGW implementations return deterministic values from `std::random_device`. Always test with `rd.entropy()` - if it returns 0, the device may not be truly random.
- **Thread safety:** Each thread should have its own `mt19937` instance. Either seed independently or use `thread_local`:

  ```cpp
  thread_local std::mt19937 gen{std::random_device{}()};
  ```

- **Reproducibility:** For testing/debugging, seed with a fixed value: `std::mt19937 gen(42);`. Same seed -> same sequence across runs (on the same platform).
- **Performance:** `mt19937` is fast (~1 ns/call). `random_device` is slow (~100+ ns) - use it only for seeding.
- **NOT cryptographically secure:** Neither `mt19937` nor `random_device` (on all platforms) are suitable for cryptographic key generation. Use OS-specific crypto APIs for security-sensitive applications.
- **`shuffle`:** Use `std::ranges::shuffle(v, gen)` instead of `std::random_shuffle` (removed in C++17).
- Compile with `-std=c++20 -Wall -Wextra`.
