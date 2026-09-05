# Use Google Mock / trompeloeil for mock objects in unit tests

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/rollbear/trompeloeil>  

---

## Topic Overview

This file focuses on **trompeloeil** - a header-only C++14 mocking framework that integrates natively with Catch2 and doctest. (See files #584 and #681 for Google Mock's MOCK_METHOD, EXPECT_CALL, ON_CALL patterns.)

If you are using Catch2 or doctest rather than Google Test, trompeloeil is probably the most natural choice for mocking. It is a single header with no build dependency on Google Test, and its expectation syntax integrates directly into your test framework's failure reporting so failures read as part of the test, not as mysterious assertion errors from a different library.

### gMock vs trompeloeil

The table below covers the practical differences. Both frameworks cover the same use cases; the choice mostly comes down to which test framework you are already using.

| Feature | Google Mock | trompeloeil |
| --- | :---: | :---: |
| Dependencies | Part of Google Test | Header-only, standalone |
| Test framework | GTest only | Catch2, doctest, GTest, or any |
| Syntax style | Macro-based | Expression templates |
| C++ standard | C++11+ | C++14+ |
| Sequence enforcement | `InSequence` | `trompeloeil::sequence` |
| Argument matching | Built-in matchers | Lambda matchers |
| Learning curve | Moderate | Lower (more readable) |

---

## Self-Assessment

### Q1: Write a mock for an interface using MOCK_METHOD and verify a method is called exactly once

This example uses gMock. `ConfigWriter` wraps a filesystem interface, and the test verifies that exactly one `write` call happens with the correct path and value. The mock's destructor takes care of the verification - you never have to call `verify` yourself.

```cpp
// gMock: MOCK_METHOD + Times(1)
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

class IFileSystem {
public:
    virtual ~IFileSystem() = default;
    virtual bool write(const std::string& path, const std::string& data) = 0;
    virtual std::string read(const std::string& path) = 0;
    virtual bool exists(const std::string& path) const = 0;
};

class MockFileSystem : public IFileSystem {
public:
    MOCK_METHOD(bool, write, (const std::string&, const std::string&), (override));
    MOCK_METHOD(std::string, read, (const std::string&), (override));
    MOCK_METHOD(bool, exists, (const std::string&), (const, override));
};

// Code under test
class ConfigWriter {
    IFileSystem& fs_;
public:
    explicit ConfigWriter(IFileSystem& fs) : fs_(fs) {}

    bool save(const std::string& key, const std::string& value) {
        return fs_.write("/etc/config/" + key, value);
    }
};

TEST(ConfigWriter, SaveWritesExactlyOnce) {
    MockFileSystem mock_fs;

    // Expect write() called exactly once with specific path
    EXPECT_CALL(mock_fs, write("/etc/config/theme", "dark"))
        .Times(1)
        .WillOnce(::testing::Return(true));

    ConfigWriter writer(mock_fs);
    EXPECT_TRUE(writer.save("theme", "dark"));
    // Destructor of mock_fs verifies: exactly 1 call confirmed
}
```

### Q2: Use EXPECT_CALL with argument matchers to verify the correct arguments are passed

Matchers let you be precise about individual arguments while being flexible about others. In this test, the latency must be positive (we use `Gt(0.0)`) but we do not care about its exact value, while the path must start with `/api`. Mixing precise and loose matchers in the same call is entirely normal.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

using ::testing::_;
using ::testing::HasSubstr;
using ::testing::StartsWith;
using ::testing::Gt;
using ::testing::AllOf;
using ::testing::Return;

class IMetrics {
public:
    virtual ~IMetrics() = default;
    virtual void record(const std::string& name, double value) = 0;
    virtual void tag(const std::string& key, const std::string& val) = 0;
};

class MockMetrics : public IMetrics {
public:
    MOCK_METHOD(void, record, (const std::string&, double), (override));
    MOCK_METHOD(void, tag, (const std::string&, const std::string&), (override));
};

class RequestHandler {
    IMetrics& metrics_;
public:
    explicit RequestHandler(IMetrics& m) : metrics_(m) {}

    void handle(const std::string& path, int status, double latency_ms) {
        metrics_.record("http.latency_ms", latency_ms);
        metrics_.record("http.status", static_cast<double>(status));
        metrics_.tag("http.path", path);
        if (status >= 500)
            metrics_.tag("http.error", "server_error");
    }
};

TEST(RequestHandler, RecordsMetricsWithCorrectArgs) {
    MockMetrics mock;

    // Argument matchers for precise verification
    EXPECT_CALL(mock, record("http.latency_ms", Gt(0.0)));  // latency > 0
    EXPECT_CALL(mock, record("http.status", 200.0));

    EXPECT_CALL(mock, tag("http.path", StartsWith("/api")));
    EXPECT_CALL(mock, tag("http.error", _)).Times(0);  // No error for 200

    RequestHandler handler(mock);
    handler.handle("/api/users", 200, 42.5);
}

TEST(RequestHandler, ServerErrorTagsError) {
    MockMetrics mock;

    EXPECT_CALL(mock, record(_, _)).Times(2);  // Don't care about values

    // Care about specific tag calls
    EXPECT_CALL(mock, tag("http.path", "/crash"));
    EXPECT_CALL(mock, tag("http.error", "server_error"));

    RequestHandler handler(mock);
    handler.handle("/crash", 500, 999.0);
}
```

### Q3: Show how trompeloeil integrates with Catch2 for expectation failure reporting

trompeloeil uses a different vocabulary from gMock, but the concepts map across cleanly. `REQUIRE_CALL` is the equivalent of `EXPECT_CALL` - the call must happen or the test fails. `ALLOW_CALL` is the equivalent of `ON_CALL` - the call may or may not happen. `FORBID_CALL` maps to `EXPECT_CALL(...).Times(0)`. Sequence enforcement uses a `trompeloeil::sequence` object and the `.IN_SEQUENCE(seq)` modifier.

One particularly nice feature shown here is `trompeloeil::re("pattern")` - a built-in regex matcher that lets you match strings without writing a lambda.

```cpp
// trompeloeil with Catch2
// Single header: #include <trompeloeil.hpp>
// Catch2 adapter: #include <catch2/trompeloeil.hpp>

#include <catch2/catch.hpp>
#include <trompeloeil.hpp>
#include <catch2/trompeloeil.hpp>  // Adapter for Catch2 reporting
#include <string>

// Interface
class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual void connect(const std::string& url) = 0;
    virtual std::string query(const std::string& sql) = 0;
    virtual void disconnect() = 0;
};

// trompeloeil mock
class MockDatabase : public IDatabase {
public:
    MAKE_MOCK1(connect, void(const std::string&), override);
    MAKE_MOCK1(query, std::string(const std::string&), override);
    MAKE_MOCK0(disconnect, void(), override);
};

// Code under test
class ReportGenerator {
    IDatabase& db_;
public:
    explicit ReportGenerator(IDatabase& db) : db_(db) {}

    std::string generate(const std::string& report_id) {
        db_.connect("postgres://localhost/mydb");
        auto data = db_.query("SELECT * FROM reports WHERE id='" + report_id + "'");
        db_.disconnect();
        return "Report: " + data;
    }
};

// Tests with trompeloeil
TEST_CASE("ReportGenerator queries database in order") {
    MockDatabase mock_db;
    ReportGenerator gen(mock_db);

    // Sequence enforcement
    trompeloeil::sequence seq;

    // Expectations use REQUIRE_CALL (Catch2-style)
    REQUIRE_CALL(mock_db, connect("postgres://localhost/mydb"))
        .IN_SEQUENCE(seq);

    REQUIRE_CALL(mock_db, query(trompeloeil::re(".*reports.*")))  // regex matcher
        .IN_SEQUENCE(seq)
        .RETURN("test_data");

    REQUIRE_CALL(mock_db, disconnect())
        .IN_SEQUENCE(seq);

    auto result = gen.generate("42");
    REQUIRE(result == "Report: test_data");
    // Expectations auto-verified when REQUIRE_CALL objects go out of scope
}

TEST_CASE("trompeloeil with ALLOW_CALL for optional calls") {
    MockDatabase mock_db;
    ReportGenerator gen(mock_db);

    // ALLOW_CALL: won't fail if not called (like ON_CALL in gMock)
    ALLOW_CALL(mock_db, connect(trompeloeil::_));
    ALLOW_CALL(mock_db, disconnect());

    // REQUIRE_CALL: MUST be called (like EXPECT_CALL in gMock)
    REQUIRE_CALL(mock_db, query(trompeloeil::_))
        .RETURN("data");

    gen.generate("99");
}

TEST_CASE("trompeloeil TIMES for call count") {
    MockDatabase mock_db;

    REQUIRE_CALL(mock_db, connect(trompeloeil::_))
        .TIMES(2);  // Must be called exactly twice

    mock_db.connect("url1");
    mock_db.connect("url2");
    // If called 1 or 3 times, Catch2 reports a clear failure message
}

// Failure output looks like:
// test.cpp:42: FAILED:
//   Unfulfilled expectation:
//   Expected disconnect() to be called 1 time, but it was called 0 times
//   Set up at test.cpp:38
```

Notice that when trompeloeil fails, the failure message includes both where the expectation was set and what went wrong - which makes diagnosing test failures much faster than a raw assertion failure with no context.

---

## Notes

- trompeloeil: `#include <trompeloeil.hpp>` - single header, no build step
- `MAKE_MOCKn` (n = arg count): `MAKE_MOCK2(foo, int(double, int), override)`
- `REQUIRE_CALL` = must be called (test fails if not) - like `EXPECT_CALL` in gMock
- `ALLOW_CALL` = may or may not be called - like `ON_CALL` in gMock
- `FORBID_CALL` = must NOT be called - like `EXPECT_CALL(...).Times(0)` in gMock
- trompeloeil matchers: `trompeloeil::_` (any), `trompeloeil::re("pattern")` (regex), `trompeloeil::gt(n)` (greater than)
