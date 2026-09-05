# Write testable C++ code by designing for dependency injection

**Category:** Testing and Verification  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ri-abstract>  

---

## Topic Overview

**Dependency Injection (DI)** means passing dependencies into a class instead of having it create them internally. This is the single most important design technique for testability - it lets you swap real implementations for test doubles without changing any production code.

The reason this matters so much in testing is that classes which create their own dependencies are inherently hard to test in isolation. A `Timer` that calls `std::chrono::steady_clock::now()` directly cannot be tested without actually waiting for real time to pass. A `Cache` that opens a real network connection cannot be unit tested without a network. DI breaks this coupling.

### Untestable vs Testable

```cpp
UNTESTABLE (hardcoded dependency):         TESTABLE (injected dependency):
.......................                    ......................
| class Timer {       |                    | class Timer {        |
|   SystemClock clk_; | <- can't mock!    |   IClock& clk_;     | <- inject!
|   Timer() {         |                    |   Timer(IClock& c)  |
|     clk_ = ...;     |                    |     : clk_(c) {}    |
|   }                 |                    | };                   |
| };                  |                    ......................
.......................                          ^           ^
                                           SystemClock   FakeClock
                                          (production)     (test)
```

### Three DI Styles in C++

C++ offers three distinct flavors of dependency injection, each with different trade-offs. The table below summarizes them; the sections below go into detail:

| Style | Mechanism | Overhead | Flexibility |
| --- | --- | --- | --- |
| **Interface injection** | Virtual functions (`IClock&`) | vtable | Runtime polymorphism |
| **Constructor injection** | Same + stored in member | vtable | Most common |
| **Template injection** | `template<Clock C> class Timer` | Zero | Compile-time, fastest |

---

## Self-Assessment

### Q1: Refactor a class that directly instantiates a Clock to accept a Clock concept or interface

**Answer:**

The original `CacheBAD` class calls `std::chrono::steady_clock::now()` directly in its constructor and `is_expired` method. There is no way to control what time it sees during a test. The refactored version accepts any object that satisfies the `IClock` interface - or the `ClockLike` concept in the C++20 template version:

```cpp
#include <chrono>
#include <string>
#include <concepts>

// BEFORE: Untestable - hardcoded SystemClock
class CacheBAD {
    std::chrono::steady_clock::time_point last_refresh_;
    int ttl_seconds_;
public:
    explicit CacheBAD(int ttl) : ttl_seconds_(ttl) {
        last_refresh_ = std::chrono::steady_clock::now();  // Hardcoded!
    }
    bool is_expired() const {
        auto now = std::chrono::steady_clock::now();  // Hardcoded!
        auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
            now - last_refresh_).count();
        return elapsed > ttl_seconds_;
    }
    // Testing this requires actual waiting (sleep) - SLOW and flaky!
};

// AFTER (Option A): Interface injection
class IClock {
public:
    virtual ~IClock() = default;
    virtual std::chrono::steady_clock::time_point now() const = 0;
};

class SystemClock : public IClock {
public:
    std::chrono::steady_clock::time_point now() const override {
        return std::chrono::steady_clock::now();
    }
};

class Cache {
    const IClock& clock_;
    std::chrono::steady_clock::time_point last_refresh_;
    int ttl_seconds_;
public:
    Cache(const IClock& clock, int ttl)
        : clock_(clock), ttl_seconds_(ttl) {
        last_refresh_ = clock_.now();
    }

    bool is_expired() const {
        auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
            clock_.now() - last_refresh_).count();
        return elapsed > ttl_seconds_;
    }
};

// AFTER (Option B): Template/Concept injection (C++20)
template<typename T>
concept ClockLike = requires(const T& c) {
    { c.now() } -> std::same_as<std::chrono::steady_clock::time_point>;
};

template<ClockLike C>
class CacheT {
    const C& clock_;
    std::chrono::steady_clock::time_point last_refresh_;
    int ttl_seconds_;
public:
    CacheT(const C& clock, int ttl)
        : clock_(clock), ttl_seconds_(ttl) {
        last_refresh_ = clock_.now();
    }

    bool is_expired() const {
        auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
            clock_.now() - last_refresh_).count();
        return elapsed > ttl_seconds_;
    }
    // No vtable overhead - everything resolved at compile time
};
```

The C++20 concept version gives you a cleaner error message if you accidentally pass something that does not have a `now()` method - the compiler tells you the concept constraint failed rather than giving a wall of template substitution errors.

### Q2: Show that injecting a fake clock enables deterministic time-dependent tests without sleep()

**Answer:**

This is the payoff. With a `FakeClock` you can advance time instantly in a test - no `sleep_for(61s)` needed. Each test runs in microseconds regardless of what TTL values are involved:

```cpp
#include <gtest/gtest.h>
#include <chrono>

using TimePoint = std::chrono::steady_clock::time_point;
using namespace std::chrono_literals;

// Fake clock - fully controlled by test
class FakeClock : public IClock {
    mutable TimePoint current_;
public:
    FakeClock() : current_(std::chrono::steady_clock::now()) {}

    TimePoint now() const override { return current_; }

    // Test controls: advance time instantly
    void advance(std::chrono::seconds delta) {
        current_ += delta;
    }
    void set(TimePoint tp) { current_ = tp; }
};

// Tests run in microseconds, no sleep() needed
TEST(CacheTest, NotExpiredInitially) {
    FakeClock clock;
    Cache cache(clock, 60);  // 60 second TTL

    EXPECT_FALSE(cache.is_expired());  // Just created - not expired
}

TEST(CacheTest, ExpiresAfterTTL) {
    FakeClock clock;
    Cache cache(clock, 60);

    clock.advance(59s);                 // 59 seconds later
    EXPECT_FALSE(cache.is_expired());   // Still valid

    clock.advance(2s);                  // 61 seconds total
    EXPECT_TRUE(cache.is_expired());    // Now expired!
}

TEST(CacheTest, ExpiresExactlyAtBoundary) {
    FakeClock clock;
    Cache cache(clock, 10);

    clock.advance(10s);                 // Exactly at TTL
    EXPECT_FALSE(cache.is_expired());   // Not > 10, just ==

    clock.advance(1s);                  // 11 seconds
    EXPECT_TRUE(cache.is_expired());
}

// Template version - same pattern, zero overhead
TEST(CacheTemplateTest, ExpiresAfterTTL) {
    FakeClock clock;
    CacheT cache(clock, 30);  // CTAD deduces CacheT<FakeClock>

    clock.advance(31s);
    EXPECT_TRUE(cache.is_expired());
}

// Without DI, this test would need:
//   std::this_thread::sleep_for(61s);  <- 61 seconds per test!
// With FakeClock:
//   clock.advance(61s);  <- instant
```

The `ExpiresExactlyAtBoundary` test is the important one here - it tests the off-by-one at `elapsed > ttl_seconds_`. Without a fake clock you would never bother writing this test because you cannot advance time to a precise second boundary reliably with `sleep_for`.

### Q3: Explain the difference between interface injection, constructor injection, and template injection

**Answer:**

The three styles serve different purposes and involve real trade-offs. Understanding when to reach for each one is the practical skill here.

| Aspect | Interface Injection | Constructor Injection | Template Injection |
| --- | :---: | :---: | :---: |
| **Mechanism** | Pass `IBase&` | Store `IBase&` in ctor | `template<class T>` |
| **Binding** | Runtime (vtable) | Runtime (vtable) | Compile-time |
| **Overhead** | Virtual call per use | Virtual call per use | Zero (inlined) |
| **Swap deps at runtime?** | Yes | No (set at construction) | No |
| **Binary size** | One instantiation | One instantiation | Per-type instantiation |
| **Testability** | Mock via virtual | Mock via virtual | Mock via duck typing |

```cpp
// 1. Interface injection (method parameter)
class Renderer {
public:
    void render(ICanvas& canvas) {     // Dependency passed per call
        canvas.draw_line(0, 0, 100, 100);
    }
    // Different calls can use different canvases
};

// 2. Constructor injection (most common)
class Renderer2 {
    ICanvas& canvas_;
public:
    explicit Renderer2(ICanvas& canvas) : canvas_(canvas) {}  // Set once
    void render() {
        canvas_.draw_line(0, 0, 100, 100);
    }
    // Canvas is fixed for lifetime of Renderer2
};

// 3. Template injection (zero-overhead)
template<typename Canvas>
class Renderer3 {
    Canvas& canvas_;
public:
    explicit Renderer3(Canvas& canvas) : canvas_(canvas) {}
    void render() {
        canvas_.draw_line(0, 0, 100, 100);  // No vtable! Direct call
    }
    // Fastest, but Renderer3<RealCanvas> and Renderer3<MockCanvas>
    // are DIFFERENT TYPES - can't store in same container
};

// When to use each:
// Interface injection:   When dependency varies per operation
// Constructor injection: DEFAULT choice - clean, well-understood
// Template injection:    Performance-critical paths (tight loops, real-time)

// Bonus: std::function injection
class Renderer4 {
    std::function<void(int,int,int,int)> draw_line_;
public:
    explicit Renderer4(std::function<void(int,int,int,int)> fn)
        : draw_line_(std::move(fn)) {}
    void render() { draw_line_(0, 0, 100, 100); }
    // Lightweight - no interface class needed
    // Good for simple callbacks, bad for complex deps
};
```

The reason template injection trips people up is the type erasure consequence: `Renderer3<RealCanvas>` and `Renderer3<MockCanvas>` are completely different types. You cannot store them in the same `std::vector` or return them from the same factory function. Interface injection does not have this problem. So for performance-critical inner loops, template injection wins; for everything else, constructor injection with virtual interfaces is the practical default.

---

## Notes

- Constructor injection is the go-to for most C++ code - it is clean, explicit, and universally understood.
- Template injection is C++'s unique advantage: zero-overhead DI with no equivalent in Java or C#. Use it when the virtual call overhead genuinely matters.
- C++20 concepts replace SFINAE for constraining template DI parameters - much cleaner error messages.
- Avoid the "service locator" anti-pattern, where a class pulls its dependencies from a global registry. It makes dependencies hidden and tests fragile.
- Use `std::unique_ptr<IBase>` for ownership transfer; `IBase&` for non-owning references. The choice between them clarifies ownership intent.
- DI containers (like Boost.DI) exist for C++ but are rarely needed - constructor injection with a few factory functions usually suffices even in large codebases.
