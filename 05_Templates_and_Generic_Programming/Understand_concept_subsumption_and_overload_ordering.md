# Understand Concept Subsumption and Overload Ordering

**Category:** Templates and Generic Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constraints#Partial_ordering_of_constraints>  

---

## Topic Overview

### What Is Concept Subsumption

Concept **subsumption** is the mechanism that lets the compiler automatically pick the most specific overload when multiple constrained templates match. If concept A logically implies concept B - meaning anything that satisfies A must also satisfy B - then A **subsumes** B, and overloads constrained by A are preferred.

Think of it as a specialization order for constraints. The more constrained overload wins, as long as the compiler can prove that the constraint truly is stricter:

```cpp
std::integral -> std::regular -> std::copyable -> std::movable -> std::destructible
  (more constrained)                                         (less constrained)
```

When the compiler sees two viable overloads, it picks the **most constrained** one - the one whose constraints subsume the other's.

### How Subsumption Works

The compiler normalizes constraints into **atomic constraints** connected by `&&` and `||`. Concept A subsumes B if A's normal form logically implies B's normal form.

The reason this trips people up is the word "logically implies." The compiler does not do a semantic analysis of whether your concept is truly stricter - it works structurally, by checking whether A's normalized form contains all the atoms of B's normalized form. That is why you need to build concepts by referencing each other rather than independently writing the same `requires` clause twice:

```cpp
template <typename T> concept Movable  = /* ... */;
template <typename T> concept Copyable = Movable<T> && /* copy ops */;
template <typename T> concept Regular  = Copyable<T> && std::equality_comparable<T>;

// Regular subsumes Copyable (Regular = Copyable + more)
// Copyable subsumes Movable (Copyable = Movable + more)
```

### Rules for Subsumption to Work

If the table feels like a lot, the one rule that matters most is the last row: two independently written `requires` blocks that look identical do **not** subsume each other, even if they express the same constraint.

| Condition | Subsumes? | Why |
| --- | :---: | --- |
| `concept B = A<T> && extra` | B subsumes A | B includes all of A's atoms + more |
| `concept B = A<T> \|\| extra` | A subsumes B | A is stricter (B is looser) |
| `concept B` defined independently with same meaning | **No** | Must share atomic constraints by reference |
| `requires(T t) { t.x(); }` in two places | **No** | Each `requires` is a distinct atomic constraint |

**Key rule:** Subsumption only works when concepts **share** atomic constraints via the same concept definition. Two identical-looking `requires` expressions written independently do **not** subsume each other.

---

## Self-Assessment

### Q1: Show that a more constrained overload is preferred over a less constrained one in overload resolution

Here we build a three-level hierarchy: `Printable` at the bottom, `PrintableNumber` in the middle (adds `std::integral`), and `PrintableSignedNumber` at the top (adds `std::signed_integral`). Because each concept builds on the previous one, the compiler can rank them:

```cpp
#include <iostream>
#include <concepts>
#include <type_traits>
#include <string>

// Define a concept hierarchy via refinement
template <typename T>
concept Printable = requires(std::ostream& os, const T& t) {
    { os << t } -> std::same_as<std::ostream&>;
};

template <typename T>
concept PrintableNumber = Printable<T> && std::integral<T>;
// PrintableNumber subsumes Printable (it includes Printable + more)

template <typename T>
concept PrintableSignedNumber = PrintableNumber<T> && std::signed_integral<T>;
// PrintableSignedNumber subsumes PrintableNumber subsumes Printable

// Three overloads with increasing constraints:
template <Printable T>
std::string classify(const T&) { return "Printable (least constrained)"; }

template <PrintableNumber T>
std::string classify(const T&) { return "PrintableNumber (more constrained)"; }

template <PrintableSignedNumber T>
std::string classify(const T&) { return "PrintableSignedNumber (most constrained)"; }

int main() {
    std::cout << "=== Subsumption: most constrained wins ===\n\n";

    // std::string: only Printable matches
    std::cout << "string:       " << classify(std::string("hi")) << "\n";

    // unsigned int: Printable + PrintableNumber match -> PrintableNumber wins
    std::cout << "unsigned int: " << classify(42u) << "\n";

    // int: all three match -> PrintableSignedNumber wins (most constrained)
    std::cout << "int:          " << classify(42) << "\n";

    // short: signed integral -> PrintableSignedNumber
    std::cout << "short:        " << classify(short(1)) << "\n";

    // unsigned char: unsigned integral -> PrintableNumber
    std::cout << "unsigned char:" << classify(static_cast<unsigned char>(65)) << "\n";

    return 0;
}
```

Each call lands on the most specific overload that applies. The compiler knows `PrintableSignedNumber` is stricter than `PrintableNumber` because it literally includes `PrintableNumber<T>` in its definition - the atoms are shared.

**Expected output:**

```text
=== Subsumption: most constrained wins ===

string:       Printable (least constrained)
unsigned int: PrintableNumber (more constrained)
int:          PrintableSignedNumber (most constrained)
short:        PrintableSignedNumber (most constrained)
unsigned char:PrintableNumber (more constrained)
```

### Q2: Demonstrate the ambiguity when two overloads are equally constrained but neither subsumes the other

This is the classic pitfall with concepts. If you write two independent concepts that happen to be satisfied by the same type, the call is ambiguous - the compiler has no way to rank them:

```cpp
#include <iostream>
#include <concepts>
#include <cstddef>
#include <vector>

// Two concepts defined INDEPENDENTLY - no shared atomic constraints
template <typename T>
concept HasSize = requires(const T& t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

template <typename T>
concept HasEmpty = requires(const T& t) {
    { t.empty() } -> std::same_as<bool>;
};

// Neither subsumes the other - they share NO atomic constraints

// Overload 1: constrained by HasSize
template <HasSize T>
void inspect(const T& c) {
    std::cout << "HasSize: size = " << c.size() << "\n";
}

// Overload 2: constrained by HasEmpty
template <HasEmpty T>
void inspect(const T& c) {
    std::cout << "HasEmpty: empty = " << c.empty() << "\n";
}

// For a type that satisfies BOTH (e.g., std::vector), the call is AMBIGUOUS!
// Uncomment to see the error:
// std::vector<int> v;
// inspect(v);  // ERROR: ambiguous - neither HasSize nor HasEmpty subsumes the other

// Fix: create a combined concept that subsumes both
template <typename T>
concept HasSizeAndEmpty = HasSize<T> && HasEmpty<T>;
// This subsumes both HasSize AND HasEmpty

template <HasSizeAndEmpty T>
void inspect_fixed(const T& c) {
    std::cout << "HasSizeAndEmpty: size=" << c.size() << " empty=" << c.empty() << "\n";
}

// Now add the other two overloads alongside:
template <HasSize T>
void inspect_fixed(const T& c) requires (!HasEmpty<T>) {
    std::cout << "HasSize only: size = " << c.size() << "\n";
}

template <HasEmpty T>
void inspect_fixed(const T& c) requires (!HasSize<T>) {
    std::cout << "HasEmpty only: empty = " << c.empty() << "\n";
}

// Type with only size()
struct SizeOnly {
    int size() const { return 5; }
};

// Type with only empty()
struct EmptyOnly {
    bool empty() const { return true; }
};

int main() {
    std::cout << "=== Ambiguity from non-subsumption ===\n\n";

    std::cout << "Problem: vector satisfies both HasSize and HasEmpty,\n";
    std::cout << "but neither concept subsumes the other -> AMBIGUOUS\n\n";

    std::cout << "=== Fix: add a combined concept ===\n\n";

    std::vector<int> v{1, 2, 3};
    inspect_fixed(v);           // HasSizeAndEmpty: size=3 empty=false

    SizeOnly so;
    inspect_fixed(so);          // HasSize only: size = 5

    EmptyOnly eo;
    inspect_fixed(eo);          // HasEmpty only: empty = true

    return 0;
}
```

The fix - combining the two into `HasSizeAndEmpty` - works because that combined concept includes the atoms from both, so the compiler can prove it is strictly more constrained than either alone.

### Q3: Use concept subsumption to create a three-way dispatch: unconstrained < Copyable < Regular

The standard library concepts already form a proper subsumption chain. That means you can write a multi-level dispatch just by using them as constraints, and the compiler handles the ranking automatically:

```cpp
#include <iostream>
#include <concepts>
#include <string>
#include <memory>

// std::regular subsumes std::copyable
// std::copyable subsumes std::movable
// These are DEFINED in <concepts> with proper subsumption chains!

// Three-way dispatch using the standard concepts hierarchy:
template <typename T>
std::string dispatch(const T&) {
    return "unconstrained (fallback)";
}

template <std::movable T>
std::string dispatch(const T&) {
    return "movable (move-only)";
}

template <std::copyable T>
std::string dispatch(const T&) {
    return "copyable (copy + move)";
}

template <std::regular T>
std::string dispatch(const T&) {
    return "regular (copy + move + == + default-constructible)";
}

// Test types
struct MoveOnly {
    MoveOnly() = default;
    MoveOnly(MoveOnly&&) = default;
    MoveOnly& operator=(MoveOnly&&) = default;
    MoveOnly(const MoveOnly&) = delete;
    MoveOnly& operator=(const MoveOnly&) = delete;
};

struct CopyableType {
    CopyableType() = default;
    CopyableType(const CopyableType&) = default;
    CopyableType& operator=(const CopyableType&) = default;
    // No operator== -> not regular
};

struct RegularType {
    int value = 0;
    RegularType() = default;
    RegularType(const RegularType&) = default;
    RegularType& operator=(const RegularType&) = default;
    bool operator==(const RegularType&) const = default;
};

// A non-movable type
struct Immovable {
    Immovable() = default;
    Immovable(const Immovable&) = delete;
    Immovable(Immovable&&) = delete;
};

int main() {
    std::cout << "=== Three-way dispatch: unconstrained < movable < copyable < regular ===\n\n";

    // Subsumption chain: regular -> copyable -> movable -> unconstrained
    std::cout << "Immovable:    " << dispatch(Immovable{}) << "\n";
    std::cout << "MoveOnly:     " << dispatch(MoveOnly{}) << "\n";
    std::cout << "CopyableType: " << dispatch(CopyableType{}) << "\n";
    std::cout << "RegularType:  " << dispatch(RegularType{}) << "\n";
    std::cout << "int:          " << dispatch(42) << "\n";
    std::cout << "string:       " << dispatch(std::string("hi")) << "\n";

    std::cout << "\n=== Why this works ===\n";
    std::cout << "std::regular  subsumes std::copyable  (regular = copyable + == + default_init)\n";
    std::cout << "std::copyable subsumes std::movable   (copyable = movable + copy ops)\n";
    std::cout << "std::movable  subsumes unconstrained  (any constraint subsumes none)\n";
    std::cout << "Compiler picks most constrained viable overload automatically.\n";

    return 0;
}
```

This only works cleanly because the standard library concepts are defined to build on each other using the exact same atomic constraints. If you tried to replicate this with hand-written `enable_if` conditions, you would get ambiguity errors instead.

**Expected output:**

```text
=== Three-way dispatch: unconstrained < movable < copyable < regular ===

Immovable:    unconstrained (fallback)
MoveOnly:     movable (move-only)
CopyableType: copyable (copy + move)
RegularType:  regular (copy + move + == + default-constructible)
int:          regular (copy + move + == + default-constructible)
string:       regular (copy + move + == + default-constructible)
```

---

## Notes

- **Subsumption** = concept A logically implies concept B -> A's overload is preferred.
- Subsumption only works through **shared atomic constraints** - two independently written identical `requires` expressions do **not** subsume each other.
- Standard library concepts form a subsumption hierarchy: `regular` > `semiregular` > `copyable` > `movable` > `destructible`.
- If two overloads are constrained by concepts that don't share atoms, calling with a type that satisfies both is **ambiguous** - add a combined concept.
- The compiler normalizes constraints to conjunctive/disjunctive normal form for subsumption checking.
