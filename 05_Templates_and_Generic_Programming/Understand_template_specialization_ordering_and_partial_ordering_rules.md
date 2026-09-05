# Understand Template Specialization Ordering and Partial Ordering Rules

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/partial_specialization>  

---

## Topic Overview

### How the Compiler Chooses Among Specializations

When you have multiple specializations that all match a given set of template arguments, the compiler picks the most specific one using **partial ordering**. Think of it as a specificity contest - a specialization that matches a narrower set of types wins over one that matches a broader set. The priority order is:

```cpp
1. Full (explicit) specialization    — exact match
2. More specialized partial spec     — tighter pattern
3. Less specialized partial spec     — looser pattern
4. Primary template                  — catch-all
```

### Partial Ordering Algorithm

The reason this is called "partial" ordering is that not every pair of specializations can be compared - sometimes neither is more specific than the other, and you get an ambiguity error. For the cases that can be ordered, the rule is:

The compiler determines which specialization is "more specialized" by checking: can one's pattern be used as arguments for the other, **but not vice versa**?

```cpp
Is Spec-A an instance of Spec-B?  AND  Is Spec-B NOT an instance of Spec-A?
-> Spec-A is more specialized than Spec-B
```

In plain language: if every type that matches A also matches B, but not the reverse, then A is more specific.

### Ambiguity

If neither specialization is more specialized than the other, the compiler reports an **ambiguity error**. The fix is almost always to add a third specialization that covers the overlapping case and is more specific than both.

| Spec1 pattern | Spec2 pattern | For `<int*, int*>` |
|:--:|:--:|:--:|
| `<T*, U>` | `<T, U*>` | Ambiguous - neither subsumes the other |
| `<T*, T*>` | `<T*, U>` | `<T*,T*>` wins - more specialized |

---

## Self-Assessment

### Q1: Show that a more specialized partial specialization is preferred over a less specialized one

This example builds up a chain of specializations from most general to most specific. Watch how each new specialization adds a constraint, and notice that `<int*, int*>` triggers the full specialization - not any of the partial ones:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// Primary: general case
template <typename T, typename U>
struct TypePair {
    static std::string name() { return "Primary<T, U>"; }
};

// Partial: pointer + anything
template <typename T, typename U>
struct TypePair<T*, U> {
    static std::string name() { return "Partial<T*, U>"; }
};

// Partial: pointer + pointer (more specialized than T*, U)
template <typename T, typename U>
struct TypePair<T*, U*> {
    static std::string name() { return "Partial<T*, U*>"; }
};

// Partial: same pointer type (most specialized)
template <typename T>
struct TypePair<T*, T*> {
    static std::string name() { return "Partial<T*, T*>"; }
};

// Full: exact match
template <>
struct TypePair<int*, int*> {
    static std::string name() { return "Full<int*, int*>"; }
};

int main() {
    std::cout << "=== Specialization ordering ===\n\n";

    // TypePair<int, double>     -> Primary (no pointer)
    std::cout << "<int, double>:       " << TypePair<int, double>::name() << "\n";

    // TypePair<int*, double>    -> Partial<T*, U> (first is pointer)
    std::cout << "<int*, double>:      " << TypePair<int*, double>::name() << "\n";

    // TypePair<int*, double*>   -> Partial<T*, U*> (both pointers, different types)
    std::cout << "<int*, double*>:     " << TypePair<int*, double*>::name() << "\n";

    // TypePair<double*, double*> -> Partial<T*, T*> (both pointers, same type)
    std::cout << "<double*, double*>:  " << TypePair<double*, double*>::name() << "\n";

    // TypePair<int*, int*>      -> Full<int*, int*> (exact match)
    std::cout << "<int*, int*>:        " << TypePair<int*, int*>::name() << "\n";

    std::cout << "\nOrdering chain: Full > T*,T* > T*,U* > T*,U > Primary\n";

    return 0;
}
```

**Expected output:**

```text
=== Specialization ordering ===

<int, double>:       Primary<T, U>
<int*, double>:      Partial<T*, U>
<int*, double*>:     Partial<T*, U*>
<double*, double*>:  Partial<T*, T*>
<int*, int*>:        Full<int*, int*>
```

### Q2: Explain what happens when two partial specializations are equally specialized (ambiguity)

The two partial specializations below are each more specific than the primary template, but they're not more specific than each other - `<T*, U>` doesn't subsume `<T, U*>` and vice versa. The fix is to add a third specialization that covers their intersection:

```cpp
#include <iostream>
#include <string>

// Primary
template <typename T, typename U>
struct Grid {
    static std::string name() { return "Primary<T, U>"; }
};

// Partial A: first is pointer
template <typename T, typename U>
struct Grid<T*, U> {
    static std::string name() { return "Partial<T*, U>"; }
};

// Partial B: second is pointer
template <typename T, typename U>
struct Grid<T, U*> {
    static std::string name() { return "Partial<T, U*>"; }
};

// For Grid<int*, double*>:
//   Partial A matches: T=int, U=double*  (T*=int*, U=double*)
//   Partial B matches: T=int*, U=double  (T=int*, U*=double*)
//   Neither is more specialized! -> AMBIGUITY
//
// Uncomment to see the error:
// auto name = Grid<int*, double*>::name();
// Error: ambiguous partial specialization

// FIX: add a specialization that covers the overlap
template <typename T, typename U>
struct Grid<T*, U*> {
    static std::string name() { return "Partial<T*, U*> (resolves ambiguity)"; }
};

int main() {
    std::cout << "=== Ambiguity in partial specialization ===\n\n";

    std::cout << "<int, double>:     " << Grid<int, double>::name() << "\n";     // Primary
    std::cout << "<int*, double>:    " << Grid<int*, double>::name() << "\n";    // Partial A
    std::cout << "<int, double*>:    " << Grid<int, double*>::name() << "\n";    // Partial B

    // This WAS ambiguous, now resolved by the T*,U* specialization:
    std::cout << "<int*, double*>:   " << Grid<int*, double*>::name() << "\n";   // T*,U*

    std::cout << "\nAmbiguity happens when:\n";
    std::cout << "  - Two partial specs both match\n";
    std::cout << "  - Neither is more specialized than the other\n";
    std::cout << "  - No full specialization breaks the tie\n\n";
    std::cout << "Fix: add a specialization that covers the overlapping case.\n";

    return 0;
}
```

### Q3: Write three partial specializations of a template and predict which instantiation is chosen

Here's a nice example of recursive specialization selection - `const int*` triggers `Classify<T*>` with `T=const int`, which then triggers `Classify<const T>` with `T=int`, which triggers the full specialization for `int`. The result builds up from the innermost match outward:

```cpp
#include <iostream>
#include <string>
#include <vector>

// Primary template
template <typename T>
struct Classify {
    static std::string result() { return "generic type"; }
};

// Partial spec 1: pointer types
template <typename T>
struct Classify<T*> {
    static std::string result() { return "pointer to " + Classify<T>::result(); }
};

// Partial spec 2: const types
template <typename T>
struct Classify<const T> {
    static std::string result() { return "const " + Classify<T>::result(); }
};

// Partial spec 3: reference types
template <typename T>
struct Classify<T&> {
    static std::string result() { return "reference to " + Classify<T>::result(); }
};

// Full spec: int
template <>
struct Classify<int> {
    static std::string result() { return "int"; }
};

int main() {
    std::cout << "=== Predict which specialization is chosen ===\n\n";

    // int -> Full spec (exact match)
    std::cout << "int:           " << Classify<int>::result() << "\n";

    // double -> Primary (no match for any partial)
    std::cout << "double:        " << Classify<double>::result() << "\n";

    // int* -> Partial<T*> with T=int -> "pointer to int"
    std::cout << "int*:          " << Classify<int*>::result() << "\n";

    // const int -> Partial<const T> with T=int -> "const int"
    std::cout << "const int:     " << Classify<const int>::result() << "\n";

    // int& -> Partial<T&> with T=int -> "reference to int"
    std::cout << "int&:          " << Classify<int&>::result() << "\n";

    // const int* -> Partial<T*> with T=const int -> recursive!
    //   -> "pointer to " + Classify<const int> -> "pointer to const int"
    std::cout << "const int*:    " << Classify<const int*>::result() << "\n";

    // int** -> Partial<T*> with T=int* -> recursive!
    //   -> "pointer to " + Classify<int*> -> "pointer to pointer to int"
    std::cout << "int**:         " << Classify<int**>::result() << "\n";

    // const int*& -> Partial<T&> with T=const int*
    //   -> "reference to " + Classify<const int*>
    //   -> "reference to pointer to const int"
    std::cout << "const int*&:   " << Classify<const int*&>::result() << "\n";

    return 0;
}
```

**Expected output:**

```text
=== Predict which specialization is chosen ===

int:           int
double:        generic type
int*:          pointer to int
const int:     const int
int&:          reference to int
const int*:    pointer to const int
int**:         pointer to pointer to int
const int*&:   reference to pointer to const int
```

---

## Notes

- **Partial ordering**: the compiler checks if one specialization's pattern is a strict subset of another.
- Selection priority: full specialization > more specialized partial > less specialized partial > primary.
- **Ambiguity** occurs when two partial specializations both match and neither is more specialized. Fix by adding a specialization covering the overlap.
- Partial specializations can be recursive - `Classify<const int*>` -> `Classify<T*>` -> `Classify<const int>` -> `Classify<const T>` -> `Classify<int>`.
- For function templates, the compiler uses **partial ordering of function templates** (different rules than class template partial specialization).
