# Test API usability with compile-fail tests using static_assert and SFINAE

**Category:** API and Library Design  
**Standard:** C++17, C++20  
**Reference:** <https://cmake.org/cmake/help/latest/prop_test/WILL_FAIL.html>  

---

## Topic Overview

Good APIs prevent misuse at compile time. **Compile-fail tests** verify that incorrect usage produces a compiler error - not a silent bug, not a runtime crash, but an immediate, caught-at-build-time failure. If your library says "Handle is non-copyable," you should have a test that proves it.

### static_assert Tests

The simplest form of compile-time verification is `static_assert` combined with type traits. You state a property your type must have, and the compiler checks it every time the file is compiled. Here's a class with deleted copy operations and the assertions that enforce its contract:

```cpp
#include <type_traits>
#include <memory>

class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
    NonCopyable(NonCopyable&&) = default;
    NonCopyable& operator=(NonCopyable&&) = default;
};

// Compile-time verification:
static_assert(!std::is_copy_constructible_v<NonCopyable>,
              "NonCopyable must not be copyable");
static_assert(std::is_move_constructible_v<NonCopyable>,
              "NonCopyable must be movable");
static_assert(std::is_nothrow_move_constructible_v<NonCopyable>,
              "NonCopyable move must be noexcept");
```

If someone later accidentally adds a copy constructor - maybe during a refactor - the `static_assert` fires immediately with a clear message. No test runner required; the compiler itself is the test framework.

### CMake Compile-Fail Tests

`static_assert` only tests properties you can query via type traits. Sometimes you need to verify that a particular expression outright fails to compile - for example, that `Handle h2 = h1;` is rejected by the compiler. For that you need a separate translation unit that you expect to fail, with CMake's `WILL_FAIL` property to invert the pass/fail logic:

```cmake
# CMakeLists.txt — test that certain code FAILS to compile

# Write a small source that should not compile
file(WRITE ${CMAKE_BINARY_DIR}/test_no_copy.cpp "
#include <mylib/handle.hpp>
int main() {
    mylib::Handle h1;
    mylib::Handle h2 = h1;  // Should FAIL — Handle is non-copyable
}
")

add_executable(test_no_copy EXCLUDE_FROM_ALL ${CMAKE_BINARY_DIR}/test_no_copy.cpp)
target_link_libraries(test_no_copy mylib)

add_test(NAME handle_no_copy
         COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target test_no_copy)
set_tests_properties(handle_no_copy PROPERTIES WILL_FAIL TRUE)
# Test PASSES if compilation FAILS
```

The `WILL_FAIL TRUE` property tells CTest that this test is expected to return a non-zero exit code. If compilation succeeds (meaning someone accidentally made `Handle` copyable), the test fails - which is exactly what you want.

### SFINAE-Based Trait Detection

Before C++20 concepts, SFINAE with `std::void_t` was the standard way to detect whether a type has a particular member or operation. It's more verbose than concepts, but it still works in C++17 and gives you a reusable boolean trait you can use in `static_assert` or as a template constraint:

```cpp
#include <type_traits>
#include <string>

// Detect if a type has a serialize() method
template<typename T, typename = void>
struct has_serialize : std::false_type {};

template<typename T>
struct has_serialize<T, std::void_t<decltype(std::declval<const T&>().serialize())>>
    : std::true_type {};

struct Good {
    std::string serialize() const { return "{}"; }
};

struct Bad {
    int value;
};

static_assert(has_serialize<Good>::value, "Good must be serializable");
static_assert(!has_serialize<Bad>::value, "Bad must NOT be serializable");

// C++20 concepts make this cleaner:
template<typename T>
concept Serializable20 = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

static_assert(Serializable20<Good>);
static_assert(!Serializable20<Bad>);
```

The C++20 `requires` expression does the same detection in a fraction of the code, and the error message when a constraint fails is far more readable. If you're on C++20, prefer concepts; if you're on C++17, the `void_t` pattern is the standard tool.

---

## Self-Assessment

### Q1: Why test that code FAILS to compile

A library that says "Handle is non-copyable" should verify this in CI. If someone accidentally adds a copy constructor, the compile-fail test catches it immediately. Without compile-fail tests, API guarantees exist only in documentation - they're promises you're not actually checking. With compile-fail tests, they're enforced contracts.

### Q2: Show concept-based compile-time diagnostics

Concepts let you name the constraint and get a clear error message pointing at the exact unsatisfied requirement:

```cpp
template<typename T>
concept HasNoThrowMove = std::is_nothrow_move_constructible_v<T>;

template<HasNoThrowMove T>
void store(std::vector<T>& vec, T&& item) {
    vec.push_back(std::move(item));
}

// Calling store with a type without noexcept move:
// error: constraints not satisfied [HasNoThrowMove<BadType>]
```

Compare this to the SFINAE equivalent, which would produce a wall of "no matching function for call to 'store'" with template instantiation backtraces that bury the actual reason. Concepts surface the failure at the point of the call with a name attached.

### Q3: How to test deleted overloads

Use `std::is_constructible_v` to verify which constructor overloads exist and which don't:

```cpp
static_assert(!std::is_constructible_v<Buffer, int>,
              "Buffer(int) should be deleted");
static_assert(!std::is_constructible_v<Buffer, double>,
              "Buffer(double) should be deleted");
static_assert(std::is_constructible_v<Buffer, size_t>,
              "Buffer(size_t) should work");
```

This pattern is especially useful when you've explicitly deleted conversions from signed or floating-point types to prevent accidental misuse - the assertions make sure those deletions actually stick.

---

## Notes

- Compile-fail tests are a form of **negative testing** - you're verifying that incorrect usage is rejected, not just that correct usage works.
- Use `static_assert` for same-translation-unit checks and CMake `WILL_FAIL` for separate-compilation checks that involve actual misuse expressions.
- C++20 concepts provide much better error messages than SFINAE when constraints are not met - if you're on C++20, prefer them for new code.
- Run compile-fail tests in CI to prevent accidental API contract violations from slipping through unnoticed.
