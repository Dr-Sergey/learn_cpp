# Understand interface segregation: prefer small focused interfaces

**Category:** Modern OOP Patterns  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ri-segregate>  

---

## Topic Overview

The **Interface Segregation Principle (ISP)** - the "I" in SOLID - states: *No client should be forced to depend on methods it does not use.* In C++, this means prefer small, focused abstract interfaces over large "god" interfaces.

The practical consequence is simple: if you have a class that only reads from a database, it shouldn't need to implement or even know about `backup()` and `migrate()`. When interfaces are fat, every class that implements them drags in responsibilities it doesn't own, and every mock in your tests has to stub out methods it doesn't care about.

### Fat Interface vs Segregated Interfaces

Here's the structural difference. On the left, one big interface forces every implementor to handle everything. On the right, three small interfaces let each client declare exactly what it needs.

```cpp
// BAD: FAT interface:                // GOOD: SEGREGATED:
┌──────────────────────┐            ┌──────────────┐  ┌──────────────┐
│     IDatabase         │            │   IReader     │  │   IWriter     │
│  + read()             │            │  + read()     │  │  + write()    │
│  + write()            │            │  + query()    │  │  + update()   │
│  + query()            │            └──────────────┘  │  + delete_()   │
│  + update()           │                              └──────────────┘
│  + delete_()          │
│  + backup()           │            ┌──────────────┐
│  + migrate()          │            │   IAdmin      │
│  + getStats()         │            │  + backup()   │
└──────────────────────┘            │  + migrate()  │
                                     │  + getStats() │
Client that only reads must          └──────────────┘
implement backup(), migrate()...
-> Violates ISP!                     Each client implements ONLY what it needs.
```

---

## Self-Assessment

### Q1: Split a large IDatabase interface into IReader and IWriter and show how components compose them

The key payoff here is that `generate_report` takes a `const IReader&` - it literally cannot call `write`, `remove`, or `backup` by accident. The interface itself enforces the constraint. Similarly, `import_data` takes `IWriter&` and cannot accidentally read.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

// BAD: Fat interface forces all implementations to handle everything
class IDatabase_Fat {
public:
    virtual ~IDatabase_Fat() = default;
    virtual std::string read(int id) = 0;
    virtual void write(int id, const std::string& data) = 0;
    virtual void backup(const std::string& path) = 0;
    virtual void migrate(int version) = 0;
};

// GOOD: Segregated interfaces
class IReader {
public:
    virtual ~IReader() = default;
    virtual std::string read(int id) const = 0;
    virtual std::vector<std::string> query(const std::string& filter) const = 0;
};

class IWriter {
public:
    virtual ~IWriter() = default;
    virtual void write(int id, const std::string& data) = 0;
    virtual void remove(int id) = 0;
};

class IAdmin {
public:
    virtual ~IAdmin() = default;
    virtual void backup(const std::string& path) = 0;
    virtual size_t record_count() const = 0;
};

// Full database implements ALL interfaces through multiple inheritance
class SqlDatabase : public IReader, public IWriter, public IAdmin {
    std::vector<std::string> data_{10, ""};  // simple storage
public:
    std::string read(int id) const override {
        return (id < 10) ? data_[id] : "";
    }
    std::vector<std::string> query(const std::string& filter) const override {
        std::vector<std::string> results;
        for (const auto& d : data_)
            if (!d.empty() && d.find(filter) != std::string::npos)
                results.push_back(d);
        return results;
    }
    void write(int id, const std::string& data) override {
        if (id < 10) data_[id] = data;
    }
    void remove(int id) override {
        if (id < 10) data_[id].clear();
    }
    void backup(const std::string& path) override {
        std::cout << "  Backing up to " << path << "\n";
    }
    size_t record_count() const override {
        size_t count = 0;
        for (const auto& d : data_) if (!d.empty()) ++count;
        return count;
    }
};

// Client functions only depend on what they NEED:
void generate_report(const IReader& db) {
    // Only needs read access - can't accidentally write!
    auto results = db.query("user");
    std::cout << "  Report: found " << results.size() << " user records\n";
}

void import_data(IWriter& db) {
    // Only needs write access - can't read or admin!
    db.write(0, "user_alice");
    db.write(1, "user_bob");
    std::cout << "  Imported 2 records\n";
}

void admin_task(IAdmin& db) {
    std::cout << "  Records: " << db.record_count() << "\n";
    db.backup("/backup/daily.sql");
}

int main() {
    SqlDatabase db;

    import_data(db);         // IWriter& only
    generate_report(db);     // IReader& only
    admin_task(db);          // IAdmin& only
}
// Expected output:
//   Imported 2 records
//   Report: found 2 user records
//   Records: 2
//   Backing up to /backup/daily.sql
```

`SqlDatabase` implements all three interfaces through multiple inheritance, which is perfectly idiomatic in C++. The caller never knows or cares - it just asks for what it needs.

---

### Q2: Show that a mock in a test only needs to implement the small relevant interface

This is where the benefit really shows up. When `UserService` depends on `IReader`, the test mock only needs two methods. With a fat interface, you'd have to stub out eight methods just to test one function.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <cassert>

// Same segregated interfaces from Q1
class IReader {
public:
    virtual ~IReader() = default;
    virtual std::string read(int id) const = 0;
    virtual std::vector<std::string> query(const std::string& filter) const = 0;
};

class IWriter {
public:
    virtual ~IWriter() = default;
    virtual void write(int id, const std::string& data) = 0;
    virtual void remove(int id) = 0;
};

// Production code under test
class UserService {
    const IReader& reader_;  // only depends on IReader
public:
    explicit UserService(const IReader& r) : reader_(r) {}

    bool user_exists(int id) const {
        return !reader_.read(id).empty();
    }
};

// GOOD: Mock only implements IReader - NOT IWriter, IAdmin!
class MockReader : public IReader {
    std::vector<std::string> data_;
public:
    MockReader(std::initializer_list<std::string> data) : data_(data) {}

    std::string read(int id) const override {
        return (id >= 0 && id < static_cast<int>(data_.size()))
            ? data_[id] : "";
    }

    std::vector<std::string> query(const std::string&) const override {
        return data_;  // simplified
    }
};

// If IDatabase was fat, we'd have to mock backup(), migrate(), etc. - wasted effort!

int main() {
    // Test with minimal mock
    MockReader mock({"alice", "bob", ""});
    UserService svc(mock);

    assert(svc.user_exists(0) == true);   // "alice" exists
    assert(svc.user_exists(1) == true);   // "bob" exists
    assert(svc.user_exists(2) == false);  // empty = not found
    assert(svc.user_exists(99) == false); // out of range

    std::cout << "All tests passed!\n";
}
// Expected output:
//   All tests passed!
```

**Benefit summary:**

| With Fat Interface | With Segregated Interfaces |
| --- | --- |
| Mock must implement 8+ methods | Mock implements 2 methods |
| Test depends on unrelated interface changes | Test isolated from write/admin changes |
| Hard to see what the test actually exercises | Clear: "UserService needs a reader" |

---

### Q3: Explain how concept definitions naturally encourage interface segregation in generic code

Concepts are compile-time ISP. Each concept declares only the minimum the function actually needs, and the compiler checks it precisely. There's no vtable, no inheritance, and no wasted stub methods.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <concepts>

// Concepts define MINIMAL requirements - ISP by nature!
template <typename T>
concept Readable = requires(const T& t, int id) {
    { t.read(id) } -> std::convertible_to<std::string>;
};

template <typename T>
concept Writable = requires(T& t, int id, const std::string& data) {
    { t.write(id, data) } -> std::same_as<void>;
};

template <typename T>
concept ReadWritable = Readable<T> && Writable<T>;

// Functions constrained to MINIMAL concept:
template <Readable DB>
void print_record(const DB& db, int id) {
    std::cout << "  Record " << id << ": " << db.read(id) << "\n";
}

template <Writable DB>
void store_record(DB& db, int id, const std::string& data) {
    db.write(id, data);
}

// Read-only type satisfies Readable but NOT Writable
class ReadOnlyCache {
    std::vector<std::string> cache_{"cached_a", "cached_b"};
public:
    std::string read(int id) const {
        return (id < 2) ? cache_[id] : "";
    }
    // No write() - it's read-only!
};

// Full database satisfies both
class SimpleDB {
    std::vector<std::string> data_{10, ""};
public:
    std::string read(int id) const { return (id < 10) ? data_[id] : ""; }
    void write(int id, const std::string& d) { if (id < 10) data_[id] = d; }
};

int main() {
    ReadOnlyCache cache;
    SimpleDB db;

    print_record(cache, 0);   // ReadOnlyCache satisfies Readable
    print_record(db, 0);      // SimpleDB satisfies Readable

    store_record(db, 0, "hello");  // SimpleDB satisfies Writable
    // store_record(cache, 0, "x");  // ERROR: ReadOnlyCache not Writable

    print_record(db, 0);
}
// Expected output:
//   Record 0: cached_a
//   Record 0:
//   Record 0: hello
```

Notice that `ReadOnlyCache` doesn't inherit from anything - it doesn't need to. As long as it has `read()`, it satisfies `Readable`. The compiler enforces the constraint and gives you a clear error message if you try to call `store_record` on a read-only type.

**Why concepts = natural ISP:**

- Each concept defines the **minimum** set of operations needed.
- Types don't need to "implement" anything - they just need to have the right member functions.
- Composing concepts (`ReadWritable = Readable && Writable`) mirrors composing interfaces.
- Compiler error messages are clear: "type X does not satisfy concept Y."

---

## Notes

- **ISP applies to templates too:** A template that requires `begin()`, `end()`, `size()`, `push_back()`, `insert()`, `erase()` forces all containers to provide everything. Better: constrain to just what's used.
- **C++ Core Guidelines I.25:** "Prefer abstract classes as interfaces to class hierarchies."
- **Multiple inheritance of interfaces** is idiomatic C++ - unlike Java/C# which have an `interface` keyword, C++ uses pure abstract classes.
- **ISP reduces recompilation:** Changing `IAdmin::backup()` doesn't force recompilation of code that only uses `IReader`.
- **Concepts (C++20) eliminate the runtime cost:** Virtual ISP has vtable overhead; concept-based ISP has zero overhead (templates, fully inlined).
