# Use trompeloeil for header-only mocking with Catch2 or doctest

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/rollbear/trompeloeil>  

---

## Topic Overview

**Trompeloeil** is a header-only C++14 mocking framework that integrates seamlessly with Catch2 and doctest. Unlike gMock (which requires Google Test), trompeloeil works with any testing framework via adaptors - you just include a different header and everything wires up automatically.

### Why Trompeloeil with Catch2/doctest

If you are already using Catch2 or doctest and need mocking, trompeloeil is the natural choice. The table below compares the two ecosystems; the headline differences are that trompeloeil is header-only and framework-agnostic, while gMock is tightly coupled to Google Test:

| Feature | trompeloeil + Catch2 | gMock + GTest |
| --- | :---: | :---: |
| Header-only | Yes | No (requires building gtest) |
| Works with Catch2/doctest | Yes (native) | No |
| C++14 expression templates | Yes | GTest macros |
| Compile time (100 mocks) | Fast | Slower |
| Expectation syntax | `REQUIRE_CALL` | `EXPECT_CALL` |
| Sequence enforcement | `trompeloeil::sequence` | `InSequence` |
| Matchers | `eq`, `ne`, `gt`, `ANY` | `Eq`, `Ne`, `Gt`, `_` |

### Setup

Setup is as simple as it gets. There is no library to build or link - just two includes:

```cpp
// Just two includes - no library to build
#include <catch2/catch_test_macros.hpp>
#include <catch2/trompeloeil.hpp>
// OR for doctest:
// #include <doctest/trompeloeil.hpp>
```

---

## Self-Assessment

### Q1: Define a mock with MAKE_MOCK2 and set an expectation with REQUIRE_CALL

**Answer:**

The pattern is always: define an interface, create a mock class with `MAKE_MOCK*` macros, write the system under test against the interface, then set expectations in the test. The `MAKE_MOCK2` macro takes a method name, the full function signature, and optionally `override`. The number in the macro name is just the parameter count:

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/trompeloeil.hpp>
#include <string>
#include <memory>

// Interface
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(int level, const std::string& message) = 0;
    virtual bool is_enabled(int level) const = 0;
};

// Mock - MAKE_MOCK<N> where N = param count
class MockLogger : public ILogger {
public:
    MAKE_MOCK2(log, void(int, const std::string&), override);
    MAKE_CONST_MOCK1(is_enabled, bool(int), override);
    // MAKE_MOCK2: 2 parameters + return type signature + override
    // MAKE_CONST_MOCK1: const method with 1 parameter
};

// System under test
class OrderProcessor {
    ILogger& logger_;
public:
    explicit OrderProcessor(ILogger& log) : logger_(log) {}

    bool process(int order_id) {
        logger_.log(1, "Processing order " + std::to_string(order_id));
        // ... business logic ...
        logger_.log(0, "Order " + std::to_string(order_id) + " completed");
        return true;
    }
};

// Test with REQUIRE_CALL
TEST_CASE("OrderProcessor logs start and end") {
    MockLogger mock;

    // REQUIRE_CALL: must be called exactly once (default)
    // Expectations are verified when the REQUIRE_CALL object goes out of scope
    REQUIRE_CALL(mock, log(1, trompeloeil::_))  // level=1, any message
        .WITH(_2.find("Processing") != std::string::npos);

    REQUIRE_CALL(mock, log(0, trompeloeil::_))  // level=0, any message
        .WITH(_2.find("completed") != std::string::npos);

    OrderProcessor proc(mock);
    REQUIRE(proc.process(42));
    // Mock expectations auto-verified at end of scope
}

TEST_CASE("OrderProcessor checks log level") {
    MockLogger mock;

    // ALLOW_CALL: may be called 0 or more times
    ALLOW_CALL(mock, is_enabled(trompeloeil::_))
        .RETURN(true);

    REQUIRE_CALL(mock, log(trompeloeil::_, trompeloeil::_))
        .TIMES(2);  // Called exactly 2 times

    OrderProcessor proc(mock);
    proc.process(99);
}
```

Notice that expectations are verified automatically when the `REQUIRE_CALL` object goes out of scope - you do not need to call any explicit "verify" method at the end of the test.

### Q2: Use trompeloeil::eq, ne, lt matchers to constrain call arguments

**Answer:**

Matchers let you express exactly what argument values are acceptable. `trompeloeil::_` is the wildcard (any value). For anything more precise, use the named matchers or `.WITH()` for custom predicates. Here is the full catalog in action on a metrics interface:

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/trompeloeil.hpp>
#include <string>

// Interface
class IMetrics {
public:
    virtual ~IMetrics() = default;
    virtual void record(const std::string& name, double value) = 0;
    virtual void increment(const std::string& counter, int delta) = 0;
};

class MockMetrics : public IMetrics {
public:
    MAKE_MOCK2(record, void(const std::string&, double), override);
    MAKE_MOCK2(increment, void(const std::string&, int), override);
};

// System under test
class RequestHandler {
    IMetrics& metrics_;
public:
    explicit RequestHandler(IMetrics& m) : metrics_(m) {}

    void handle(int status_code, double latency_ms) {
        metrics_.record("latency", latency_ms);
        if (status_code >= 400) {
            metrics_.increment("errors", 1);
        }
        if (latency_ms > 1000.0) {
            metrics_.increment("slow_requests", 1);
        }
    }
};

TEST_CASE("Matchers constrain arguments") {
    MockMetrics mock;

    SECTION("trompeloeil::eq - exact match") {
        REQUIRE_CALL(mock, record(trompeloeil::eq("latency"s), trompeloeil::_));
        // Only matches if first arg is exactly "latency"

        ALLOW_CALL(mock, increment(trompeloeil::_, trompeloeil::_));

        RequestHandler handler(mock);
        handler.handle(200, 50.0);
    }

    SECTION("trompeloeil::ne - not equal") {
        REQUIRE_CALL(mock, record(trompeloeil::ne(""s), trompeloeil::_));
        // Matches any non-empty string

        ALLOW_CALL(mock, increment(trompeloeil::_, trompeloeil::_));

        RequestHandler handler(mock);
        handler.handle(200, 50.0);
    }

    SECTION("trompeloeil::gt, lt, ge, le - comparisons") {
        REQUIRE_CALL(mock, record(trompeloeil::_, trompeloeil::gt(0.0)));
        // latency must be > 0.0

        REQUIRE_CALL(mock, increment("errors"s, trompeloeil::ge(1)));
        // delta must be >= 1

        RequestHandler handler(mock);
        handler.handle(500, 50.0);
    }

    SECTION("trompeloeil::re - regex matcher") {
        REQUIRE_CALL(mock, record(trompeloeil::re("^lat.*"), trompeloeil::_));
        // Name must match regex "^lat.*"

        ALLOW_CALL(mock, increment(trompeloeil::_, trompeloeil::_));

        RequestHandler handler(mock);
        handler.handle(200, 50.0);
    }

    SECTION(".WITH() for custom predicates") {
        REQUIRE_CALL(mock, record(trompeloeil::_, trompeloeil::_))
            .WITH(_1.size() > 3 && _2 < 10000.0);
        // _1 = first arg (string), _2 = second arg (double)
        // Custom compound predicate

        ALLOW_CALL(mock, increment(trompeloeil::_, trompeloeil::_));

        RequestHandler handler(mock);
        handler.handle(200, 50.0);
    }
}

// Matcher quick reference:
// trompeloeil::eq(x)   - == x
// trompeloeil::ne(x)   - != x
// trompeloeil::gt(x)   - > x
// trompeloeil::ge(x)   - >= x
// trompeloeil::lt(x)   - < x
// trompeloeil::le(x)   - <= x
// trompeloeil::re(pat) - regex match
// trompeloeil::_       - any value (wildcard)
// .WITH(predicate)     - custom lambda using _1, _2, etc.
```

The `.WITH()` clause is the escape hatch for anything the named matchers cannot express. The `_1`, `_2` placeholders refer to the first and second arguments respectively.

### Q3: Explain how trompeloeil integrates with Catch2 CHECK macros for non-fatal expectation failures

**Answer:**

Trompeloeil reports violations through the test framework's **failure reporting** mechanism. With Catch2, this means mock failures look exactly like normal assertion failures in the test output - same format, same source location reporting.

**Fatal vs Non-Fatal:**

| trompeloeil Macro | Catch2 Equivalent | Behavior on Failure |
| --- | --- | --- |
| `REQUIRE_CALL` | Like `REQUIRE` | **Fatal** - test stops immediately if unmet at scope exit |
| `ALLOW_CALL` | - | Non-fatal - 0 calls is fine |
| `FORBID_CALL` | - | **Fatal** - any call is an error |

The wiring happens through the adaptor header. When you include `<catch2/trompeloeil.hpp>`, it registers a reporter that maps trompeloeil's violation callbacks to Catch2's failure mechanism. You do not have to do anything special - it just works:

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/trompeloeil.hpp>  // <- This adaptor header does the magic

// The adaptor (#include <catch2/trompeloeil.hpp>) defines:
// - trompeloeil::reporter that maps to Catch2's FAIL() / CHECK()
// - Violations appear in Catch2's normal test output with source location
// - Stack trace shows WHERE the unexpected call happened

class INotifier {
public:
    virtual ~INotifier() = default;
    virtual void notify(const std::string& msg) = 0;
};

class MockNotifier : public INotifier {
public:
    MAKE_MOCK1(notify, void(const std::string&), override);
};

TEST_CASE("Violation reported through Catch2") {
    MockNotifier mock;

    // FORBID_CALL: if notify() is called, Catch2 reports FAILURE
    FORBID_CALL(mock, notify(trompeloeil::_));

    // If code under test calls mock.notify("oops"), output is:
    //
    // test.cpp:42: FAILED:
    //   Match of forbidden call of notify with signature void(const std::string&)
    //     param  _1 == "oops"
    //
    // (Same format as any Catch2 REQUIRE failure)
}

TEST_CASE("Unsatisfied REQUIRE_CALL reported at scope exit") {
    MockNotifier mock;

    {
        REQUIRE_CALL(mock, notify("hello"s));
        // If we forget to call mock.notify("hello") before } ...
    }
    // At }, the REQUIRE_CALL destructor checks satisfaction
    // If unsatisfied: Catch2 reports:
    //   Unfulfilled expectation:
    //     test.cpp:50: Expected notify("hello") to be called 1 time, was called 0 times
}

// doctest works identically
// Just change the include:
// #include <doctest/trompeloeil.hpp>
// Same macros, same behavior, different reporting backend
```

**Key integration details:**

1. The adaptor header registers a `trompeloeil::reporter` during static init.
2. Unmet expectations use Catch2's `FAIL()` - reported as test failures in the normal output.
3. Source location points to the `REQUIRE_CALL` line, not the mock definition.
4. Multiple violations in one test are all reported (Catch2 section isolation handles this cleanly).

---

## Notes

- **Header-only**: just `#include <trompeloeil.hpp>` - no library to link or build step required.
- `MAKE_MOCK0` through `MAKE_MOCK15` cover methods with 0 to 15 parameters.
- `MAKE_CONST_MOCK*` is for const member functions - do not forget this or you will get a confusing compile error.
- Expectations are checked in **LIFO** order (last defined = first matched), which matters when you have multiple expectations for the same method.
- Use `NAMED_REQUIRE_CALL` to store an expectation in a variable so you can add `.RETURN()` changes after the fact.
- `TIMES(n)` for exact count; `TIMES(AT_LEAST(1))` and `TIMES(AT_MOST(3))` for ranges.
- Compile with `-std=c++14` minimum; C++17 is recommended for CTAD support.
