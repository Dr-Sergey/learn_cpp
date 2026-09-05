# Use the Pimpl idiom to reduce compilation dependencies

**Category:** Best Practices and Idioms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/pimpl>  

---

## Topic Overview

**Pimpl** (Pointer to Implementation) is an idiom that hides a class's private members behind a forward-declared pointer. Clients only see the public interface in the header - all the messy private fields and their heavy includes live exclusively in the `.cpp` file. The payoff is that changing anything in the implementation doesn't recompile the clients.

```cpp
// +------------------+     +-------------------+
// | Widget.h (public)|     | Widget.cpp        |
// | class Widget {   |     | struct Widget::Impl {
// |   struct Impl;   |     |   std::string name;
// |   unique_ptr<>   |---->|   std::vector data;
// |   void doWork(); |     |   Database db;
// | };               |     | };                |
// +------------------+     +-------------------+
```

Clients include `Widget.h` - they never see `string`, `vector`, or `Database` headers.

The reason this matters in large projects is that C++ recompiles every translation unit that transitively includes a changed header. If `Widget.h` mentions `Database.h`, every file using `Widget` gets recompiled whenever `Database.h` changes. Pimpl breaks that chain.

---

## Self-Assessment

### Q1: Implement a class with Pimpl

This is split across three files the way it would look in a real project. Pay attention to where the destructor and move operations are defined - they *must* be in the `.cpp` file, not the header, because `unique_ptr`'s deleter needs the complete `Impl` type.

**widget.h:**

```cpp
#pragma once
#include <memory>
#include <string>

class Widget {
public:
    explicit Widget(const std::string& name);
    ~Widget();  // defined in .cpp (Impl is incomplete here)

    // Move operations
    Widget(Widget&& other) noexcept;
    Widget& operator=(Widget&& other) noexcept;

    // No copy (or implement in .cpp)
    Widget(const Widget&) = delete;
    Widget& operator=(const Widget&) = delete;

    void doWork();
    std::string name() const;

private:
    struct Impl;                  // forward declaration only!
    std::unique_ptr<Impl> pImpl_; // pointer to implementation
};
```

**widget.cpp:**

```cpp
#include "widget.h"
#include <iostream>
#include <vector>
#include <algorithm>
// Heavy headers only in .cpp!

struct Widget::Impl {
    std::string name;
    std::vector<int> data;
    int counter = 0;

    Impl(const std::string& n) : name(n), data{1, 2, 3} {}

    void process() {
        std::sort(data.begin(), data.end());
        ++counter;
        std::cout << name << ": processed (count=" << counter << ")\n";
    }
};

// Constructor/destructor MUST be in .cpp where Impl is complete
Widget::Widget(const std::string& name)
    : pImpl_(std::make_unique<Impl>(name)) {}

Widget::~Widget() = default;  // unique_ptr needs complete type here

Widget::Widget(Widget&& other) noexcept = default;
Widget& Widget::operator=(Widget&& other) noexcept = default;

void Widget::doWork() { pImpl_->process(); }
std::string Widget::name() const { return pImpl_->name; }
```

**main.cpp:**

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
#include "widget.h"
#include <iostream>
// Note: we did NOT include <vector>, <algorithm>, etc.!

int main() {
    Widget w("MyWidget");
    w.doWork();
    w.doWork();
    std::cout << "Name: " << w.name() << '\n';
}
// Expected output:
// MyWidget: processed (count=1)
// MyWidget: processed (count=2)
// Name: MyWidget
```

`main.cpp` has no idea that `vector` or `algorithm` are involved. If you later swap the `vector` for a different data structure, `main.cpp` doesn't recompile.

### Q2: Pimpl tradeoffs

Pimpl isn't free - you're trading compilation speed and interface stability for a heap allocation and a pointer indirection on every call. Here's the full picture:

| Benefit | Cost |
| --- | --- |
| Reduced compile time (changes to Impl don't affect clients) | Heap allocation for Impl |
| ABI stability (sizeof Widget doesn't change) | Pointer indirection on every method call |
| Cleaner headers (no private includes) | More boilerplate (constructor/destructor in .cpp) |
| Faster incremental builds | Cannot inline implementation methods |

**When to use Pimpl:**

- Large projects where build times matter
- Library APIs where ABI stability is critical
- Classes with many private dependencies

**When NOT to use Pimpl:**

- Performance-critical inner loops (indirection cost)
- Small utility classes (overhead not worth it)
- Header-only libraries

### Q3: Pimpl with `unique_ptr` and forward-declared incomplete type

This example condenses the full pattern into a single file to make the mechanics easier to study. In a real project the `// Implementation` section would live in a separate `.cpp`. The key rules at the bottom of the code are worth committing to memory - violating any of them produces confusing compiler errors about incomplete types.

```cpp
#include <iostream>
#include <memory>
#include <string>

// Header: only forward-declare Impl
class Database {
public:
    Database(const std::string& connection);
    ~Database();  // MUST be in .cpp

    Database(Database&&) noexcept;
    Database& operator=(Database&&) noexcept;

    void query(const std::string& sql);
    bool is_connected() const;

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};

// Implementation (normally in .cpp)
struct Database::Impl {
    std::string connection_;
    bool connected_ = false;

    Impl(const std::string& conn) : connection_(conn), connected_(true) {}
};

Database::Database(const std::string& conn)
    : impl_(std::make_unique<Impl>(conn)) {}
Database::~Database() = default;
Database::Database(Database&&) noexcept = default;
Database& Database::operator=(Database&&) noexcept = default;

void Database::query(const std::string& sql) {
    std::cout << "[" << impl_->connection_ << "] " << sql << '\n';
}
bool Database::is_connected() const { return impl_->connected_; }

int main() {
    Database db("localhost:5432");
    std::cout << "Connected: " << db.is_connected() << '\n';
    db.query("SELECT * FROM users");
}
// Expected output:
// Connected: 1
// [localhost:5432] SELECT * FROM users
```

**Key rules for unique_ptr + Pimpl:**

1. Destructor **must** be defined in `.cpp` (where `Impl` is complete)
2. Move operations **must** be defined in `.cpp` too
3. Use `std::make_unique<Impl>(...)` in constructor

---

## Notes

- The destructor, move constructor, and move assignment must be in the `.cpp` file.
- For copy semantics, implement deep copy of `Impl` in `.cpp`.
- `std::unique_ptr<Impl>` requires `Impl` to be complete at destruction point.
- Alternative: `std::shared_ptr<Impl>` works without .cpp-defined destructor (type-erases deleter).
- Fast Pimpl: use a stack buffer + placement new to avoid heap allocation.
