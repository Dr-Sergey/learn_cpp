# Know the lifetime extension rules for temporaries bound to const references

**Category:** Core Language Fundamentals  
**Standard:** C++98, C++11, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/reference_initialization#Lifetime_of_a_temporary>  

---

## Topic Overview

### The Rule

When a temporary is bound directly to a `const` lvalue reference or an rvalue reference, the temporary's lifetime is **extended** to match the lifetime of the reference:

```cpp
const std::string& ref = std::string("hello");  // Temporary lives as long as ref
// ref is valid here - temporary still alive
```

Without binding to a reference, the temporary would be destroyed at the end of the full-expression (the semicolon).

### When Lifetime Extension DOES Work

These all bind directly to the reference, which is the key requirement:

```cpp
const int& r1 = 42;                        // Literal creates a temporary
const std::string& r2 = "hello";           // Implicit conversion creates temporary
const auto& r3 = std::vector{1,2,3};       // Direct binding to temporary
std::string&& r4 = std::string("world");   // Rvalue reference also extends lifetime

struct Base {};
struct Derived : Base {};
const Base& r5 = Derived{};                // Extends even through base class binding
```

### When Lifetime Extension Does NOT Work

The rule is strict: extension only happens for **direct** binding. Any indirection - a function call, a member access - breaks it.

| Scenario | Extends? | Why |
| --- | --- | --- |
| Direct binding `const T& r = T{}` | Yes | Standard rule |
| Returned from function `const T& r = f()` | No | Temporary dies end of statement |
| Member reference initialized `T{}.member` | No | Accessing member of temporary |
| Passed to function parameter | No | Lifetime = duration of function call only |
| ternary `? :` (in some cases) | Depends | Complex rules |
| Aggregate init member `{.ref = T{}}` | No (before C++26) | Temporary dies |

```cpp
const std::string& dangerous() {
    return std::string("oops");  // Temporary destroyed at return!
}

struct S { const int& ref; };
S s{42};  // Before C++26: temporary int dies, s.ref dangles
```

---

## Self-Assessment

### Q1: Demonstrate that `const T& ref = T{}` extends the lifetime

The `Verbose` type announces every construction and destruction, so you can see exactly when the lifetime ends:

```cpp
#include <iostream>
#include <string>

struct Verbose {
    std::string name;
    Verbose(std::string n) : name(std::move(n)) {
        std::cout << "  Construct: " << name << "\n";
    }
    ~Verbose() {
        std::cout << "  Destroy:   " << name << "\n";
    }
};

int main() {
    std::cout << "=== Before binding ===\n";

    {
        std::cout << "--- Scope start ---\n";
        const Verbose& ref = Verbose("extended");  // Lifetime extended!
        std::cout << "Using ref: " << ref.name << "\n";
        std::cout << "--- Scope end ---\n";
        // Temporary destroyed HERE - when ref goes out of scope
    }

    std::cout << "\n=== Without binding ===\n";
    {
        std::cout << "--- Scope start ---\n";
        Verbose("not_extended");  // Temporary destroyed immediately (end of expression)
        std::cout << "--- After temporary ---\n";
    }

    std::cout << "\n=== Rvalue reference extends too ===\n";
    {
        Verbose&& rref = Verbose("rvalue_extended");
        std::cout << "Using rref: " << rref.name << "\n";
        // Destroyed when rref goes out of scope
    }

    return 0;
}
```

**Output:**

```text
=== Before binding ===
--- Scope start ---
  Construct: extended
Using ref: extended
--- Scope end ---
  Destroy:   extended

=== Without binding ===
--- Scope start ---
  Construct: not_extended
  Destroy:   not_extended
--- After temporary ---

=== Rvalue reference extends too ===
  Construct: rvalue_extended
Using rref: rvalue_extended
  Destroy:   rvalue_extended
```

Notice how "extended" stays alive until the closing brace of its scope, while "not_extended" is gone immediately after its expression. That's lifetime extension in action.

Key observations:

- With `const T& ref = T{}`, the temporary survives until `ref` goes out of scope.
- Without binding, the temporary is destroyed at the semicolon.
- `T&& rref = T{}` also extends lifetime (rvalue references bind to temporaries).

### Q2: Show a case where lifetime extension does NOT occur

Here's where people get burned. The BUGGY versions look harmless but produce dangling references:

```cpp
#include <iostream>
#include <string>

struct Verbose {
    std::string name;
    Verbose(std::string n) : name(std::move(n)) {
        std::cout << "  Construct: " << name << "\n";
    }
    ~Verbose() {
        std::cout << "  Destroy:   " << name << "\n";
    }
    Verbose(const Verbose& o) : name(o.name + "_copy") {
        std::cout << "  Copy: " << name << "\n";
    }
};

// Case 1: Returning by const reference - DANGLING!
const Verbose& make_verbose_BAD() {
    return Verbose("returned_temp");  // Temporary dies at return!
}

// Case 2: Returning by value - SAFE (copy/move, no dangling)
Verbose make_verbose_GOOD() {
    return Verbose("returned_safe");  // Copy elision (RVO/NRVO)
}

// Case 3: Function parameter - temporary lives only during the call
void use_ref(const Verbose& v) {
    std::cout << "  In function: " << v.name << "\n";
    // v is valid here - temporary alive for the duration of the function call
}

// Case 4: Storing reference to temporary's member - DANGLING!
struct Pair {
    std::string first, second;
};

int main() {
    // Case 1: DANGLING - the temporary is destroyed before we use the reference
    // const Verbose& bad = make_verbose_BAD();
    // std::cout << bad.name << "\n";  // UB! Accessing destroyed object
    // Compiler warning: returning reference to local temporary

    // Case 2: SAFE - value is returned, copy elided
    std::cout << "=== Case 2: Return by value ===\n";
    const Verbose& good = make_verbose_GOOD();  // Lifetime extended!
    std::cout << "Using good: " << good.name << "\n";

    // Case 3: Temporary lives during function call only
    std::cout << "\n=== Case 3: Parameter binding ===\n";
    use_ref(Verbose("param_temp"));
    std::cout << "After function call - temp is already destroyed\n";

    // Case 4: Member access on temporary
    std::cout << "\n=== Case 4: Member of temporary ===\n";
    // const std::string& ref = Pair{"hello", "world"}.first;
    // The temporary Pair is destroyed, ref dangles!
    // In C++23, this may be extended in some cases, but it's still dangerous.

    // SAFE alternative: bind the whole temporary
    const Pair& pair_ref = Pair{"hello", "world"};  // Lifetime extended
    std::cout << pair_ref.first << "\n";  // OK!

    return 0;
}
```

The safe pattern at the end is the key takeaway: if you want to keep a member alive, bind the whole object.

Rules of thumb:

- Direct binding `const T& r = T{}` -> extends lifetime
- Returned from function `const T& r = f()` where f returns by value -> extends! (the return value is the temporary)
- Function returning reference `const T& f() { return T{}; }` -> DANGLING
- Member of temporary `const int& r = Pair{}.first` -> usually dangling (before C++26)
- Through intermediate function `const T& r = identity(T{})` -> dangling (temporary dies after `identity` returns)

### Q3: C++23 deducing-this and reference binding

C++23's **deducing this** (`this auto&& self`) lets member functions deduce whether they're called on an lvalue or rvalue:

```cpp
#include <iostream>
#include <string>

struct Widget {
    std::string data = "hello";

    // C++23: deducing this - no need for const/non-const overloads
    template<typename Self>
    auto&& get_data(this Self&& self) {
        return std::forward<Self>(self).data;
    }

    // Equivalent to writing all of:
    // std::string& get_data() & { return data; }
    // const std::string& get_data() const& { return data; }
    // std::string&& get_data() && { return std::move(data); }
};

int main() {
    Widget w;
    auto& ref = w.get_data();           // Self = Widget& -> returns std::string&
    const Widget cw;
    auto& cref = cw.get_data();         // Self = const Widget& -> returns const std::string&

    // Rvalue: calling on a temporary
    auto val = Widget{}.get_data();     // Self = Widget -> returns std::string&& -> moved into val

    // Lifetime caution with deducing-this on temporaries:
    // auto&& dangerous = Widget{}.get_data();
    // Widget{} is a temporary, .get_data() returns a reference to its member
    // The temporary Widget is destroyed -> dangerous is a dangling reference!

    std::cout << ref << "\n";   // "hello"
    std::cout << val << "\n";   // "hello" (moved from the temporary)

    return 0;
}
```

**Key interaction:** Deducing-this makes it easier to write perfect-forwarding accessors, but the lifetime rules for temporaries still apply. Binding `auto&&` to the result of a member function called on a temporary does NOT extend the temporary's lifetime.

---

## Notes

- `const T&` and `T&&` binding extends temporary lifetime to the reference's scope.
- Lifetime extension ONLY works for **direct binding** - not through function returns, member access, or casts.
- Common trap: `const auto& s = get_string().substr(0, 5)` - the `get_string()` temporary dies first if it returns by value and you chain calls.
- C++23 deducing-this simplifies overload sets but doesn't change lifetime rules.
- When in doubt, bind by **value** (`auto x = expr;`) to avoid dangling references.
