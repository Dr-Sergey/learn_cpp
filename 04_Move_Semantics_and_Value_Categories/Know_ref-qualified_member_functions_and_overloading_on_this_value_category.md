# Know Ref-Qualified Member Functions and Overloading on `*this` Value Category

**Category:** Move Semantics and Value Categories  
**Standard:** C++11 and later  
**Reference:** https://en.cppreference.com/w/cpp/language/member_functions#ref-qualified_member_functions  

---

## Topic Overview

Ref-qualified member functions let you overload a member function based on whether `*this` is an lvalue or an rvalue. By appending `&` or `&&` after the parameter list (and after any cv-qualifiers), you tell the compiler which overload to pick depending on the value category of the object the function is called on.

The reason this matters is that calling a function on a temporary can be dangerous. If a getter returns a reference into the object, and the object is a temporary, that reference dangles the moment the expression is done. Ref-qualifiers let you catch that at compile time instead of at runtime.

Here is the resolution table:

```cpp
┌──────────────────────────────────────────────────────────────┐
│            Ref-Qualifier Overload Resolution                 │
├──────────────────┬───────────────────────────────────────────┤
│  Qualifier       │  Called when *this is ...                 │
├──────────────────┼───────────────────────────────────────────┤
│  void f() &      │  lvalue  (named object, lvalue ref)      │
│  void f() &&     │  rvalue  (temporary, std::move(obj))     │
│  void f() const& │  const lvalue OR rvalue (fallback)       │
│  void f()        │  any value category (no restriction)     │
└──────────────────┴───────────────────────────────────────────┘
```

A critical rule: if **any** overload of a given member function name is ref-qualified, then **all** overloads of that name must be ref-qualified. You cannot mix qualified and unqualified overloads - the compiler will reject it outright.

The most important practical use case is **preventing accidental operations on temporaries**. Another powerful pattern is **stealing resources from expiring objects** - an `&&`-qualified overload can move internal data out, while the `&`-qualified overload copies it.

Ref-qualifiers also compose with `const`. The full ordering of specificity looks like this:

```cpp
  void f() &;         // non-const lvalue only
  void f() const &;   // const lvalue, or rvalue if no && overload
  void f() &&;        // non-const rvalue only
  void f() const &&;  // const rvalue (rare, but legal)
```

Note that `const&` can bind to rvalues (just like a `const T&` parameter), so if you only provide `const&` without `&&`, rvalues will still call the `const&` overload. To **forbid** rvalue calls, you must either provide a deleted `&&` overload or only provide the non-const `&` overload.

---

## Self-Assessment

### Q1: How do you overload a getter to return by reference for lvalues and by value for rvalues

The idea here is that when you call a getter on a named object, you want a cheap reference. When you call it on a temporary, the object is about to die anyway - so you might as well move the data out rather than return a dangling reference.

```cpp
#include <iostream>
#include <string>
#include <utility>

class Document {
    std::string title_;

public:
    explicit Document(std::string title) : title_(std::move(title)) {}

    // Lvalue: return a const reference - no copy, caller can read
    const std::string& title() const & {
        std::cout << "  [title() const & - returning ref]\n";
        return title_;
    }

    // Rvalue: move the title out - the Document is expiring anyway
    std::string title() && {
        std::cout << "  [title() && - moving out]\n";
        return std::move(title_);
    }
};

Document make_document() {
    return Document{"Temporary Report"};
}

int main() {
    Document doc{"Persistent Report"};

    // Calls const& overload - no copy, no move
    const std::string& ref = doc.title();
    std::cout << "From lvalue: " << ref << "\n\n";

    // Calls && overload - moves the string out of the temporary
    std::string stolen = make_document().title();
    std::cout << "From rvalue: " << stolen << "\n\n";

    // Explicit move triggers && overload
    std::string moved = std::move(doc).title();
    std::cout << "After std::move: " << moved << "\n";

    return 0;
}
```

The key insight: the same function name, two behaviors. Named objects get a reference (zero cost), and temporaries get their data moved out (still zero copies, just a move).

### Q2: How do you prevent calling a dangerous member function on a temporary

When a function returns a raw pointer or reference into the object's internals, calling it on a temporary is undefined behavior - the object vanishes and you're left holding a dangling pointer. The `= delete` trick turns that runtime disaster into a compile-time error.

```cpp
#include <iostream>
#include <vector>
#include <numeric>

class DataBuffer {
    std::vector<int> data_;

public:
    explicit DataBuffer(std::vector<int> data) : data_(std::move(data)) {}

    // Safe: object lives long enough for the span/pointer to remain valid
    const int* data_ptr() const & {
        return data_.data();
    }

    // Dangerous on temporaries - delete the rvalue overload
    const int* data_ptr() const && = delete;

    // Similarly for a method returning a reference to internal state
    std::vector<int>& raw_data() & {
        return data_;
    }

    // Prevent: DataBuffer{...}.raw_data() would return a dangling ref
    std::vector<int>& raw_data() && = delete;

    // But we CAN allow extracting data from an rvalue by moving
    std::vector<int> extract_data() && {
        return std::move(data_);
    }

    int sum() const {
        // No ref-qualifier needed - pure computation, no dangling risk
        // But note: if ANY overload of sum() were ref-qualified, ALL must be.
        return std::accumulate(data_.begin(), data_.end(), 0);
    }
};

int main() {
    DataBuffer buf({1, 2, 3, 4, 5});

    // OK: lvalue, pointer is valid as long as buf lives
    const int* p = buf.data_ptr();
    std::cout << "First element: " << *p << "\n";

    // COMPILE ERROR: calling deleted rvalue overload
    // const int* bad = DataBuffer({1,2,3}).data_ptr();

    // OK: extract data from a temporary (moves it out)
    std::vector<int> extracted = DataBuffer({10, 20, 30}).extract_data();
    std::cout << "Extracted size: " << extracted.size() << "\n";

    // OK: sum() has no ref-qualifier, works on any value category
    int s = DataBuffer({1, 2, 3}).sum();
    std::cout << "Sum of temp: " << s << "\n";

    return 0;
}
```

Notice the distinction: `data_ptr()` and `raw_data()` are deleted for rvalues because returning references into a temporary is always wrong. But `extract_data()` is allowed on rvalues because it moves the data out - that's actually the right thing to do.

### Q3: What happens when you mix ref-qualified and non-ref-qualified overloads

The short answer is: you can't. As soon as you ref-qualify one overload of a name, the compiler requires all overloads of that name to be ref-qualified too.

```cpp
#include <iostream>

struct Widget {
    // If you ref-qualify ANY overload, ALL overloads of that name
    // must be ref-qualified. Uncommenting the unqualified overload
    // below produces a compiler error.

    void process() & {
        std::cout << "process() & - lvalue\n";
    }

    void process() && {
        std::cout << "process() && - rvalue\n";
    }

    // ERROR: cannot overload ref-qualified with non-ref-qualified
    // void process() const { }  // won't compile!

    // This IS allowed - it's const-qualified AND ref-qualified:
    void process() const & {
        std::cout << "process() const& - const lvalue or rvalue fallback\n";
    }
};

int main() {
    Widget w;
    const Widget cw;

    w.process();              // process() &
    std::move(w).process();   // process() &&
    cw.process();             // process() const &

    // Temporary: prefers && over const&
    Widget{}.process();       // process() &&

    // const temporary: only const& matches (const&& not provided)
    std::move(cw).process();  // process() const &

    return 0;
}
```

The reason `Widget{}.process()` picks `&&` over `const&` is that an unqualified temporary is non-const, so `&&` is the better match. For a const temporary (`std::move(cw)`), there is no `const&&` overload, so it falls back to `const&`.

---

## Notes

- Ref-qualifiers go **after** cv-qualifiers: `void f() const &;` not `void f() & const;`.
- If any overload of a name is ref-qualified, all overloads of that same name must be ref-qualified - mixing is a hard error.
- The `const &` overload acts as a catch-all for rvalues when no `&&` overload exists, mirroring how `const T&` binds to rvalues in function parameters.
- Use `= delete` on the `&&` overload to prevent calling a function on temporaries - this is the primary safety pattern.
- Ref-qualifiers are orthogonal to `virtual` - you can have virtual ref-qualified member functions, but the ref-qualifier must match exactly in the override.
- The deduced-this parameter (`this auto&& self`) in C++23 subsumes ref-qualifiers, but ref-qualifiers remain the idiomatic C++11/17 approach.
