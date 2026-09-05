# Master Google Mock - matchers, actions, and expectations

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Google Mock (gmock)** lets you create mock objects that verify how your code interacts with its collaborators. Unlike a simple stub that just returns a fixed value, a mock also checks that the right methods were called with the right arguments, the right number of times, in the right order.

The framework is built around three concepts. **Matchers** constrain what argument values are acceptable. **Actions** define what the mock does when a matching call happens (return a value, throw, call a lambda). **Expectations** declare how many times a call is expected to occur and in what sequence. You combine all three in a single `EXPECT_CALL` statement.

### Core Concepts

| Concept | Purpose | Example |
| --- | --- | --- |
| **MOCK_METHOD** | Declare a mockable virtual method | `MOCK_METHOD(int, get, (int id), (const, override))` |
| **EXPECT_CALL** | Set expectation on a mock method | `EXPECT_CALL(mock, get(42))` |
| **Matcher** | Constrain expected arguments | `Eq(42)`, `_` (any), `HasSubstr("hello")` |
| **Action** | Define return value or side effect | `Return(5)`, `Throw(...)`, `Invoke(fn)` |
| **Cardinality** | How many calls expected | `Times(1)`, `Times(AtLeast(2))` |

### EXPECT_CALL Syntax

Here is the full shape of an `EXPECT_CALL` statement with all optional clauses. In practice you rarely use all of them at once - the most common form is just `EXPECT_CALL(mock, method(matcher)).WillOnce(Return(value))`.

```cpp
EXPECT_CALL(mock_object, method_name(matchers))
    .Times(cardinality)
    .WillOnce(action)
    .WillRepeatedly(action)
    .InSequence(seq)
    .RetiresOnSaturation();
```

---

## Self-Assessment

### Q1: Create mock objects and set basic expectations

**Answer:**

The workflow is always the same: define the interface, define the mock class using `MOCK_METHOD` macros, write the code under test against the interface, then in tests create mock instances and set `EXPECT_CALL` expectations before handing the mock to the code under test.

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include <string>
#include <memory>
#include <vector>

using ::testing::_;
using ::testing::Return;
using ::testing::Throw;
using ::testing::AtLeast;
using ::testing::Exactly;
using ::testing::HasSubstr;
using ::testing::Gt;
using ::testing::InSequence;

// === Interface to mock ===
class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual bool connect(const std::string& connStr) = 0;
    virtual std::string query(const std::string& sql) = 0;
    virtual int execute(const std::string& sql) = 0;
    virtual void disconnect() = 0;
};

// === Mock class ===
class MockDatabase : public IDatabase {
public:
    MOCK_METHOD(bool, connect, (const std::string& connStr), (override));
    MOCK_METHOD(std::string, query, (const std::string& sql), (override));
    MOCK_METHOD(int, execute, (const std::string& sql), (override));
    MOCK_METHOD(void, disconnect, (), (override));
};

// === Production code under test ===
class UserRepository {
public:
    explicit UserRepository(IDatabase& db) : db_(db) {}

    bool initialize() {
        return db_.connect("prod://db");
    }

    std::string find_user(int id) {
        return db_.query("SELECT * FROM users WHERE id=" + std::to_string(id));
    }

    int delete_user(int id) {
        return db_.execute("DELETE FROM users WHERE id=" + std::to_string(id));
    }

    void shutdown() {
        db_.disconnect();
    }

private:
    IDatabase& db_;
};

// === Tests ===

TEST(UserRepositoryTest, InitializeConnectsToDatabase) {
    MockDatabase mock_db;

    // Expect connect to be called exactly once with the right string
    EXPECT_CALL(mock_db, connect("prod://db"))
        .Times(1)
        .WillOnce(Return(true));

    UserRepository repo(mock_db);
    EXPECT_TRUE(repo.initialize());
}

TEST(UserRepositoryTest, FindUserExecutesCorrectQuery) {
    MockDatabase mock_db;

    // Match any string containing "WHERE id=42"
    EXPECT_CALL(mock_db, query(HasSubstr("WHERE id=42")))
        .WillOnce(Return("{\"name\": \"Alice\"}"));

    UserRepository repo(mock_db);
    auto result = repo.find_user(42);
    EXPECT_THAT(result, HasSubstr("Alice"));
}

TEST(UserRepositoryTest, DeleteUserReturnsAffectedRows) {
    MockDatabase mock_db;

    EXPECT_CALL(mock_db, execute(_))  // Accept any SQL
        .WillOnce(Return(1));  // 1 row affected

    UserRepository repo(mock_db);
    EXPECT_EQ(repo.delete_user(42), 1);
}

TEST(UserRepositoryTest, ConnectionFailureReturnsFalse) {
    MockDatabase mock_db;

    EXPECT_CALL(mock_db, connect(_))
        .WillOnce(Return(false));

    UserRepository repo(mock_db);
    EXPECT_FALSE(repo.initialize());
}
```

Notice how `HasSubstr` lets you verify that the SQL contains the right user ID without hard-coding the exact query string. This makes the test more resilient to irrelevant formatting changes while still catching the meaningful regression.

Expectations are checked when the mock object is destroyed - not when `EXPECT_CALL` is written, not when the code runs, but at destruction time. Make sure the mock object outlives the code under test or the verification will never happen.

### Q2: Use advanced matchers, actions, and sequence verification

**Answer:**

Real tests often need more than `Return(value)`. You might need to save an argument for later inspection, call a lambda based on what was passed in, or verify that a sequence of calls happens in a specific order. Here are the tools for all three.

```cpp
using ::testing::Invoke;
using ::testing::DoAll;
using ::testing::SetArgReferee;
using ::testing::SaveArg;
using ::testing::AllOf;
using ::testing::AnyOf;
using ::testing::Not;
using ::testing::StartsWith;
using ::testing::MatchesRegex;
using ::testing::NiceMock;
using ::testing::StrictMock;

// === Advanced Matchers ===
TEST(MatcherExamples, CompositeMatchers) {
    MockDatabase mock;

    // Combine matchers with AllOf, AnyOf, Not
    EXPECT_CALL(mock, query(AllOf(
            StartsWith("SELECT"),
            HasSubstr("users"),
            Not(HasSubstr("DROP"))  // Safety: no SQL injection
        )))
        .WillOnce(Return("result"));

    // This matches:
    mock.query("SELECT * FROM users WHERE id=1");
}

// === Advanced Actions ===
TEST(ActionExamples, InvokeRealFunction) {
    MockDatabase mock;
    std::string captured_sql;

    // SaveArg: capture argument for later inspection
    EXPECT_CALL(mock, execute(_))
        .WillOnce(DoAll(
            SaveArg<0>(&captured_sql),  // Save first arg
            Return(1)                   // And return 1
        ));

    UserRepository repo(mock);
    repo.delete_user(99);

    EXPECT_THAT(captured_sql, HasSubstr("99"));
}

TEST(ActionExamples, InvokeCustomLambda) {
    MockDatabase mock;

    // Invoke: call a real function/lambda instead
    EXPECT_CALL(mock, query(_))
        .WillRepeatedly(Invoke([](const std::string& sql) -> std::string {
            if (sql.find("id=1") != std::string::npos)
                return "Alice";
            return "not found";
        }));

    EXPECT_EQ(mock.query("SELECT WHERE id=1"), "Alice");
    EXPECT_EQ(mock.query("SELECT WHERE id=999"), "not found");
}

// === Sequence Verification ===
TEST(SequenceTest, MethodsCalledInOrder) {
    MockDatabase mock;

    InSequence seq;  // All EXPECT_CALLs below must happen in order

    EXPECT_CALL(mock, connect(_)).WillOnce(Return(true));
    EXPECT_CALL(mock, query(_)).WillOnce(Return("data"));
    EXPECT_CALL(mock, disconnect());

    UserRepository repo(mock);
    repo.initialize();
    repo.find_user(1);
    repo.shutdown();
}

// === Nice, Strict, and Naggy Mocks ===
TEST(MockTypes, NiceMockIgnoresUnexpectedCalls) {
    // NiceMock: doesn't warn about uninteresting calls
    NiceMock<MockDatabase> nice_mock;
    EXPECT_CALL(nice_mock, connect(_)).WillOnce(Return(true));

    UserRepository repo(nice_mock);
    repo.initialize();
    // repo.find_user(1);  // Would not cause failure even without EXPECT_CALL
}

TEST(MockTypes, StrictMockFailsOnUnexpectedCalls) {
    // StrictMock: ANY unregistered call is a test failure
    StrictMock<MockDatabase> strict_mock;
    EXPECT_CALL(strict_mock, connect(_)).WillOnce(Return(true));

    UserRepository repo(strict_mock);
    repo.initialize();
    // repo.find_user(1);  // Would FAIL - no EXPECT_CALL for query()
}
```

The reason `NiceMock` vs `StrictMock` matters is that the default behavior (called "Naggy") prints a warning for uninteresting calls but does not fail. For focused unit tests, `StrictMock` makes the test fail immediately if any unexpected call slips through. For high-level tests where you only care about one specific interaction, `NiceMock` reduces noise.

### Q3: Show mocking patterns for complex real-world scenarios

**Answer:**

Here are two practical patterns you will reach for often: using `ON_CALL` in `SetUp` to establish sensible defaults for a whole test fixture, and wrapping free functions in an interface so they become mockable.

```cpp
// === Pattern 1: Mock with ON_CALL defaults + specific overrides ===

class MockFileSystem : public IFileSystem {
public:
    MOCK_METHOD(bool, exists, (const std::string& path), (const, override));
    MOCK_METHOD(std::string, read, (const std::string& path), (const, override));
    MOCK_METHOD(bool, write, (const std::string& path, const std::string& data), (override));
};

class ConfigServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Set defaults with ON_CALL - applied unless overridden by EXPECT_CALL
        ON_CALL(fs_, exists(_)).WillByDefault(Return(false));
        ON_CALL(fs_, read(_)).WillByDefault(Return("{}"));
        ON_CALL(fs_, write(_, _)).WillByDefault(Return(true));
    }

    MockFileSystem fs_;
};

TEST_F(ConfigServiceTest, LoadsDefaultWhenConfigMissing) {
    // fs_.exists() returns false by default (set in SetUp)
    // Only need to verify the behavior we care about
    ConfigService svc(fs_);
    EXPECT_EQ(svc.get("key"), "default_value");
}

TEST_F(ConfigServiceTest, LoadsConfigWhenFileExists) {
    // Override default for this specific test
    EXPECT_CALL(fs_, exists("/etc/config.json")).WillOnce(Return(true));
    EXPECT_CALL(fs_, read("/etc/config.json"))
        .WillOnce(Return(R"({"key": "custom_value"})"));

    ConfigService svc(fs_);
    EXPECT_EQ(svc.get("key"), "custom_value");
}


// === Pattern 2: Mocking free functions via wrapper ===

// Can't mock free functions directly. Wrap them:
class ITimeProvider {
public:
    virtual ~ITimeProvider() = default;
    virtual int64_t now_ms() = 0;
};

class SystemTime : public ITimeProvider {
public:
    int64_t now_ms() override {
        return std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now().time_since_epoch()
        ).count();
    }
};

class MockTime : public ITimeProvider {
public:
    MOCK_METHOD(int64_t, now_ms, (), (override));
};

// Now you can control time in tests:
TEST(RateLimiterTest, AllowsRequestWithinLimit) {
    MockTime time;
    EXPECT_CALL(time, now_ms())
        .WillOnce(Return(1000))     // First call
        .WillOnce(Return(2000));    // Second call: 1 second later

    RateLimiter limiter(time, /*max_per_second=*/5);
    EXPECT_TRUE(limiter.allow());
    EXPECT_TRUE(limiter.allow());
}
```

The `ON_CALL` / `EXPECT_CALL` distinction is subtle but important. `ON_CALL` sets a behavior without registering an expectation - calls may happen zero or more times and the test does not care. `EXPECT_CALL` sets a behavior *and* registers an expectation that the mock will verify at destruction time. Use `ON_CALL` in `SetUp` for defaults, `EXPECT_CALL` in individual tests for the interactions that actually matter for that test's scenario.

---

## Notes

- `MOCK_METHOD` is the modern syntax (GMock 1.10+). The old `MOCK_METHODn` family of macros is deprecated and should not appear in new code.
- The default mock type is "Naggy" - it warns about uninteresting calls but does not fail. Use `NiceMock` to suppress those warnings or `StrictMock` to make them failures.
- `ON_CALL` sets default behaviors; `EXPECT_CALL` sets expectations. Keep `ON_CALL` in `SetUp()` for fixture-wide defaults, and `EXPECT_CALL` in individual tests for what matters in that scenario.
- Expectations are verified in the mock destructor - make sure mocks outlive the code under test so verification actually runs.
- `EXPECT_CALL` expectations are matched in reverse order - the last matching expectation wins. This is useful for setting a default and then overriding it for specific argument values, but can be surprising if you forget it.
- Mock only at boundaries (interfaces) - do not mock everything in sight. If you find yourself needing many mocks for a single unit test, the code under test probably has too many responsibilities.
- If you find yourself writing mocks for things that are simple to just instantiate directly, a hand-written fake struct might be cleaner than a full Google Mock class.
