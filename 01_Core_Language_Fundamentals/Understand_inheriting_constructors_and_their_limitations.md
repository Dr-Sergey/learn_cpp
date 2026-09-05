# Understand inheriting constructors and their limitations

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/using_declaration>  

---

## Topic Overview

In C++11, base class constructors are **not** inherited by default. A derived class must declare its own constructors or use `using Base::Base;` to **inherit** all of the base class's constructors.

### The Problem

Without the `using` declaration, the derived class has no idea how to forward arguments to its base:

```cpp
struct Base {
    Base(int x) : val(x) {}
    Base(int x, int y) : val(x + y) {}
    int val;
};

struct Derived : Base {
    // No constructors declared
    // Derived d(42);  // ERROR: no matching constructor
};
```

### The Solution: `using Base::Base;`

One line pulls in all the base class constructors. Members added only in the derived class should get default member initializers so they're never left uninitialized:

```cpp
struct Derived : Base {
    using Base::Base;   // inherit ALL Base constructors
    int extra = 0;      // use default member initializer
};

Derived d1(42);         // OK: calls Base(int)
Derived d2(10, 20);     // OK: calls Base(int, int)
```

### What Happens Under the Hood

The compiler synthesizes a forwarding constructor for each one it inherits - think of it as the compiler writing these for you:

```cpp
// Conceptually, using Base::Base; generates:
Derived(int x) : Base(x) {}           // forwards to Base(int)
Derived(int x, int y) : Base(x, y) {} // forwards to Base(int, int)
```

### Key Limitations

Here is where people get tripped up - inherited constructors know nothing about the derived class's own members:

1. **Derived-only members are NOT initialized** by inherited constructors (unless they have default member initializers).
2. **Copy and move constructors** are never inherited.
3. If a derived-class constructor has the **same signature** as an inherited one, the derived version wins.
4. **Default constructors** - if only `using Base::Base;` is declared and `Base` has a default constructor, `Derived` gets it too.

```cpp
struct Base {
    Base(int x) : val(x) {}
    int val;
};

struct Derived : Base {
    using Base::Base;
    int extra;     // NO default initializer!
};

Derived d(42);
// d.val = 42 (from Base)
// d.extra = ??? (UNINITIALIZED -- undefined behavior if read)
```

---

## Self-Assessment

### Q1: Show that base class constructors are NOT inherited by default; use `using Base::Base;` to inherit them

Notice the contrast - `Dog_Manual` has to duplicate every constructor signature, while `Dog_Inherited` gets them all for free:

```cpp
#include <iostream>
#include <string>

struct Animal {
    std::string name;
    int age;

    Animal(std::string n, int a) : name(std::move(n)), age(a) {}
    Animal(std::string n) : name(std::move(n)), age(0) {}
};

// Without using Base::Base -- must write constructors manually
struct Dog_Manual : Animal {
    std::string breed;

    Dog_Manual(std::string n, int a, std::string b)
        : Animal(std::move(n), a), breed(std::move(b)) {}

    // Cannot do Dog_Manual("Rex", 5) -- no matching constructor
};

// With using Base::Base -- inherit all Animal constructors
struct Dog_Inherited : Animal {
    using Animal::Animal;    // inherit Animal(string,int) and Animal(string)
    std::string breed = "unknown";   // default member initializer
};

int main() {
    // Dog_Manual dm("Rex", 5);  // ERROR: no matching constructor

    Dog_Inherited d1("Rex", 5);     // OK: uses Animal(string, int)
    Dog_Inherited d2("Buddy");      // OK: uses Animal(string)

    std::cout << d1.name << ", age " << d1.age << ", breed: " << d1.breed << "\n";
    // Rex, age 5, breed: unknown

    std::cout << d2.name << ", age " << d2.age << ", breed: " << d2.breed << "\n";
    // Buddy, age 0, breed: unknown
}
```

**How it works:**

- Without `using Animal::Animal;`, `Dog_Manual` has no constructor matching `(string, int)` - only its own.
- With `using Animal::Animal;`, all non-copy/move constructors are inherited automatically.
- `breed` gets its value from the default member initializer since inherited constructors don't know about it.

### Q2: Explain that inheriting constructors do not initialize members declared only in the derived class

**Answer:**

Inherited constructors only forward arguments to the base class - they have **no knowledge** of derived-class members. Such members are:

- **Default-initialized** if they have a default member initializer (`int x = 0;`)
- **Left uninitialized** if they are scalar types without a default member initializer (`int x;`)
- **Default-constructed** if they are class types with a default constructor

The distinction matters because reading `c` below is undefined behavior:

```cpp
#include <iostream>

struct Base {
    int a;
    Base(int x) : a(x) {}
};

struct Derived : Base {
    using Base::Base;

    int b = 99;         // has default member initializer -> will be 99
    int c;              // NO initializer -> UNINITIALIZED
    std::string d;      // class type -> default constructed (empty string)
};

int main() {
    Derived obj(42);

    std::cout << "a = " << obj.a << "\n";   // 42 (from Base)
    std::cout << "b = " << obj.b << "\n";   // 99 (default member initializer)
    // std::cout << "c = " << obj.c << "\n"; // UB! c is uninitialized
    std::cout << "d = '" << obj.d << "'\n"; // "" (default constructed)
}
```

**Best practice:** Always provide default member initializers for all derived-class members when using inherited constructors.

### Q3: Show that inherited constructors participate in overload resolution as if declared in the derived class

Overload resolution treats inherited constructors as if they were native to the derived class, which means a derived-class constructor with the same signature wins:

```cpp
#include <iostream>
#include <string>

struct Base {
    Base(int x)         { std::cout << "Base(int): " << x << "\n"; }
    Base(double x)      { std::cout << "Base(double): " << x << "\n"; }
    Base(std::string s) { std::cout << "Base(string): " << s << "\n"; }
};

struct Derived : Base {
    using Base::Base;   // inherits all three constructors

    // Override one specific signature
    Derived(double x) : Base(static_cast<int>(x)) {
        std::cout << "Derived(double): truncated to int\n";
    }
};

int main() {
    Derived d1(42);        // calls inherited Base(int)
    // Output: Base(int): 42

    Derived d2(3.14);      // calls Derived(double) -- derived wins same signature
    // Output: Base(int): 3
    //         Derived(double): truncated to int

    Derived d3(std::string("hello"));  // calls inherited Base(string)
    // Output: Base(string): hello

    // Overload resolution works normally:
    // d1 -> int -> exact match: inherited Base(int)
    // d2 -> double -> exact match: Derived(double) shadows inherited Base(double)
    // d3 -> string -> exact match: inherited Base(string)
}
```

**How it works:**

- Inherited constructors are treated as if they were declared in the derived class for overload resolution purposes.
- If the derived class declares a constructor with the **same parameter types**, the derived version hides the inherited one.
- The normal overload resolution rules (exact match > promotion > conversion) apply.

---

## Notes

- `using Base::Base;` was introduced in C++11 and refined in C++17 (constructors from multiple bases can be inherited).
- Copy/move constructors are **never** inherited - the compiler always generates (or deletes) its own.
- Inherited constructors have the same access (public/protected/private) and `explicit`-ness as in the base class.
- If a base constructor is `explicit`, the inherited version is also `explicit`.
- With multiple inheritance: `using A::A; using B::B;` - if ambiguous signatures exist, the derived class must declare its own to resolve.
