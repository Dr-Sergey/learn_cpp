# Understand constexpr variables and when they are evaluated at compile time

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md#constexpr>  

---

## Topic Overview

`constexpr` (C++11) declares that a variable's value can be computed at **compile time**.
The compiler evaluates it during compilation, not at runtime. The practical payoff is that
you can use the value anywhere a compile-time constant is required - template arguments,
array sizes, `static_assert` conditions - without any runtime overhead.

### Basic Usage

Once a variable is `constexpr`, you can plug it anywhere that demands a constant expression:

```cpp
constexpr int max_size = 100;        // computed at compile time
constexpr double pi = 3.14159265;    // computed at compile time

// Can use in contexts that REQUIRE compile-time values:
std::array<int, max_size> arr;       // template argument must be constexpr
static_assert(max_size == 100);      // static_assert is compile-time
int buffer[max_size];                // C array size must be constant
```

### constexpr Functions

A `constexpr` function can be called at compile time or at runtime, depending on whether
its arguments are known at compile time. The compiler chooses:

```cpp
constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

// Called at compile time:
constexpr int f5 = factorial(5);     // evaluated at compile time -> 120
static_assert(f5 == 120);           // verified at compile time

// Called at runtime (when argument is not constexpr):
int n;
std::cin >> n;
int fn = factorial(n);               // evaluated at runtime
```

The same function, two behaviors - which one you get depends entirely on the call site.

### constexpr vs const

People often mix these up. The key difference is the guarantee: `const` only promises
immutability; `constexpr` also promises compile-time initialization:

```cpp
const int a = 42;          // constant, but MAY be computed at runtime
constexpr int b = 42;      // MUST be computed at compile time

const int c = get_value();     // OK: runtime initialization of const
// constexpr int d = get_value(); // ERROR: get_value() is not constexpr

// Both are const (cannot modify after initialization):
// a = 10;  // ERROR
// b = 10;  // ERROR
```

### constexpr vs static

These are orthogonal concepts that are easy to conflate because both can appear in a
function body. Remember: `constexpr` is about *when* a value is known; `static` is about
*how long* storage lives:

```cpp
void example() {
    constexpr int x = 42;   // compile-time constant, may exist on stack
    static int y = 42;      // runtime initialization, persists across calls

    // constexpr is NOT static - it's a separate concept:
    // - constexpr = value known at compile time
    // - static = storage persists across function calls

    // A constexpr variable inside a function template is separate per instantiation:
    // template<int N>
    // void f() { constexpr int val = N * N; }
    // f<3>() has val=9, f<4>() has val=16 - separate per instantiation
}
```

### Forcing Compile-Time Evaluation

A `constexpr` function is *allowed* to run at compile time, but not forced to. Here's
how to guarantee it actually happens:

```cpp
// Method 1: Assign to constexpr variable
constexpr auto result = factorial(10); // guaranteed compile-time

// Method 2: Use static_assert
static_assert(factorial(10) == 3628800);

// Method 3: Use as template argument
std::array<int, factorial(5)> arr; // must be compile-time

// Method 4: Use consteval (C++20)
consteval int compile_only(int n) { return n * n; }
// compile_only() MUST be evaluated at compile time - error otherwise
```

### constexpr With Classes (C++11 and later)

`constexpr` extends to constructors and member functions too. Once a constructor is
`constexpr`, you can create objects of that type at compile time:

```cpp
struct Point {
    double x, y;
    constexpr Point(double x, double y) : x(x), y(y) {}
    constexpr double distance_squared() const { return x*x + y*y; }
};

constexpr Point origin{0.0, 0.0};
constexpr Point p{3.0, 4.0};
constexpr double d2 = p.distance_squared();  // 25.0, computed at compile time
static_assert(d2 == 25.0);
```

### Evolution Across Standards

`constexpr` functions started very restrictive and have been gradually relaxed with each
standard - what's impossible in C++11 is routine in C++20:

| Version | What constexpr functions can do |
| --- | --- |
| C++11 | Single return statement only |
| C++14 | Loops, local variables, multiple returns |
| C++17 | if constexpr, constexpr lambda |
| C++20 | Virtual calls, try-catch, dynamic_cast, new/delete, std::vector, std::string |
| C++23 | Even fewer restrictions |

---

## Self-Assessment

### Q1: Write a constexpr factorial and verify it's computed at compile time using static_assert

The `static_assert` calls are the proof: if the compiler can't evaluate `factorial` at
compile time, the `static_assert` won't compile. The template argument use is equally
definitive:

```cpp
#include <iostream>

constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

// Verify at compile time with static_assert:
static_assert(factorial(0) == 1);
static_assert(factorial(1) == 1);
static_assert(factorial(5) == 120);
static_assert(factorial(10) == 3628800);

// Use as template argument - MUST be compile-time:
template<int N>
struct FactorialResult {
    static constexpr int value = factorial(N);
};

static_assert(FactorialResult<6>::value == 720);

int main() {
    // Also works at runtime:
    int n;
    std::cout << "Enter n: ";
    std::cin >> n;
    std::cout << n << "! = " << factorial(n) << "\n"; // runtime evaluation

    // To prove compile-time evaluation, check the assembly:
    // constexpr values appear as immediate constants (e.g., mov eax, 120)
    // Use: g++ -O2 -S file.cpp && grep "120" file.s
    constexpr int f5 = factorial(5);
    std::cout << f5 << "\n"; // 120 - embedded in binary as literal
}
```

### Q2: Explain why a constexpr variable inside a function is not the same as a static variable

Here's the key distinction shown in code. Notice especially that you can combine both
keywords when you need a compile-time constant with a stable address:

```cpp
#include <iostream>

void demo_constexpr() {
    constexpr int x = 42;
    // x is a compile-time constant
    // It does NOT persist across function calls
    // It may be placed on the stack or optimized away entirely
    // Every call to demo_constexpr() "re-creates" x (conceptually)
}

void demo_static() {
    static int y = compute_once();
    // y is initialized ONCE (first call), persists in static storage
    // Subsequent calls see the same y
    // y is computed at RUNTIME (unless compute_once is constexpr)
    y++;
    std::cout << y << "\n";
}

// Key difference:
// - constexpr -> compile-time value, no runtime state
// - static    -> runtime storage, persists across calls

// They can be combined:
void demo_both() {
    static constexpr int z = 42;
    // z is a compile-time constant AND lives in static storage
    // This is useful when you need the address of a compile-time constant
}

// In a template, constexpr is per-instantiation:
template<int N>
constexpr int squared = N * N;
// squared<3> == 9, squared<4> == 16 - different values

int main() {
    demo_static(); // prints 1
    demo_static(); // prints 2 (y persists - static)
    demo_static(); // prints 3

    std::cout << squared<5> << "\n"; // 25
}
```

### Q3: Show a case where constexpr falls back to runtime evaluation and how to prevent it

A `constexpr` function silently falls back to runtime evaluation when called with a
non-constant argument. Here's the problem, and three ways to prevent it:

```cpp
#include <iostream>
#include <array>

constexpr int compute(int n) {
    return n * n + 1;
}

int main() {
    // Compile-time evaluation (argument is constexpr):
    constexpr int a = compute(5);  // definitely compile-time
    static_assert(a == 26);

    // Runtime fallback (argument is NOT constexpr):
    int n;
    std::cin >> n;
    int b = compute(n);  // runtime evaluation - n is unknown at compile time

    // How to PREVENT runtime fallback:

    // Method 1: consteval (C++20) - FORCES compile-time evaluation
    // consteval int must_be_compiletime(int n) { return n * n; }
    // consteval functions cannot be called with runtime values

    // Method 2: Assign to constexpr variable - error if can't be compile-time
    // constexpr int c = compute(n); // ERROR: n is not a constant expression

    // Method 3: Use in a template argument
    // std::array<int, compute(n)> arr; // ERROR: n is not constant

    // C++20 consteval example:
    consteval int forced(int n) { return n * n; }
    constexpr int d = forced(5);  // OK - 5 is constant
    // int e = forced(n);          // ERROR - n is not constant
    //                              // consteval NEVER falls back to runtime

    std::cout << a << " " << b << "\n";
}
```

---

## Notes

- `constexpr` implies `const` for variables but NOT for functions (a `constexpr` function can be called at runtime).
- `consteval` (C++20) is `constexpr` that **never** falls back to runtime.
- `constinit` (C++20) guarantees **static initialization** but does not make the variable const.
- In debug builds (`-O0`), `constexpr` functions may be evaluated at runtime unless forced by `static_assert` or template arguments.
- `std::is_constant_evaluated()` (C++20) / `if consteval` (C++23) lets a function branch on whether it's being evaluated at compile time.
