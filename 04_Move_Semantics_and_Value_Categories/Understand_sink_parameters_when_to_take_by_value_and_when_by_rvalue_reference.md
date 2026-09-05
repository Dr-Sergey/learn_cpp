# Understand Sink Parameters: When to Take by Value and When by Rvalue Reference

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/value_category>  

---

## Topic Overview

### The Sink Parameter Decision Tree

When a function needs to store or consume its argument, the choice of parameter type actually matters for performance. The decision is not arbitrary - it follows directly from the type's move cost and how the function is used. Here is the reasoning as a decision tree:

```cpp
Want to store/consume the argument?
├─ YES (sink parameter):
│   ├─ Type is move-only (unique_ptr)? -> Take by T&&
│   ├─ Type has cheap move (string, vector)? -> Take by value + move
│   ├─ Hot path / tight loop setter? -> Two overloads (const T& + T&&)
│   └─ Large trivially-copyable? -> Take by const T&
└─ NO (just read):
    └─ Take by const T& or std::string_view/std::span
```

### Quick Reference

If the table feels like a lot, the rule of thumb is: by-value for most things, rvalue reference for move-only types, const reference when move is as expensive as copy.

| Scenario | Signature | Why |
| --- | --- | --- |
| Constructor storing string | `f(std::string s)` | Simple, handles both lvalue/rvalue |
| Setter called in loop | `f(const std::string&)` | Reuses existing buffer |
| unique_ptr transfer | `f(std::unique_ptr<T>&&)` | Can't copy, signals ownership transfer |
| Generic library code | `template<class T> f(T&&)` | Perfect forwarding |

---

## Self-Assessment

### Q1: Explain why `void setName(std::string name) { name_ = std::move(name); }` is the preferred sink idiom

By-value sink works because the parameter itself becomes the "staging area." The caller either copies into it (lvalue case) or moves into it (rvalue case), and then the function body moves from it once more into the member. That second move is cheap - it is just a pointer swap for `std::string`. The net result is near-optimal with a single function:

```cpp
#include <iostream>
#include <string>

class Employee {
    std::string name_;
    std::string department_;
public:
    // THE PREFERRED SINK IDIOM: take by value, move into member
    Employee(std::string name, std::string dept)
        : name_(std::move(name))
        , department_(std::move(dept))
    {}

    // Works optimally for both lvalues and rvalues:
    void setName(std::string name) {
        name_ = std::move(name);
    }

    const std::string& name() const { return name_; }
};

int main() {
    std::cout << "=== By-Value Sink Idiom ===\n\n";

    // Rvalue: string constructed in-place -> moved once into param -> moved into member
    // Total: ~2 moves (nearly free for string)
    std::cout << "1. Rvalue argument:\n";
    Employee e1("Alice", "Engineering");
    std::cout << "  " << e1.name() << "\n";

    // Lvalue: string copied into param -> moved into member
    // Total: 1 copy + 1 move (copy is necessary, move is nearly free)
    std::cout << "\n2. Lvalue argument:\n";
    std::string name = "Bob";
    Employee e2(name, std::string("Sales"));
    std::cout << "  " << e2.name() << ", original: \"" << name << "\"\n";

    // Why this is PREFERRED:
    std::cout << "\n=== Why this works ===\n";
    std::cout << "  1. Single function handles both lvalues and rvalues\n";
    std::cout << "  2. No template complexity, no overload explosion\n";
    std::cout << "  3. The 'extra' move is O(1) -- typically ~5ns\n";
    std::cout << "  4. Caller's intent is clear: function CONSUMES the value\n";
    std::cout << "  5. Works with implicit conversions: setName(\"literal\")\n";

    // setName with implicit conversion from const char*
    std::cout << "\n3. Implicit conversion (const char* -> string):\n";
    e1.setName("Charlie");  // Constructs string from literal -> moves into member
    std::cout << "  " << e1.name() << "\n";

    return 0;
}
```

Point 5 in the output is easy to overlook but practically important: taking by value means implicit conversions work for free. `setName("Charlie")` constructs a `std::string` from the string literal and then moves it in - no extra overload required.

### Q2: Show when two overloads (`const T&` + `T&&`) are better than by-value sink

The two-overload approach wins when the target member already has allocated capacity and you can reuse it on the lvalue path. With a single `const std::string&` overload, assigning directly into the member can skip allocation entirely if the capacity is large enough. The by-value path always brings a freshly-allocated copy and discards the existing buffer:

```cpp
#include <iostream>
#include <string>
#include <chrono>

class MessageLog {
    std::string last_message_;
    size_t log_count_ = 0;
public:
    // TWO OVERLOADS: optimal for repeated calls
    void log_optimal(const std::string& msg) {
        last_message_ = msg;  // Can REUSE last_message_'s buffer!
        ++log_count_;
    }
    void log_optimal(std::string&& msg) {
        last_message_ = std::move(msg);
        ++log_count_;
    }

    // BY-VALUE: simpler but wastes buffer reuse opportunity
    void log_byval(std::string msg) {
        last_message_ = std::move(msg);  // Always discards old buffer
        ++log_count_;
    }

    size_t count() const { return log_count_; }
};

int main() {
    std::cout << "=== When Two Overloads Win ===\n\n";

    constexpr int N = 1'000'000;
    std::string msg(100, 'x');  // 100-char message

    // Benchmark: repeated lvalue calls (the case where overloads win)
    MessageLog log;

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        log.log_optimal(msg);  // Reuses buffer after first call
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < N; ++i) {
        log.log_byval(msg);  // Copies string each time, wastes old buffer
    }
    auto t3 = std::chrono::high_resolution_clock::now();

    auto opt_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    auto val_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t3 - t2).count();

    std::cout << "Repeated lvalue calls (" << N << " iterations):\n";
    std::cout << "  Two overloads: " << opt_ms << " ms\n";
    std::cout << "  By-value:      " << val_ms << " ms\n";

    std::cout << "\n=== When to use two overloads ===\n";
    std::cout << "  1. Setter called in tight loops with lvalue args\n";
    std::cout << "  2. Target member can reuse allocated capacity\n";
    std::cout << "  3. Performance-critical path (measured, not guessed)\n";
    std::cout << "\n=== When by-value is fine ===\n";
    std::cout << "  1. Constructors (no existing buffer to reuse)\n";
    std::cout << "  2. Rarely-called setters\n";
    std::cout << "  3. Code clarity is more important than micro-perf\n";

    return 0;
}
```

The benchmark is reproducible on your machine - try it. The gap between the two approaches depends on the string size and allocation behavior of your standard library implementation, but for repeated calls with an existing buffer, `log_optimal` will typically win by a measurable margin.

### Q3: Benchmark sink-by-value vs sink-by-rvalue-reference for a type with expensive move

This example targets a different failure mode: a type where `move` and `copy` cost the same because there is no heap buffer to steal. `std::array<double, 1024>` is 8 KB of stack data - moving it memcpys all 8 KB just like copying does. Taking it by value in a sink therefore pays two memcpys where one would do:

```cpp
#include <iostream>
#include <chrono>
#include <array>
#include <utility>
#include <algorithm>

// A type where move is EXPENSIVE (not just pointer-swap)
struct BigMatrix {
    std::array<double, 1024> data{};  // 8 KB -- trivially copyable

    // Move == Copy for trivially copyable types!
    // No pointers to steal -- must memcpy the entire array
    BigMatrix() = default;
    BigMatrix(const BigMatrix&) = default;
    BigMatrix(BigMatrix&&) = default;
    BigMatrix& operator=(const BigMatrix&) = default;
    BigMatrix& operator=(BigMatrix&&) = default;
};

class Solver {
    BigMatrix matrix_;
public:
    // By-value: lvalue -> copy into param + move(=copy) into member = 2 copies
    void set_byval(BigMatrix m) {
        matrix_ = std::move(m);  // move == memcpy for BigMatrix
    }

    // By-rvalue-ref: 1 move(=copy) directly
    void set_rref(BigMatrix&& m) {
        matrix_ = std::move(m);
    }

    // By-const-ref: 1 copy directly (best for lvalues!)
    void set_cref(const BigMatrix& m) {
        matrix_ = m;
    }
};

int main() {
    std::cout << "=== Expensive Move: By-Value vs Rvalue-Ref ===\n\n";

    constexpr int N = 100'000;
    BigMatrix src;
    src.data[0] = 3.14;

    Solver s;

    // Benchmark lvalue path
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        s.set_cref(src);  // 1 copy
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < N; ++i) {
        s.set_byval(src);  // 1 copy + 1 move(=copy) = 2 copies
    }
    auto t3 = std::chrono::high_resolution_clock::now();

    auto cref_ms = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();
    auto byval_ms = std::chrono::duration_cast<std::chrono::microseconds>(t3 - t2).count();

    std::cout << "sizeof(BigMatrix) = " << sizeof(BigMatrix) << " bytes\n\n";
    std::cout << "Lvalue path (" << N << " calls):\n";
    std::cout << "  const-ref:  " << cref_ms << " µs (1 copy)\n";
    std::cout << "  by-value:   " << byval_ms << " µs (2 copies!)\n";
    std::cout << "  Ratio: ~" << (byval_ms > 0 ? byval_ms / std::max(cref_ms, 1L) : 0) << "x\n";

    std::cout << "\n=== Conclusion ===\n";
    std::cout << "  For trivially-copyable types (move == copy):\n";
    std::cout << "    - By-value sink is ~2x slower for lvalues\n";
    std::cout << "    - Use const-ref for lvalues, rvalue-ref for rvalues\n";
    std::cout << "  For types with cheap move (string, vector, unique_ptr):\n";
    std::cout << "    - By-value sink overhead is negligible (~5ns extra)\n";
    std::cout << "    - By-value is the cleaner default\n";

    return 0;
}
```

The ratio printed at the end will be close to 2x for large trivially-copyable types on any modern machine. This is the empirical confirmation of the decision tree at the top: "large trivially-copyable? -> take by const T&".

---

## Notes

- **By-value sink** = simple default for constructors with cheap-move types (string, vector).
- **Rvalue reference** (`T&&`) = required for move-only types, signals "I will consume this".
- **Two overloads** = optimal when repeated setter calls benefit from buffer reuse.
- **Never take `unique_ptr` by value** - it's move-only, use `unique_ptr&&` to signal ownership transfer.
- For large trivially-copyable types, by-value sink doubles the copy cost - use `const T&`.
