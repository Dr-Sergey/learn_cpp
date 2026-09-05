# Use test fixtures for shared setup and teardown in Google Test

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Test fixtures** in Google Test provide shared setup/teardown logic for a group of related tests. Each `TEST_F` creates a **fresh instance** of the fixture class, ensuring test isolation. Fixtures support four levels of setup/teardown granularity.

The key thing to understand is that fixture isolation is per-test, not per-suite. Every `TEST_F` gets its own brand-new fixture object with `SetUp` called fresh. This means you can have twenty tests that all start with the same three items in a vector, and none of them can accidentally affect the others - the vector is never shared.

### Fixture Lifecycle

| Level | Methods | When | Use Case |
| --- | --- | --- | --- |
| **Per-test** | `SetUp()` / `TearDown()` | Before/after each TEST_F | Default - isolate each test |
| **Per-suite** | `SetUpTestSuite()` / `TearDownTestSuite()` | Before first / after last test in suite | Expensive shared resources |
| **Global** | `::testing::Environment` | Before/after all suites | Database connections, process-level init |
| **Constructor/Destructor** | ctor / dtor | Same as SetUp/TearDown | RAII-managed resources |

### Execution Order

This diagram shows exactly when each hook fires. Read it top-to-bottom to understand when your setup and teardown code will run relative to the test body.

```cpp
Global SetUp
  └─ TestSuite::SetUpTestSuite()
       ├─ Fixture ctor -> SetUp() -> TEST_F body -> TearDown() -> Fixture dtor
       ├─ Fixture ctor -> SetUp() -> TEST_F body -> TearDown() -> Fixture dtor
       └─ ...
  └─ TestSuite::TearDownTestSuite()
Global TearDown
```

---

## Self-Assessment

### Q1: Implement test fixtures at different granularity levels

**Answer:**

The example below shows two fixture patterns side by side. `StackTest` uses per-test `SetUp` - the stack is rebuilt before every test, which is cheap and gives full isolation. `DatabaseTest` uses `SetUpTestSuite` - the "database" is constructed once and shared across all tests in the suite, because connecting to a database is expensive. Per-test `SetUp` then clears only the test-specific data while keeping the connection alive.

```cpp
#include <gtest/gtest.h>
#include <memory>
#include <vector>
#include <string>
#include <iostream>

// === Production code ===
class Database {
public:
    explicit Database(const std::string& connStr) : conn_(connStr) {
        std::cout << "DB connected: " << conn_ << "\n";
    }
    ~Database() { std::cout << "DB disconnected\n"; }

    void insert(const std::string& table, const std::string& data) {
        records_[table].push_back(data);
    }

    std::vector<std::string> query(const std::string& table) const {
        auto it = records_.find(table);
        if (it == records_.end()) return {};
        return it->second;
    }

    void clear_table(const std::string& table) {
        records_.erase(table);
    }

private:
    std::string conn_;
    std::map<std::string, std::vector<std::string>> records_;
};

// === Per-Test Fixture (most common) ===

class StackTest : public ::testing::Test {
protected:
    // SetUp() runs before EACH test - fresh stack every time
    void SetUp() override {
        stack.push_back(10);
        stack.push_back(20);
        stack.push_back(30);
    }

    // TearDown() is optional - useful for non-RAII cleanup
    void TearDown() override {
        // Verify no corruption (defensive)
        // In practice, RAII handles this automatically
    }

    std::vector<int> stack;  // Each test gets its own copy
};

TEST_F(StackTest, InitialSizeIsThree) {
    EXPECT_EQ(stack.size(), 3);
}

TEST_F(StackTest, PopRemovesLastElement) {
    stack.pop_back();
    EXPECT_EQ(stack.size(), 2);
    EXPECT_EQ(stack.back(), 20);
}

TEST_F(StackTest, PreviousTestDidNotAffectState) {
    // This passes because SetUp() creates a fresh stack
    EXPECT_EQ(stack.size(), 3);
    EXPECT_EQ(stack.back(), 30);
}


// === Per-Suite Fixture (shared expensive resource) ===

class DatabaseTest : public ::testing::Test {
protected:
    // Static setup - runs ONCE for the entire test suite
    static void SetUpTestSuite() {
        // Expensive: connect to database once
        db_ = std::make_unique<Database>("test://localhost:5432");
        db_->insert("seed", "base_data");
    }

    static void TearDownTestSuite() {
        db_.reset();  // Disconnect once
    }

    // Per-test setup - clean up test-specific data but keep connection
    void SetUp() override {
        db_->clear_table("test_data");
    }

    // Helper: access the shared database
    Database& db() { return *db_; }

private:
    static std::unique_ptr<Database> db_;  // Shared across tests
};

std::unique_ptr<Database> DatabaseTest::db_;  // Define static member

TEST_F(DatabaseTest, InsertAndQuery) {
    db().insert("test_data", "record_1");
    auto results = db().query("test_data");
    EXPECT_EQ(results.size(), 1);
}

TEST_F(DatabaseTest, SeedDataAvailable) {
    // Seed data from SetUpTestSuite persists
    auto seeds = db().query("seed");
    EXPECT_EQ(seeds.size(), 1);
}

TEST_F(DatabaseTest, TestDataIsolatedBetweenTests) {
    // Per-test SetUp() cleared test_data
    auto results = db().query("test_data");
    EXPECT_TRUE(results.empty());
}
```

Note that `db_` must be declared static inside the class and then defined outside. If you forget the out-of-class definition you will get a linker error - this is a common gotcha.

### Q2: When to use constructor/destructor vs SetUp/TearDown

**Answer:**

| Aspect | Constructor / Destructor | SetUp / TearDown |
| --- | --- | --- |
| **Exception safety** | Exception in ctor -> test not run, no leak | Exception in SetUp -> TearDown still called |
| **Virtual calls** | Can't call virtual methods in ctor | Can call virtual methods |
| **RAII resources** | Natural fit - manages lifetime | Must manually release in TearDown |
| **Fatal assertions** | Can't use `ASSERT_*` in constructor | Can use `ASSERT_*` (skips test on failure) |
| **Derived fixtures** | Base ctor runs first | Base SetUp runs first |

The practical rule is simple: use constructors when your resources are RAII types and you don't need assertions. Use `SetUp` when you need to call `ASSERT_*` to bail out early if setup fails - you cannot use `ASSERT_*` in a constructor because it just returns from the current function, and the constructor does not return a meaningful value.

```cpp
// Constructor approach - natural for RAII resources
class FileTest : public ::testing::Test {
protected:
    FileTest()
        : temp_dir_(create_temp_directory())  // RAII: cleaned up by dtor
        , file_(temp_dir_ / "test.txt")
    {
        // Can't use ASSERT_* here!
        std::ofstream(file_) << "test content";
    }

    // ~FileTest() automatically cleans up temp_dir_ via RAII

    std::filesystem::path temp_dir_;
    std::filesystem::path file_;
};

// SetUp approach - when you need ASSERT or virtual calls
class NetworkTest : public ::testing::Test {
protected:
    void SetUp() override {
        connection_ = connect_to_server();
        ASSERT_TRUE(connection_.is_open())  // Fatal: skip test if can't connect
            << "Server unavailable";
    }

    void TearDown() override {
        if (connection_.is_open())
            connection_.close();
    }

    Connection connection_;
};
```

**Rule of thumb:**

- Use **constructor/destructor** for RAII resources and simple initialization.
- Use **SetUp/TearDown** when you need `ASSERT_*` or virtual dispatch.

### Q3: Show fixture inheritance for layered test setup

**Answer:**

Fixture inheritance lets you build up setup in layers - a base fixture handles infrastructure common to all tests, mid-level fixtures add domain-specific state, and leaf fixtures set up the exact scenario for a particular test suite. The critical rule is that each `SetUp` override must call the base class `SetUp` first, otherwise the infrastructure the base sets up will not be ready.

```cpp
#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <memory>

// === Base fixture: common to all service tests ===
class ServiceTestBase : public ::testing::Test {
protected:
    void SetUp() override {
        // Create mock infrastructure
        log_messages_.clear();
    }

    void log(const std::string& msg) {
        log_messages_.push_back(msg);
    }

    void expect_logged(const std::string& msg) {
        EXPECT_NE(
            std::find(log_messages_.begin(), log_messages_.end(), msg),
            log_messages_.end()
        ) << "Expected log message: " << msg;
    }

    std::vector<std::string> log_messages_;
};

// === Mid-level fixture: adds database for data tests ===
class DataServiceTest : public ServiceTestBase {
protected:
    void SetUp() override {
        ServiceTestBase::SetUp();  // MUST call base SetUp!
        db_records_.clear();
    }

    void seed_data(const std::string& record) {
        db_records_.push_back(record);
    }

    std::vector<std::string> db_records_;
};

// === Leaf fixture: specific test suite ===
class UserServiceTest : public DataServiceTest {
protected:
    void SetUp() override {
        DataServiceTest::SetUp();  // Chain base setup
        seed_data("admin:admin@test.com");
        seed_data("user:user@test.com");
    }
};

TEST_F(UserServiceTest, HasSeededUsers) {
    EXPECT_EQ(db_records_.size(), 2);
}

TEST_F(UserServiceTest, LoggingWorks) {
    log("User created");
    expect_logged("User created");
}

// === Typed test fixtures (test same logic against multiple types) ===

template<typename T>
class ContainerTest : public ::testing::Test {
protected:
    T container;
};

using ContainerTypes = ::testing::Types<
    std::vector<int>,
    std::deque<int>,
    std::list<int>
>;

TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, EmptyOnConstruction) {
    EXPECT_TRUE(this->container.empty());
}

TYPED_TEST(ContainerTest, SizeIncreasesAfterInsert) {
    this->container.push_back(42);
    EXPECT_EQ(this->container.size(), 1);
}
```

`TYPED_TEST` is the right tool when you have a generic interface and want to verify that multiple concrete types satisfy its contract. Every test in the suite runs once for each type in the list, and the test name in the output tells you exactly which type failed.

---

## Notes

- Always call `Base::SetUp()` at the start of derived `SetUp()` - forgetting this is a common bug.
- `TEST_F` creates a new fixture instance per test - member variables are never shared.
- Static members in fixtures ARE shared - use `SetUpTestSuite` to initialize them.
- Prefer many small fixture classes over one giant fixture with many members.
- Use `TYPED_TEST` to run the same tests against multiple types (containers, allocators).
- `::testing::Environment` is for process-level setup (database server, temp directories).
- Keep fixture setup minimal - if tests don't all need the same setup, split into multiple fixtures.
