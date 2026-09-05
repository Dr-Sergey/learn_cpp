# Use AI to explain and document complex C++ template metaprogramming

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

Template metaprogramming (TMP) is one of the most complex areas of C++, and it's also one of the areas where AI assistants shine the brightest. The reason is simple: TMP code tends to be dense, symbolic, and heavily recursive - and AI is very good at tracing through that kind of mechanical logic and putting it into plain English. AI excels at **explaining** existing TMP code, **generating documentation** for template libraries, **simplifying** complex SFINAE patterns into C++20 concepts, and **teaching** TMP concepts with step-by-step instantiation traces. Code that used to be impenetrable becomes approachable when you have an AI walking you through the instantiation chain.

### AI TMP Assistance Areas

If you're wondering where AI is actually useful here versus where you should be skeptical, the table below is a good starting point. Explanation and tracing tasks are where AI is most reliable; writing new TMP from scratch is where you need to be most careful.

| Task | AI Effectiveness | Key Prompt Strategy |
| --- | --- | --- |
| Explain existing TMP code | Excellent | Paste code + "explain step by step" |
| Generate instantiation traces | Excellent | "Show how this instantiates for T=int" |
| Simplify SFINAE to concepts | Good | Paste SFINAE + target standard |
| Document template APIs | Excellent | Paste template + "document for users" |
| Debug template errors | Good | Paste full error + source |
| Write new TMP from scratch | Medium | Very specific requirements needed |

---

## Self-Assessment

### Q1: Use AI to explain complex TMP code

**Answer:**

The prompt below is how you ask for an explanation. The key technique is asking the AI to show an instantiation trace for a concrete type - that forces it to reason step by step rather than just describe the code abstractly.

```cpp
=== PROMPT ===

"Explain this template metaprogramming code step by step.
Show the instantiation trace for TypeList<int, double, string>.
Explain what each template does and why."
```

Here's the TMP library being explained. Notice how the templates compose: `TypeList` holds the types, `TypeAt` walks the list recursively, `Transform` maps a metafunction over the list, and `Filter` selects elements by predicate. Taken individually each piece is understandable; together they form a compile-time list manipulation toolkit.

```cpp
// Complex TMP code to explain:
template<typename... Ts> struct TypeList {};

// Get element at index
template<size_t N, typename List> struct TypeAt;
template<size_t N, typename Head, typename... Tail>
struct TypeAt<N, TypeList<Head, Tail...>>
    : TypeAt<N-1, TypeList<Tail...>> {};
template<typename Head, typename... Tail>
struct TypeAt<0, TypeList<Head, Tail...>> {
    using type = Head;
};

// Transform each type in a list
template<template<typename> class F, typename List>
struct Transform;
template<template<typename> class F, typename... Ts>
struct Transform<F, TypeList<Ts...>> {
    using type = TypeList<typename F<Ts>::type...>;
};

// Filter types matching a predicate
template<template<typename> class Pred, typename List>
struct Filter;
template<template<typename> class Pred>
struct Filter<Pred, TypeList<>> {
    using type = TypeList<>;
};
template<template<typename> class Pred, typename Head, typename... Tail>
struct Filter<Pred, TypeList<Head, Tail...>> {
    using Rest = typename Filter<Pred, TypeList<Tail...>>::type;
    using type = std::conditional_t<
        Pred<Head>::value,
        typename Prepend<Head, Rest>::type,
        Rest>;
};

// === AI EXPLANATION ===
//
// "This is a compile-time type list manipulation library.
//
// 1. TypeList<Ts...>: A variadic template that holds a
//    list of types. Like std::tuple but for types only.
//
// 2. TypeAt<N, List>: Gets the Nth type from a TypeList.
//    Works by recursion:
//    - TypeAt<2, TypeList<int,double,string>>
//    - -> TypeAt<1, TypeList<double,string>>     (N-1, drop Head)
//    - -> TypeAt<0, TypeList<string>>             (N-1, drop Head)
//    - -> type = string                           (N=0, take Head)
//
// 3. Transform<F, List>: Applies a metafunction F to each
//    type. Like std::transform but at compile time.
//    Transform<std::add_pointer, TypeList<int,double>>
//    -> TypeList<int*, double*>
//
// 4. Filter<Pred, List>: Keeps only types where Pred<T>
//    is true. Like std::copy_if at compile time.
//    Filter<std::is_integral, TypeList<int,double,char>>
//    -> TypeList<int, char>"
```

The instantiation trace for `TypeAt` is the most instructive part of the AI explanation. You can see exactly how C++ peels elements off the front of the list one at a time until `N` hits zero - at which point the base case fires and yields the answer. This is the standard recursive template technique, and once you see it traced once you'll recognize it everywhere.

### Q2: Generate documentation for template libraries

**Answer:**

The following prompt pattern works well for documenting templates because it asks for everything a user needs: parameter descriptions, constraints, examples with output, common error messages, and performance notes. Giving the AI a concrete checklist like this keeps it from writing vague fluff.

```cpp
=== PROMPT ===

"Generate user-facing documentation for this template
matrix class. Include:

- Brief description of each template parameter
- Constraints and requirements
- Usage examples with expected output
- Common error messages and what they mean
- Performance characteristics"
```

The class being documented uses a `requires` constraint to restrict `T` to arithmetic types, and encodes the matrix dimensions as non-type template parameters so mismatched multiplications fail at compile time rather than at runtime. The AI-generated documentation below captures all of that for a reader who just wants to use the class without reading the implementation.

```cpp
// Code to document:
template<typename T, size_t Rows, size_t Cols,
         MatrixLayout Layout = MatrixLayout::RowMajor>
    requires std::is_arithmetic_v<T>
class Matrix { /*...*/ };

// === AI-generated documentation ===

/// @brief Fixed-size matrix with compile-time dimensions.
///
/// @tparam T      Element type. Must be arithmetic (int, float,
///                double, etc.). Complex types not supported.
/// @tparam Rows   Number of rows (must be > 0).
/// @tparam Cols   Number of columns (must be > 0).
/// @tparam Layout Memory layout: RowMajor (default) or ColMajor.
///                RowMajor: elements in a row are contiguous.
///                ColMajor: elements in a column are contiguous.
///                Choose ColMajor for interop with Fortran/LAPACK.
///
/// ## Quick Start
/// ```cpp
/// // 3x3 identity matrix
/// Matrix<double, 3, 3> I = Matrix<double,3,3>::identity();
///
/// // Matrix multiplication (dimensions checked at compile time)
/// Matrix<float, 2, 3> A = {{1,2,3}, {4,5,6}};
/// Matrix<float, 3, 2> B = {{1,2}, {3,4}, {5,6}};
/// auto C = A * B;  // Matrix<float, 2, 2>
///
/// // Dimension mismatch = compile error
/// // auto bad = A * A;  // ERROR: 2x3 * 2x3 doesn't match
/// ```
///
/// ## Common Errors
/// | Error Message | Cause | Fix |
/// |---|---|---|
/// | "constraints not satisfied" | T is not arithmetic | Use int/float/double |
/// | "no match for operator*" | Dimension mismatch | Check Cols_A == Rows_B |
/// | "cannot convert" | Layout mismatch | Explicit convert or match layouts |
///
/// ## Performance
/// - Stack-allocated for small matrices (no heap)
/// - Dimensions known at compile time enable SIMD vectorization
/// - RowMajor: fast row iteration, slow column iteration
/// - ColMajor: fast column iteration, slow row iteration
```

Notice how the documentation includes both the happy path and the failure cases. The "Common Errors" table is especially valuable because C++ template error messages are notoriously hard to read - translating them into "you passed a non-arithmetic type" is genuinely useful for anyone using the library.

### Q3: Use AI to trace template instantiation

**Answer:**

Instantiation tracing is one of the most powerful things you can ask an AI to do with TMP code. The prompt below asks the AI to work through every overload tried, in order, with concrete types substituted in - exactly what the compiler does, but written in human-readable form.

```cpp
=== PROMPT ===

"Trace the template instantiation of this expression:
  auto result = serialize(std::vector<std::pair<int, std::string>>{});

Show every template that gets instantiated, in order,
with the concrete types substituted."
```

The serialization library below uses SFINAE to dispatch to different `serialize` overloads based on what the type supports. The important thing to notice is that SFINAE failures are not errors - they just remove a candidate from consideration, and the next overload gets tried. The AI trace makes this visible.

```cpp
// Serialization library with ADL + SFINAE
template<typename T>
auto serialize(const T& val) -> decltype(val.serialize()) {
    return val.serialize();  // Has .serialize() method
}

template<typename T>
auto serialize(const T& val)
    -> std::enable_if_t<std::is_arithmetic_v<T>, std::string> {
    return std::to_string(val);  // Arithmetic types
}

template<typename T>
std::string serialize(const std::vector<T>& vec) {
    std::string result = "[";
    for (size_t i = 0; i < vec.size(); ++i) {
        if (i > 0) result += ",";
        result += serialize(vec[i]);  // Recurse
    }
    return result + "]";
}

template<typename K, typename V>
std::string serialize(const std::pair<K, V>& p) {
    return "{" + serialize(p.first) + ":" + serialize(p.second) + "}";
}

// === AI INSTANTIATION TRACE ===
//
// serialize(std::vector<std::pair<int, std::string>>{})
//
// 1. Try overload 1: val.serialize() -> SFINAE fail
//    (vector has no .serialize())
// 2. Try overload 2: is_arithmetic_v<vector<...>> -> false
//    SFINAE fail
// 3. Try overload 3: serialize(const vector<T>&)
//    -> MATCH with T = pair<int, string>
//    Instantiates: serialize<pair<int,string>>(vector<...>)
//
// Inside the loop, calls serialize(vec[i]):
// 4. Try overload 1: pair has no .serialize() -> fail
// 5. Try overload 2: is_arithmetic_v<pair> -> false -> fail
// 6. Try overload 3: not a vector -> fail
// 7. Try overload 4: serialize(const pair<K,V>&)
//    -> MATCH with K=int, V=string
//
// For p.first (int):
// 8. Overload 2 matches: is_arithmetic_v<int> = true
//    Returns: std::to_string(42) -> "42"
//
// For p.second (string):
// 9. Overload 1: string has no .serialize() -> fail
// 10. Overload 2: is_arithmetic_v<string> -> false -> fail
// 11. No match! COMPILATION ERROR.
//     Need to add: std::string serialize(const std::string& s)
```

The trace reveals a real bug: there is no overload for `std::string`. The AI found this by following the instantiation chain all the way down to the leaf types - exactly the kind of systematic walkthrough that humans tend to skip when reading this kind of code. This is the reason asking for a full trace is so useful; bugs in TMP code often hide several recursion levels deep.

---

## Notes

- AI is exceptionally good at explaining existing TMP code - paste it and ask "what does this do?" and you'll usually get a clear, correct answer.
- For instantiation traces, always specify concrete types: "trace with `T=std::vector<int>`" gives a much more useful response than a generic trace.
- Ask AI to simplify TMP to C++20 concepts - the results are often dramatically cleaner and easier to maintain.
- AI can generate `static_assert` test cases for TMP: "verify these types pass/fail the trait" is a good prompt pattern.
- For Boost.Hana or Boost.Mp11, AI knows the idioms well and can explain or generate code effectively.
- When AI-generated TMP doesn't compile, paste the full error back and ask for a fix - the error message is crucial context.
- Use AI to generate tutorial progressions: "explain SFINAE in 5 steps, from basic to advanced" works surprisingly well as a learning exercise.
