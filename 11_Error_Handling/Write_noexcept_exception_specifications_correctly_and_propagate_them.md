# Write noexcept exception specifications correctly and propagate them

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/noexcept_spec>  

---

## Topic Overview

`noexcept` is both a **specifier** (declaring that a function won't throw) and an **operator** (testing whether an expression can throw). Correct use of `noexcept` is critical for performance (enables optimizations) and correctness (move operations in containers). Conditional `noexcept` allows propagating the noexcept guarantee from the types you depend on.

### `noexcept` Specifier

```cpp
void always_noexcept() noexcept;          // unconditionally noexcept
void conditional() noexcept(true);         // same as noexcept
void may_throw() noexcept(false);          // same as no annotation
void computed() noexcept(sizeof(int) > 2); // compile-time boolean
```

### `noexcept` Operator

The `noexcept(expr)` form is an operator that evaluates at compile time to `true` or `false` - it does not execute `expr`. You use it inside a conditional specifier to say "this function is noexcept if and only if this other operation is noexcept":

```cpp
// noexcept(expr) evaluates to true/false at compile time
static_assert(noexcept(42));            // literals never throw
static_assert(!noexcept(new int));      // new can throw

// Use in conditional noexcept:
template <typename T>
void wrapper(T& x) noexcept(noexcept(x.do_work())) {
    x.do_work();  // noexcept iff x.do_work() is noexcept
}
```

### When to Mark `noexcept`

| Function Type | Should be `noexcept`? | Why |
| --- | --- | --- |
| Destructors | Always (implicit since C++11) | Throwing destructor -> `terminate()` during unwinding |
| Move constructor | Yes, if possible | `vector::push_back` falls back to copy if move isn't `noexcept` |
| Move assignment | Yes, if possible | Same as above - affects container reallocation |
| `swap()` | Always | Used in copy-and-swap idiom - must not fail |
| `size()`, `empty()`, `data()` | Yes | Simple accessors that can't fail |
| Leaf functions with no allocation | Usually | Enables better codegen |

### What Happens When `noexcept` Is Violated

This is the part that surprises people. A `noexcept` violation does not simply convert the thrown exception into a different kind of exception - it calls `std::terminate()` directly, and the stack may not even be unwound. Destructors for local variables are not guaranteed to run:

```cpp
void f() noexcept {
    throw std::runtime_error("oops");
}
// -> std::terminate() is called IMMEDIATELY
// -> Stack is NOT necessarily unwound (implementation-defined)
// -> Destructors may NOT run (unlike normal exception propagation)
```

---

## Self-Assessment

### Q1: Add conditional `noexcept`: `noexcept(noexcept(T{std::move(v)}))` to a wrapper

**Solution - Conditional noexcept Propagation:**

Conditional `noexcept` is how you write generic code that correctly propagates the noexcept-ness of whatever type `T` you are wrapping. Without it, you would either unconditionally mark everything `noexcept` (wrong - might be a lie for some `T`) or mark nothing (wrong - loses the optimization for types that do have noexcept moves):

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <string>
#include <vector>

// A generic wrapper that propagates noexcept from T's operations

template <typename T>
class Wrapper {
    T value_;

public:
    // Conditional noexcept: noexcept iff T's move constructor is noexcept
    Wrapper(T&& v) noexcept(noexcept(T{std::move(v)}))
        : value_(std::move(v))
    {
        std::cout << "Wrapper constructed (noexcept="
                  << noexcept(T{std::move(v)}) << ")\n";
    }

    // Conditional noexcept on move constructor
    Wrapper(Wrapper&& other)
        noexcept(std::is_nothrow_move_constructible_v<T>)
        : value_(std::move(other.value_))
    {}

    // Conditional noexcept on move assignment
    Wrapper& operator=(Wrapper&& other)
        noexcept(std::is_nothrow_move_assignable_v<T>)
    {
        value_ = std::move(other.value_);
        return *this;
    }

    // Conditional noexcept on swap
    void swap(Wrapper& other)
        noexcept(std::is_nothrow_swappable_v<T>)
    {
        using std::swap;
        swap(value_, other.value_);
    }

    // Accessor - always noexcept
    const T& get() const noexcept { return value_; }
};

// Type with noexcept move
struct LightType {
    int x;
    LightType(LightType&&) noexcept = default;
};

// Type with potentially-throwing move
struct HeavyType {
    std::string data;
    HeavyType(HeavyType&& other) /* NOT noexcept */ : data(std::move(other.data)) {
        // Some implementations may not mark string move as noexcept
    }
};

int main() {
    // Check propagation at compile time:
    static_assert(std::is_nothrow_move_constructible_v<Wrapper<int>>);
    static_assert(std::is_nothrow_move_constructible_v<Wrapper<LightType>>);

    Wrapper<int> w1(42);
    std::cout << "w1.get() = " << w1.get() << "\n";

    Wrapper<std::string> w2(std::string("hello"));
    std::cout << "w2.get() = " << w2.get() << "\n";

    // Demonstrate the noexcept operator:
    std::cout << "\nnoexcept checks:\n";
    std::cout << "  int move: " << std::is_nothrow_move_constructible_v<Wrapper<int>> << "\n";
    std::cout << "  string move: " << std::is_nothrow_move_constructible_v<Wrapper<std::string>> << "\n";
    std::cout << "  vector move: " << std::is_nothrow_move_constructible_v<Wrapper<std::vector<int>>> << "\n";
}
// Expected output:
//   Wrapper constructed (noexcept=1)
//   w1.get() = 42
//   Wrapper constructed (noexcept=1)
//   w2.get() = hello
//
//   noexcept checks:
//     int move: 1
//     string move: 1
//     vector move: 1
```

**Common Conditional noexcept Patterns:**

```cpp
// Pattern 1: Forward noexcept from sub-expression
template <typename T>
void f(T& x) noexcept(noexcept(x.operation())) {
    x.operation();
}

// Pattern 2: Use type traits
template <typename T>
T copy(const T& x) noexcept(std::is_nothrow_copy_constructible_v<T>) {
    return x;
}

// Pattern 3: Combine conditions
template <typename T, typename U>
void assign(T& dst, U&& src)
    noexcept(std::is_nothrow_assignable_v<T&, U&&>)
{
    dst = std::forward<U>(src);
}
```

---

### Q2: Show that `std::vector<T>::push_back` behavior changes based on T's move constructor `noexcept`

**Solution - vector Reallocation Strategy:**

This is one of the most practically important consequences of `noexcept`. When `std::vector` needs to grow its buffer, it has to move all existing elements to the new location. If the move constructor is `noexcept`, it can do so safely. If it might throw, a half-moved buffer would be unrecoverable - so it falls back to copying, which is much slower but maintains the strong guarantee:

```cpp
#include <iostream>
#include <vector>
#include <string>

// Type with noexcept move -> vector uses MOVE on reallocation
struct Safe {
    int id;
    Safe(int i) : id(i) {}
    Safe(const Safe& other) : id(other.id) {
        std::cout << "    Safe COPY " << id << "\n";
    }
    Safe(Safe&& other) noexcept : id(other.id) {
        other.id = -1;
        std::cout << "    Safe MOVE " << id << "\n";
    }
};

// Type WITHOUT noexcept move -> vector falls back to COPY
struct Risky {
    int id;
    Risky(int i) : id(i) {}
    Risky(const Risky& other) : id(other.id) {
        std::cout << "    Risky COPY " << id << "\n";
    }
    Risky(Risky&& other) /* NOT noexcept */ : id(other.id) {
        other.id = -1;
        std::cout << "    Risky MOVE " << id << "\n";
    }
};

int main() {
    std::cout << "=== Safe (noexcept move) ===\n";
    {
        std::vector<Safe> v;
        v.reserve(2);  // initial capacity: 2
        v.emplace_back(1);
        v.emplace_back(2);
        std::cout << "  push_back triggers reallocation:\n";
        v.emplace_back(3);  // reallocation! -> MOVES elements 1,2
    }

    std::cout << "\n=== Risky (throwing move) ===\n";
    {
        std::vector<Risky> v;
        v.reserve(2);
        v.emplace_back(1);
        v.emplace_back(2);
        std::cout << "  push_back triggers reallocation:\n";
        v.emplace_back(3);  // reallocation! -> COPIES elements 1,2
    }
}
// Expected output:
//   === Safe (noexcept move) ===
//     push_back triggers reallocation:
//     Safe MOVE 1
//     Safe MOVE 2
//
//   === Risky (throwing move) ===
//     push_back triggers reallocation:
//     Risky COPY 1
//     Risky COPY 2
```

**Why vector needs `noexcept` move:**

```cpp
Reallocation with MOVE (noexcept):
  Old buffer: [A, B, C]
  New buffer: [A', B', <- C'...]
  If move of C fails -> too late! A,B already moved (destroyed in old buffer)
  With noexcept: guaranteed no failure -> safe to move

Reallocation with COPY (potentially-throwing):
  Old buffer: [A, B, C]     <- still intact
  New buffer: [A', B', <- C'...]
  If copy of C fails -> destroy A',B' -> old buffer still intact -> strong guarantee!
```

**Performance Impact:**

```cpp
vector<T> with 1M elements, reallocation:
  T has noexcept move: ~1M pointer-swap moves     -> fast
  T has throwing move:  ~1M deep copies            -> slow

  Adding noexcept to your move constructor can be a 10-100x speedup
  for vector operations involving reallocation.
```

---

### Q3: Explain why marking a function `noexcept` when it can throw causes `std::terminate`

**Solution - `noexcept` Violation and Its Consequences:**

The danger of incorrectly marking a function `noexcept` is that you are making a promise the runtime enforces by terminating the program, not by catching the exception. There is no recovery. The code below illustrates this and shows the safe alternatives:

```cpp
#include <iostream>
#include <exception>
#include <stdexcept>
#include <vector>
#include <string>

// DANGEROUS: noexcept on a function that CAN throw
int parse_int(const std::string& s) noexcept {
    // std::stoi can throw std::invalid_argument or std::out_of_range
    return std::stoi(s);  // BUG: throws from noexcept function
}

// What happens:
// 1. std::stoi("abc") throws std::invalid_argument
// 2. Exception propagation sees noexcept boundary
// 3. std::terminate() is called IMMEDIATELY
// 4. Stack may NOT be unwound (implementation-defined)
// 5. Destructors may NOT run!

// === Illustration of the danger ===
struct Logger {
    std::string name;
    Logger(const std::string& n) : name(n) {
        std::cout << "  Logger '" << name << "' created\n";
    }
    ~Logger() {
        std::cout << "  Logger '" << name << "' destroyed\n";
    }
};

void bad_noexcept() noexcept {
    Logger log("inside bad_noexcept");  // destructor may NOT run!
    throw std::runtime_error("escaping noexcept");
}

// === How to check BEFORE calling ===
int safe_parse_int(const std::string& s) noexcept {
    try {
        return std::stoi(s);
    } catch (...) {
        return 0;  // fallback value
    }
    // Now it's TRULY noexcept - exceptions are handled internally
}

// === Conditional noexcept avoids the problem ===
template <typename F>
auto invoke_safely(F&& f) noexcept(noexcept(f())) -> decltype(f()) {
    return f();
    // noexcept only if f() is noexcept - no false promises
}

int main() {
    // Safe version works:
    std::cout << "safe_parse_int(\"42\") = " << safe_parse_int("42") << "\n";
    std::cout << "safe_parse_int(\"abc\") = " << safe_parse_int("abc") << "\n";

    // Demonstrate noexcept operator:
    auto lambda_nothrow = []() noexcept { return 42; };
    auto lambda_throw = []() { throw 42; };

    std::cout << "\nnoexcept checks:\n";
    std::cout << "  nothrow lambda: " << noexcept(lambda_nothrow()) << "\n";
    std::cout << "  throwing lambda: " << noexcept(lambda_throw()) << "\n";

    // WARNING: Uncommenting the next line would call std::terminate():
    // bad_noexcept();
    // parse_int("not a number");
}
// Expected output:
//   safe_parse_int("42") = 42
//   safe_parse_int("abc") = 0
//
//   noexcept checks:
//     nothrow lambda: 1
//     throwing lambda: 0
```

**The `noexcept` Contract:**

```cpp
void f() noexcept;  // PROMISE: "I will never throw"

If violated:
  +------------------------------------------------+
  | 1. Exception thrown inside noexcept function   |
  | 2. Runtime calls std::terminate()              |
  | 3. Stack unwinding is implementation-defined   |
  |    (may NOT happen -> destructors may NOT run!)|
  | 4. terminate_handler executes                  |
  | 5. std::abort() -> process dies                |
  +------------------------------------------------+

This is WORSE than a normal uncaught exception because:

  - Normal uncaught: stack IS unwound, destructors DO run
  - noexcept violation: stack unwinding NOT guaranteed
```

**Guidelines:**

```cpp
Mark noexcept:                  Do NOT mark noexcept:
  Destructors                     Functions calling operator new
  Move constructors (if possible) Functions calling virtual methods
  Swap functions                  Functions calling unknown callbacks
  Simple getters/accessors        Functions parsing strings
  Comparison operators            Functions doing I/O
  Hash functions                  Template functions (use conditional)
```

---

## Notes

- **`noexcept` enables compiler optimizations:** The compiler can skip generating stack-unwinding code for noexcept functions, leading to smaller and faster code.
- **`noexcept` is part of the type system (C++17):** `void(*)() noexcept` and `void(*)()` are different types. A noexcept function pointer can bind to a regular function pointer slot, but not vice versa.
- **Implicit `noexcept`:** Destructors are implicitly `noexcept` since C++11. Deallocation functions (`operator delete`) are also implicitly `noexcept`.
- **`noexcept(auto)` (C++26 proposal):** Would automatically deduce noexcept from the function body - reducing boilerplate.
- **`std::move_if_noexcept(x)`** returns `T&&` if T's move is noexcept, else `const T&` - used internally by `std::vector`.
- **Testing noexcept:** Use `static_assert(noexcept(expr))` to verify at compile time. Use `std::is_nothrow_move_constructible_v<T>` for type traits.
