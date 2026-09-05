# Write Concepts-constrained library interfaces with good diagnostics

**Category:** API and Library Design  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constraints>  

---

## Topic Overview

Concepts replace SFINAE for constraining template parameters. They produce dramatically better error messages and make API requirements explicit right in the function signature - so a reader can see what a function expects without digging through pages of `enable_if` boilerplate.

The reason SFINAE error messages are so painful is that the compiler doesn't know you were trying to express a constraint. It sees a substitution failure deep in a template instantiation stack and reports "no matching function for call to 'save'" along with many lines of "candidate template ignored" - none of which say what was actually wrong. With concepts, the compiler knows a constraint was supposed to hold and it tells you exactly which requirement wasn't satisfied.

### Before and After

Here's the same constraint expressed with SFINAE and then with a concept - and the difference in what happens when you pass the wrong type:

```cpp
#include <concepts>
#include <type_traits>
#include <iostream>
#include <vector>
#include <string>

// BEFORE (SFINAE): cryptic "no matching function" errors
template<typename T,
         typename = std::enable_if_t<std::is_arithmetic_v<T>>>
T old_add(T a, T b) { return a + b; }

// AFTER (Concepts): clear "constraint not satisfied" message
template<std::integral T>
T add(T a, T b) { return a + b; }

// Custom concept with good diagnostics
template<typename T>
concept Serializable = requires(const T& t, std::ostream& os) {
    { t.serialize(os) } -> std::same_as<void>;
    { T::type_name() } -> std::convertible_to<std::string>;
};

template<Serializable T>
void save(const T& obj, std::ostream& os) {
    os << T::type_name() << ":";
    obj.serialize(os);
}

// Error message when constraint fails:
// "note: constraints not satisfied"
// "note: the required expression 't.serialize(os)' is invalid"
// vs SFINAE: "no matching function for call to 'save'"
```

The concept also serves as documentation. Looking at `template<Serializable T>`, a caller immediately knows the type needs `serialize(os)` and a static `type_name()`. With SFINAE, none of that is visible from the signature.

### Layered Concepts for Clear Diagnostics

Monolithic concepts with five requirements bundled together are hard to diagnose when one fails. A better approach is to build concepts incrementally - each layer adds one named requirement - so a failing error message tells you exactly which piece is missing:

```cpp
#include <concepts>
#include <ranges>
#include <iterator>

// Build concepts incrementally - each layer adds one requirement
template<typename T>
concept HasSize = requires(const T& t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

template<typename T>
concept HasIterators = requires(T& t) {
    { t.begin() } -> std::input_or_output_iterator;
    { t.end() } -> std::sentinel_for<decltype(std::declval<T&>().begin())>;
};

template<typename T>
concept HasPushBack = requires(T& t, typename T::value_type v) {
    t.push_back(std::move(v));
};

template<typename T>
concept Collection = HasSize<T> && HasIterators<T>;

template<typename T>
concept GrowableCollection = Collection<T> && HasPushBack<T>;

// If Collection fails, the error says WHICH sub-concept failed:
// "note: 'HasSize<T>' was not satisfied"
// Much better than a monolithic concept with 5 requirements!
```

When `GrowableCollection<T>` fails for a type, the compiler drills down through `Collection<T>` -> `HasSize<T>` and pinpoints the actual missing piece. Compare that to a single `requires` block with all five requirements - the compiler would list every failing expression and you'd have to figure out which one matters.

---

## Self-Assessment

### Q1: Write a concept for "JSON-serializable" types

A type is JSON-serializable if you can convert it to a JSON string and reconstruct it from one. Here's how to express that as a concept:

```cpp
template<typename T>
concept JsonSerializable = requires(const T& t) {
    { to_json(t) } -> std::convertible_to<std::string>;
} && requires(const std::string& s) {
    { from_json<T>(s) } -> std::same_as<T>;
};
```

This uses two separate `requires` clauses to check both directions of the round-trip. Note that `to_json` and `from_json` are expected to be ADL-visible free functions, not members.

### Q2: Show how concept subsumption gives better overload selection

Concept subsumption means that if concept `B` implies concept `A` (because `B` is defined as `A && something_extra`), then a function constrained by `B` is considered more specialized than one constrained by `A`. The compiler picks the more specialized overload automatically - no partial specialization tricks needed:

```cpp
template<typename T>
concept Printable = requires(const T& t, std::ostream& os) { os << t; };

template<typename T>
concept FormattedPrintable = Printable<T> && requires(const T& t) {
    { t.format() } -> std::convertible_to<std::string>;
};

// FormattedPrintable subsumes Printable -> more specific overload wins
template<Printable T>
void print(const T& val) { std::cout << val; }

template<FormattedPrintable T>
void print(const T& val) { std::cout << val.format(); } // Preferred when both match
```

Because `FormattedPrintable` is defined in terms of `Printable`, the compiler knows the second overload is strictly more constrained. When both match, the more constrained one wins. This is a cleaner version of what you'd do with tag dispatch or partial specialization in C++17.

### Q3: Constrain return types with concepts

You can use concepts inside `requires` to constrain not just argument types but return types and the types returned by factory functions:

```cpp
template<typename Factory>
concept WidgetFactory = requires(Factory& f, int config) {
    { f.create(config) } -> std::derived_from<Widget>;
    { f.validate(config) } -> std::same_as<bool>;
};
```

This says: a `WidgetFactory` is any type whose `create` method returns something derived from `Widget` and whose `validate` method returns exactly `bool`. The `->` inside a `requires` expression checks the return type after the call expression is evaluated.

---

## Notes

- Prefer layered, named concepts over deeply nested `requires` expressions - the error messages are dramatically better when each concept has its own name.
- Concept names should read like adjectives: `Sortable`, `Serializable`, `Hashable`. They describe what the type is capable of, not what it is.
- Always test concept satisfaction with `static_assert(MyConcept<MyType>)` - it's a one-line check that your type meets the requirements you intended.
- Concepts are zero-cost at runtime - they only participate in overload resolution and constraint checking during compilation.
