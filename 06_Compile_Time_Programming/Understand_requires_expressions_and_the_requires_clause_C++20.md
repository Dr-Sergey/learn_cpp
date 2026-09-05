# Understand `requires` Expressions and the `requires` Clause (C++20)

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/requires>  

---

## Topic Overview

### What Is a `requires` Expression

A `requires` expression is a compile-time predicate that checks whether a set of **syntactic and semantic requirements** are satisfied. It evaluates to `true` or `false` at compile time. You'll see it most often inside a concept definition, but it can appear anywhere a compile-time boolean is needed.

```cpp
template <typename T>
concept Serializable = requires(T t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};
```

Reading this out loud: "T is Serializable if, given a value `t` of type T, the expression `t.serialize()` is valid and its return type is convertible to `std::string`."

### The Two `requires` Keywords

This is where people get confused - C++20 uses the word `requires` in two completely different roles:

| Context | Purpose | Example |
| --- | --- | --- |
| **`requires` expression** | A compile-time boolean expression that tests constraints | `requires(T t) { t.size(); }` |
| **`requires` clause** | A constraint applied to a template/function | `template <typename T> requires Integral<T>` |

They can even appear together: `requires requires(T t) { t.foo(); }` (a requires clause containing a requires expression). Yes, it looks odd, but each `requires` is doing a different job.

### Four Kinds of Requirements

Inside a `requires` expression, there are four requirement types. Each tests something different about the type:

| Kind | Syntax | Checks |
| --- | --- | --- |
| **Simple** | `expr;` | Expression is valid (compiles) |
| **Type** | `typename T::type;` | Type exists |
| **Compound** | `{ expr } -> concept;` | Expression is valid AND its return type satisfies a concept |
| **Nested** | `requires predicate;` | An additional boolean constraint must hold |

Here's all four in one concept:

```cpp
template <typename T>
concept FullyConstrained = requires(T a, T b) {
    a + b;                                       // Simple: a+b compiles
    typename T::value_type;                      // Type: T has a value_type member
    { a.size() } -> std::convertible_to<size_t>; // Compound: size() returns something convertible to size_t
    requires std::copyable<T>;                   // Nested: T must be copyable
};
```

---

## Self-Assessment

### Q1: Write a `requires` expression that checks for a `.serialize()` member that returns `std::string`

The code below builds two concepts: `Serializable` (just needs `serialize()`) and `FullySerializable` (needs both `serialize()` and a static `deserialize()`). Then it shows three types that do/don't satisfy them, and uses `static_assert` to confirm the checks happen at compile time.

```cpp
#include <iostream>
#include <string>
#include <concepts>

// === Concept: Serializable ===
// Uses a compound requirement to check:
// 1. t.serialize() is a valid expression
// 2. Its return type is convertible to std::string
template <typename T>
concept Serializable = requires(T t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

// === Also check for deserialize ===
template <typename T>
concept FullySerializable = requires(T t, const std::string& s) {
    { t.serialize() } -> std::convertible_to<std::string>;
    { T::deserialize(s) } -> std::convertible_to<T>;
};

// === Types that do/don't satisfy the concept ===
struct Config {
    std::string name;
    int value;
    std::string serialize() const {
        return name + "=" + std::to_string(value);
    }
    static Config deserialize(const std::string& s) {
        auto pos = s.find('=');
        return {s.substr(0, pos), std::stoi(s.substr(pos + 1))};
    }
};

struct NotSerializable {
    int data;
    // No serialize() method
};

struct WrongReturn {
    int serialize() const { return 42; }  // Returns int, not string
};

// === Constrained function ===
template <Serializable T>
void save(const T& obj) {
    std::string data = obj.serialize();
    std::cout << "Saved: " << data << "\n";
}

// === Constrained overload for FullySerializable ===
template <FullySerializable T>
T round_trip(const T& obj) {
    std::string data = obj.serialize();
    std::cout << "Serialized: " << data << "\n";
    T restored = T::deserialize(data);
    std::cout << "Deserialized successfully\n";
    return restored;
}

int main() {
    // Compile-time checks
    static_assert(Serializable<Config>);
    static_assert(!Serializable<NotSerializable>);
    static_assert(!Serializable<WrongReturn>);      // int not convertible_to<string>
    static_assert(FullySerializable<Config>);

    Config cfg{"timeout", 30};
    save(cfg);

    Config restored = round_trip(cfg);
    std::cout << "Restored: " << restored.name << "=" << restored.value << "\n";

    // These would NOT compile:
    // save(NotSerializable{42});   // Error: constraint not satisfied
    // save(WrongReturn{});         // Error: int not convertible to string

    return 0;
}
```

**Expected output:**

```text
Saved: timeout=30
Serialized: timeout=30
Deserialized successfully
Restored: timeout=30
```

### Q2: Explain the four kinds of requirements: simple, type, compound, and nested

This example defines a separate concept for each requirement kind, then combines all four into a single `WellBehavedContainer` concept. Reading through it is a good way to internalize when each form is appropriate.

```cpp
#include <iostream>
#include <string>
#include <concepts>
#include <vector>
#include <type_traits>

// === 1. SIMPLE REQUIREMENT ===
// Checks that an expression is valid (compiles)
template <typename T>
concept HasPushBack = requires(T t, typename T::value_type v) {
    t.push_back(v);    // Simple: push_back(v) must compile
};

// === 2. TYPE REQUIREMENT ===
// Checks that a type name/alias exists
template <typename T>
concept HasValueType = requires {
    typename T::value_type;      // Type: T::value_type must exist
    typename T::iterator;        // Type: T::iterator must exist
};

// === 3. COMPOUND REQUIREMENT ===
// Checks expression validity AND constrains return type
template <typename T>
concept SizedContainer = requires(T t) {
    { t.size() } noexcept -> std::convertible_to<std::size_t>;  // Must be noexcept AND return size_t-compatible
    { t.empty() } -> std::same_as<bool>;                        // Must return exactly bool
    { *t.begin() } -> std::same_as<typename T::reference>;      // Dereference yields reference
};

// === 4. NESTED REQUIREMENT ===
// Tests a boolean predicate (often using type traits)
template <typename T>
concept SafeContainer = requires {
    requires std::is_nothrow_destructible_v<T>;    // T's destructor must be noexcept
    requires sizeof(T) > 0;                        // T must not be zero-sized
    requires std::copyable<T>;                     // T must be copyable
};

// === Combined: all four in one concept ===
template <typename T>
concept WellBehavedContainer = requires(T t, typename T::value_type v) {
    // Simple requirements
    t.push_back(v);
    t.clear();

    // Type requirements
    typename T::value_type;
    typename T::iterator;
    typename T::size_type;

    // Compound requirements
    { t.size() } -> std::convertible_to<std::size_t>;
    { t.front() } -> std::same_as<typename T::reference>;

    // Nested requirements
    requires std::movable<T>;
    requires std::is_nothrow_destructible_v<T>;
};

// === Usage ===
template <WellBehavedContainer C>
void process(C& container) {
    std::cout << "Container size: " << container.size() << "\n";
    if (!container.empty()) {
        std::cout << "Front: " << container.front() << "\n";
    }
}

int main() {
    // Verify concepts at compile time
    static_assert(HasPushBack<std::vector<int>>);
    static_assert(HasValueType<std::vector<int>>);
    static_assert(SizedContainer<std::vector<int>>);
    static_assert(SafeContainer<std::vector<int>>);
    static_assert(WellBehavedContainer<std::vector<int>>);

    // int doesn't satisfy any container concept
    static_assert(!HasPushBack<int>);
    static_assert(!HasValueType<int>);

    std::vector<int> v = {1, 2, 3};
    process(v);

    std::cout << "\n=== Four Requirement Types ===\n";
    std::cout << "1. Simple:   expr;                       - just checks compilability\n";
    std::cout << "2. Type:     typename T::type;           - checks type exists\n";
    std::cout << "3. Compound: { expr } -> concept;        - checks expr + return type\n";
    std::cout << "4. Nested:   requires bool_expression;   - checks predicate is true\n";

    return 0;
}
```

**Expected output:**

```text
Container size: 3
Front: 1

=== Four Requirement Types ===

1. Simple:   expr;                       - just checks compilability
2. Type:     typename T::type;           - checks type exists
3. Compound: { expr } -> concept;        - checks expr + return type
4. Nested:   requires bool_expression;   - checks predicate is true
```

### Q3: Show how a `requires` clause on a function differs from a concept definition

The most important practical difference is **subsumption**: when you have two overloads and one's constraints are a strict superset of the other's, the more-constrained overload wins. This only works with named concepts. Ad-hoc inline `requires` expressions do NOT participate in subsumption - the compiler treats them as opaque, so you'd get an ambiguity error.

```cpp
#include <iostream>
#include <concepts>
#include <type_traits>
#include <string>

// === CONCEPT DEFINITION ===
// - Named, reusable constraint
// - Can be used in multiple places
// - Participates in overload resolution (subsumption)
template <typename T>
concept Printable = requires(std::ostream& os, T t) {
    { os << t } -> std::same_as<std::ostream&>;
};

template <typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// === REQUIRES CLAUSE ON A FUNCTION ===
// - Ad-hoc, inline constraint
// - Not named, not reusable
// - Good for one-off constraints

// Method 1: requires clause after template params
template <typename T>
    requires Printable<T>
void print_v1(const T& val) {
    std::cout << val;
}

// Method 2: trailing requires clause
template <typename T>
void print_v2(const T& val) requires Printable<T> {
    std::cout << val;
}

// Method 3: concept as type constraint (abbreviated template)
void print_v3(const Printable auto& val) {
    std::cout << val;
}

// Method 4: ad-hoc requires clause (NOT a named concept)
template <typename T>
    requires requires(T t) { t.to_string(); }  // inline requires expression
void print_v4(const T& val) {
    std::cout << val.to_string();
}

// === Key Difference: Subsumption ===
// Named concepts participate in overload resolution via subsumption
// More constrained overload is preferred

template <typename T>
    requires Printable<T>          // Less constrained
void display(const T& val) {
    std::cout << "[generic] " << val << "\n";
}

template <typename T>
    requires Printable<T> && Numeric<T>  // More constrained (subsumes above)
void display(const T& val) {
    std::cout << "[numeric] " << val << "\n";
}

// === Without concepts, subsumption doesn't work ===
// These would be AMBIGUOUS (neither subsumes the other):
/*
template <typename T>
    requires requires(std::ostream& os, T t) { os << t; }
void broken(const T& val) { ... }

template <typename T>
    requires requires(std::ostream& os, T t) { os << t; } && std::integral<T>
void broken(const T& val) { ... }  // Ambiguous! inline requires don't subsume
*/

struct MyType {
    std::string to_string() const { return "MyType"; }
    friend std::ostream& operator<<(std::ostream& os, const MyType& m) {
        return os << m.to_string();
    }
};

int main() {
    // All print methods work
    print_v1(42);          std::cout << "\n";
    print_v2(3.14);        std::cout << "\n";
    print_v3("hello");     std::cout << "\n";
    print_v4(MyType{});    std::cout << "\n";

    // Subsumption: numeric overload is selected for int
    display(42);           // [numeric] 42 - more constrained wins
    display("hello");      // [generic] hello - only one matches

    std::cout << "\n=== Concept vs Requires Clause ===\n";
    std::cout << "Concept:        Named, reusable, supports subsumption\n";
    std::cout << "Requires clause: Inline, ad-hoc, no subsumption between ad-hoc constraints\n";
    std::cout << "\nRule of thumb:\n";
    std::cout << "  - Use named concepts for constraints you use more than once\n";
    std::cout << "  - Use requires clause for one-off constraints\n";
    std::cout << "  - Prefer concepts for overload sets (subsumption)\n";

    return 0;
}
```

**Expected output:**

```text
42
3.14
hello
MyType
[numeric] 42
[generic] hello

=== Concept vs Requires Clause ===
Concept:        Named, reusable, supports subsumption
Requires clause: Inline, ad-hoc, no subsumption between ad-hoc constraints

Rule of thumb:

  - Use named concepts for constraints you use more than once
  - Use requires clause for one-off constraints
  - Prefer concepts for overload sets (subsumption)
```

---

## Notes

- `requires` expressions are compile-time predicates - they never generate runtime code.
- The `requires requires` pattern (clause + expression) is valid but prefer naming the concept.
- Compound requirements can include `noexcept`: `{ expr } noexcept -> concept;`.
- Named concepts enable **subsumption** - the compiler prefers more constrained overloads.
- Ad-hoc `requires` expressions do NOT participate in subsumption - use named concepts for overload sets.
- Always prefer concepts over `enable_if` in new C++20 code.
