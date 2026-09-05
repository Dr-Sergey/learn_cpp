# Understand Template Argument Substitution Order and Its Effect on Errors

**Category:** Templates and Generic Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/sfinae>  

---

## Topic Overview

### Substitution Order Matters

When the compiler substitutes template arguments, it processes them **left to right** in the template parameter list and then through the function signature. This ordering matters because a later parameter's default can depend on an earlier one - and if the earlier substitution fails, the later one never happens.

Here's the simplest illustration of that dependency:

```cpp
template <typename T, typename U = typename T::value_type>
void f(T, U);
// If T=int: T::value_type is invalid - substitution failure on U
// Order: T substituted first, then U depends on T
```

The reason this trips people up is that you might look at `f(someInt, someDouble)` and think both types are spelled out explicitly - but the default for `U` still involves `T`, and that dependency is checked left-to-right.

### Three Places to Put `enable_if`

If you're on C++17 or earlier, you have a few options for where to put an `enable_if` constraint. The choice mostly comes down to readability. C++20's `requires` clause renders all of them obsolete, but you'll see the older forms in real codebases:

| Location | Syntax | Readability | Recommendation |
| --- | --- | :---: | :---: |
| **Return type** | `enable_if_t<cond, R> f(T)` | Low - clutters return | Avoid |
| **Extra template param** | `template<class T, enable_if_t<cond,int>=0>` | Medium | **OK (C++11/17)** |
| **requires clause** | `template<class T> requires cond` | **High** | **Best (C++20)** |

### Why `requires` Is Better

The table above points to `requires` as the winner, and here's why each advantage actually matters in practice:

- `requires` clauses are **always** in the immediate context, which means substitution failures there are never accidentally promoted to hard errors.
- Error messages name the failing constraint by concept, so you know exactly what went wrong.
- Concepts support **subsumption** - the compiler can use constraint relationships to resolve overload ambiguities that `enable_if` couldn't.
- The syntax is just cleaner than `enable_if` in any position.

---

## Self-Assessment

### Q1: Show that substitution of one template parameter can affect whether another is well-formed

The interesting part of this example isn't the individual functions - it's watching how the substitution chain propagates left to right, and how a failure at one link silently removes the whole overload:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// U depends on T: U defaults to T::value_type
// If T has no value_type, U's default is invalid - SFINAE
template <typename T, typename U = typename T::value_type>
U first_element(const T& container) {
    return *container.begin();
}

// Fallback
template <typename T>
T identity(const T& val) {
    return val;
}

// More complex dependency chain:
// T -> T::iterator -> iterator's value_type
template <typename T,
          typename Iter = typename T::iterator,
          typename Val = typename std::iterator_traits<Iter>::value_type>
Val get_first_v2(T& container) {
    return *container.begin();
}

// Show how substitution order creates dependencies:
// template <typename T, typename U = decay_t<T>, typename V = typename U::type>
// T is substituted first, then U (depends on T), then V (depends on U)
// Failure at any stage removes the overload

int main() {
    std::cout << "=== Template parameter dependency ===\n\n";

    std::vector<int> v{10, 20, 30};
    std::string s = "hello";

    // T=vector<int>, U=vector<int>::value_type=int - OK
    std::cout << "first_element(vector): " << first_element(v) << "\n";  // 10

    // T=string, U=string::value_type=char - OK
    std::cout << "first_element(string): " << first_element(s) << "\n";  // h

    // T=int: int::value_type doesn't exist - SFINAE removes first_element
    // Falls through to identity:
    std::cout << "identity(42):          " << identity(42) << "\n";       // 42

    // Chain dependency: T -> T::iterator -> value_type
    std::cout << "get_first_v2(vector):  " << get_first_v2(v) << "\n";   // 10

    std::cout << "\n=== Substitution order ===\n";
    std::cout << "Parameters are substituted left to right:\n";
    std::cout << "  template<T, U = T::value_type, V = U::nested>\n";
    std::cout << "  1. Substitute T\n";
    std::cout << "  2. Substitute U (depends on T) — failure here = SFINAE\n";
    std::cout << "  3. Substitute V (depends on U) — failure here = SFINAE\n";
    std::cout << "  If any step fails -> entire overload removed\n";

    return 0;
}
```

### Q2: Explain why `enable_if` in default template arguments is preferred over return type SFINAE for readability

The core issue is that return type SFINAE buries the actual return type inside a wall of `enable_if` boilerplate. This demo lines up all three approaches side by side so you can see the difference clearly:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === Version 1: enable_if in RETURN TYPE (hard to read) ===
template <typename T>
std::enable_if_t<std::is_integral_v<T>, T>
add_return(T a, T b) {
    return a + b;
}
// Problem: return type is cluttered with enable_if noise
// "What does this function return?" -> hard to tell at a glance

// === Version 2: enable_if as DEFAULT TEMPLATE ARGUMENT (better) ===
template <typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
T add_param(T a, T b) {
    return a + b;
}
// Better: return type is clearly T
// The enable_if is "out of the way" in the template parameter list

// === Version 3: requires clause (BEST — C++20) ===
template <typename T>
    requires std::is_integral_v<T>
T add_requires(T a, T b) {
    return a + b;
}
// Best: constraint is separate, return type is clean, reads naturally

// === More complex example showing the readability difference ===

// Return type SFINAE (confusing):
template <typename T>
std::enable_if_t<
    std::is_arithmetic_v<T> && !std::is_same_v<T, bool>,
    std::conditional_t<std::is_floating_point_v<T>, double, long long>
>
compute_return(T a, T b) { return a * b + a; }

// Default template arg (clearer):
template <typename T,
          std::enable_if_t<std::is_arithmetic_v<T> && !std::is_same_v<T, bool>, int> = 0>
auto compute_param(T a, T b) {
    if constexpr (std::is_floating_point_v<T>) return double(a * b + a);
    else return static_cast<long long>(a * b + a);
}

// requires (cleanest):
template <typename T>
    requires (std::is_arithmetic_v<T> && !std::is_same_v<T, bool>)
auto compute_requires(T a, T b) {
    if constexpr (std::is_floating_point_v<T>) return double(a * b + a);
    else return static_cast<long long>(a * b + a);
}

int main() {
    std::cout << "=== Readability comparison ===\n\n";

    std::cout << "Return SFINAE:    " << add_return(3, 4) << "\n";
    std::cout << "Default arg:      " << add_param(3, 4) << "\n";
    std::cout << "requires clause:  " << add_requires(3, 4) << "\n";

    std::cout << "\nComplex example:\n";
    std::cout << "Return SFINAE:   " << compute_return(3.5, 2.0) << "\n";
    std::cout << "Default arg:     " << compute_param(3.5, 2.0) << "\n";
    std::cout << "requires clause: " << compute_requires(3.5, 2.0) << "\n";

    std::cout << "\nRanking (readability):\n";
    std::cout << "  1. requires clause    -> return type clean, constraint separate\n";
    std::cout << "  2. Default template arg -> return type clean, enable_if hidden\n";
    std::cout << "  3. Return type SFINAE -> return type buried in enable_if\n";

    return 0;
}
```

### Q3: Show how `requires` clauses diagnose substitution failures with better messages than SFINAE

The SFINAE version gives you "no matching function" - which tells you nothing about *why*. The `requires` version names exactly which constraint failed. Named concepts go even further and can show you the failing sub-expression. Here's that three-way comparison:

```cpp
#include <iostream>
#include <concepts>
#include <string>
#include <vector>

// === SFINAE version — terrible error messages ===
template <typename T,
          std::enable_if_t<std::is_integral_v<T> && (sizeof(T) >= 4), int> = 0>
void process_sfinae(T val) {
    std::cout << "SFINAE: " << val << "\n";
}

// Calling process_sfinae(std::string("hi")) gives:
// error: no matching function for call to 'process_sfinae'
// note: candidate template ignored: requirement 'is_integral_v<string>' was not satisfied
// (Only on good compilers — older ones just say "no matching function")

// === requires version — excellent error messages ===
template <typename T>
    requires std::integral<T> && (sizeof(T) >= 4)
void process_requires(T val) {
    std::cout << "requires: " << val << "\n";
}

// Calling process_requires(std::string("hi")) gives:
// error: constraints not satisfied for function 'process_requires'
// note: because 'std::integral<std::string>' evaluated to false
// (Pinpoints EXACTLY which constraint failed)

// === Named concept — even better ===
template <typename T>
concept LargeInteger = std::integral<T> && (sizeof(T) >= 4);

void process_concept(LargeInteger auto val) {
    std::cout << "concept: " << val << "\n";
}

// Calling process_concept(short(1)) gives:
// error: constraints not satisfied for 'LargeInteger'
// note: because 'sizeof(short) >= 4' (2 >= 4) evaluated to false
// (Names the concept AND shows the failing sub-expression!)

// === Complex constraint with nested concepts ===
template <typename T>
concept Sortable = requires(T& t) {
    { t.begin() } -> std::input_or_output_iterator;
    { t.end() } -> std::input_or_output_iterator;
    { t.size() } -> std::convertible_to<std::size_t>;
    requires std::totally_ordered<typename T::value_type>;
};

template <Sortable T>
void sort_container(T& c) {
    std::cout << "Sorting container of " << c.size() << " elements\n";
}

int main() {
    std::cout << "=== requires vs SFINAE error quality ===\n\n";

    // These work:
    process_sfinae(42);
    process_requires(42);
    process_concept(42);

    // These would fail with different error quality:
    // process_sfinae(std::string("hi"));
    //   -> "no matching function" (unhelpful)
    //
    // process_requires(std::string("hi"));
    //   -> "constraints not satisfied: integral<string> is false" (helpful!)
    //
    // process_concept(short(1));
    //   -> "LargeInteger not satisfied: sizeof(short) >= 4 is false" (great!)

    std::cout << "\nError message comparison:\n\n";
    std::cout << "SFINAE:         'no matching function for call to process_sfinae'\n";
    std::cout << "                 (Why? What went wrong? No clue.)\n\n";
    std::cout << "requires:       'constraints not satisfied: integral<string> is false'\n";
    std::cout << "                 (Tells you EXACTLY which constraint failed.)\n\n";
    std::cout << "Named concept:  'LargeInteger not satisfied: sizeof(short) >= 4 is false'\n";
    std::cout << "                 (Names the concept AND shows the failing expression.)\n";

    std::vector<int> v{3, 1, 4, 1, 5};
    sort_container(v);  // OK

    return 0;
}
```

---

## Notes

- Template arguments are substituted **left to right** - later parameters can depend on earlier ones.
- `enable_if` in default template parameters is cleaner than in return types (return type stays readable).
- C++20 `requires` clauses are the best approach: clean syntax, great errors, subsumption support.
- `requires` expressions are always in the immediate context - no risk of accidental hard errors.
- Error message quality: named concept > requires clause > enable_if default arg > return type SFINAE.
