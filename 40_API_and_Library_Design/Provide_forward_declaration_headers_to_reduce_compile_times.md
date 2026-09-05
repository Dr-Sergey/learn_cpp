# Provide forward declaration headers to reduce compile times

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#sf-source-files>  

---

## Topic Overview

Heavy headers slow compilation down because every translation unit that includes them pays the full cost of parsing all the types, templates, and transitive includes they pull in. Forward declaration headers - typically named `_fwd.h` or `_fwd.hpp` - let consumers declare types without including their full definitions. A header that only needs to hold a pointer to a type doesn't need to know the type's layout.

### Pattern

The idea is to split your library's declarations into two tiers: a lightweight `_fwd` header with just enough to name the types, and the full header with the actual definitions. Here's what that looks like in practice:

```cpp
// mylib/types_fwd.hpp — lightweight, include anywhere
#pragma once
namespace mylib {
    class Connection;
    class Request;
    class Response;
    struct Config;
    enum class Status : int;
}

// mylib/types.hpp — full definitions, include only in .cpp
#pragma once
#include "types_fwd.hpp"
#include <string>
#include <vector>
#include <memory>

namespace mylib {
    class Connection {
        // ... heavy implementation
    };
    // ...
}

// consumer.hpp — can use forward declarations only
#include <mylib/types_fwd.hpp>
#include <memory>

class Consumer {
    std::unique_ptr<mylib::Connection> conn_;  // OK: unique_ptr of incomplete type
public:
    void process(const mylib::Request& req);   // OK: reference to incomplete type
};

// consumer.cpp — needs full definitions for method bodies
#include "consumer.hpp"
#include <mylib/types.hpp>

void Consumer::process(const mylib::Request& req) {
    // Now Connection and Request are complete
}
```

The `.hpp` file only needs the `_fwd` header because it only uses pointers and references - it doesn't call any methods or store types by value. The `.cpp` file is where the full header finally appears, and that's fine because it's only compiled once. This is the compilation firewall pattern in action.

### What Can Use Incomplete Types

Not every use of a type needs to know its full definition. Here's the quick reference:

| Usage | Incomplete OK? |
| --- | --- |
| `T*`, `T&`, `const T&` | Yes |
| `std::unique_ptr<T>` | Yes (delete in .cpp) |
| `std::shared_ptr<T>` | Yes |
| `std::vector<T>` | No (needs `sizeof(T)`) |
| `T` by value | No |
| `sizeof(T)` | No |
| Calling `T` methods | No |

The rule of thumb: if you need to know how big the type is or what methods it has, you need the full definition. If you only need to talk about the type - pass a pointer, store a reference, return a handle - a forward declaration is enough.

---

## Self-Assessment

### Q1: Why does `unique_ptr<T>` work with incomplete types but `vector<T>` doesn't

`unique_ptr<T>` only stores a raw `T*` internally - a pointer has a fixed size regardless of what `T` is. The destructor calls `delete` on that pointer, which does need `T` to be complete, but you can put that destructor definition in a `.cpp` file where `T` is fully defined. `vector<T>` is different: it needs `sizeof(T)` to allocate its element storage and needs `T`'s destructor to destroy elements during reallocation - both of those require a complete type at the point of instantiation.

### Q2: Measure compile time improvement from forward declarations

The gains are multiplicative - every translation unit that includes the lighter header saves time, so the improvement scales with your project size:

```cpp
// Before: consumer.hpp includes full mylib/types.hpp
// Including 50 headers transitively -> 200ms per TU

// After: consumer.hpp includes only mylib/types_fwd.hpp
// Including 2 headers -> 20ms per TU

// With 100 TUs including consumer.hpp:
// Before: 100 x 200ms = 20s
// After:  100 x 20ms  = 2s (10x faster)
```

A 10x improvement from one header split is not unusual in real projects. The more widely-included your header is, the more every millisecond saved per inclusion multiplies up.

### Q3: When can't you use forward declarations

You cannot use a forward declaration when the header needs the type's size - for example, if you store a `mylib::Config` as a value member (not a pointer), the compiler needs to know `sizeof(Config)` to lay out your class. The same applies to inheritance: a derived class needs the complete base class definition. And of course, any place you call a method on an object or instantiate a container like `vector<T>` also requires the complete type.

---

## Notes

- The standard library itself provides `<iosfwd>` as a forward declaration header for streams - you can include it in headers that only need `std::ostream&` parameters, saving the cost of the full `<iostream>` parse.
- A good rule of thumb: headers should include as little as possible; `.cpp` files include everything they need.
- Combining the Pimpl idiom with forward declaration headers gives you maximum compilation insulation - your `.hpp` exposes zero implementation details and pulls in zero implementation headers.
