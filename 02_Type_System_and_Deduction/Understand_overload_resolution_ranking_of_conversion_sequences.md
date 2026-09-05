# Understand Overload Resolution: Ranking of Conversion Sequences

**Category:** Type System and Deduction  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/overload_resolution>  

---

## Topic Overview

### What Is Overload Resolution

When a function call matches multiple overloaded functions, the compiler performs **overload resolution** to select the best candidate. The core mechanism is **ranking conversion sequences** - how well each argument converts to each candidate's parameter type. If you've ever been surprised by which overload the compiler chose, this is the topic that explains it.

### The Five Conversion Ranks (Best to Worst)

| Rank | Name | Examples |
| --- | --- | --- |
| **1** | Exact match | No conversion, lvalue-to-rvalue, array-to-pointer, function-to-pointer, top-level cv qualification |
| **2** | Promotion | `short` -> `int`, `float` -> `double`, `bool` -> `int`, `char` -> `int` |
| **3** | Standard conversion | `int` -> `double`, `double` -> `int`, pointer conversions, `int` -> `bool` |
| **4** | User-defined conversion | Conversion constructors, `operator T()` conversions |
| **5** | Ellipsis | Match via `...` parameter |

The compiler picks the candidate whose conversion sequence has the **best (lowest) rank**. If there's a tie, additional tiebreakers apply.

### Exact Match Sub-Ranking

Not all exact matches are equal. Within the exact-match category:

| Sub-rank | Description |
| --- | --- |
| Identity | No conversion at all (`int` -> `int`) |
| Lvalue-to-rvalue | Read value from variable |
| Array-to-pointer | `int[10]` -> `int*` |
| Function-to-pointer | `void f()` -> `void(*f)()` |
| Qualification adjustment | `int*` -> `const int*` |

### Standard Conversion Sequence Structure

A full standard conversion sequence has up to three steps:

```cpp
[lvalue transform] -> [numeric conversion/promotion] -> [qualification adjustment]
     (optional)              (at most one)                   (optional)
```

Example: `char[]` -> `const int*`

1. Array-to-pointer: `char[]` -> `char*`
2. Standard conversion: `char*` -> `int*` (not valid - just illustrative)
3. Qualification: `int*` -> `const int*`

### Tiebreaking Rules

When two candidates have the same rank, the compiler uses these tiebreakers (in order):

```cpp
1. Subset of cv-qualification: int* beats const int* for non-const argument
2. Derived-to-base: closer base wins
3. Reference binding: non-const ref preferred for non-const lvalue
4. Same rank but one is promotion vs conversion: promotion wins
5. Direct reference binding beats indirect
```

### How the Compiler Selects the Best Viable Function

```cpp
Step 1: Name lookup -> find candidate set
Step 2: Filter to viable functions (argument count matches, each arg convertible)
Step 3: Rank each argument's conversion sequence
Step 4: Compare candidates pairwise:

        - Candidate A is BETTER than B if:

          For ALL arguments, A's conversion is no worse than B's
          AND for AT LEAST ONE argument, A's conversion is strictly better
Step 5: If exactly one candidate is better than all others -> selected
        If no single best exists -> AMBIGUOUS -> compile error
```

The "better for all, strictly better for one" rule is why multi-parameter overloads can go ambiguous even when each individual argument seems to have a clear winner.

### Promotion vs Standard Conversion

This distinction catches many people off guard. `float` -> `double` is a **promotion** (rank 2), but `int` -> `long` is a **standard conversion** (rank 3) - even though both feel like "widening":

```cpp
void f(int);      // #1
void f(double);   // #2

f(3.14f);  // float -> double is PROMOTION (rank 2)
           // float -> int is STANDARD CONVERSION (rank 3)
           // Result: #2 wins (promotion beats conversion)

void g(long);     // #1
void g(double);   // #2

g(42);     // int -> long is STANDARD CONVERSION (rank 3)
           // int -> double is STANDARD CONVERSION (rank 3)
           // Same rank, no tiebreaker -> AMBIGUOUS!
```

The reason `int` -> `long` is not a promotion comes down to the standard's exact definition: promotions are a specific fixed set (`char/short/bool` -> `int`, `float` -> `double`), and `int` -> `long` is not in it.

### Const and Reference Overloading

Reference-qualified overloads are resolved based on value category and constness of the object being called on:

```cpp
void f(int&);        // #1: non-const lvalue ref
void f(const int&);  // #2: const lvalue ref
void f(int&&);       // #3: rvalue ref

int x = 42;
const int cx = 10;

f(x);            // #1 wins: non-const lvalue binds better to int& than const int&
f(cx);           // #2 wins: const lvalue can only bind to const int&
f(42);           // #3 wins: rvalue binds better to int&& than const int&
f(std::move(x)); // #3 wins: xvalue (rvalue) prefers int&&
```

### User-Defined Conversions

When a user-defined conversion gets you to the parameter type, the second conversion step still matters for ranking within the user-defined tier:

```cpp
struct A {
    operator int() const { return 42; }  // user-defined conversion to int
};

void f(int);    // #1
void f(double); // #2

A a;
f(a);  // A -> int via operator int() (user-defined + identity)
       // A -> double via operator int() + int->double (user-defined + conversion)
       // Both require user-defined conversion, but #1 has shorter second step
       // Result: #1 wins
```

**Key rule:** At most **one** user-defined conversion is allowed per conversion sequence. Two user-defined conversions in a chain are **never** attempted.

---

## Self-Assessment

### Q1: Explain the ranking: exact match > promotion > standard conversion > user-defined conversion > ellipsis

Work through the output predictions before running this. The `compete` calls at the end are the most instructive - they show all three tiers racing against each other:

```cpp
#include <iostream>
#include <string>

// Demonstrating each rank level

// --- Rank 1: Exact match ---
void rank_demo(int x)            { std::cout << "exact match (int)\n"; }

// --- Rank 2: Promotion ---
void promo_demo(int x)           { std::cout << "promotion: short->int\n"; }
void promo_demo(double x)        { std::cout << "promotion: float->double\n"; }

// --- Rank 3: Standard conversion ---
void std_conv(double x)          { std::cout << "standard conversion: int->double\n"; }

// --- Rank 4: User-defined conversion ---
struct MyInt {
    int val;
    MyInt(int v) : val(v) {}       // converting constructor (user-defined)
};
void user_conv(MyInt m)           { std::cout << "user-defined: int->MyInt\n"; }

// --- Rank 5: Ellipsis ---
void ellipsis_demo(...)           { std::cout << "ellipsis match\n"; }

// Competition among ranks:
void compete(int x)               { std::cout << "  compete(int)    — exact\n"; }
void compete(double x)            { std::cout << "  compete(double) — standard conv\n"; }
void compete(...)                  { std::cout << "  compete(...)    — ellipsis\n"; }

int main() {
    std::cout << "=== Rank demonstrations ===\n";

    // Exact match (rank 1)
    rank_demo(42);           // int -> int: identity

    // Promotion (rank 2)
    short s = 5;
    promo_demo(s);           // short -> int: promotion (NOT conversion)
    float fl = 1.5f;
    promo_demo(fl);          // float -> double: promotion

    // Standard conversion (rank 3)
    std_conv(42);            // int -> double: standard conversion

    // User-defined conversion (rank 4)
    user_conv(42);           // int -> MyInt via converting constructor

    // Ellipsis (rank 5)
    ellipsis_demo("anything");  // anything matches ...

    std::cout << "\n=== Competition ===\n";
    std::cout << "compete(42):\n";
    compete(42);             // exact match wins
    std::cout << "compete(3.14):\n";
    compete(3.14);           // exact match (double->double)
    std::cout << "compete(\"hello\"):\n";
    compete("hello");        // const char* -> no match for int or double -> ellipsis

    return 0;
}
```

**Output:**

```text
=== Rank demonstrations ===
exact match (int)
promotion: short->int
promotion: float->double
standard conversion: int->double
user-defined: int->MyInt
ellipsis match

=== Competition ===
compete(42):
  compete(int)    — exact
compete(3.14):
  compete(double) — standard conv
compete("hello"):
  compete(...)    — ellipsis
```

**Summary of the ranking hierarchy:**

1. **Exact match** - no conversion or only trivial adjustments (lvalue-to-rvalue, array decay, cv-qualification)
2. **Promotion** - widening within the same "family" (`short`->`int`, `float`->`double`) as defined by the standard
3. **Standard conversion** - numeric conversions across families (`int`->`double`, pointer conversions, `bool` conversions)
4. **User-defined conversion** - conversion constructors or `operator T()` - at most one per sequence
5. **Ellipsis** - last resort, matches anything but is the worst rank

The compiler always picks the highest-ranked (lowest number) viable conversion. Within the same rank, tiebreaking rules apply (cv-qualification, reference binding, etc.).

### Q2: Show a case where two overloads are equally ranked and the call is ambiguous

The commented-out calls are the real content here - they show you the three classic patterns that produce ambiguity errors. The fix at the bottom is the practical takeaway:

```cpp
#include <iostream>

// Case 1: Two standard conversions of the same rank
void f(long x)   { std::cout << "f(long)\n"; }
void f(double x) { std::cout << "f(double)\n"; }

// Case 2: Two user-defined conversions, equally ranked
struct A {};
struct B {};
struct C {
    operator A() const { return A{}; }
    operator B() const { return B{}; }
};
void g(A) { std::cout << "g(A)\n"; }
void g(B) { std::cout << "g(B)\n"; }

// Case 3: Multiple parameters, mixed ranking
void h(int, double)  { std::cout << "h(int, double)\n"; }
void h(double, int)  { std::cout << "h(double, int)\n"; }

int main() {
    // Case 1: int->long and int->double are both standard conversions (rank 3)
    // Neither is a promotion (int->long is NOT a promotion, it's a conversion!)
    // f(42);  // AMBIGUOUS: equally ranked standard conversions

    // Case 2: C->A and C->B are both user-defined conversions (rank 4)
    C c;
    // g(c);   // AMBIGUOUS: two equally-ranked user-defined conversions

    // Case 3: h(1, 2)
    // Candidate h(int, double): arg1=exact, arg2=int->double (standard conv)
    // Candidate h(double, int): arg1=int->double (standard conv), arg2=exact
    // Neither is better for ALL arguments -> AMBIGUOUS
    // h(1, 2);  // AMBIGUOUS

    // All three calls above are commented out because they cause compile errors.
    // Uncommenting any one will produce an error like:
    //   error: call of overloaded 'f(int)' is ambiguous

    // RESOLUTION strategies:
    // 1. Explicit cast
    f(static_cast<long>(42));    // Forces f(long)
    f(static_cast<double>(42));  // Forces f(double)

    // 2. Add a better overload
    // void f(int x) { ... }  // Exact match would win

    // 3. Use explicit template arguments (if applicable)

    std::cout << "Ambiguity resolved with explicit casts.\n";
    return 0;
}
```

**Output:**

```text
f(long)
f(double)
Ambiguity resolved with explicit casts.
```

**How this works:**

- **Case 1:** `int` -> `long` and `int` -> `double` are both standard conversions (rank 3). Since `int` -> `long` is NOT a promotion (only `short` -> `int` is), both are tied, causing ambiguity.
- **Case 2:** Both `operator A()` and `operator B()` are user-defined conversions of equal rank. The compiler cannot prefer one over the other.
- **Case 3:** With multiple parameters, one candidate must be **at least as good on ALL arguments AND strictly better on AT LEAST ONE**. When each candidate wins on different arguments, neither dominates, causing ambiguity.
- **Fix:** Use explicit casts, add an exact-match overload, or restructure the overload set.

### Q3: Demonstrate how adding a const overload resolves an ambiguity between lvalue and rvalue overloads

The three-overload pattern (`&`, `const &`, `&&`) is worth understanding deeply because you'll encounter it in standard library types. Without all three, some call sites simply don't compile:

```cpp
#include <iostream>
#include <string>

// Scenario: a container with element access
class Container {
    std::string data_ = "Hello";
public:
    // Without const overload, we might have:
    //   std::string& get()       — for mutable access
    //   std::string  get() &&    — for move from rvalue
    // But what about: const Container& c; c.get()?
    // Neither overload works for const objects!

    // Solution: add all three overloads
    std::string&       get() &        { std::cout << "  non-const lvalue\n"; return data_; }
    const std::string& get() const &  { std::cout << "  const lvalue\n"; return data_; }
    std::string        get() &&       { std::cout << "  rvalue (move)\n"; return std::move(data_); }
};

// Another example: operator<< overloading
struct Logger {
    // Without const version: void print(std::string&) can't take temporaries
    // Adding const ref resolves it:
    void print(std::string& s)       { std::cout << "  mutable ref: " << s << "\n"; }
    void print(const std::string& s) { std::cout << "  const ref: " << s << "\n"; }
};

// CV-qualification tiebreaker example
void process(int* p)       { std::cout << "  process(int*)\n"; }
void process(const int* p) { std::cout << "  process(const int*)\n"; }

int main() {
    std::cout << "=== Container ref-qualified overloads ===\n";

    Container c;
    const Container& cc = c;

    c.get();                    // non-const lvalue — calls &
    cc.get();                   // const lvalue — calls const &
    Container{}.get();          // rvalue — calls &&
    std::move(c).get();         // rvalue — calls &&

    std::cout << "\n=== Logger const resolution ===\n";

    Logger log;
    std::string msg = "hello";

    log.print(msg);             // non-const lvalue -> print(string&)
    log.print("world");         // temporary -> const string& (can't bind to string&)

    std::cout << "\n=== Pointer cv-qualification tiebreaker ===\n";

    int x = 42;
    const int cx = 10;

    process(&x);   // int* -> int*: exact match
                    // int* -> const int*: qualification conversion
                    // int* wins: less cv-qualified is preferred for non-const
    process(&cx);  // const int* -> const int*: exact match
                    // const int* -> int*: NOT valid (can't drop const)
                    // const int* wins: it's the only viable candidate

    return 0;
}
```

**Output:**

```text
=== Container ref-qualified overloads ===
  non-const lvalue
  const lvalue
  rvalue (move)
  rvalue (move)

=== Logger const resolution ===
  mutable ref: hello
  const ref: world

=== Pointer cv-qualification tiebreaker ===
  process(int*)
  process(const int*)
```

**How this works:**

- **Ref-qualified overloads** (`&`, `const &`, `&&`) let the compiler choose based on the value category and constness of the object being called on
- Without the `const &` overload, calling `.get()` on a const object would fail - neither `&` (requires non-const) nor `&&` (requires rvalue) would match
- **CV-qualification tiebreaker:** When both `T*` and `const T*` match, the compiler prefers the less cv-qualified version for non-const arguments (closer match)
- For const arguments, only the `const` overload is viable, so there's no ambiguity
- This three-overload pattern (`&`, `const &`, `&&`) is common in modern C++ containers and smart pointers

---

## Notes

- **Implicit conversions are limited to one user-defined step.** `A -> B -> C` with two user-defined conversions is **never** attempted. This prevents combinatorial explosion and surprising implicit chains.
- **Narrowing conversions** (e.g., `double` -> `int`) are standard conversions that succeed in overload resolution but may generate warnings. In braced initialization, they are errors.
- **Template vs non-template:** When a template instantiation and a non-template function are equally ranked, the **non-template wins**.
- **Deleted functions participate in overload resolution.** A deleted overload can still be selected - it then causes a hard compile error (not SFINAE).
- Memorize the promotions: `char/short/bool` -> `int`, `float` -> `double`. Everything else between arithmetic types is a standard conversion.
- Use `-Wconversion` (GCC/Clang) to catch surprising implicit conversions.
