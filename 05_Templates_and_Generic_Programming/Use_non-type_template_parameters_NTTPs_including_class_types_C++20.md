# Use Non-Type Template Parameters (NTTPs) Including Class Types (C++20)

**Category:** Templates and Generic Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/template_parameters#Non-type_template_parameter>  

---

## Topic Overview

### What Are Non-Type Template Parameters

You already know that template parameters can be types. NTTPs are the other kind - they let you parameterize templates on **values** rather than types. The key constraint is that the value must be known at compile time:

```cpp
template <int N>          // N is a non-type template parameter
struct Array {
    int data[N];           // N is known at compile time
};
Array<10> a;               // Creates array of 10 ints
```

`Array<10>` and `Array<20>` are two completely different types, just like `vector<int>` and `vector<double>` are. Each unique value of `N` creates a distinct instantiation.

### NTTP Types Across Standards

The set of types you can use as NTTPs has grown with each standard. C++20 is particularly notable because it allows floating-point and structural class types, which were previously forbidden:

| Standard | Allowed NTTP Types |
| --- | --- |
| C++98/03 | Integral types, pointers, references, enums |
| C++11 | + `nullptr_t`, scoped enums |
| C++17 | + `auto` NTTPs (deduced type) |
| C++20 | + **Floating-point**, **class types** (if structural), `auto` with constraints |

### Structural Types (C++20)

For a class type to be usable as an NTTP, it must be a **structural type**. The requirements exist because the compiler needs to be able to compare two NTTP values for equality (to know whether two template instantiations are the same), and to encode the value into the mangled name:

- All base classes and non-static data members are **public** and non-mutable
- All base classes and members are themselves structural types or arrays thereof
- Has no user-provided constructors (but can have `constexpr` constructors via defaulted or aggregate)

```cpp
struct FixedString {          // Structural type
    char data[32]{};
    std::size_t len{};

    constexpr FixedString(const char* s) : len{0} {
        while (s[len]) { data[len] = s[len]; ++len; }
    }
};

template <FixedString Tag>   // Class type as NTTP!
void log() { /* ... */ }

log<"INFO">();                // String literal -> FixedString NTTP
```

---

## Self-Assessment

### Q1: Write a template that takes a `size_t N` and creates a `std::array<int, N>`

Notice that `buf5` and `buf3` are different types even though they look similar - the size is baked into the type, which is what makes `static_assert` work at compile time:

```cpp
#include <iostream>
#include <array>
#include <numeric>
#include <algorithm>

// Basic NTTP: size_t
template <std::size_t N>
struct FixedBuffer {
    std::array<int, N> data{};  // N known at compile time

    void fill_sequential() {
        std::iota(data.begin(), data.end(), 1);
    }

    void print(const char* label) const {
        std::cout << label << " [";
        for (std::size_t i = 0; i < N; ++i) {
            if (i > 0) std::cout << ", ";
            std::cout << data[i];
        }
        std::cout << "] (size=" << N << ")\n";
    }

    constexpr std::size_t size() const { return N; }
};

// auto NTTP (C++17) — compiler deduces the type
template <auto Value>
struct Constant {
    static constexpr auto value = Value;
    void print() const {
        std::cout << "Constant<" << value << "> (type size=" << sizeof(Value) << ")\n";
    }
};

// Multiple NTTPs for a matrix
template <std::size_t Rows, std::size_t Cols>
struct Matrix {
    std::array<std::array<double, Cols>, Rows> data{};

    void print() const {
        std::cout << "Matrix<" << Rows << "x" << Cols << ">:\n";
        for (const auto& row : data) {
            for (double v : row) std::cout << "  " << v;
            std::cout << "\n";
        }
    }
};

int main() {
    std::cout << "=== FixedBuffer<N> ===\n";
    FixedBuffer<5> buf5;
    buf5.fill_sequential();
    buf5.print("buf5");   // [1, 2, 3, 4, 5]

    FixedBuffer<3> buf3;
    buf3.fill_sequential();
    buf3.print("buf3");   // [1, 2, 3]

    // N is compile-time -> different N = different TYPE
    static_assert(buf5.size() == 5);
    static_assert(buf3.size() == 3);

    std::cout << "\n=== auto NTTP ===\n";
    Constant<42> c1;    c1.print();     // int
    Constant<'A'> c2;   c2.print();     // char
    Constant<true> c3;  c3.print();     // bool

    std::cout << "\n=== Matrix ===\n";
    Matrix<2, 3> m;
    m.data[0] = {1.0, 2.0, 3.0};
    m.data[1] = {4.0, 5.0, 6.0};
    m.print();

    return 0;
}
```

### Q2: Explain what structural types are in C++20 NTTPs and why floating-point was disallowed before C++20

The floating-point restriction before C++20 came down to a fundamental problem with identity. The language needs to decide whether `Array<0.1 + 0.2>` and `Array<0.3>` are the same type or not - and IEEE 754 arithmetic makes that ambiguous in ways that would produce confusing behavior. C++20 resolved this by switching to bitwise comparison for float NTTPs:

**Structural Types (C++20):**

| Requirement | Why |
| --- | --- |
| All members public and non-mutable | Compiler must read all members for template argument comparison |
| No user-provided copy/move constructors | Must be trivially copyable for mangling |
| All members are themselves structural | Recursive requirement |
| Destructor is trivial or constexpr | Must be usable at compile time |

```cpp
// Structural: all public, no user constructors
struct Point { int x; int y; };
template <Point P> void f();
f<Point{1, 2}>();

// NOT structural: private member
struct BadPoint {
private: int x; int y;
public: BadPoint(int a, int b) : x(a), y(b) {}
};
// template <BadPoint P> void g();  // ERROR
```

**Why floating-point was disallowed before C++20:**

The problem was **identity comparison**. Template arguments must be compared for **equality** to determine if two specializations are the same:

```cpp
// Is Array<0.1 + 0.2> the same as Array<0.3>?
// In IEEE 754: 0.1 + 0.2 == 0.30000000000000004, not 0.3
// This makes template identity ambiguous!

// Also: +0.0 == -0.0 in IEEE, but they have different bit patterns
// And: NaN != NaN, so template<NaN> would never match itself

// C++20 resolved this by defining float NTTP comparison as bitwise comparison,
// not IEEE ==. So +0.0 and -0.0 are DIFFERENT template arguments.
```

```cpp
// C++20: float NTTPs allowed
template <double D>
struct FloatTag {
    static constexpr double value = D;
};

FloatTag<3.14> a;     // OK in C++20
FloatTag<2.718> b;    // Different type from a
```

### Q3: Use a string literal as a NTTP in C++20 to create a fixed-name logging tag at compile time

You can't pass a raw string literal as an NTTP directly because `const char*` is a pointer (and pointer NTTPs have tricky identity rules). The standard workaround is a `FixedString` structural type that copies the characters into an array - then each unique string literal produces a distinct type at zero runtime cost:

```cpp
#include <iostream>
#include <cstring>
#include <source_location>

// === FixedString: a structural type that wraps a string literal ===
template <std::size_t N>
struct FixedString {
    char data[N]{};

    constexpr FixedString(const char (&str)[N]) {
        for (std::size_t i = 0; i < N; ++i)
            data[i] = str[i];
    }

    constexpr const char* c_str() const { return data; }
    constexpr std::size_t size() const { return N - 1; }  // exclude null terminator

    constexpr bool operator==(const FixedString&) const = default;
};

// Deduction guide: FixedString("hello") -> FixedString<6>
template <std::size_t N>
FixedString(const char (&)[N]) -> FixedString<N>;

// === Logger with compile-time tag ===
template <FixedString Tag>
struct Logger {
    static void info(const char* msg) {
        std::cout << "[" << Tag.c_str() << "] INFO: " << msg << "\n";
    }

    static void warn(const char* msg) {
        std::cout << "[" << Tag.c_str() << "] WARN: " << msg << "\n";
    }

    static void error(const char* msg) {
        std::cout << "[" << Tag.c_str() << "] ERROR: " << msg << "\n";
    }
};

// === Different tags = different types (zero runtime overhead) ===
using AppLog = Logger<"APP">;
using NetLog = Logger<"NET">;
using DbLog  = Logger<"DB">;

// === Compile-time string manipulation ===
template <FixedString A, FixedString B>
consteval bool same_tag() {
    return A == B;
}

int main() {
    std::cout << "=== String literal as NTTP ===\n";

    AppLog::info("Application started");
    NetLog::info("Listening on port 8080");
    DbLog::warn("Connection pool nearly full");
    AppLog::error("Unhandled exception");

    std::cout << "\n=== Compile-time tag comparison ===\n";
    static_assert(same_tag<"APP", "APP">());
    static_assert(!same_tag<"APP", "NET">());
    std::cout << "same_tag<APP,APP> = " << same_tag<"APP", "APP">() << "\n";
    std::cout << "same_tag<APP,NET> = " << same_tag<"APP", "NET">() << "\n";

    std::cout << "\n=== Tag properties ===\n";
    constexpr FixedString tag{"HELLO"};
    std::cout << "tag = " << tag.c_str() << ", len = " << tag.size() << "\n";

    return 0;
}
// Expected output:
//   [APP] INFO: Application started
//   [NET] INFO: Listening on port 8080
//   [DB] WARN: Connection pool nearly full
//   [APP] ERROR: Unhandled exception
//   same_tag<APP,APP> = 1
//   same_tag<APP,NET> = 0
//   tag = HELLO, len = 5
```

---

## Notes

- NTTPs parameterize templates on **values**: `template<int N>`, `template<auto V>`.
- C++20 adds **floating-point** and **structural class types** as NTTPs.
- **Structural type** = all public members, trivially copyable, no user-provided copy/move constructors.
- String literals can't be NTTPs directly - wrap them in a `FixedString` structural type.
- Float NTTPs use **bitwise comparison** (not IEEE `==`), so `+0.0` and `-0.0` are distinct.
- Each unique NTTP value creates a distinct template instantiation - use judiciously to avoid code bloat.
