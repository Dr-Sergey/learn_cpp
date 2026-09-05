# Understand two-phase name lookup in templates

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/two-phase_lookup>  

---

## Topic Overview

When the compiler parses a template, it performs name lookup in **two phases**. This is one of those rules that feels abstract until you hit a mysterious compile error - then it suddenly makes a lot of sense.

### Phase 1: Template Definition Time

**Non-dependent names** (names that don't depend on template parameters) are looked up immediately when the template is defined.

```cpp
void helper() { std::cout << "global helper\n"; }

template<typename T>
void foo(T x) {
    helper();  // Non-dependent: looked up NOW at definition time
    x.bar();   // Dependent: looked up later at instantiation time
}
```

The compiler can resolve `helper()` right away because it doesn't involve `T` at all. `x.bar()` has to wait because the compiler doesn't yet know what `T` is.

### Phase 2: Template Instantiation Time

**Dependent names** (names that depend on template parameters) are looked up when the template is actually instantiated with a specific type. ADL is also performed at this point.

### The typename Disambiguator

When a dependent name refers to a **type**, you must use `typename` - otherwise the compiler assumes it's a value:

```cpp
template<typename T>
void process(T& container) {
    // T::iterator could be a type or a static member
    // typename tells the compiler: "this is a type"
    typename T::iterator it = container.begin();

    // C++20 relaxes this requirement in some contexts
}
```

### The template Disambiguator

When a dependent name calls a **template member**, you must use `template` to tell the compiler that the following `<` is not the less-than operator:

```cpp
template<typename T>
void call_method(T& obj) {
    // obj.method<int>() - compiler thinks < is less-than operator!
    obj.template method<int>();  // Correct: .template disambiguates
}
```

---

## Self-Assessment

### Q1: Show a dependent name that requires the `typename` keyword to parse correctly

Without `typename`, the compiler defaults to treating `Container::const_iterator` as a value - which causes a parse error. The `typename` keyword is your explicit promise that this dependent name is a type:

```cpp
#include <iostream>
#include <vector>
#include <map>

template<typename Container>
void print_first(const Container& c) {
    // Container::const_iterator is a DEPENDENT name
    // Without typename, the compiler assumes it's a value, not a type
    // typename Container::const_iterator it = c.begin();  // C++11/14/17 required

    // C++20: typename is not required in many contexts (but still allowed)
    typename Container::const_iterator it = c.begin();
    if (it != c.end()) {
        std::cout << *it << "\n";
    }
}

// More complex example: nested dependent type
template<typename T>
struct Wrapper {
    // T::value_type is dependent on T
    using element_type = typename T::value_type;

    // Return type is dependent
    typename T::const_iterator find_first(const T& container) {
        return container.begin();
    }
};

// Where typename is STILL required (even in C++20):
template<typename T>
typename T::size_type get_size(const T& c) {
    return c.size();
}

int main() {
    std::vector<int> v{10, 20, 30};
    print_first(v);  // 10

    Wrapper<std::vector<int>> w;
    auto it = w.find_first(v);
    std::cout << *it << "\n";  // 10

    auto sz = get_size(v);
    std::cout << "Size: " << sz << "\n";  // 3
}
```

**How this works:**

- `Container::const_iterator` is a dependent name - it depends on the template parameter `Container`.
- Without `typename`, the compiler parses it as a non-type (value/variable) - which breaks the code.
- `typename` explicitly tells the compiler: "I promise this is a type."
- C++20 relaxed the requirement in many common contexts (variable declarations, return types, parameter types), but the programmer can still write `typename` for clarity.

### Q2: Use the `template` keyword to disambiguate a dependent template member

When `parser` has a dependent type, the compiler can't tell whether `parser.parse<int>` means "call the template `parse` with `int`" or "compare `parser.parse` against `int` with less-than." The `.template` keyword resolves the ambiguity:

```cpp
#include <iostream>
#include <string>

struct Parser {
    template<typename T>
    T parse(const std::string& input) {
        // Simplified: just default-construct T
        return T{};
    }

    template<typename T>
    static T create() {
        return T{};
    }
};

template<typename P>
void use_parser(P& parser) {
    // parser.parse<int>("42")
    // The compiler sees: parser.parse < int > ("42")
    // Is '<' a less-than operator or a template argument list?
    // Answer: it's ambiguous! The compiler assumes less-than.

    // Fix: use .template to disambiguate
    auto val = parser.template parse<int>("42");
    std::cout << "Parsed: " << val << "\n";

    // Same with static member through dependent type:
    auto obj = P::template create<double>();
    std::cout << "Created: " << obj << "\n";
}

// Another example with nested templates
template<typename T>
struct Container {
    template<typename U>
    U convert() const { return U{}; }
};

template<typename C>
void convert_from(const C& c) {
    // c.convert<int>()  - ambiguous
    auto result = c.template convert<int>();
    std::cout << "Converted: " << result << "\n";
}

int main() {
    Parser p;
    use_parser(p);

    Container<std::string> c;
    convert_from(c);
}
```

**How this works:**

- When `parser` has a dependent type, `parser.parse<int>` is ambiguous: is `<` comparison or template arg?
- `.template` before `parse` tells the compiler: "the following `<` starts a template argument list."
- Same applies to `::template` for static members and `->template` for pointer access.

### Q3: Explain why non-dependent names are looked up at template definition, not instantiation

**Answer:**

Non-dependent names are resolved at definition time for two critical reasons:

**1. Early error detection:**

If the compiler waited until instantiation to check non-dependent names, you'd only find bugs in templates that were actually used - leaving latent errors in unused code paths. Phase 1 catches them immediately:

```cpp
template<typename T>
void foo() {
    undeclared_function();  // ERROR: caught at definition time (Phase 1)
    // No need to wait for instantiation to find this bug
}
// Even if foo() is never instantiated, this error is detected
```

**2. Preventing silent behavior changes:**

Without Phase 1 lookup, adding a new overload later in a file could silently change what an already-defined template calls. That would be a nightmare to debug:

```cpp
void helper() { std::cout << "Version A\n"; }

template<typename T>
void foo() {
    helper();  // Bound to Version A at definition time
}

// Later in the file:
void helper(int) { std::cout << "Version B\n"; }

// If foo<int>() used instantiation-time lookup for helper(),
// it might find Version B instead - changing behavior silently!
// Phase 1 lookup prevents this.
```

**3. Examples of dependent vs non-dependent:**

Here's a quick side-by-side showing which lookup phase applies to each name:

```cpp
int global = 42;

template<typename T>
struct Demo {
    void method() {
        std::cout << global;     // Non-dependent -> Phase 1 lookup
        T x;                     // Dependent -> Phase 2 lookup
        x.member();              // Dependent -> Phase 2 (ADL possible)
        typename T::type y;      // Dependent -> Phase 2
        foo();                   // Non-dependent -> Phase 1
        this->bar();             // Dependent (this->) -> Phase 2
    }
};
```

**Rule of thumb:** If a name involves `T` (or depends on it in any way), it's dependent -> Phase 2. Otherwise it's non-dependent -> Phase 1.

---

## Notes

- MSVC historically didn't implement two-phase lookup correctly - it delayed all lookups to Phase 2. Use `/permissive-` to enable standard-conformant behavior.
- C++20 significantly relaxed `typename` requirements - in most positions where only a type is valid, `typename` is now optional.
- `this->` makes a name dependent, which can be useful in templates to access base class members: `this->inherited_method()`.
- ADL only applies during Phase 2 for dependent function calls - not during Phase 1.
- Two-phase lookup is the reason template definition must be in headers (or visible at definition point) - the compiler needs to resolve non-dependent names immediately.
