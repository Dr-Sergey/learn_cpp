# Design testable C++ code with dependency injection and interfaces

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Writing testable C++ code begins at **design time**, not after implementation. The core principle is to separate *what* your code does from *how* it interacts with external systems. This means coding against abstract interfaces and injecting concrete implementations, which allows test doubles to replace real I/O, network, database, and hardware in your unit tests.

The reason this matters is simple: code that creates its own dependencies internally cannot be tested in isolation. If your `ReportGenerator` calls `std::time()` directly and opens a real file, your test has to control the clock and the filesystem - both of which are painful to arrange and slow to run. If instead you inject an `IClock` and an `IFileWriter`, your test can hand in lightweight fakes and run in microseconds.

### Testability Smells vs Fixes

If you spot any of these patterns in existing code, they are a sign that the code will be hard to test:

| Untestable Pattern | Testable Alternative |
| --- | --- |
| `new` inside business logic | Constructor-inject a factory or ready-made object |
| Singleton for shared state | Inject the shared resource |
| Calling `std::time()` directly | Inject `IClock` interface |
| Direct file I/O in logic | Inject `IFileSystem` interface |
| Hard-coded URLs/paths | Inject configuration object |
| `static` functions with side effects | Wrap in mockable interface |

### Layers of Testable Architecture

The architecture that makes this all work has three layers. Business logic sits in the middle and depends only on abstract interfaces - it never touches infrastructure directly. Infrastructure adapters (the real implementations) live at the bottom and are tested with integration tests. The composition root at the top is the only place where everything is wired together.

```cpp
┌────────────────────────────────────────┐
│            Application / main()        │  <- Composition root: wires deps
├────────────────────────────────────────┤
│         Business Logic Layer           │  <- Pure logic, no I/O
│   (depends on abstract interfaces)     │  <- Easy to unit test
├────────────────────────────────────────┤
│     Infrastructure Adapters            │  <- Real implementations
│  (SqlRepo, HttpClient, FileSystem)     │  <- Integration tested
└────────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Refactor untestable code to be dependency-injectable

**Answer:**

The before/after below shows the transformation in concrete terms. The "before" version is completely impossible to test without a real database and a real clock. The "after" version accepts all three dependencies through its constructor, making each one replaceable.

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include <string>
#include <chrono>
#include <memory>

using ::testing::Return;
using ::testing::_;

// === BEFORE: Untestable - hard-coded dependencies ===
/*
class ReportGenerator {
public:
    std::string generate() {
        // Hard-coded: can't test without real DB and clock
        auto now = std::chrono::system_clock::now();
        auto data = SqlDatabase("prod://db").query("SELECT * FROM sales");
        std::ofstream file("/reports/" + format_date(now) + ".csv");
        file << format_csv(data);
        return file_path;
    }
};
*/

// === AFTER: Testable - all dependencies injected ===

// Interface for time
class IClock {
public:
    virtual ~IClock() = default;
    virtual std::string now_formatted() const = 0;
};

// Interface for data source
class IDataSource {
public:
    virtual ~IDataSource() = default;
    virtual std::vector<std::string> fetch_sales() = 0;
};

// Interface for file output
class IFileWriter {
public:
    virtual ~IFileWriter() = default;
    virtual bool write(const std::string& path, const std::string& content) = 0;
};

// Business logic: depends only on interfaces
class ReportGenerator {
public:
    ReportGenerator(IClock& clock, IDataSource& data, IFileWriter& writer)
        : clock_(clock), data_(data), writer_(writer) {}

    std::string generate() {
        auto timestamp = clock_.now_formatted();
        auto sales = data_.fetch_sales();

        std::string csv;
        for (const auto& row : sales)
            csv += row + "\n";

        std::string path = "/reports/" + timestamp + ".csv";
        if (!writer_.write(path, csv))
            throw std::runtime_error("Failed to write report");

        return path;
    }

private:
    IClock& clock_;
    IDataSource& data_;
    IFileWriter& writer_;
};

// === Mocks ===
class MockClock : public IClock {
public:
    MOCK_METHOD(std::string, now_formatted, (), (const, override));
};

class MockDataSource : public IDataSource {
public:
    MOCK_METHOD(std::vector<std::string>, fetch_sales, (), (override));
};

class MockFileWriter : public IFileWriter {
public:
    MOCK_METHOD(bool, write, (const std::string&, const std::string&), (override));
};

// === Tests: fast, deterministic, no I/O ===

TEST(ReportGeneratorTest, GeneratesCorrectPath) {
    MockClock clock;
    MockDataSource data;
    MockFileWriter writer;

    EXPECT_CALL(clock, now_formatted()).WillOnce(Return("2024-01-15"));
    EXPECT_CALL(data, fetch_sales()).WillOnce(Return(
        std::vector<std::string>{"widget,100", "gadget,200"}));
    EXPECT_CALL(writer, write("/reports/2024-01-15.csv", _))
        .WillOnce(Return(true));

    ReportGenerator gen(clock, data, writer);
    EXPECT_EQ(gen.generate(), "/reports/2024-01-15.csv");
}

TEST(ReportGeneratorTest, ThrowsOnWriteFailure) {
    MockClock clock;
    MockDataSource data;
    MockFileWriter writer;

    EXPECT_CALL(clock, now_formatted()).WillOnce(Return("2024-01-15"));
    EXPECT_CALL(data, fetch_sales()).WillOnce(Return(std::vector<std::string>{}));
    EXPECT_CALL(writer, write(_, _)).WillOnce(Return(false));

    ReportGenerator gen(clock, data, writer);
    EXPECT_THROW(gen.generate(), std::runtime_error);
}
```

These tests run with zero file I/O and a deterministic clock. The `ThrowsOnWriteFailure` test is especially hard to write without DI - how would you make a real file write fail on demand? With the injected `IFileWriter`, it is a one-liner.

### Q2: Compare testing with virtual interfaces vs template-based injection

**Answer:**

Virtual interfaces are not the only way to inject dependencies. Template-based injection uses compile-time polymorphism instead: the type of the dependency is a template parameter, and the test just passes a struct with the right methods - no vtable needed.

```cpp
// === Template-based DI: zero-cost, compile-time polymorphism ===

template<typename Clock, typename DataSource, typename FileWriter>
class ReportGeneratorT {
public:
    ReportGeneratorT(Clock& clock, DataSource& data, FileWriter& writer)
        : clock_(clock), data_(data), writer_(writer) {}

    std::string generate() {
        auto timestamp = clock_.now_formatted();
        auto sales = data_.fetch_sales();
        std::string csv;
        for (const auto& row : sales) csv += row + "\n";
        std::string path = "/reports/" + timestamp + ".csv";
        writer_.write(path, csv);
        return path;
    }

private:
    Clock& clock_;
    DataSource& data_;
    FileWriter& writer_;
};

// Test double: no vtable overhead
struct FakeClock {
    std::string now_formatted() const { return "2024-01-15"; }
};

struct FakeDataSource {
    std::vector<std::string> fetch_sales() { return {"a,1", "b,2"}; }
};

struct FakeFileWriter {
    std::string last_path;
    std::string last_content;
    bool write(const std::string& path, const std::string& content) {
        last_path = path;
        last_content = content;
        return true;
    }
};

TEST(TemplateDI, WritesCorrectContent) {
    FakeClock clock;
    FakeDataSource data;
    FakeFileWriter writer;

    ReportGeneratorT gen(clock, data, writer);
    gen.generate();

    EXPECT_EQ(writer.last_path, "/reports/2024-01-15.csv");
    EXPECT_EQ(writer.last_content, "a,1\nb,2\n");
}
```

The trade-off between the two approaches comes down to what you value more. Here is a quick comparison to help decide:

| Approach | Overhead | Mockability | ABI Stable | Readability |
| --- | --- | --- | --- | --- |
| Virtual interface | vtable (1-3ns/call) | Google Mock | Yes | Familiar OOP |
| Template injection | Zero (inlined) | Manual fakes | No (header-only) | More verbose |
| `std::function` | Possible alloc | Lambda/functor | Yes | Best for single ops |

For most production code, virtual interfaces are fine - the vtable overhead is negligible compared to any real I/O. Template injection is worth considering for hot loops or when you want the fakes to be simple structs that live right next to the test code.

### Q3: Design a complete composition root with production and test wiring

**Answer:**

Here is the full picture: production implementations, the composition root that wires them together, and a reminder of how the same business logic class is used in tests with mocks.

```cpp
#include <memory>
#include <iostream>
#include <string>
#include <vector>

// === Production implementations ===
class SystemClock : public IClock {
public:
    std::string now_formatted() const override {
        // Real implementation using chrono
        return "2024-06-15";  // Simplified
    }
};

class SqlDataSource : public IDataSource {
public:
    explicit SqlDataSource(const std::string& connStr) : conn_(connStr) {}
    std::vector<std::string> fetch_sales() override {
        // Real SQL query
        return {"real,data"};
    }
private:
    std::string conn_;
};

class DiskFileWriter : public IFileWriter {
public:
    bool write(const std::string& path, const std::string& content) override {
        // Real file write
        std::cout << "Writing to " << path << "\n";
        return true;
    }
};

// === Composition Root: wires everything together ===

// Production composition (in main.cpp)
int main() {
    // Create infrastructure
    SystemClock clock;
    SqlDataSource data("prod://salesdb");
    DiskFileWriter writer;

    // Wire into business logic
    ReportGenerator generator(clock, data, writer);

    // Run
    auto path = generator.generate();
    std::cout << "Report generated: " << path << "\n";
}

// Test composition (in test file) - already shown above:
// MockClock, MockDataSource, MockFileWriter -> ReportGenerator
// Fast, deterministic, no external dependencies
```

Notice that `main()` is the only place where `#include "sql_data_source.h"` or `#include "disk_file_writer.h"` ever appears. Test files only ever include the interfaces - they know nothing about the real implementations. That is the boundary that makes the architecture work.

---

## Notes

- The **Composition Root** is the single place where all dependencies are wired together - this is typically `main()` for applications or a factory/service locator for libraries.
- Never `#include` production implementations in test files - only interfaces. Test files should be completely unaware of which concrete class they are indirectly testing.
- Store injected dependencies as references (caller owns the lifetime) or `unique_ptr` (the class owns the lifetime). Avoid raw pointers.
- For legacy code, the best entry point is wrapping the hardest-to-test dependency first - that is usually the I/O or external service call.
- The [**Dependency Inversion Principle**](../29_OOP_Design/Apply_the_Dependency_Inversion_Principle_with_abstract_interfaces_and_dependency.md) is the theoretical foundation behind DI.
- The cost of virtual dispatch is negligible outside hot loops - do not prematurely optimize away interfaces because of perceived vtable overhead.
- Consider C++20 concepts as an alternative to virtual interfaces for zero-cost DI with better error messages at compile time.
