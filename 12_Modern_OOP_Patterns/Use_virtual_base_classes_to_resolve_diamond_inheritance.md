# Use virtual base classes to resolve diamond inheritance

**Category:** Modern OOP Patterns  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/derived_class#Virtual_base_classes>  

---

## Topic Overview

The **diamond problem** occurs when a class inherits from two classes that share a common base. Without `virtual` inheritance, the common base is duplicated - its members exist twice, causing ambiguity. `virtual` base classes ensure a single shared instance of the common base.

The reason this trips people up is that the ambiguity is structural, not just a naming conflict you can resolve with a cast. Two copies of `Animal` literally exist in the object, with separate data, and the compiler cannot know which one you mean when you write `bat.name`. The `virtual` keyword tells both intermediate classes to share the same base subobject rather than each creating their own copy.

```cpp
WITHOUT virtual:                     WITH virtual:

    Animal                               Animal (single instance)
    /    \                               /    \
 Mammal  WingedAnimal              Mammal(v)  WingedAnimal(v)
    \    /                               \    /
     Bat                                  Bat
  (TWO copies of Animal!)           (ONE copy of Animal)
```

### Memory Layout

| Inheritance | `sizeof(Bat)` includes... |
| --- | --- |
| Non-virtual | `Animal` (copy 1) + `Mammal` data + `Animal` (copy 2) + `WingedAnimal` data + `Bat` data |
| Virtual | `Animal` (shared) + `Mammal` data + vbptr + `WingedAnimal` data + vbptr + `Bat` data |

Virtual base classes add a **virtual base pointer (vbptr)** in each intermediate class, pointing to the shared base subobject.

---

## Self-Assessment

### Q1: Show a diamond inheritance ambiguity and fix it using virtual base classes

In the non-virtual version, constructing a `Bat` constructs `Animal` twice - once through `Mammal` and once through `WingedAnimal`. Both copies get the same name string, but they are genuinely separate subobjects, so accessing `bat.name` is ambiguous:

```cpp
#include <iostream>
#include <string>

// === PROBLEM: Diamond WITHOUT virtual ===
namespace bad {
    struct Animal {
        std::string name;
        Animal(std::string n) : name(std::move(n)) {
            std::cout << "  Animal(" << name << ") constructed\n";
        }
    };

    struct Mammal : Animal {
        Mammal(std::string n) : Animal(std::move(n)) {}
    };

    struct WingedAnimal : Animal {
        WingedAnimal(std::string n) : Animal(std::move(n)) {}
    };

    struct Bat : Mammal, WingedAnimal {
        Bat(std::string n)
            : Mammal(n), WingedAnimal(n) {}
        // TWO Animal subobjects! Mammal::Animal and WingedAnimal::Animal
    };
}

// === FIX: Diamond WITH virtual ===
namespace good {
    struct Animal {
        std::string name;
        Animal(std::string n = "") : name(std::move(n)) {
            std::cout << "  Animal(" << name << ") constructed\n";
        }
    };

    struct Mammal : virtual Animal {    // <- virtual!
        Mammal(std::string n) : Animal(std::move(n)) {}
    };

    struct WingedAnimal : virtual Animal {  // <- virtual!
        WingedAnimal(std::string n) : Animal(std::move(n)) {}
    };

    struct Bat : Mammal, WingedAnimal {
        // Most-derived class MUST initialize the virtual base directly!
        Bat(std::string n)
            : Animal(n),          // <- Bat initializes Animal
              Mammal(n),
              WingedAnimal(n) {}
        // ONE Animal subobject -- no ambiguity!
    };
}

int main() {
    std::cout << "=== Without virtual (TWO Animal copies) ===\n";
    bad::Bat b1("Batty");
    // b1.name;  // ERROR: ambiguous -- which Animal's name?
    std::cout << "Mammal::name = " << b1.Mammal::name << "\n";
    std::cout << "WingedAnimal::name = " << b1.WingedAnimal::name << "\n";

    std::cout << "\n=== With virtual (ONE Animal copy) ===\n";
    good::Bat b2("Batty");
    std::cout << "name = " << b2.name << "\n";  // unambiguous!
}
// Expected output:
//   === Without virtual (TWO Animal copies) ===
//     Animal(Batty) constructed
//     Animal(Batty) constructed
//   Mammal::name = Batty
//   WingedAnimal::name = Batty
//
//   === With virtual (ONE Animal copy) ===
//     Animal(Batty) constructed
//   name = Batty
```

**Notice:** Without `virtual`, `Animal` is constructed **twice**. With `virtual`, only **once**.

---

### Q2: Explain the construction order of virtual bases and why they are constructed by the most-derived class

This is the part that surprises people most. Each intermediate class specifies `Base(v)` in its initializer list - but when those classes are not the most-derived class, those initializers are silently ignored. The most-derived class is the only one whose initializer for the virtual base actually runs:

```cpp
#include <iostream>

struct Base {
    int value;
    Base(int v) : value(v) {
        std::cout << "1. Base(" << v << ") constructed\n";
    }
};

struct Left : virtual Base {
    Left(int v) : Base(v) {  // Base(v) IGNORED when Left is not most-derived!
        std::cout << "2. Left constructed\n";
    }
};

struct Right : virtual Base {
    Right(int v) : Base(v) {  // Base(v) IGNORED when Right is not most-derived!
        std::cout << "3. Right constructed\n";
    }
};

struct Bottom : Left, Right {
    // Bottom is the most-derived class -> IT initializes Base
    Bottom(int v) : Base(v * 10),  // <- THIS is the one that actually runs!
                    Left(v),        // Left's Base(v) initializer is IGNORED
                    Right(v) {      // Right's Base(v) initializer is IGNORED
        std::cout << "4. Bottom constructed\n";
    }
};

int main() {
    Bottom b(5);

    std::cout << "\nBase::value = " << b.value << "\n";
    // value is 50 (from Bottom's Base(v*10)), NOT 5!
}
// Expected output:
//   1. Base(50) constructed
//   2. Left constructed
//   3. Right constructed
//   4. Bottom constructed
//
//   Base::value = 50
```

The value is 50, not 5. `Left` and `Right` both pass `v` to `Base`, but `Bottom` passes `v * 10`. Since `Bottom` is the most-derived class, its initializer wins - the others are discarded. If `Bottom` had not listed `Base(...)` at all, `Base`'s default constructor would have been called.

**Construction order rules:**

1. **Virtual bases are constructed FIRST**, in declaration order (depth-first, left-to-right)
2. Then non-virtual bases, in declaration order
3. Then data members, in declaration order
4. Then the constructor body

**Why the most-derived class initializes virtual bases:**

- Both `Left` and `Right` specify `Base(v)` - they disagree!
- Only ONE `Base` exists (shared) - someone must decide the arguments
- The most-derived class (`Bottom`) is the single point of truth
- Intermediate classes' virtual base initializers are **silently ignored**
- If `Bottom` doesn't initialize `Base`, the **default constructor** is called

---

### Q3: Describe when to prefer composition over virtual bases for the same diamond problem

Virtual inheritance solves a real problem, but it adds memory overhead (vbptr), construction complexity, and surprising behavior around initialization. In most practical situations, composition gives you the same behavior with none of that baggage:

```cpp
#include <iostream>
#include <string>
#include <memory>

// === Approach 1: Virtual inheritance ===
struct Device {
    std::string id;
    Device(std::string i) : id(std::move(i)) {}
    virtual ~Device() = default;
};

struct Printer : virtual Device {
    Printer(std::string i) : Device(std::move(i)) {}
    void print_doc() { std::cout << id << ": printing\n"; }
};

struct Scanner : virtual Device {
    Scanner(std::string i) : Device(std::move(i)) {}
    void scan_doc() { std::cout << id << ": scanning\n"; }
};

struct AllInOne : Printer, Scanner {
    AllInOne(std::string i) : Device(std::move(i)), Printer(i), Scanner(i) {}
};

// === Approach 2: Composition (preferred!) ===
struct PrinterModule {
    void print_doc(const std::string& id) { std::cout << id << ": printing\n"; }
};

struct ScannerModule {
    void scan_doc(const std::string& id) { std::cout << id << ": scanning\n"; }
};

struct AllInOneComposed {
    std::string id;
    PrinterModule printer;
    ScannerModule scanner;

    AllInOneComposed(std::string i) : id(std::move(i)) {}

    void print_doc() { printer.print_doc(id); }
    void scan_doc() { scanner.scan_doc(id); }
};

int main() {
    // Virtual inheritance approach
    AllInOne a1("HP-VI");
    a1.print_doc();
    a1.scan_doc();

    std::cout << "---\n";

    // Composition approach
    AllInOneComposed a2("HP-Comp");
    a2.print_doc();
    a2.scan_doc();
}
// Expected output:
//   HP-VI: printing
//   HP-VI: scanning
//   ---
//   HP-Comp: printing
//   HP-Comp: scanning
```

The composed version produces the same output, but `PrinterModule` and `ScannerModule` have no constructor ordering surprises, no vbptr overhead, and can be tested independently.

**When to prefer each:**

| Criterion | Virtual Inheritance | Composition |
| --- | --- | --- |
| **Need polymorphism?** | Yes - `Device*` can point to any | No base pointer needed |
| **Construction complexity** | High - most-derived initializes base | Low - standard members |
| **Memory overhead** | vbptr per virtual inheritance edge | None |
| **Performance** | Slower (vbptr indirection) | Faster (direct access) |
| **Coupling** | Tight - tied to class hierarchy | Loose - modules are independent |
| **Testability** | Harder - need full hierarchy | Easy - mock modules independently |
| **When to use** | Framework/library hierarchies requiring IS-A | **Most cases** - default choice |

**Rule of thumb:** Prefer composition. Use virtual inheritance only when:

1. You genuinely need IS-A polymorphism through a shared base pointer
2. The hierarchy is stable and well-understood (e.g., `iostream` in the standard library)
3. Interface-only virtual bases (no data members) - reduces construction order surprises

---

## Notes

- **Standard library example:** `std::iostream` inherits virtually from `std::ios_base` through `std::istream` and `std::ostream`.
- **Default constructors:** If the most-derived class doesn't explicitly initialize a virtual base, the virtual base's **default constructor** is called - even if intermediate classes specify arguments.
- **`dynamic_cast` with virtual bases:** Unlike `static_cast`, `dynamic_cast` can navigate across virtual inheritance edges (requires RTTI).
- **Initialization list order:** Virtual base initializers in the most-derived class must come before non-virtual base initializers.
- **Performance cost:** Access to virtual base members requires an extra indirection through the vbptr. This matters in tight loops but is negligible in most application code.
