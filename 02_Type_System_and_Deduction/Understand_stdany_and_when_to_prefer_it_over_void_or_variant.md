# Understand `std::any` and When to Prefer It Over `void*` or `variant`

**Category:** Type System and Deduction  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/any>  

---

## Topic Overview

### What Is `std::any`

`std::any` is a type-safe container for a **single value of any copy-constructible type**. It uses type-erasure internally - think of it as a safe replacement for `void*` that knows its stored type and can enforce correct casts at runtime.

```cpp
#include <any>
#include <string>

std::any a = 42;             // stores int
a = std::string("hello");    // now stores string
a = 3.14;                    // now stores double
```

Each assignment replaces the old value and its type entirely. The `any` object tracks the type for you.

### Core API

| Operation | Description |
| --- | --- |
| `std::any a;` | Empty any (holds nothing) |
| `std::any a = value;` | Construct with a value |
| `a = new_value;` | Assign new value (any type) |
| `a.has_value()` | Check if non-empty |
| `a.type()` | Returns `std::type_info` of stored type |
| `a.reset()` | Clear contents (become empty) |
| `std::any_cast<T>(a)` | Extract value - throws `std::bad_any_cast` on type mismatch |
| `std::any_cast<T>(&a)` | Extract pointer - returns `nullptr` on type mismatch (no throw) |
| `std::make_any<T>(args...)` | Construct in-place |
| `a.emplace<T>(args...)` | Destroy old, construct new in-place |

### `std::any_cast` - Safe Extraction

There are three flavors of `any_cast`. The pointer version is usually the right choice for conditional access because it never throws:

```cpp
std::any a = 42;

// By value (copies out)
int val = std::any_cast<int>(a);                // OK: 42

// By reference (avoids copy)
int& ref = std::any_cast<int&>(a);              // OK: reference to stored int
ref = 100;                                       // modifies the stored value

// By pointer (safe, never throws)
int* ptr = std::any_cast<int>(&a);              // returns pointer or nullptr
if (ptr) std::cout << *ptr;                     // safe

// Wrong type
// double d = std::any_cast<double>(a);         // THROWS std::bad_any_cast
double* dp = std::any_cast<double>(&a);         // returns nullptr (safe)
```

### Type-Erasure Mechanics

`std::any` internally stores:

1. A **type descriptor** (typically a function pointer table or vtable pointer)
2. A **storage area** for the held value

```cpp
┌──────────────────────────────────────┐
│ std::any                             │
├──────────────────────────────────────┤
│ manager_ptr -> [clone/destroy/type]  │  <- type-erasure function table
├──────────────────────────────────────┤
│ storage (union):                     │
│   small buffer [~32 bytes]           │  <- SBO: small types stored inline
│   OR                                 │
│   heap pointer -> allocated object   │  <- large types heap-allocated
└──────────────────────────────────────┘
```

### Small-Buffer Optimization (SBO)

Most implementations (GCC, Clang, MSVC) provide SBO - small types are stored **inline** within the `any` object, avoiding heap allocation:

| Implementation | SBO buffer size |
| --- | --- |
| libstdc++ (GCC) | Typically fits types <= `sizeof(void*) * 2` (16 bytes on 64-bit) |
| libc++ (Clang) | 3 x `sizeof(void*)` (24 bytes on 64-bit) |
| MSVC | 64 bytes (generous!) |

Types larger than the SBO buffer are heap-allocated. Types that are not nothrow-move-constructible may also be heap-allocated even if they fit.

### `std::any` vs `void*` vs `std::variant`

| Feature | `void*` | `std::any` | `std::variant<Ts...>` |
| --- | --- | --- | --- |
| Type safety | None | Runtime | Compile-time |
| Knows stored type | No | Yes (`type()`) | Yes (index) |
| Cast safety | Undefined behavior | Exception/nullptr | Compile-time checked |
| Fixed type set | No | No | Yes |
| Heap allocation | Manual | Maybe (SBO) | Never |
| `std::visit` | No | No | Yes (exhaustive) |
| Size | `sizeof(void*)` | ~32-64 bytes | `max(sizeof(Ts...))` + discriminant |
| Copy semantics | Shallow | Deep (value) | Deep (value) |
| Requires copy-constructible | No | Yes | Per alternative |

### When to Use Each

Here's the practical decision matrix. If the table feels like a lot, it boils down to: use `variant` when you know all the types at compile time, use `any` when you don't:

```cpp
// USE std::any when:
//   Type set is open-ended (plugin systems, property maps)
//   External code provides types you don't know at compile time
//   Message passing with heterogeneous payloads
//   Configuration storage (key-value with mixed types)

// USE std::variant when:
//   Type set is fixed and known at compile time
//   You want compile-time exhaustiveness checking (visit)
//   Performance matters (no heap allocation)
//   You want pattern matching

// USE void* when:
//   C API interop (callback user data)
//   Extreme performance constraints
//   You control all creation and consumption (type is "known")

// NEVER USE void* when:
//   Ownership transfer needed
//   Types are unknown or extensible
//   Safety matters more than performance
```

---

## Self-Assessment

### Q1: Store heterogeneous values in a `std::vector<std::any>` and extract them safely with `std::any_cast`

Notice how `print_any` uses the pointer cast in a chain of `if/else` rather than catching exceptions - that's the idiomatic pattern for type dispatch on `any`:

```cpp
#include <any>
#include <iostream>
#include <string>
#include <vector>
#include <typeinfo>

// A heterogeneous property bag
void print_any(const std::any& a) {
    if (!a.has_value()) {
        std::cout << "  [empty]\n";
        return;
    }

    // Try known types with pointer cast (no-throw)
    if (auto p = std::any_cast<int>(&a)) {
        std::cout << "  int: " << *p << "\n";
    } else if (auto p = std::any_cast<double>(&a)) {
        std::cout << "  double: " << *p << "\n";
    } else if (auto p = std::any_cast<std::string>(&a)) {
        std::cout << "  string: " << *p << "\n";
    } else if (auto p = std::any_cast<bool>(&a)) {
        std::cout << "  bool: " << std::boolalpha << *p << "\n";
    } else {
        std::cout << "  [unknown type: " << a.type().name() << "]\n";
    }
}

int main() {
    // Store heterogeneous values
    std::vector<std::any> bag;
    bag.push_back(42);
    bag.push_back(3.14);
    bag.push_back(std::string("hello"));
    bag.push_back(true);
    bag.emplace_back(std::in_place_type<std::string>, 5, 'x');  // "xxxxx" in-place
    bag.push_back(std::any{});  // empty

    std::cout << "=== Property bag contents ===\n";
    for (const auto& item : bag) {
        print_any(item);
    }

    // Safe extraction with exception handling
    std::cout << "\n=== Safe extraction ===\n";
    try {
        int val = std::any_cast<int>(bag[0]);
        std::cout << "bag[0] as int: " << val << "\n";

        // Wrong type — will throw
        double wrong = std::any_cast<double>(bag[0]);
    } catch (const std::bad_any_cast& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    // Pointer-based extraction (preferred for conditional access)
    std::cout << "\n=== Pointer-based extraction (no throw) ===\n";
    for (std::size_t i = 0; i < bag.size(); ++i) {
        if (auto* s = std::any_cast<std::string>(&bag[i])) {
            std::cout << "bag[" << i << "] is string: " << *s << "\n";
        }
    }

    // Modify through reference cast
    std::any_cast<int&>(bag[0]) = 999;
    std::cout << "\nModified bag[0]: " << std::any_cast<int>(bag[0]) << "\n";

    return 0;
}
```

**Output:**

```text
=== Property bag contents ===
  int: 42
  double: 3.14
  string: hello
  bool: true
  string: xxxxx
  [empty]

=== Safe extraction ===
bag[0] as int: 42
Caught: bad any_cast

=== Pointer-based extraction (no throw) ===
bag[2] is string: hello
bag[4] is string: xxxxx

Modified bag[0]: 999
```

**How this works:**

- `std::vector<std::any>` stores values of any copy-constructible type in a single container
- Pointer overload `std::any_cast<T>(&a)` returns `nullptr` on type mismatch - safe, no exception
- Reference overload `std::any_cast<T&>(a)` allows in-place modification
- `emplace` with `std::in_place_type<T>` constructs directly, avoiding temporaries

### Q2: Explain the type-erasure mechanics of `std::any` and its small-buffer optimization

`std::any` implements type erasure using an internal **manager function** (or virtual dispatch table):

**Step 1 - On construction/assignment:**

- The `any` object records a pointer to a "manager" function specialized for type `T`
- The manager knows how to: copy `T`, destroy `T`, return `typeid(T)`

**Step 2 - Storage:**

- If `sizeof(T) <= SBO_size` AND `T` is nothrow-move-constructible: stored **inline** (no heap)
- Otherwise: `new T(...)` on the heap, `any` stores the pointer

**Step 3 - On `any_cast<T>`:**

- Compare `typeid` of requested type vs stored type
- If match: return pointer to stored object (cast from internal storage)
- If mismatch: throw `bad_any_cast` (or return `nullptr` for pointer overload)

The code below lets you observe the SBO threshold on your actual implementation:

```cpp
#include <any>
#include <iostream>
#include <string>
#include <array>

struct Small { int x; };                     // 4 bytes — fits SBO
struct Large { std::array<char, 256> data; }; // 256 bytes — likely heap-allocated

int main() {
    std::cout << "sizeof(std::any): " << sizeof(std::any) << " bytes\n";

    // Small: likely SBO (no heap allocation)
    std::any a = Small{42};
    std::cout << "Small stored: " << std::any_cast<Small>(a).x << "\n";

    // Large: likely heap-allocated
    std::any b = Large{};
    std::cout << "Large stored: OK\n";

    // SBO threshold check (implementation-specific)
    std::cout << "\nSBO analysis:\n";
    std::cout << "  sizeof(Small) = " << sizeof(Small)
              << (sizeof(Small) <= sizeof(std::any) - sizeof(void*) ? " -> likely SBO" : " -> likely heap")
              << "\n";
    std::cout << "  sizeof(Large) = " << sizeof(Large)
              << (sizeof(Large) <= sizeof(std::any) - sizeof(void*) ? " -> likely SBO" : " -> likely heap")
              << "\n";

    // The type info is preserved even after reassignment
    a = std::string("hello");
    std::cout << "\nAfter reassignment: type = " << a.type().name() << "\n";
    std::cout << "Value: " << std::any_cast<std::string>(a) << "\n";

    return 0;
}
```

**Key takeaways:**

- SBO eliminates heap allocation for small types (typically <= 16-64 bytes depending on implementation)
- nothrow-move-constructible is often required for SBO eligibility
- The type-erasure overhead is similar to a virtual function call per operation
- `type()` returns a `std::type_info&` which enables runtime type checking

### Q3: List three cases where `std::variant` is preferable to `std::any` from a safety perspective

**Case 1 - Compile-time exhaustiveness with `std::visit`:**

```cpp
#include <variant>
#include <string>
#include <iostream>

using Value = std::variant<int, double, std::string>;

void process(const Value& v) {
    std::visit([](auto&& arg) {
        using T = std::decay_t<decltype(arg)>;
        if constexpr (std::is_same_v<T, int>)
            std::cout << "Integer: " << arg << "\n";
        else if constexpr (std::is_same_v<T, double>)
            std::cout << "Double: " << arg << "\n";
        else
            std::cout << "String: " << arg << "\n";
    }, v);
}
// If a new type is added to the variant, the compiler forces you to handle it.
// With std::any, forgotten types silently fail at runtime.
```

**Case 2 - No possibility of `bad_any_cast` at runtime:**

```cpp
// std::any: runtime failure if you forget a type
std::any a = 42;
std::string s = std::any_cast<std::string>(a);  // THROWS at runtime!

// std::variant: impossible to extract wrong type
std::variant<int, std::string> v = 42;
// std::string s = std::get<std::string>(v);  // THROWS, but...
auto* p = std::get_if<std::string>(&v);        // Returns nullptr safely
// And std::visit handles ALL types by design — no casts needed!
```

**Case 3 - No heap allocation, deterministic size:**

```cpp
#include <any>
#include <variant>
#include <string>
#include <iostream>
#include <vector>

struct Config {
    // Variant: size known at compile time, no heap alloc
    using Setting = std::variant<int, double, bool, std::string>;
    std::vector<Setting> settings;  // Predictable memory layout

    // any: size unknown, may heap-allocate per element
    // std::vector<std::any> settings;  // Unpredictable, slower
};

int main() {
    using V = std::variant<int, double, std::string>;
    std::cout << "sizeof(variant<int,double,string>): " << sizeof(V) << "\n";
    std::cout << "sizeof(any): " << sizeof(std::any) << "\n";
    // variant is typically smaller AND never allocates
    return 0;
}
```

**Summary:**

| Safety aspect | `std::variant` | `std::any` |
| --- | --- | --- |
| Type checking | Compile-time (visit) | Runtime (any_cast) |
| Exhaustiveness | Compiler-enforced | Manually checked |
| Memory | Stack-only, predictable | May heap-allocate |
| Wrong-type access | Compile error (usually) | Runtime exception |
| Code review | Types visible in signature | Types hidden |

---

## Notes

- **`std::any` requires copy-constructibility.** Move-only types cannot be stored in `any`. Use `std::unique_ptr<void, Deleter>` or a custom type-erased wrapper for move-only types.
- **Performance:** `any_cast` is typically as fast as a `dynamic_cast` (one `typeid` comparison). For hot loops, prefer `variant` with `visit`.
- **`std::any` is not a replacement for inheritance.** If your types share a common interface, use virtual functions or concepts.
- **Thread safety:** `std::any` has the same thread-safety guarantees as other standard containers - concurrent reads are safe, concurrent writes require synchronization.
- **Avoid `std::any` in public APIs** when possible - it hides the type contract from callers. Prefer `variant` for closed type sets.
