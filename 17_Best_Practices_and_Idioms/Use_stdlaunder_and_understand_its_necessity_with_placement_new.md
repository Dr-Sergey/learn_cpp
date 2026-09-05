# Use std::launder and understand its necessity with placement new

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/launder>  

---

## Topic Overview

`std::launder` tells the compiler: "this pointer now refers to a new object at the same address." It is needed when you use placement `new` to create a new object in storage that already held a different object - especially one with `const` or reference members.

This topic is one of the trickier corners of C++. The reason it trips people up is that the problem is invisible: the code compiles, the pointers look right, but the compiler may silently use stale cached values because its optimizer proved - correctly, given the original object - that certain values could never change. `std::launder` exists to break that proof.

```cpp
#include <new>
const int x = 42;
new (const_cast<int*>(&x)) int(99);
// &x still "refers" to const 42 in compiler's view
int* p = std::launder(const_cast<int*>(&x));  // p refers to new object (99)
```

---

## Self-Assessment

### Q1: Explain pointer-interconvertibility and why placement new can produce unusable pointers

The root issue is that a pointer is not just an address - it also carries the compiler's knowledge about what object lives there. When you use placement `new` to construct a new object at an address, old pointers to that address are not automatically updated in the compiler's mental model. If the destroyed object had `const` members, the compiler may have already decided those values are fixed forever.

```cpp
#include <iostream>
#include <new>
#include <cstring>

struct A {
    const int id;  // const member!
    int value;
};

int main() {
    // Problem: const member + placement new = stale pointer
    alignas(A) unsigned char buf[sizeof(A)];

    // Construct first object
    A* p1 = new (buf) A{1, 100};
    std::cout << "p1->id = " << p1->id << '\n';  // 1

    // Destroy and construct a NEW object at same address
    p1->~A();
    A* p2 = new (buf) A{2, 200};  // different const member value!

    // p1 is now stale! It still "knows" id was 1.
    // The compiler may have cached p1->id = 1 (because it's const).
    // Using p1 here is technically UB.

    // std::launder fixes this:
    A* p3 = std::launder(reinterpret_cast<A*>(buf));
    std::cout << "p3->id = " << p3->id << '\n';    // 2 (correct!)
    std::cout << "p3->value = " << p3->value << '\n'; // 200

    // Without launder, compiler might optimize to print 1 for id!
}
// Expected output:
// p1->id = 1
// p3->id = 2
// p3->value = 200
```

**Why it happens:** When a struct has `const` or reference members, the compiler assumes those values never change through a given pointer. Placement new creates a new object, but old pointers don't know that. The old pointer is "stale" - it is pointer-interconvertible with the storage, but not a valid pointer to the new object.

### Q2: Show when `std::launder` is needed after placement new

The rule of thumb is: if the newly constructed object has the same type and the same const/reference member values as the old one, and you use the pointer returned by placement `new` itself, you do not need `std::launder`. But if you are reusing an old pointer, or the const/reference members have changed, you do.

```cpp
#include <iostream>
#include <new>
#include <type_traits>

struct Config {
    const int version;   // const member
    int& ref;            // reference member
};

// Scenario 1: std::launder IS needed (const/ref members change)
void needs_launder() {
    int v1 = 1, v2 = 2;
    alignas(Config) unsigned char storage[sizeof(Config)];

    // First object
    Config* c1 = new (storage) Config{10, v1};
    std::cout << c1->version << ", ref=" << c1->ref << '\n';  // 10, ref=1
    c1->~Config();

    // Second object at same address with different const/ref
    new (storage) Config{20, v2};

    // MUST use launder - old pointer assumes version==10
    Config* c2 = std::launder(reinterpret_cast<Config*>(storage));
    std::cout << c2->version << ", ref=" << c2->ref << '\n';  // 20, ref=2
}

// Scenario 2: std::launder NOT needed (no const/ref members)
struct Point { int x, y; };

void no_launder_needed() {
    alignas(Point) unsigned char buf[sizeof(Point)];
    Point* p = new (buf) Point{3, 4};
    p->~Point();

    Point* p2 = new (buf) Point{5, 6};
    // p2 is the return of placement new - always valid
    std::cout << p2->x << ", " << p2->y << '\n';  // 5, 6
}

int main() {
    needs_launder();
    no_launder_needed();
}
// Expected output:
// 10, ref=1
// 20, ref=2
// 5, 6
```

The key observation in `no_launder_needed` is that `p2` comes directly from the return value of placement `new`. That pointer is always a valid pointer to the new object. The problem only arises when you try to reuse an old pointer (like accessing the storage through `buf` cast back to `Point*`) after the object has been replaced.

### Q3: What compiler optimizations does `std::launder` defeat

`std::launder` is a no-op at runtime - it is a fence for the optimizer, not an instruction. Its effect is to prevent the compiler from carrying assumptions about the pointed-to object across the `launder` call.

```cpp
#include <iostream>
#include <new>

struct Widget {
    const int id;  // compiler assumes this NEVER changes
};

int main() {
    alignas(Widget) unsigned char buf[sizeof(Widget)];

    Widget* w1 = new (buf) Widget{42};

    // Compiler optimization: since w1->id is const,
    // the compiler can cache "42" and never re-read memory.
    int cached = w1->id;  // compiler thinks this is always 42

    w1->~Widget();
    new (buf) Widget{99};  // new object with id=99

    // WITHOUT launder: compiler uses cached value = 42 (wrong!)
    // WITH launder: compiler is forced to re-read memory
    Widget* w2 = std::launder(reinterpret_cast<Widget*>(buf));
    std::cout << "w2->id = " << w2->id << '\n';  // 99 (correct!)
}
// Expected output:
// w2->id = 99
```

The table below summarizes what each optimization the compiler might apply without `std::launder`, and what it is forced to do instead:

**Optimizations defeated by `std::launder`:**

| Optimization | Without launder | With launder |
| --- | --- | --- |
| Const propagation | Compiler caches const values | Forced to re-read |
| Devirtualization | Compiler caches vtable ptr | Forced to re-read |
| Reference propagation | Compiler caches reference target | Forced to re-read |
| Dead store elimination | May skip placement new | Respects new object |

---

## Notes

- `std::launder` is in `<new>` since C++17.
- You need it only when: placement new + const/reference members + reusing old pointers.
- The return value of placement `new` itself is always valid - no launder needed for it.
- `std::launder` is a no-op at runtime; it's a compiler hint.
- In C++20, `std::bit_cast` may be preferred for type punning instead of launder.
