# Understand conversion sequences and overload resolution ranking

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/overload_resolution>  

---

## Topic Overview

When multiple overloaded functions match a call, C++ uses **overload resolution** to pick the
best match. The compiler ranks each candidate by the **implicit conversion sequence** needed
for each argument. Understanding this ranking is what lets you predict - or diagnose - which
overload gets called.

### Conversion Sequence Ranking (Best to Worst)

If the table feels like a lot, it boils down to: the compiler prefers doing less work.
An exact match beats a small widening, which beats any other built-in conversion, which
beats a user-defined type conversion, which beats the catch-all `...`.

| Rank | Category | Examples |
| --- | --- | --- |
| 1 | **Exact match** | No conversion; lvalue-to-rvalue; array/function-to-pointer; cv-qualification |
| 2 | **Promotion** | `short` -> `int`, `float` -> `double`, `bool` -> `int` |
| 3 | **Standard conversion** | `int` -> `double`, `double` -> `int`, pointer conversions, `Derived*` -> `Base*` |
| 4 | **User-defined conversion** | Conversion constructor, conversion operator |
| 5 | **Ellipsis** | `...` (C-style varargs) |

A candidate with a **better rank for at least one argument** and **no worse rank for all
other arguments** wins. If no single candidate is strictly better, the call is **ambiguous**.

### Exact Match Examples

Adding `const` to a reference is considered a trivial qualification adjustment - it still
counts as an exact match:

```cpp
void f(int x);
f(42);           // exact match - int -> int

void g(const int& x);
int a = 5;
g(a);            // exact match - adding const to reference is trivial qualification
```

### Promotion Examples

Promotions are widening conversions that are always value-preserving. The compiler prefers
them over other standard conversions because they never lose information:

```cpp
void f(int x);
void f(double x);

f(short{5});     // short -> int (promotion, rank 2) beats short -> double (conversion, rank 3)
f(true);         // bool -> int (promotion) beats bool -> double (conversion)
f(3.14f);        // float -> double (promotion) - calls f(double)
```

### Standard Conversion Examples

Standard conversions cover the rest of the built-in conversions - including ones that can
lose data, like `double` to `int`:

```cpp
void f(double x);
f(42);           // int -> double - standard conversion (rank 3)

void g(Base* p);
Derived* d = ...;
g(d);            // Derived* -> Base* - standard conversion (pointer conversion)
```

### User-Defined Conversion

A converting constructor or conversion operator is rank 4 - the compiler will only consider
it if no built-in path is available. At most one user-defined conversion is allowed per
implicit conversion sequence:

```cpp
struct MyInt {
    MyInt(int x) : val(x) {}    // converting constructor (implicit)
    int val;
};
void f(MyInt m);
f(42);           // int -> MyInt via converting constructor - user-defined conversion (rank 4)
```

### Ambiguity

When two candidates tie across all arguments, the compiler refuses to guess. This is the
right behavior - it forces you to be explicit:

```cpp
void f(int, double);
void f(double, int);

f(1, 1);         // AMBIGUOUS!
// For f(int, double):  arg1 exact match, arg2 int->double (conversion)
// For f(double, int):  arg1 int->double (conversion), arg2 exact match
// Neither is strictly better -> compiler error
```

---

## Self-Assessment

### Q1: Explain the ranking: exact match > promotion > standard conversion > user-defined conversion

**Answer:**

The compiler evaluates each argument's conversion independently:

1. **Exact match (rank 1):** No conversion needed, or trivial adjustments (adding `const`, array-to-pointer decay). This is the best possible match.

2. **Promotion (rank 2):** Widening conversions defined by the standard that preserve the value exactly: `short`/`char` -> `int`, `float` -> `double`. These are preferred because they are always lossless.

3. **Standard conversion (rank 3):** Any other built-in conversion: `int` -> `double`, `double` -> `int` (may lose precision!), pointer conversions (`Derived*` -> `Base*`), boolean conversions.

4. **User-defined conversion (rank 4):** Requires calling a converting constructor or conversion operator. At most one user-defined conversion is allowed per sequence.

5. **Ellipsis match (rank 5):** C `...` varargs - worst match, rarely used in modern C++.

Here you can see promotion winning over a standard conversion for the same `short` argument:

```cpp
#include <iostream>

void process(int x)    { std::cout << "exact\n"; }    // rank 1 for int
void process(double x) { std::cout << "standard\n"; } // rank 3 for int

int main() {
    short s = 5;
    process(s);    // short -> int (promotion, rank 2) beats short -> double (rank 3)
                   // Output: "exact"

    process(42);   // exact match for int
                   // Output: "exact"

    process(3.14); // exact match for double
                   // Output: "standard"
}
```

### Q2: Show a case where two overloads are equally good and the compiler reports an ambiguity

Both `int -> long` and `int -> double` are rank-3 standard conversions - neither is better
than the other, so the compiler refuses to pick:

```cpp
#include <iostream>

void f(long x)   { std::cout << "long\n"; }
void f(double x) { std::cout << "double\n"; }

// Both int->long and int->double are standard conversions (rank 3)
// Neither is better -> AMBIGUOUS

void g(int, double) { std::cout << "g(int,double)\n"; }
void g(double, int) { std::cout << "g(double,int)\n"; }

int main() {
    // f(42);     // ERROR: ambiguous - int->long and int->double are both rank 3

    // Fix 1: explicit cast
    f(static_cast<long>(42));     // OK: exact match for long overload

    // Fix 2: add an int overload
    // void f(int x);   // would be exact match

    // g(1, 1);  // ERROR: ambiguous
    //   g(int,double): arg1=exact, arg2=int->double (rank 3)
    //   g(double,int): arg1=int->double (rank 3), arg2=exact
    //   Neither dominates the other

    // Fix: make one argument unambiguous
    g(1, 2.0);    // OK: g(int,double) - both args exact/promotion
}
```

**How it works:**

- The compiler reports "call to `f` is ambiguous" when two candidates have equally-ranked conversion sequences.
- Fixes: add an exact-match overload, use explicit casts, or change argument types.

### Q3: Demonstrate how explicit constructors are excluded from implicit conversion sequences

`explicit` on a constructor pulls it out of the overload resolution process for implicit
conversions entirely - the compiler won't even consider it as a rank-4 user-defined
conversion. This is what prevents accidental "I meant to pass a count but my class took it
as data" style bugs:

```cpp
#include <iostream>
#include <string>

struct Implicit {
    Implicit(int x) : val(x) {}     // implicit converting constructor
    int val;
};

struct Explicit {
    explicit Explicit(int x) : val(x) {}   // explicit - blocks implicit conversions
    int val;
};

void take_implicit(Implicit m) { std::cout << "Implicit: " << m.val << "\n"; }
void take_explicit(Explicit e) { std::cout << "Explicit: " << e.val << "\n"; }

int main() {
    // Implicit constructor participates in overload resolution:
    take_implicit(42);            // OK: int -> Implicit via implicit constructor
    Implicit a = 42;              // OK: copy-initialization with implicit conversion

    // Explicit constructor does NOT participate:
    // take_explicit(42);         // ERROR: no implicit conversion from int to Explicit
    // Explicit b = 42;           // ERROR: copy-initialization requires implicit conversion

    // Must use direct-initialization:
    take_explicit(Explicit{42});  // OK: direct-initialization
    Explicit c{42};               // OK: direct-initialization
    Explicit d(42);               // OK: direct-initialization

    // Why this matters for overload resolution:
    // If a function has overloads for Implicit and Explicit,
    // passing an int will ONLY match the Implicit overload.
}
```

**How it works:**

- `explicit` on a constructor removes it from the set of implicit conversion sequences entirely.
- The compiler never considers `explicit Explicit(int)` when trying to convert `int` -> `Explicit` implicitly.
- This prevents accidental conversions (e.g., `std::vector<int> v = 42;` is blocked because `vector(size_type)` is `explicit`).

---

## Notes

- When two conversions have the same rank, additional tiebreakers apply: e.g., `const` vs non-`const` reference binding, derived-to-base conversion depth.
- Template overloads are considered after non-template overloads when both are exact matches (non-template wins).
- `std::string(const char*)` is not explicit, which is why `std::string s = "hello";` works (user-defined conversion from `const char*`).
- Use `explicit` on single-argument constructors by default to avoid surprises in overload resolution.
