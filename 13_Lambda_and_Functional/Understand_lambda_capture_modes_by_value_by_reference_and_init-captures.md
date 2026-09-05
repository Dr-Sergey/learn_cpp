# Understand lambda capture modes: by value, by reference, and init-captures

**Category:** Lambda and Functional  
**Standard:** C++11, C++14 (basic captures)  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

Lambda capture determines which outer variables a lambda can access and how. There are three fundamental modes, and it's worth understanding each one clearly before reaching for the convenient defaults like `[=]` or `[&]`:

| Capture | Syntax | Effect |
| --- | --- | --- |
| By value | `[x]` or `[=]` | Copies `x` into closure at capture time |
| By reference | `[&x]` or `[&]` | Stores reference to `x`; sees mutations |
| Init-capture (C++14) | `[y = expr]` | Creates a new member initialized from `expr` |

To make this concrete, here's a visual breakdown of a mixed capture list. Each element in the brackets does something distinct:

```cpp
// [x, &y, z = std::move(p)]
//  |   |    |
//  |   |    +- init-capture: moves p into z
//  |   +------ by reference: y is a ref
//  +---------- by value: x is a copy
```

### Capture Defaults

Rather than listing every variable individually, you can use a default that covers all the variables the lambda body actually uses:

| Syntax | Meaning |
| --- | --- |
| `[=]` | Capture all used variables by value |
| `[&]` | Capture all used variables by reference |
| `[=, &x]` | All by value except `x` by reference |
| `[&, x]` | All by reference except `x` by value |

**Danger:** `[&]` can silently capture locals that outlive the lambda - this leads to dangling references, and the compiler will not warn you.

---

## Self-Assessment

### Q1: Show a dangling reference bug in a lambda that captures a local variable by reference and is stored

This is probably the single most common lambda bug in real code. The issue is that `[&]` or `[&x]` only stores a reference - it doesn't keep the referred-to object alive. If the lambda outlives the local variable, you have undefined behavior.

```cpp
#include <iostream>
#include <functional>

std::function<int()> make_counter_buggy() {
    int count = 0;
    // BUG: captures 'count' by reference, but 'count' is destroyed
    // when make_counter_buggy() returns!
    return [&count]() {
        return ++count;  // DANGLING REFERENCE - UB!
    };
}

std::function<int()> make_counter_fixed() {
    int count = 0;
    // FIX: capture by VALUE (with mutable to allow modification)
    return [count]() mutable {
        return ++count;  // OK - count is a member of the closure
    };
}

int main() {
    // BUGGY version - undefined behavior
    auto bad = make_counter_buggy();
    // std::cout << bad() << "\n";  // UB! 'count' no longer exists
    std::cout << "bad counter: UB (don't call!)\n";

    // FIXED version - safe
    auto good = make_counter_fixed();
    std::cout << "good counter: " << good() << "\n";  // 1
    std::cout << "good counter: " << good() << "\n";  // 2
    std::cout << "good counter: " << good() << "\n";  // 3

    // Another common bug: capturing loop variable by reference
    std::vector<std::function<int()>> funcs;
    for (int i = 0; i < 3; ++i) {
        funcs.push_back([&i]() { return i; });  // BUG: all share same 'i'
    }
    // After loop: i == 3 (or destroyed)
    // funcs[0]() would return 3, not 0!

    // FIX: capture by value
    funcs.clear();
    for (int i = 0; i < 3; ++i) {
        funcs.push_back([i]() { return i; });  // each gets its own copy
    }
    for (const auto& f : funcs)
        std::cout << f() << " ";
    std::cout << "\n";
}
// Expected output:
//   bad counter: UB (don't call!)
//   good counter: 1
//   good counter: 2
//   good counter: 3
//   0 1 2
```

The loop version is especially tricky because `i` exists for the whole loop - it just ends up with the value `3` when the loop is done, so every lambda in the vector returns `3` instead of the individual values. Capturing by value gives each lambda its own independent snapshot.

---

### Q2: Use init-capture `[x = std::move(y)]` to move a `unique_ptr` into a lambda

The reason init-captures exist is specifically for move-only types. You can't copy a `unique_ptr`, and you can't capture a reference to a local and then return the lambda safely. The init-capture gives you a third option: move the object directly into the closure.

```cpp
#include <iostream>
#include <memory>
#include <functional>
#include <vector>

struct Resource {
    std::string name;
    Resource(std::string n) : name(std::move(n)) {
        std::cout << "Resource(" << name << ") created\n";
    }
    ~Resource() {
        std::cout << "Resource(" << name << ") destroyed\n";
    }
    void use() const { std::cout << "Using " << name << "\n"; }
};

int main() {
    auto res = std::make_unique<Resource>("GPU_Buffer");

    // Can't capture unique_ptr by value (no copy)!
    // auto bad = [res]() { res->use(); };  // ERROR: deleted copy ctor

    // Init-capture MOVES the unique_ptr into the lambda
    auto task = [r = std::move(res)]() {
        r->use();
    };

    // res is now nullptr - ownership transferred to lambda
    std::cout << "res is null: " << (res == nullptr) << "\n";

    task();  // lambda owns the resource

    // Practical use: store move-only lambda in a task queue
    // (requires std::function alternative or move-only wrapper)
    std::cout << "\n--- Init-capture with transform ---\n";
    std::vector<int> data = {1, 2, 3, 4, 5};
    int multiplier = 10;

    // Init-capture can rename and compute
    auto transform = [m = multiplier, base = data.size()](int x) {
        return x * m + static_cast<int>(base);
    };

    for (int x : data)
        std::cout << x << " -> " << transform(x) << "\n";
}
// Expected output:
//   Resource(GPU_Buffer) created
//   res is null: 1
//   Using GPU_Buffer
//
//   --- Init-capture with transform ---
//   1 -> 15
//   2 -> 25
//   3 -> 35
//   4 -> 45
//   5 -> 55
//   Resource(GPU_Buffer) destroyed
```

Notice that `base` in the init-capture doesn't need to be the same name as anything in the outer scope - you can rename and transform at capture time. This is often cleaner than capturing a reference just to read a single value.

---

### Q3: Explain the difference between `[=]` and `[this]` for capturing in member functions

This is where a lot of people get surprised. Inside a member function, `[=]` does NOT copy the member variables - it copies `this` (the pointer). So `[=]` and `[this]` have essentially the same dangling pointer risk. The C++17 `[*this]` capture is what you want when you need a true copy.

```cpp
#include <iostream>
#include <functional>
#include <string>

struct Widget {
    std::string name = "Widget";
    int counter = 0;

    std::function<void()> get_lambda_this() {
        // [this] captures the 'this' POINTER by value
        // The lambda holds a raw pointer to 'this'
        // If Widget is destroyed -> dangling pointer!
        return [this]() {
            std::cout << "[this] name=" << name
                      << " counter=" << ++counter << "\n";
        };
    }

    std::function<void()> get_lambda_eq() {
        // [=] in a member function captures 'this' by value (pointer copy!)
        // It does NOT copy the Widget members!
        // Same dangling risk as [this]!
        return [=]() {
            std::cout << "[=] name=" << name
                      << " counter=" << counter << "\n";
            // 'name' and 'counter' accessed through captured 'this' pointer
        };
    }

    std::function<void()> get_lambda_star_this() {
        // [*this] (C++17) captures the ENTIRE Widget BY VALUE
        // Safe even if original Widget is destroyed!
        return [*this]() mutable {
            std::cout << "[*this] name=" << name
                      << " counter=" << ++counter << "\n";
        };
    }
};

int main() {
    std::function<void()> f1, f2, f3;

    {
        Widget w;
        w.name = "Original";
        w.counter = 10;

        f1 = w.get_lambda_this();
        f2 = w.get_lambda_eq();
        f3 = w.get_lambda_star_this();

        f1();  // OK - w still alive
        f2();  // OK - w still alive
        f3();  // OK - has its own copy of w
    }
    // w is destroyed here!

    // f1();  // UB! 'this' pointer dangles!
    // f2();  // UB! also uses 'this' pointer!
    std::cout << "After Widget destroyed:\n";
    f3();     // Safe! Has its own copy
    f3();     // counter increments independently
}
// Expected output:
//   [this] name=Original counter=11
//   [=] name=Original counter=11
//   [*this] name=Original counter=11
//   After Widget destroyed:
//   [*this] name=Original counter=12
//   [*this] name=Original counter=13
```

The reason `[=]` surprises people is that it looks like "copy everything" but inside a member function it really means "copy the implicit `this` pointer" - because `name` and `counter` are accessed as `this->name` and `this->counter` under the hood. The `[*this]` syntax was added in C++17 specifically to address this confusion.

**Summary table:**

| Capture | What's captured | Copies Widget data? | Dangling risk? |
| --- | --- | --- | --- |
| `[this]` | `this` pointer | No | Yes - if Widget dies |
| `[=]` | `this` pointer (implicitly) | No | Yes - same as `[this]`! |
| `[*this]` (C++17) | Entire Widget object | Yes | No - independent copy |

> **C++20 deprecation warning:** `[=]` implicitly capturing `this` is deprecated. Use `[=, this]` or `[=, *this]` explicitly.

---

## Notes

- **`mutable` keyword:** By-value captures are `const` by default. Use `mutable` to modify them inside the lambda body.
- **Init-capture names:** `[x = expr]` creates a new name `x` in the closure - it doesn't need to match any outer variable.
- **`std::move` in init-capture:** `[v = std::move(vec)]` transfers ownership. The original is left in a moved-from state.
- **Pack capture (C++20):** `[...args = std::move(args)]` captures a parameter pack by move.
- **Reference lifetime:** A lambda captured by `[&]` never extends the lifetime of the referenced objects. Always ensure references outlive the lambda.
