# Write test fixtures that use RAII for resource management

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rt-raii>  

---

## Topic Overview

**RAII test fixtures** acquire resources in the constructor and release them in the destructor. The guarantee is unconditional: even if the test throws an exception, the destructor still runs. With the traditional `setUp`/`tearDown` approach, if the test body throws, `tearDown` never executes and you leak resources.

```cpp
// Traditional:                    RAII:
//  setUp() { open(db); }          TestFixture() { open(db); }
//  test()  { use(db); throw! }    ~TestFixture() { close(db); }
//  tearDown() { close(db); }      // destructor ALWAYS runs
//  // tearDown SKIPPED on throw!  // cleanup guaranteed!
```

The reason this matters more in tests than in production code is that tests are expected to exercise failure paths. If your fixture leaks a temp file or an open connection on failure, you end up with broken state that corrupts subsequent tests.

---

## Self-Assessment

### Q1: Replace manual setUp/tearDown with RAII constructor/destructor

Here you can see both approaches side by side. The critical flaw in `BadTestFixture` is that if `test()` throws, `tearDown()` is never called. The `GoodTestFixture` doesn't have this problem - cleanup is in the destructor, which C++ always calls when the object goes out of scope.

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <cassert>
#include <filesystem>

// ============ BAD: manual setUp/tearDown ============
class BadTestFixture {
    std::ofstream file_;
    std::string path_;
public:
    void setUp() {
        path_ = "test_output.txt";
        file_.open(path_);
        std::cout << "[setUp] File opened\n";
    }

    void test() {
        file_ << "test data";
        // If this throws, tearDown is NEVER called!
        assert(true);
    }

    void tearDown() {
        file_.close();
        std::filesystem::remove(path_);
        std::cout << "[tearDown] File cleaned up\n";
    }
};

// ============ GOOD: RAII fixture ============
class GoodTestFixture {
    std::ofstream file_;
    std::string path_;
public:
    GoodTestFixture() : path_("test_output.txt") {
        file_.open(path_);
        std::cout << "[ctor] File opened\n";
    }

    ~GoodTestFixture() {
        file_.close();
        std::filesystem::remove(path_);
        std::cout << "[dtor] File cleaned up\n";
    }

    void test() {
        file_ << "test data";
        file_.flush();
        assert(std::filesystem::file_size(path_) > 0);
        std::cout << "[test] Assertions passed\n";
    }
};

int main() {
    {
        GoodTestFixture fixture;  // constructor = setUp
        fixture.test();
    }  // destructor = tearDown (ALWAYS runs)

    std::cout << "File exists: "
              << std::filesystem::exists("test_output.txt") << '\n';
}
// Expected output:
// [ctor] File opened
// [test] Assertions passed
// [dtor] File cleaned up
// File exists: 0
```

The braces around `fixture` are deliberate - they create a scope so the destructor runs before the `file_exists` check, demonstrating that cleanup happened.

### Q2: RAII fixtures guarantee cleanup even when tests throw

This is the scenario the traditional approach fails at. The fixture acquires two database connections in the constructor. When `run_test()` throws, stack unwinding destroys the fixture and the destructor cleans up both connections - in reverse order of construction, as always with C++ destructors.

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

// RAII resource: simulates a database connection
class TestDatabase {
    std::string name_;
public:
    explicit TestDatabase(const std::string& name) : name_(name) {
        std::cout << "[" << name_ << "] DB connected\n";
    }
    ~TestDatabase() {
        std::cout << "[" << name_ << "] DB disconnected (cleanup!)\n";
    }
    void query(const std::string& sql) {
        std::cout << "[" << name_ << "] Query: " << sql << '\n';
    }
};

// RAII fixture with multiple resources
class IntegrationTestFixture {
    TestDatabase db_;
    TestDatabase cache_;
public:
    IntegrationTestFixture()
        : db_("main_db"), cache_("cache_db") {}
    // Destructor automatically cleans up both!

    void run_test() {
        db_.query("INSERT INTO users VALUES ('alice')");
        cache_.query("SET user:alice:active true");

        // Simulate test failure:
        throw std::runtime_error("assertion failed!");
        // Both db_ and cache_ are STILL cleaned up!
    }
};

int main() {
    try {
        IntegrationTestFixture fixture;
        fixture.run_test();
    } catch (const std::exception& e) {
        std::cout << "Test failed: " << e.what() << '\n';
    }
    std::cout << "Resources cleaned up despite exception\n";
}
// Expected output:
// [main_db] DB connected
// [cache_db] DB connected
// [main_db] Query: INSERT INTO users VALUES ('alice')
// [cache_db] Query: SET user:alice:active true
// [cache_db] DB disconnected (cleanup!)
// [main_db] DB disconnected (cleanup!)
// Test failed: assertion failed!
// Resources cleaned up despite exception
```

Notice that `cache_db` disconnects before `main_db` - that's reverse-construction order, which is the standard C++ rule. Also note that the disconnection messages appear *before* the "Test failed" output, confirming that cleanup runs during stack unwinding, not after catching the exception.

### Q3: Scope exit guard for integration test cleanup

Sometimes you don't want a full fixture class - you just need a guaranteed cleanup for something ad-hoc. A `ScopeExit` guard captures a cleanup lambda and runs it in the destructor, giving you RAII without writing a whole class. This is especially handy for one-off test helpers.

```cpp
#include <iostream>
#include <fstream>
#include <filesystem>
#include <functional>
#include <string>

// Simple scope_exit guard (similar to GSL final_action)
class ScopeExit {
    std::function<void()> action_;
public:
    explicit ScopeExit(std::function<void()> action)
        : action_(std::move(action)) {}
    ~ScopeExit() { if (action_) action_(); }
    ScopeExit(const ScopeExit&) = delete;
    ScopeExit& operator=(const ScopeExit&) = delete;
};

void integration_test() {
    // Create temp file
    std::string path = "integration_test_data.json";
    std::ofstream(path) << R"({"test": true})";

    // Guarantee cleanup no matter what happens
    ScopeExit cleanup([&path] {
        std::filesystem::remove(path);
        std::cout << "Cleaned up: " << path << '\n';
    });

    // Test logic
    auto size = std::filesystem::file_size(path);
    std::cout << "File size: " << size << " bytes\n";

    // Even if we throw here, cleanup runs!
    // throw std::runtime_error("test failed");

    std::cout << "Test passed\n";
}  // cleanup runs here

int main() {
    integration_test();
    std::cout << "File exists: "
              << std::filesystem::exists("integration_test_data.json") << '\n';
}
// Expected output:
// File size: 14 bytes
// Test passed
// Cleaned up: integration_test_data.json
// File exists: 0
```

The copy operations are explicitly deleted because a `ScopeExit` that runs its action twice (once per copy) would be a nasty bug. C++26 may standardize `std::scope_exit` with similar semantics.

---

## Notes

- RAII fixtures work with any test framework: Google Test, Catch2, doctest.
- Google Test uses `TEST_F` which creates the fixture on the stack (RAII-compatible).
- Stack unwinding during exceptions destroys local objects in reverse order.
- Use `ScopeExit`/`scope_guard` for ad-hoc cleanup (C++26 may standardize `std::scope_exit`).
- Never do important cleanup in a `tearDown()` method - put it in the destructor.
