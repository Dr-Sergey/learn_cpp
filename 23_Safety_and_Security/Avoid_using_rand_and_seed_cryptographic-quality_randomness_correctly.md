# Avoid using rand() and seed cryptographic-quality randomness correctly

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/random/random_device>  

---

## Topic Overview

This topic focuses on proper seeding of mt19937 and OS-level entropy for key material - complementing #737 (rand() problems, random_device basics, cross-platform API).

The reason this trips people up is that mt19937 looks secure on the surface - it produces numbers that pass every statistical test, it has a period of 2^19937, and its output shows no obvious patterns. But "statistical quality" and "cryptographic security" are completely different things. The difference comes down to this: given enough outputs, can an attacker reconstruct the internal state and predict every future value? For mt19937 the answer is yes, after only 624 consecutive outputs. That makes it completely unsuitable for any security use.

The seeding situation is equally important and equally misunderstood. Even if you decide to use mt19937 for non-security purposes, seeding it badly means your entire output sequence is guessable from the start.

```cpp
Seeding mt19937 correctly:

  BAD:   mt19937 gen(rd());           // 32-bit seed -> 2^32 possible states
  BAD:   mt19937 gen(time(0));        // ~1M possible states per day
  GOOD:  mt19937 gen(seed_seq{...});  // full 624-word state seeded

  mt19937 state: 624 x 32-bit words = 19968 bits
  Single rd() call: 32 bits
  -> Must use seed_seq with 624+ random values!
```

| Seeding method | Entropy | Security | Performance |
| --- | --- | --- | --- |
| `srand(time(0))` | ~20 bits | None | N/A (rand() broken) |
| `mt19937(rd())` | 32 bits | Weak (2^32 states) | Good |
| `mt19937(seed_seq)` | ~19968 bits | Statistical only | Good |
| `random_device` directly | Full OS entropy | Cryptographic | Slow |

---

## Self-Assessment

### Q1: Why rand() is broken

`rand()` uses a Linear Congruential Generator (LCG):

```cpp
next = (a * current + c) % m
```

This formula is entirely deterministic given the current state. The reason this is a security problem is that once you know a single output and the LCG parameters (which are public), you can compute every future and past output. There is no secret involved.

The full list of problems:

1. **Predictable**: Knowing any output + the LCG parameters reveals all future outputs.
2. **Low period**: Often 2^32 at best; some implementations have 2^15.
3. **Correlated outputs**: Sequential values are correlated (not independent).
4. **Modulo bias**: `rand() % N` is biased unless `(RAND_MAX + 1) % N == 0`.
5. **Global state**: `srand()` sets global state. Thread-unsafe. Library calls may reset it.
6. **Low bits repeat**: In many LCGs, `rand() & 1` alternates 0,1,0,1.

This short program makes the predictability concrete - run it and watch both lines come out identical:

```cpp
#include <iostream>
#include <cstdlib>

int main() {
    // Demonstrate: same seed = same sequence (attacker predicts output)
    for (int trial = 0; trial < 2; ++trial) {
        srand(12345);
        std::cout << "Trial " << trial << ": ";
        for (int i = 0; i < 5; ++i)
            std::cout << rand() << ' ';
        std::cout << '\n';
    }
    // Both lines identical! If attacker knows seed, game over.

    // Modulo bias example:
    // If RAND_MAX=32767, rand()%10:
    //   0-7 appear 3277 times in full cycle
    //   8-9 appear 3276 times
    //   Bias: 0.03% (small but real for crypto)
    std::cout << "\nRAND_MAX = " << RAND_MAX << '\n';
    std::cout << "Bias in rand()%10: " << (RAND_MAX + 1) % 10 << " extra values for low digits\n";
}
```

Both trials print the same sequence because `srand(12345)` resets the generator to its starting state each time. The modulo bias feels small in percentage terms, but in cryptography even a 0.03% deviation from uniform is enough to exploit in a chosen-plaintext attack.

### Q2: Properly seed mt19937

This example shows side by side what bad seeding looks like versus full seeding, and ends with the critical reminder about why mt19937 is still not safe for crypto even when seeded perfectly.

```cpp
#include <iostream>
#include <random>
#include <array>
#include <algorithm>
#include <functional>

int main() {
    // === BAD SEEDING ===
    {
        std::random_device rd;
        std::mt19937 gen(rd());  // Only 32 bits of entropy!
        // mt19937 has 19968-bit state, but only 2^32 possible initial states
        // An attacker can enumerate all 2^32 seeds in seconds
        std::cout << "Bad seed (32-bit): first value = " << gen() << '\n';
    }

    // === GOOD SEEDING (full state) ===
    {
        std::random_device rd;

        // Generate 624 random 32-bit values from OS entropy
        std::array<uint32_t, std::mt19937::state_size> seed_data;
        std::generate(seed_data.begin(), seed_data.end(), std::ref(rd));

        // Use seed_seq to properly initialize all 624 state words
        std::seed_seq seed_seq(seed_data.begin(), seed_data.end());
        std::mt19937 gen(seed_seq);

        std::cout << "Good seed (full): first value = " << gen() << '\n';
        // 2^19968 possible initial states - infeasible to enumerate
    }

    // === IMPORTANT: mt19937 is NOT crypto-safe! ===
    {
        std::mt19937 gen(42);
        // After observing 624 consecutive outputs, the ENTIRE internal
        // state can be recovered. After that, ALL future outputs are known.
        std::cout << "\nWARNING: mt19937 is NOT cryptographic!\n";
        std::cout << "624 observed outputs -> full state recovery\n";
        std::cout << "Use for: simulations, games, shuffling\n";
        std::cout << "NOT for: tokens, keys, passwords, nonces\n";
    }

    // === Use uniform_int_distribution (not %) ===
    {
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_int_distribution<int> d6(1, 6);  // fair die
        std::cout << "\nFair d6: ";
        for (int i = 0; i < 10; ++i) std::cout << d6(gen) << ' ';
        std::cout << '\n';
    }
}
```

The key takeaway from the "NOT crypto-safe" block: mt19937's state recovery after 624 outputs is a known and published algorithm. If an attacker can observe 624 values from your generator - and in many scenarios involving web tokens, session IDs, or game seeds, they can - they own the entire future output stream. Good seeding does not fix this; it just makes the initial state harder to brute-force. But state recovery from output is a different attack that bypasses the seed entirely.

### Q3: OS entropy for cryptographic keys

When you genuinely need unpredictable bytes for keys, nonces, or tokens, you have to go to the OS. Here is the full cross-platform version with proper platform detection:

```cpp
#include <iostream>
#include <cstring>
#include <iomanip>

#ifdef _WIN32
  #include <windows.h>
  #include <bcrypt.h>
  #pragma comment(lib, "bcrypt.lib")
#elif defined(__linux__)
  #include <sys/random.h>
#elif defined(__APPLE__)
  #include <stdlib.h>  // arc4random_buf
#endif

// Generate crypto-quality random bytes
bool secure_random(void* buf, size_t len) {
#ifdef _WIN32
    return BCRYPT_SUCCESS(
        BCryptGenRandom(nullptr, (PUCHAR)buf, (ULONG)len,
                        BCRYPT_USE_SYSTEM_PREFERRED_RNG));
#elif defined(__linux__)
    // getrandom: blocks until entropy pool initialized
    // Flags: 0 = /dev/urandom (non-blocking after boot)
    //        GRND_RANDOM = /dev/random (may block)
    return getrandom(buf, len, 0) == (ssize_t)len;
#elif defined(__APPLE__)
    arc4random_buf(buf, len);  // always succeeds
    return true;
#else
    // Fallback: read from /dev/urandom
    FILE* f = fopen("/dev/urandom", "rb");
    if (!f) return false;
    bool ok = fread(buf, 1, len, f) == len;
    fclose(f);
    return ok;
#endif
}

int main() {
    // Generate a 256-bit key
    unsigned char key[32];
    if (!secure_random(key, sizeof(key))) {
        std::cerr << "Failed!\n";
        return 1;
    }

    std::cout << "256-bit crypto key:\n  ";
    for (int i = 0; i < 32; ++i)
        std::cout << std::hex << std::setw(2) << std::setfill('0')
                  << (int)key[i];
    std::cout << "\n\n";

    std::cout << std::dec;
    std::cout << "OS entropy sources:\n";
    std::cout << "  Linux: getrandom(2) feeds from kernel CSPRNG\n";
    std::cout << "  Windows: BCryptGenRandom uses CNG provider\n";
    std::cout << "  macOS: arc4random_buf uses ChaCha20-based CSPRNG\n";
    std::cout << "  Hardware: RDRAND/RDSEED instructions feed the pool\n";
}
```

Each of these OS calls ultimately draws from a kernel-maintained entropy pool seeded by hardware events. They all use a CSPRNG (Cryptographically Secure Pseudo-Random Number Generator) internally, meaning even after initialization they do not merely replay hardware noise - they use a cryptographically strong algorithm to stretch it. The output is computationally indistinguishable from true randomness.

---

## Notes

- Complementary to #737 (rand() demos, random_device usage, BCryptGenRandom).
- `std::random_device` on some platforms (MinGW) may be deterministic. Always verify.
- For tokens/nonces: use OS entropy directly (getrandom, arc4random_buf).
- For simulations: mt19937 with full seed_seq seeding is excellent.
- CWE-330: Use of Insufficiently Random Values. CWE-338: Weak PRNG.
