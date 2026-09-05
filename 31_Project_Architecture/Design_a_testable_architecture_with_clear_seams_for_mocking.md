# Design a testable architecture with clear seams for mocking

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

A **seam** is a point where you can change behavior without editing the code itself - typically an interface boundary. The idea is simple: if every external dependency goes through an interface, you can swap in a test double during testing without touching the production code at all.

The reason this matters is that the hardest code to test is usually code that calls the clock, the random number generator, the filesystem, the network, or the database. Those things are unpredictable, slow, or hard to set up in a test environment. When you put an interface in front of them, you regain control. You can freeze time, make random numbers deterministic, simulate network failures - whatever the test needs.

If a class is hard to test, that is almost always a design problem, not a testing problem. Hard-to-test code is usually tightly coupled to its dependencies. The fix is to inject those dependencies through a seam.

### Types of Seams in C++

| Seam Type | Mechanism | Overhead | Flexibility |
| --- | --- | --- | --- |
| **Virtual interface** | `virtual` + `override` | vtable dispatch | Runtime swap |
| **Template parameter** | Template injection | None (inlined) | Compile-time swap |
| **Function pointer/std::function** | Callback injection | Indirect call | Runtime swap |
| **Link-time** | Swap `.o` files / weak symbols | None | Build-time swap |
| **Preprocessor** | `#ifdef TESTING` | None | Build-time swap |

---

## Self-Assessment

### Q1: Design seams using virtual interfaces

**Answer:**

Here's a `SessionManager` that depends on two external things: the current time (for expiry checks) and a random number generator (for token creation). Both are injected through pure virtual interfaces. In production you use `SystemClock` and `RealRng`. In tests you use `FakeClock` and `DeterministicRng`. The `SessionManager` itself never changes.

```cpp
#include <chrono>
#include <string>
#include <memory>

// === Seam 1: Time provider ===
class IClock {
public:
    virtual ~IClock() = default;
    using TimePoint = std::chrono::system_clock::time_point;
    virtual TimePoint now() const = 0;
};

class SystemClock : public IClock {
public:
    TimePoint now() const override {
        return std::chrono::system_clock::now();
    }
};

class FakeClock : public IClock {
public:
    TimePoint now() const override { return fixed_time_; }
    void set_time(TimePoint t) { fixed_time_ = t; }
    void advance(std::chrono::milliseconds ms) { fixed_time_ += ms; }
private:
    TimePoint fixed_time_ = std::chrono::system_clock::now();
};

// === Seam 2: Random number generator ===
class IRng {
public:
    virtual ~IRng() = default;
    virtual int generate(int min, int max) = 0;
};

class RealRng : public IRng {
public:
    int generate(int min, int max) override {
        std::uniform_int_distribution<int> dist(min, max);
        return dist(engine_);
    }
private:
    std::mt19937 engine_{std::random_device{}()};
};

class DeterministicRng : public IRng {
public:
    explicit DeterministicRng(std::vector<int> values)
        : values_(std::move(values)) {}
    int generate(int, int) override {
        return values_[index_++ % values_.size()];
    }
private:
    std::vector<int> values_;
    size_t index_ = 0;
};

// === Service using seams ===
class SessionManager {
public:
    SessionManager(IClock& clock, IRng& rng)
        : clock_(clock), rng_(rng) {}

    std::string create_session(const std::string& user) {
        auto now = clock_.now();
        auto token = std::to_string(rng_.generate(100000, 999999));
        sessions_[token] = {user, now};
        return token;
    }

    bool is_valid(const std::string& token,
                   std::chrono::minutes max_age) const {
        auto it = sessions_.find(token);
        if (it == sessions_.end()) return false;
        return (clock_.now() - it->second.created) < max_age;
    }

private:
    struct Session {
        std::string user;
        IClock::TimePoint created;
    };
    IClock& clock_;
    IRng& rng_;
    std::unordered_map<std::string, Session> sessions_;
};
```

The constructor takes references to the interfaces, not concrete types. That single decision is what makes the whole class testable without any other changes.

### Q2: Test with controlled seams

**Answer:**

Watch how these tests become completely deterministic. `FakeClock::advance()` lets you jump forward in time without sleeping. `DeterministicRng` returns whatever sequence of values you pre-load. The tests run instantly and never flake.

```cpp
#include <gtest/gtest.h>

TEST(SessionManagerTest, CreateSessionReturnsToken) {
    FakeClock clock;
    DeterministicRng rng({123456});
    SessionManager mgr(clock, rng);

    auto token = mgr.create_session("alice");
    EXPECT_EQ(token, "123456");
}

TEST(SessionManagerTest, SessionExpiresAfterTimeout) {
    FakeClock clock;
    DeterministicRng rng({111111});
    SessionManager mgr(clock, rng);

    auto token = mgr.create_session("bob");
    EXPECT_TRUE(mgr.is_valid(token, std::chrono::minutes(30)));

    // Advance clock past expiry
    clock.advance(std::chrono::minutes(31));
    EXPECT_FALSE(mgr.is_valid(token, std::chrono::minutes(30)));
}

TEST(SessionManagerTest, InvalidTokenIsNotValid) {
    FakeClock clock;
    DeterministicRng rng({999999});
    SessionManager mgr(clock, rng);

    EXPECT_FALSE(mgr.is_valid("nonexistent", std::chrono::minutes(30)));
}
```

The `SessionExpiresAfterTimeout` test is a good example of how seams pay off. Without a `FakeClock`, you would either have to actually wait 31 minutes, or add `sleep()` hacks, or skip the test entirely. With the seam in place, the test is one line: `clock.advance(std::chrono::minutes(31))`.

### Q3: Template-based seams for zero-overhead testing

**Answer:**

Virtual dispatch has a small runtime cost. If you are on embedded or writing performance-sensitive code, you can get the same testability benefits using templates instead. The template version is resolved at compile time, so there is no vtable lookup - the fake types are inlined just like the real ones. The tradeoff is that you cannot swap them at runtime.

```cpp
// === Compile-time seams via templates ===
template<typename Clock, typename Rng>
class SessionManagerT {
public:
    SessionManagerT(Clock& clock, Rng& rng)
        : clock_(clock), rng_(rng) {}

    std::string create_session(const std::string& user) {
        auto now = clock_.now();
        auto token = std::to_string(rng_.generate(100000, 999999));
        sessions_[token] = {user, now};
        return token;
    }

    // ... same interface ...

private:
    Clock& clock_;
    Rng& rng_;
    // ...
};

// Production: uses real implementations, zero virtual dispatch
// using ProdSessionMgr = SessionManagerT<SystemClock, RealRng>;

// Test: uses fakes, still zero virtual dispatch
// using TestSessionMgr = SessionManagerT<FakeClock, DeterministicRng>;

// === Link-time seam (for C-style APIs like hardware) ===
// In production: link real_timer.o
// In tests: link fake_timer.o

// timer.h:
uint32_t get_tick_count();  // Declaration only

// real_timer.cpp (production):
// uint32_t get_tick_count() { return HAL_GetTick(); }

// fake_timer.cpp (test):
static uint32_t fake_ticks = 0;
uint32_t get_tick_count() { return fake_ticks; }
void set_fake_ticks(uint32_t t) { fake_ticks = t; }
```

The link-time seam at the bottom is worth noting for embedded work. When you have a C-style hardware API (`HAL_GetTick()`, for example) that you cannot put behind a virtual interface, the link-time approach lets you swap implementations by choosing which `.o` file to include in the build. The caller's code never changes at all.

---

## Notes

- Every external dependency should be behind a seam - time, random numbers, filesystem, network, and database are the usual suspects.
- `FakeClock` is the most common seam you will write - it unlocks testing of any time-dependent code without real delays.
- Virtual interface seams are the most flexible option, with a slight runtime cost from vtable dispatch.
- Template seams have zero overhead and work well for embedded, but cannot be swapped at runtime.
- Link-time seams are useful for replacing C-style hardware APIs without modifying the calling code.
- Avoid `static` and global state - it creates implicit dependencies that cannot be replaced through any seam.
- If a class is hard to test, that is a design problem, not a testing problem.
