# Use named constants instead of magic numbers throughout the codebase

**Category:** Best Practices and Idioms  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Res-magic>  

---

## Topic Overview

**Magic numbers** are bare literal values sitting in code with no explanation. The problem isn't just readability - it's maintainability. When `1024` appears seven times in a codebase, you don't know which ones are "max packet size" and which ones are coincidentally the same value for an unrelated reason. Replace them with `constexpr` named constants, and the intent is captured at the point of declaration, reusable everywhere, and changeable in one place.

```cpp
BAD:  buffer.resize(1024);    // Why 1024? Max packet? Page size? Random?
GOOD: buffer.resize(kMaxPacketSize);  // Self-documenting
```

---

## Self-Assessment

### Q1: Replace magic numbers with named constants

Look at `process_bad` below. The value `1024` appears three times, and a reader has no way to know whether all three refer to the same concept or whether one of them is coincidental. `process_good` makes the relationship explicit - every use of the constant visibly refers to the same named thing.

```cpp
#include <array>
#include <iostream>
#include <vector>

// BAD: magic numbers everywhere
void process_bad(const char* data, int len) {
    if (len > 1024) return;           // what is 1024?
    if (len < 64) return;             // what is 64?
    char buffer[1024];                // again 1024
    for (int i = 0; i < 1024; ++i)    // and again...
        buffer[i] = 0;
    // If we change max packet size, we must find ALL 1024s
}

// GOOD: named constants
namespace protocol {
    constexpr int kMaxPacketSize = 1024;
    constexpr int kMinPacketSize = 64;
    constexpr int kHeaderSize = 12;
    constexpr int kMaxPayload = kMaxPacketSize - kHeaderSize;
    constexpr int kDefaultPort = 8080;
    constexpr int kMaxRetries = 3;
    constexpr double kTimeoutSeconds = 5.0;
}

void process_good(const char* data, int len) {
    if (len > protocol::kMaxPacketSize) return;
    if (len < protocol::kMinPacketSize) return;
    std::array<char, protocol::kMaxPacketSize> buffer{};
    std::cout << "Processing " << len << " bytes\n";
    std::cout << "Max payload: " << protocol::kMaxPayload << '\n';
}

int main() {
    char data[100] = {};
    process_good(data, 100);
}
// Expected output:
// Processing 100 bytes
// Max payload: 1012
```

Notice also that `kMaxPayload` is defined in terms of the other constants. If you change `kHeaderSize`, `kMaxPayload` updates automatically everywhere it's used - no hunting required.

### Q2: Show maintenance cost of magic numbers vs named constants

Here's the scenario that makes the cost concrete. You need to change a timeout from 5000 ms to 3000 ms. With magic numbers, you search the codebase for `5000`, find ten matches, and now you have to figure out which ones are timeouts and which ones are buffer sizes or port numbers that happen to be the same value. With a named constant, you change one line and you're done.

```cpp
#include <iostream>

// Scenario: change timeout from 5000ms to 3000ms

// BAD: magic number - must find every 5000 in the codebase
// Some 5000s might be timeouts, others might be something else!
/*
    connect(host, 5000);       // timeout?
    buffer.resize(5000);       // buffer size? NOT a timeout!
    retry_after(5000);         // timeout
    if (elapsed > 5000) ...    // timeout
*/

// GOOD: named constant - change ONE place
constexpr int kTimeoutMs = 3000;  // changed here, done!

void connect(const char* host, int timeout) {
    std::cout << "Connecting to " << host << " timeout=" << timeout << "ms\n";
}

void retry_after(int ms) {
    std::cout << "Retry after " << ms << "ms\n";
}

int main() {
    connect("server.com", kTimeoutMs);
    retry_after(kTimeoutMs);

    int elapsed = 4000;
    if (elapsed > kTimeoutMs)
        std::cout << "Timed out!\n";
}
// Expected output:
// Connecting to server.com timeout=3000ms
// Retry after 3000ms
// Timed out!
```

Every call that uses `kTimeoutMs` picks up the change automatically. The `buffer.resize(5000)` case is left alone because it was a different constant with the same value - and with named constants, you would never have had that confusion in the first place.

### Q3: Use `constexpr` with user-defined literals for self-documenting constants

For numeric values with physical units, user-defined literals (UDLs) take named constants one step further: the unit becomes part of the value's syntax. You can't accidentally pass meters to a function expecting kilograms when the types carry their units.

```cpp
#include <chrono>
#include <iostream>

// User-defined literals for domain-specific units
constexpr long double operator""_km(long double val) { return val * 1000.0L; }
constexpr long double operator""_m(long double val) { return val; }
constexpr long double operator""_cm(long double val) { return val / 100.0L; }

constexpr long double operator""_kg(long double val) { return val; }
constexpr long double operator""_g(long double val) { return val / 1000.0L; }

int main() {
    // BAD: what unit is 42000?
    // double distance = 42000;

    // GOOD: self-documenting with UDLs
    constexpr auto marathon = 42.195_km;   // 42195.0 meters
    constexpr auto height = 180.0_cm;      // 1.8 meters
    constexpr auto weight = 75.0_kg;       // 75.0 kg
    constexpr auto coin = 5.0_g;           // 0.005 kg

    std::cout << "Marathon: " << marathon << " m\n";
    std::cout << "Height: " << height << " m\n";
    std::cout << "Weight: " << weight << " kg\n";
    std::cout << "Coin: " << coin << " kg\n";

    // std::chrono has built-in literals:
    using namespace std::chrono_literals;
    auto timeout = 500ms;
    auto delay = 2s;
    std::cout << "Timeout: " << timeout.count() << " ms\n";
    std::cout << "Delay: " << delay.count() << " s\n";
}
// Expected output:
// Marathon: 42195 m
// Height: 1.8 m
// Weight: 75 kg
// Coin: 0.005 kg
// Timeout: 500 ms
// Delay: 2 s
```

The `std::chrono` literals (`500ms`, `2s`) are the most common example of this in the standard library. They make time values unambiguous in a way that bare integers simply cannot.

---

## Notes

- `constexpr` vs `const`: `constexpr` is evaluated at compile time; `const` may be a runtime value. Prefer `constexpr` for true constants.
- Use `inline constexpr` in headers to avoid ODR violations when the constant is used across multiple translation units.
- Acceptable "magic numbers": `0`, `1`, `-1` (loop bounds and identity values), `2` (halving). Everything else should be named.
- Namespaces group related constants meaningfully: `protocol::kMaxPacketSize`, `ui::kDefaultWidth`.
