# Use OS entropy sources correctly: std::random_device and /dev/urandom

**Category:** Safety and Security  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/random/random_device>  

---

## Topic Overview

Cryptographic security depends on unpredictable randomness. `std::random_device` provides access to OS entropy sources (`/dev/urandom` on Linux, `BCryptGenRandom` on Windows), but its behavior varies across implementations. The most dangerous mistake is seeding a PRNG with predictable values like `time(NULL)`, which gives attackers only ~30 bits of entropy to brute-force.

The reason this matters so much is that randomness is the foundation of cryptographic tokens, session IDs, and encryption keys. If your "random" seed is actually guessable, your entire security model collapses. Getting this wrong is a classic vulnerability that has burned real products.

### Entropy Source Hierarchy

Here is how the different entropy sources rank from most to least secure. The jump from any OS source down to `time(NULL)` is enormous - you go from practically unguessable to a 3600-candidate brute-force:

```cpp
Best -> Worst:

1. Hardware RNG (RDRAND/RDSEED instruction)

   -> std::random_device on modern GCC/Clang/MSVC uses this

2. OS entropy pool (/dev/urandom, BCryptGenRandom)

   -> std::random_device fallback on all major platforms
   -> Non-blocking, always available after boot

3. /dev/random (Linux < 5.6)

   -> BLOCKS if entropy pool is empty (bad for servers)
   -> Linux >= 5.6: identical to /dev/urandom (no blocking)

4. time(NULL) - INSECURE

   -> ~30 bits of entropy (seconds since epoch)
   -> Attacker knows approximate time -> trivial to brute-force

5. Constant seed - NOT RANDOM AT ALL

   -> mt19937 rng(42);  // deterministic, reproducible
```

### Platform Behavior of std::random_device

The surprising entry in this table is MinGW. If you are cross-compiling for Windows on a MinGW toolchain, old versions silently fall back to a deterministic implementation. Always check:

| Platform | Implementation | Non-deterministic? |
| --- | --- | --- |
| GCC (Linux) | `/dev/urandom` or RDRAND | Yes |
| Clang (Linux) | `/dev/urandom` or RDRAND | Yes |
| MSVC (Windows) | `BCryptGenRandom` | Yes |
| MinGW (old) | **Deterministic!** (mt19937 with fixed seed) | **NO** |
| Embedded | Implementation-defined | CHECK! |

### Core Example

This shows both the entropy check and the correct way to seed mt19937 with the full state worth of randomness. The `seed_seq` approach is the important part - a single `rd()` call only seeds 32 bits of a 19,968-bit state:

```cpp
#include <random>
#include <iostream>
#include <array>

int main() {
    // Good: use std::random_device for cryptographic seeding
    std::random_device rd;
    std::cout << "Entropy: " << rd.entropy() << "\n";
    std::cout << "Random value: " << rd() << "\n";

    // Properly seed mt19937 with full state
    std::array<std::uint32_t, std::mt19937::state_size> seed_data;
    std::generate(seed_data.begin(), seed_data.end(), std::ref(rd));
    std::seed_seq seed(seed_data.begin(), seed_data.end());
    std::mt19937 gen(seed);

    std::cout << "Well-seeded mt19937: " << gen() << "\n";
}
```

---

## Self-Assessment

### Q1: Verify that std::random_device is non-deterministic on your platform by checking entropy()

**Answer:**

The reason this requires multiple checks is that `entropy()` is misleading - the standard permits it to return `0` even for truly non-deterministic sources. The only reliable test is running the program twice and comparing output:

```cpp
#include <random>
#include <iostream>
#include <set>
#include <cstdint>

int main() {
    std::random_device rd;

    // Check 1: entropy() value
    double ent = rd.entropy();
    std::cout << "std::random_device::entropy() = " << ent << "\n";
    // GCC/Linux: 32.0 (uses /dev/urandom or RDRAND - truly random)
    // MSVC:      32.0 (uses BCryptGenRandom)
    // MinGW-old: 0.0  (DETERMINISTIC - uses mt19937!)
    //
    // entropy() == 0 -> WARNING: may be deterministic!
    // entropy() > 0  -> non-deterministic (probably)
    //
    // BUT: entropy() is unreliable per the standard.
    // It MAY return 0 even for truly random implementations.
    // Practical test: run twice and compare outputs.

    if (ent == 0.0) {
        std::cerr << "WARNING: std::random_device may be deterministic!\n";
        std::cerr << "Use platform-specific API instead.\n";
    }

    // Check 2: Uniqueness test
    constexpr int SAMPLES = 1000;
    std::set<uint32_t> values;

    for (int i = 0; i < SAMPLES; ++i) {
        values.insert(rd());
    }

    std::cout << "Unique values in " << SAMPLES << " samples: "
              << values.size() << "\n";
    // Non-deterministic: ~1000 unique (all different)
    // Deterministic:     ~1000 unique (mt19937 is still uniform distribution)
    // -> This test alone doesn't prove non-determinism!

    // Check 3: Cross-run comparison
    // The REAL test: run the program twice
    std::cout << "First 5 values: ";
    std::random_device rd2;
    for (int i = 0; i < 5; ++i) {
        std::cout << rd2() << " ";
    }
    std::cout << "\n";
    // If same on every run -> DETERMINISTIC (bad!)
    // If different on every run -> non-deterministic (good!)

    // Platform-specific verification
    //
    // Linux: strace ./a.out 2>&1 | grep -i random
    //   open("/dev/urandom", O_RDONLY) -> confirms OS entropy
    //
    // Windows: std::random_device always uses BCryptGenRandom (secure)
    //
    // MinGW workaround:
    //   #ifdef __MINGW32__
    //   #include <windows.h>
    //   #include <bcrypt.h>
    //   void secure_random(void* buf, size_t len) {
    //       BCryptGenRandom(NULL, (PUCHAR)buf, len, BCRYPT_USE_SYSTEM_PREFERRED_RNG);
    //   }
    //   #endif

    // Output (typical Linux/MSVC):
    // std::random_device::entropy() = 32
    // Unique values in 1000 samples: 1000
    // First 5 values: 3847291056 192847365 ... (different each run)
}
```

**Explanation:** `entropy()` returns a double indicating estimated bits of entropy per sample. A return of `0.0` typically means the implementation is deterministic (dangerous!). However, the standard allows `0.0` even for truly random implementations, so the most reliable test is running the program multiple times - if values differ between runs, the source is non-deterministic. On Linux, use `strace` to verify the program reads from `/dev/urandom`.

### Q2: Explain why seeding mt19937 with a single time(NULL) is cryptographically weak

**Answer:**

The reason this is so dangerous is a combination of two weaknesses: the seed space is tiny (attackers can enumerate all possibilities in under a millisecond), and mt19937 is entirely deterministic - find the seed and you know every past and future output:

```cpp
#include <random>
#include <ctime>
#include <iostream>
#include <cstdint>

int main() {
    // The Problem

    // WEAK seed: time(NULL) returns seconds since epoch
    auto weak_seed = static_cast<uint32_t>(std::time(nullptr));
    std::mt19937 weak_rng(weak_seed);

    std::cout << "time(NULL) seed: " << weak_seed << "\n";
    std::cout << "First output: " << weak_rng() << "\n";

    // Why this is catastrophically weak:
    //
    // 1. LOW ENTROPY: time(NULL) has ~30 bits of entropy
    //    - Attacker knows the approximate time (+/-1 hour at most)
    //    - That's only 3600 possible seeds
    //    - A modern CPU can test ALL of them in < 1 millisecond
    //
    // 2. PREDICTABLE: mt19937 is deterministic given the seed
    //    - Same seed -> same sequence, always
    //    - If attacker finds the seed, they know ALL future outputs
    //
    // 3. SINGLE VALUE: mt19937 state is 624 x 32 bits = 19,968 bits
    //    - Seeding with one 32-bit value initializes the full state
    //      via a deterministic expansion function
    //    - Only 2^32 possible initial states (out of 2^19968)
    //    - Attacker can enumerate ALL 2^32 possibilities in hours

    // Attack demonstration

    // Server generates a session token at time T:
    uint32_t server_time = weak_seed; // attacker estimates this
    std::mt19937 server_rng(server_time);
    uint32_t session_token = server_rng();

    // Attacker brute-forces the seed:
    uint32_t attacker_guess_time = server_time - 5; // guess within 10 seconds
    for (uint32_t t = attacker_guess_time; t <= server_time + 5; ++t) {
        std::mt19937 trial(t);
        if (trial() == session_token) {
            std::cout << "Attacker found seed: " << t << "\n";
            std::cout << "Next token will be: " << trial() << "\n";
            // Attacker now knows ALL future tokens!
            break;
        }
    }

    // mt19937 is NOT for cryptography
    //
    // Even with a perfect seed, mt19937 is NOT cryptographically secure:
    // - Observing 624 consecutive outputs reveals the full internal state
    // - Attacker can then predict ALL future outputs
    // - There is NO recovery: the state is fully deterministic
    //
    // For cryptographic use:
    //   - Tokens/keys: use std::random_device directly (slow but secure)
    //   - High-volume: use OS CSPRNG (BCryptGenRandom, getrandom())
    //   - If you must use mt19937 for non-crypto: seed it properly (Q3)

    // Output:
    // time(NULL) seed: 1705312845
    // First output: 2947381056
    // Attacker found seed: 1705312845
    // Next token will be: 1038472951
}
```

**Explanation:** `time(NULL)` provides at most 32 bits of seed, and in practice the attacker knows the approximate time (reducing it to ~12 bits). Since mt19937 is deterministic, finding the seed reveals the entire output sequence. Additionally, mt19937's 19,968-bit state is initialized from just 32 bits, meaning only 2^32 out of 2^19968 states are reachable. For any security application, use OS entropy sources directly - mt19937 (even properly seeded) is not cryptographically secure.

### Q3: Use std::random_device to seed a std::seed_seq for a fully unpredictable mt19937 state

**Answer:**

The key insight here is that seeding mt19937 with a single `rd()` call leaves 19,936 bits of state determined by a fixed algorithm from that one 32-bit value. Filling the entire 624-word array from `std::random_device` ensures all 19,968 bits of state are independently random:

```cpp
#include <random>
#include <array>
#include <iostream>
#include <algorithm>
#include <functional>
#include <cstdint>

// WRONG ways to seed mt19937

void bad_seeding() {
    // BAD 1: time-based (predictable)
    // std::mt19937 rng(std::time(nullptr));

    // BAD 2: single random_device value (only 32 bits of 19,968-bit state)
    std::random_device rd;
    std::mt19937 rng(rd());  // BETTER than time(), but still only 2^32 states
    // mt19937 has 624 x 32 = 19,968 bits of state
    // With rd(), only 2^32 possible states are covered
    std::cout << "Bad seed (single rd()): " << rng() << "\n";
}

// CORRECT way: seed_seq from random_device

std::mt19937 make_seeded_engine() {
    std::random_device rd;

    // Fill an array with enough random data to cover mt19937's full state
    // mt19937::state_size = 624 (number of 32-bit words)
    std::array<uint32_t, std::mt19937::state_size> seed_data;
    std::generate(seed_data.begin(), seed_data.end(), std::ref(rd));

    // Create a seed_seq from the random data
    // seed_seq uses a mixing algorithm to distribute entropy across the state
    std::seed_seq seq(seed_data.begin(), seed_data.end());

    // Initialize mt19937 with the fully-seeded seed_seq
    return std::mt19937{seq};
}

// For mt19937_64

std::mt19937_64 make_seeded_engine_64() {
    std::random_device rd;
    std::array<uint32_t, std::mt19937_64::state_size * 2> seed_data;
    std::generate(seed_data.begin(), seed_data.end(), std::ref(rd));
    std::seed_seq seq(seed_data.begin(), seed_data.end());
    return std::mt19937_64{seq};
}

int main() {
    bad_seeding();

    // Properly seeded engine
    auto rng = make_seeded_engine();

    std::cout << "Well-seeded mt19937:\n";
    for (int i = 0; i < 5; ++i) {
        std::cout << "  " << rng() << "\n";
    }
    // Different every run - full 19,968 bits of state are unpredictable

    // Using with distributions
    std::uniform_int_distribution<int> dice(1, 6);
    std::cout << "\nDice rolls: ";
    for (int i = 0; i < 10; ++i) {
        std::cout << dice(rng) << " ";
    }
    std::cout << "\n";

    // When to use std::random_device directly
    //
    // Use case                    Recommended source
    // Cryptographic keys          std::random_device ONLY (or OS API)
    // Session tokens              std::random_device ONLY (or OS API)
    // password salts              std::random_device ONLY (or OS API)
    // Simulation/Monte Carlo      mt19937 + seed_seq (fast, high quality)
    // Shuffling a deck            mt19937 + seed_seq
    // Game randomness             mt19937 + seed_seq
    // Unit test reproducibility   mt19937 + fixed seed (deterministic)

    // Helper function for quick use
    auto gen = make_seeded_engine();
    std::uniform_real_distribution<double> uniform(0.0, 1.0);
    std::cout << "Random double: " << uniform(gen) << "\n";

    // Output (different each run):
    // Bad seed (single rd()): 1234567890
    // Well-seeded mt19937:
    //   3847291056
    //   192847365
    //   ...
    // Dice rolls: 3 6 1 4 2 5 3 1 6 4
    // Random double: 0.73829174
}
```

**Explanation:** `std::seed_seq` takes an array of random values and applies a mixing algorithm to distribute entropy across mt19937's full 19,968-bit state. By feeding it `state_size` (624) random `uint32_t` values from `std::random_device`, we initialize all 624 words of state with true randomness - not just expanding a single 32-bit seed. This makes brute-force search of the seed space infeasible (2^19968 states vs 2^32).

---

## Notes

- **MinGW warning:** Older MinGW implementations of `std::random_device` use a deterministic mt19937 internally. Use the Win32 `BCryptGenRandom` API directly as a workaround.
- **`/dev/urandom` vs `/dev/random`:** On Linux >= 5.6, they are identical. On older kernels, `/dev/random` blocks when the entropy pool estimate is low - always prefer `/dev/urandom`.
- **`std::seed_seq` limitations:** It uses a fixed mixing algorithm that reduces entropy slightly. For cryptographic seeding, bypass seed_seq and use raw OS entropy bytes.
- **`RDRAND/RDSEED`** Intel hardware RNG instructions: available since Ivy Bridge (2012). GCC/Clang `std::random_device` uses them when available.
- **Performance:** `std::random_device` is 10-100x slower than mt19937. Use it only for seeding, not for high-volume generation.
- Compile with `-std=c++20 -Wall -Wextra`.
