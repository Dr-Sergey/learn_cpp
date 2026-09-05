# Use std::as_const and const_cast safely

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/as_const>  

---

## Topic Overview

`std::as_const(x)` returns a `const` reference to `x`, letting you force a const overload without modifying the variable. It is the clean, self-documenting way to say "treat this as read-only right here." `const_cast` does the opposite - it removes const - and it is safe only when the original object was non-const to begin with.

```cpp
std::string s = "hello";
const std::string& cs = std::as_const(s);  // safe const view

// const_cast: safe ONLY if original was non-const
std::string& back = const_cast<std::string&>(cs);  // OK: s is non-const
```

---

## Self-Assessment

### Q1: Use `std::as_const` to call a const overload

When a class has both a const and a non-const overload of a member function, calling through a non-const object always picks the non-const version - even if you just want to read. `std::as_const` gives you a way to explicitly request the const overload without making the variable itself const.

```cpp
#include <iostream>
#include <string>
#include <utility>  // std::as_const
#include <vector>

class Container {
    std::vector<int> data_ = {10, 20, 30};
public:
    // Non-const: returns mutable reference
    std::vector<int>& items() {
        std::cout << "[mutable overload]\n";
        return data_;
    }

    // Const: returns const reference
    const std::vector<int>& items() const {
        std::cout << "[const overload]\n";
        return data_;
    }
};

int main() {
    Container c;

    // Without as_const: calls non-const overload
    auto& v1 = c.items();         // mutable overload

    // With as_const: forces const overload
    auto& v2 = std::as_const(c).items();  // const overload

    // Useful in range-for to avoid copying:
    for (const auto& item : std::as_const(c).items()) {
        std::cout << item << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// [mutable overload]
// [const overload]
// [const overload]
// 10 20 30
```

The `std::as_const` call in the range-for is especially useful: without it, iterating over `c.items()` would call the non-const overload and return a reference you could accidentally modify. Wrapping it in `std::as_const` makes your intent - read-only iteration - visible right at the call site.

### Q2: When is `const_cast` safe vs UB

This is one of those cases where the rule sounds simple but trips people up in practice. The critical distinction is not whether the pointer or reference is `const` - it is whether the **original object** was declared `const`. You can safely cast away const on a const reference to a non-const object, because the object's underlying storage is writable. But if the object itself was declared `const`, modifying it through a `const_cast` is undefined behavior, regardless of what the type system says the pointer is.

```cpp
#include <iostream>
#include <cstring>

// SAFE: casting away const on a non-const original
void safe_example() {
    int x = 42;                       // x is NON-const
    const int& cref = x;              // const reference to non-const object
    int& ref = const_cast<int&>(cref); // SAFE: x was never const
    ref = 100;                         // OK: modifying non-const object
    std::cout << "Safe: " << x << '\n';
}

// UNDEFINED BEHAVIOR: casting away const on a truly const object
void unsafe_example() {
    const int x = 42;                  // x IS const
    int& ref = const_cast<int&>(x);    // technically compiles...
    // ref = 100;                       // UB! x lives in read-only memory
    // Compiler may have inlined x=42 everywhere
    std::cout << "UB avoided (didn't modify)\n";
}

// COMMON LEGITIMATE USE: calling non-const API with const data
void legacy_api(char* buf) {  // old C API, doesn't modify buf
    std::cout << "Legacy: " << buf << '\n';
}

void call_legacy() {
    const char* msg = "hello";
    // Safe IF we know legacy_api doesn't modify the buffer
    legacy_api(const_cast<char*>(msg));
}

int main() {
    safe_example();
    unsafe_example();
    call_legacy();
}
// Expected output:
// Safe: 100
// UB avoided (didn't modify)
// Legacy: hello
```

The reason the UB case is so dangerous is that compilers use `const` declarations to propagate constant values into the generated code. If you tell the compiler `x` is `const int`, it may emit `42` directly as an immediate value everywhere `x` is used and never generate a memory load at all. Modifying `x` through a cast has no effect on those already-embedded constants.

| Scenario | Safe? |
| --- | --- |
| Cast away const, original was non-const | Yes |
| Cast away const, original was const | UB if modified |
| Cast away const to call legacy C API | Safe if API doesn't modify |
| Adding const (const_cast<const T&>) | Always safe |

### Q3: Show a design flaw where `const_cast` hints at a needed refactor

When you find yourself reaching for `const_cast<SomeClass*>(this)` inside a `const` member function, that is almost always a signal that the design needs work. The right solution is almost always to mark the member `mutable` - which is C++'s way of saying "this data is not part of the logical value of the object."

```cpp
#include <iostream>
#include <string>
#include <unordered_map>

// BAD DESIGN: needs const_cast because interface is wrong
class BadCache {
    mutable std::unordered_map<std::string, int> cache_;  // should be mutable!
    int compute(const std::string& key) const { return key.length() * 42; }
public:
    // Returns cached value, but lookup modifies cache... oops
    int get(const std::string& key) const {
        // Developer uses const_cast because get() is const but cache_ needs mutation
        auto* self = const_cast<BadCache*>(this);
        if (auto it = self->cache_.find(key); it != self->cache_.end())
            return it->second;
        int val = compute(key);
        self->cache_[key] = val;  // modifying through const_cast!
        return val;
    }
};

// GOOD DESIGN: use mutable keyword instead
class GoodCache {
    mutable std::unordered_map<std::string, int> cache_;  // mutable = logical const
    int compute(const std::string& key) const { return key.length() * 42; }
public:
    int get(const std::string& key) const {
        if (auto it = cache_.find(key); it != cache_.end())
            return it->second;
        int val = compute(key);
        cache_[key] = val;  // OK: mutable member
        return val;
    }
};

int main() {
    GoodCache cache;
    std::cout << cache.get("hello") << '\n';  // 210
    std::cout << cache.get("hello") << '\n';  // 210 (cached)
    std::cout << cache.get("world") << '\n';  // 210
}
// Expected output:
// 210
// 210
// 210
```

`mutable` is not a hack - it is the intended tool for exactly this pattern. A cache, a mutex, a lazy-initialized value, a debug counter: these are all things that change without affecting the observable value of the object, and `mutable` communicates that intent clearly.

**When you see `const_cast`, ask:**

1. Should the member be `mutable`? (caches, mutexes, debug counters)
2. Should the function be non-const? (it modifies state!)
3. Is it a legacy API mismatch? (document and isolate)

---

## Notes

- `std::as_const` is in `<utility>` since C++17.
- Prefer `std::as_const` over `static_cast<const T&>(x)` for readability.
- `const_cast` is the only cast that can remove const/volatile.
- If you need `const_cast` on `this`, use `mutable` members instead.
- `const_cast` to add const is always safe and sometimes useful with templates.
