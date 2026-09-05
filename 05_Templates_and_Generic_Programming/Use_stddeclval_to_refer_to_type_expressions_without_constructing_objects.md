# Use `std::declval` to Refer to Type Expressions Without Constructing Objects

**Category:** Templates and Generic Programming  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/declval>  

---

## Topic Overview

### What Is `std::declval`

`std::declval<T>()` produces an **rvalue reference** to `T` without actually constructing an object. It can only be used in **unevaluated contexts** (`decltype`, `sizeof`, `noexcept`, concept requirements).

The classic use case is asking "what type does `T::foo()` return?" when `T` has no default constructor and you cannot write `T{}` to get an instance:

```cpp
// T has no default constructor — can't write T{}
// But we CAN ask "what type does T::foo() return?"
using ReturnType = decltype(std::declval<T>().foo());
```

### Signature

```cpp
template <class T>
std::add_rvalue_reference_t<T> declval() noexcept;
// Never defined — only declared. Using it at runtime -> linker error.
```

The fact that it is only declared and never defined is intentional - it forces you to use it only where the compiler analyzes expressions without generating code.

### Why It Exists

| Problem | Solution with `declval` |
| --- | --- |
| `T` has no default constructor | `declval<T>()` "creates" a `T&&` without construction |
| Need return type of `T::method()` | `decltype(std::declval<T>().method())` |
| SFINAE: check if expression is valid | `std::void_t<decltype(std::declval<T>() + std::declval<U>())>` |
| Abstract class `T` | `declval<T>()` works even for abstract types |

### `declval<T>()` Returns `T&&`

For non-void types, `declval<T>()` returns `T&&`. For `T&`, it returns `T&` (reference collapsing). For `void`, it returns `void`.

---

## Self-Assessment

### Q1: Use `decltype(std::declval<T>().foo())` to detect the return type of a method without `T` being default-constructible

The key insight is that `decltype` is an unevaluated context - the compiler only looks at the type of the expression, it never executes it. So even though you cannot construct a `Database` or a `Shape`, you can still ask the compiler what their methods return.

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <string>

// A class with NO default constructor
class Database {
    std::string connection_;
public:
    // Only this constructor — no default!
    explicit Database(std::string conn) : connection_(std::move(conn)) {}

    int query_count() const { return 42; }
    std::string name() const { return connection_; }
    void close() {}
};

// Abstract base class — can NEVER be constructed
class Shape {
public:
    virtual double area() const = 0;
    virtual std::string type() const = 0;
    virtual ~Shape() = default;
};

// === Detect return types using declval ===

// Without declval, this would require:  Database{}.query_count()
// But Database has no default constructor -> compile error!

// With declval: "pretend" we have a Database object
using QueryReturnType = decltype(std::declval<Database>().query_count());
using NameReturnType = decltype(std::declval<Database>().name());

// Works even for abstract classes!
using AreaReturnType = decltype(std::declval<Shape>().area());

// === Generic return-type detector ===
template <typename T>
using method_result_t = decltype(std::declval<T>().query_count());

int main() {
    std::cout << "=== Return type detection with declval ===\n";

    std::cout << "Database::query_count() returns int: "
              << std::is_same_v<QueryReturnType, int> << "\n";  // 1

    std::cout << "Database::name() returns string: "
              << std::is_same_v<NameReturnType, std::string> << "\n";  // 1

    std::cout << "Shape::area() returns double: "
              << std::is_same_v<AreaReturnType, double> << "\n";  // 1

    std::cout << "\n=== Generic detector ===\n";
    std::cout << "method_result_t<Database> is int: "
              << std::is_same_v<method_result_t<Database>, int> << "\n";  // 1

    // declval gives T&& — check the reference type
    using DeclvalType = decltype(std::declval<Database>());
    std::cout << "declval<Database>() is Database&&: "
              << std::is_rvalue_reference_v<DeclvalType> << "\n";  // 1

    return 0;
}
```

### Q2: Explain why `std::declval<T>()` is only valid in unevaluated contexts like `decltype` and `sizeof`

`std::declval<T>()` is declared but **never defined**:

```cpp
// In <utility>, roughly:
template <class T>
std::add_rvalue_reference_t<T> declval() noexcept;
// No function body! Just a declaration.
```

The reason this design works is that the compiler only needs a declaration to analyze type expressions. Here is why it cannot work at runtime:

1. **No definition** - If you try to call it, the linker cannot find the function body - **linker error**
2. **Design intent** - It exists solely to "manufacture" a type expression for the compiler's type analysis
3. **Would be impossible for some types** - How would you construct an abstract class? A deleted-constructor type?

**Unevaluated contexts** are places where the compiler analyzes expressions but never executes them:

| Context | Example | Evaluated at runtime? |
| --- | --- | :---: |
| `decltype(expr)` | `decltype(std::declval<T>().foo())` | No |
| `sizeof(expr)` | `sizeof(std::declval<T>())` | No |
| `noexcept(expr)` | `noexcept(std::declval<T>().foo())` | No |
| `requires { expr; }` | `requires { std::declval<T>() + 1; }` | No |

Here is a minimal example showing what is and is not allowed:

```cpp
#include <iostream>
#include <utility>

class NonConstructible {
    NonConstructible() = delete;  // Cannot create!
public:
    int value() const { return 42; }
};

int main() {
    // OK: decltype is unevaluated — no object is created
    using R = decltype(std::declval<NonConstructible>().value());
    std::cout << "Return type detected: " << std::is_same_v<R, int> << "\n";

    // OK: sizeof is unevaluated
    std::cout << "Size: " << sizeof(std::declval<NonConstructible>()) << "\n";

    // ERROR: this would try to actually CALL declval at runtime
    // auto x = std::declval<NonConstructible>();  // LINKER ERROR

    return 0;
}
```

### Q3: Show a SFINAE check that uses `declval` to test for a specific member function signature

The pattern here is `std::void_t<decltype(expression)>`. The idea is that if the expression inside `decltype` is valid for type `T`, the specialization matches and the trait becomes `true_type`. If the expression would be ill-formed, SFINAE kicks in and the primary template (`false_type`) wins instead.

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <string>
#include <vector>

// === SFINAE detector: does T have .size() returning something convertible to size_t? ===
template <typename T, typename = void>
struct has_size : std::false_type {};

template <typename T>
struct has_size<T,
    std::void_t<decltype(static_cast<std::size_t>(std::declval<T>().size()))>>
    : std::true_type {};

// === Detector: does T have .push_back(U)? ===
template <typename T, typename U, typename = void>
struct has_push_back : std::false_type {};

template <typename T, typename U>
struct has_push_back<T, U,
    std::void_t<decltype(std::declval<T>().push_back(std::declval<U>()))>>
    : std::true_type {};

// === Detector: does T support T + T? ===
template <typename T, typename = void>
struct is_addable : std::false_type {};

template <typename T>
struct is_addable<T,
    std::void_t<decltype(std::declval<T>() + std::declval<T>())>>
    : std::true_type {};

// === Detector: does T have .serialize() returning std::string? ===
template <typename T, typename = void>
struct has_serialize : std::false_type {};

template <typename T>
struct has_serialize<T,
    std::enable_if_t<
        std::is_same_v<
            decltype(std::declval<const T>().serialize()),
            std::string>>>
    : std::true_type {};

// Test classes
struct Widget {
    std::string serialize() const { return "Widget"; }
};

struct Gadget {
    int serialize() const { return 42; }  // Wrong return type!
};

struct Plain {};

// === Use the detectors ===
template <typename T>
std::enable_if_t<has_size<T>::value>
print_size(const T& container) {
    std::cout << "  Size: " << container.size() << "\n";
}

template <typename T>
std::enable_if_t<!has_size<T>::value>
print_size(const T&) {
    std::cout << "  (no .size() method)\n";
}

int main() {
    std::cout << "=== has_size detection ===\n";
    std::cout << "vector<int>: " << has_size<std::vector<int>>::value << "\n";  // 1
    std::cout << "string: " << has_size<std::string>::value << "\n";            // 1
    std::cout << "int: " << has_size<int>::value << "\n";                       // 0

    std::cout << "\n=== has_push_back detection ===\n";
    std::cout << "vector<int>, int: "
              << has_push_back<std::vector<int>, int>::value << "\n";  // 1
    std::cout << "string, char: "
              << has_push_back<std::string, char>::value << "\n";      // 1
    std::cout << "int, int: "
              << has_push_back<int, int>::value << "\n";               // 0

    std::cout << "\n=== is_addable detection ===\n";
    std::cout << "int: " << is_addable<int>::value << "\n";          // 1
    std::cout << "string: " << is_addable<std::string>::value << "\n"; // 1
    std::cout << "vector<int>: " << is_addable<std::vector<int>>::value << "\n"; // 0

    std::cout << "\n=== has_serialize (exact signature) ===\n";
    std::cout << "Widget: " << has_serialize<Widget>::value << "\n";  // 1
    std::cout << "Gadget: " << has_serialize<Gadget>::value << "\n";  // 0 (returns int, not string)
    std::cout << "Plain: " << has_serialize<Plain>::value << "\n";    // 0

    std::cout << "\n=== Conditional dispatch ===\n";
    std::vector<int> v = {1, 2, 3};
    print_size(v);    // Size: 3
    print_size(42);   // (no .size() method)

    return 0;
}
```

Notice that `has_serialize` for `Gadget` returns `false` even though `Gadget` does have a `serialize` method - the return type is `int`, not `std::string`, so the expression check fails. That is exactly the kind of precise interface testing that `declval` makes possible.

---

## Notes

- `std::declval<T>()` returns `T&&` without constructing `T`. Only usable in unevaluated contexts.
- Essential for SFINAE detection: check if expressions like `t.foo()`, `t + u` are valid.
- Works with abstract classes, deleted constructors, and move-only types.
- Common patterns: `std::void_t<decltype(std::declval<T>().method())>` for member detection.
- In C++20, prefer **concepts and requires expressions** - they're cleaner and provide better errors.
- Never call `std::declval` at runtime - it has no definition (linker error).
