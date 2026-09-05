# Understand Dangling Pointers vs Dangling References — UB in Both Cases

**Category:** Memory and Ownership  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/ub>  

---

## Topic Overview

### What Is Dangling

A **dangling pointer** or **dangling reference** refers to an object whose lifetime has ended. Using it is **undefined behavior (UB)** - the program may crash, return garbage, or appear to "work" (worst case). The "appears to work" outcome is the most dangerous one because you ship the code and the bug only surfaces later under different optimization settings or on a different machine.

### Common Sources

| Source | Pointer Example | Reference Example |
| --- | --- | --- |
| Returning local address | `return &local;` | `return local_ref;` |
| Use after `delete` | `delete p; *p;` | `auto& r = *p; delete p; r;` |
| Iterator invalidation | `vec.push_back(x); *old_it;` | Same |
| Temporary lifetime | `const char* p = string().c_str();` | `auto& r = get_temp().field;` |
| Scope exit | `{ int x; p = &x; }` | `{ int x; r = x; } // r is a reference` |

### Why Both Are Equally Dangerous

References feel "safer" because they can't be null, but that is a different guarantee from "always valid." A dangling reference is just as UB as a dangling pointer. The reason this matters so much is that the compiler **assumes no UB exists** and optimizes accordingly. If you dangle, the optimizer is free to produce completely unexpected code - it has not done anything wrong, you broke the contract first.

---

## Self-Assessment

### Q1: Show a dangling pointer from returning the address of a local variable

The example below deliberately creates several dangling situations. The BAD functions are shown alongside safe alternatives so the fix is always visible next to the problem.

```cpp
#include <iostream>

// BAD: returns address of local variable
int* get_value() {
    int x = 42;
    return &x;  // WARNING: address of local variable returned
}   // x destroyed here - pointer now dangles

// BAD: returning pointer into freed buffer
const char* get_greeting() {
    std::string s = "Hello, world!";
    return s.c_str();  // pointer into s's internal buffer
}   // s destroyed here - buffer freed

// GOOD: return by value
int get_value_safe() {
    int x = 42;
    return x;  // copy of value, safe
}

// GOOD: use static (lifetime extends to program end)
int* get_value_static() {
    static int x = 42;
    return &x;  // OK - static storage duration
}

int main() {
    // === Dangling pointer from local ===
    int* p = get_value();
    // p points to destroyed stack frame - accessing it is UB
    // The value might "appear" correct by luck (stack not yet overwritten)
    std::cout << "Dangling value (UB): " << *p << "\n";  // UNDEFINED BEHAVIOR

    // === Dangling into freed string buffer ===
    const char* greeting = get_greeting();
    // greeting points to freed heap memory
    std::cout << "Dangling string (UB): " << greeting << "\n";  // UNDEFINED BEHAVIOR

    // === Use-after-delete ===
    int* q = new int(99);
    delete q;
    // q is now a dangling pointer
    // std::cout << *q;  // UB - accessing freed memory

    // === Dangling from scope exit ===
    int* r = nullptr;
    {
        int local = 7;
        r = &local;
    }   // local destroyed
    // r now dangles - accessing *r is UB

    // === Safe alternatives ===
    int safe = get_value_safe();
    std::cout << "Safe value: " << safe << "\n";

    int* safe_static = get_value_static();
    std::cout << "Static value: " << *safe_static << "\n";

    return 0;
}
```

The scope-exit case at the bottom is the one that surprises students most often. `r` looks like it should be fine because you assigned it inside the block, but the object it points to ceases to exist the moment the closing brace is reached.

### Q2: Show a dangling reference from binding to a temporary

References have one lifetime-extension rule: binding a `const` reference directly to a temporary extends that temporary's lifetime to match the reference. But this only applies when the binding is direct. Any chained access - a member, a return value of a method - breaks the extension and you are left with a dangle.

```cpp
#include <iostream>
#include <string>
#include <vector>

// === Case 1: Reference to temporary subobject ===
struct Config {
    std::string name;
    int value;
};

Config get_config() {
    return {"timeout", 30};
}

// === Case 2: Reference from range-for over temporary ===
std::vector<int> get_numbers() {
    return {1, 2, 3, 4, 5};
}

// === Case 3: Reference to element of moved-from container ===
std::string& get_first(std::vector<std::string>& v) {
    return v[0];
}

int main() {
    // Case 1: DANGLING - reference to member of returned temporary
    // const std::string& name = get_config().name;
    // Config temporary is destroyed at end of full expression
    // name now dangles!

    // Fix: capture the whole object
    Config cfg = get_config();
    const std::string& name = cfg.name;  // OK - cfg lives
    std::cout << "Config: " << name << " = " << cfg.value << "\n";

    // Case 2: const ref extends lifetime of DIRECT temporary
    const int& r = 42;  // OK - lifetime extended
    std::cout << "Extended temporary: " << r << "\n";

    // But NOT for chained accesses:
    // const std::string& bad = std::string("hello").substr(0, 3);
    // The temporary std::string is destroyed, substr returns a new string
    // that's also a temporary... result: dangling!

    // Case 3: Iterator/reference invalidation
    std::vector<std::string> v = {"alpha", "beta", "gamma"};
    std::string& ref = v[0];
    std::cout << "Before push_back: " << ref << "\n";

    // v.push_back("delta");  // May reallocate! ref could dangle!
    // std::cout << ref;       // POTENTIAL UB if reallocation happened

    // Case 4: Dangling from string_view
    // std::string_view sv = std::string("temp").substr(0, 4);
    // string destroyed - sv dangles

    std::cout << "\n=== Safe patterns ===\n";
    std::cout << "1. Return by value, not by reference to locals\n";
    std::cout << "2. Store temporaries in variables before taking references\n";
    std::cout << "3. Don't hold references across container mutations\n";
    std::cout << "4. Use string (not string_view) when source may be temporary\n";

    return 0;
}
```

The `string_view` and `span` cases in Case 4 are especially common today because those types are non-owning by design. They are great for performance but you have to be disciplined about source lifetimes.

### Q3: Explain why the compiler may optimize code assuming no dangling exists, producing unexpected results

This is the hardest part of the topic. The compiler does not just fail to catch UB - it actively uses the assumption that UB does not occur as a license to generate faster code. The null-check elimination example below is a classic: after dereferencing `p`, the compiler knows `p` cannot be null (dereferencing null would be UB, which the compiler assumes never happens), so it is free to remove any later null check on the same pointer.

```cpp
#include <iostream>

// The compiler is ALLOWED to assume your program has no UB.
// If dangling access is UB, the compiler can optimize as if it never happens.

// Example: the compiler can remove "dead" code that only executes after UB

bool is_valid(int* p) {
    // If p is dereferenced later, compiler assumes p != nullptr
    // and may REMOVE this null check!
    return p != nullptr;
}

int use_after_free_example() {
    int* p = new int(42);
    int val = *p;       // compiler notes: p must be valid (no UB assumed)
    delete p;
    // compiler may reorder or cache *p from before delete
    return val;         // may return 42, may return garbage, may crash
}

int main() {
    // === How UB breaks optimizer assumptions ===

    // The compiler assumes:
    // 1. Pointers that are dereferenced are NOT null
    // 2. References always refer to valid objects
    // 3. No use-after-free ever occurs
    // 4. Array indices are in bounds

    // Based on these assumptions, the optimizer can:
    // - Remove null checks after a dereference
    // - Reorder reads and writes
    // - Eliminate entire branches
    // - Propagate values from before lifetime ended

    std::cout << "=== Optimizer and UB ===\n\n";

    // Example 1: Null check elimination
    int* p = new int(42);
    std::cout << *p << "\n";  // dereference - compiler assumes p != null
    // After this, compiler may REMOVE any subsequent "if (p == nullptr)" check
    // because dereferencing null would be UB, so p MUST be non-null
    if (p == nullptr) {
        std::cout << "This can be optimized away!\n";
        // Compiler: "p was dereferenced above, so it can't be null"
        // This entire branch may be removed
    }
    delete p;

    // Example 2: Return value from "dead" object
    // int stale = use_after_free_example();
    // The compiler might return the cached value 42 (looks "correct")
    // or might generate code that accesses freed memory

    // Example 3: signed overflow
    // for (int i = 0; i >= 0; ++i) { ... }
    // Compiler assumes signed overflow never happens (it's UB)
    // So "i >= 0" is always true - infinite loop (intentionally optimized)

    std::cout << "\n=== Key takeaway ===\n";
    std::cout << "UB doesn't mean 'crash' - it means the compiler can do ANYTHING.\n";
    std::cout << "The optimizer WILL exploit UB assumptions for speed.\n";
    std::cout << "Code that 'works in debug' may break in release (-O2).\n";
    std::cout << "\nUse sanitizers to catch dangling:\n";
    std::cout << "  -fsanitize=address  (ASan - use-after-free)\n";
    std::cout << "  -fsanitize=undefined (UBSan - undefined behavior)\n";
    std::cout << "  -fno-omit-frame-pointer (better stack traces)\n";

    return 0;
}
```

The practical takeaway: always test with `-fsanitize=address` before shipping. Code that "works" under debug or low optimization may silently break at `-O2` because the compiler exploited a UB assumption you did not even know you were making.

---

## Notes

- Dangling references are **just as dangerous** as dangling pointers - both are UB.
- The compiler **assumes no UB** and optimizes accordingly. Dangling code that "works" in debug may break with `-O2`.
- Use smart pointers (`unique_ptr`, `shared_ptr`) to eliminate use-after-free.
- Use `-fsanitize=address` to detect dangling at runtime during testing.
- `std::string_view` and `std::span` are non-owning - they dangle if the source dies.
