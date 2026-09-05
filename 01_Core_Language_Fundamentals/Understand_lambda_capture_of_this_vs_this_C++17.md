# Understand lambda capture of *this vs this (C++17)

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

When a lambda is created inside a member function, it can capture the enclosing object. There are two ways:

| Capture | Meaning | Lifetime |
| --- | --- | --- |
| `[this]` | Captures the `this` pointer (8 bytes) | Lambda holds a **pointer** - original object must stay alive |
| `[*this]` | Captures a **copy** of the entire object (C++17) | Lambda has its own independent copy |

### The Dangling `this` Problem

If the object is destroyed before the lambda runs, `[this]` leaves you with a dangling pointer and undefined behavior:

```cpp
struct Widget {
    int value = 42;

    auto make_callback() {
        return [this] { return value; };  // captures pointer to *this*
    }
};

auto callback = Widget{}.make_callback();  // Widget is a temporary -- destroyed here!
callback();  // UNDEFINED BEHAVIOR: this pointer is dangling
```

### The Fix: `[*this]` (C++17)

Capturing by value copies the whole object into the lambda's closure. The lambda becomes self-contained:

```cpp
struct Widget {
    int value = 42;

    auto make_callback() {
        return [*this] { return value; };  // copies the Widget into the lambda
    }
};

auto callback = Widget{}.make_callback();  // Widget destroyed, but lambda has a copy
callback();  // OK: returns 42 from the lambda's internal copy
```

### C++20: `[=]` No Longer Implicitly Captures `this`

In C++20, `[=]` deprecated implicit capture of `this`. You must be explicit - this prevents the surprising case where `[=]` silently captured a pointer:

```cpp
// C++17: [=] captures this implicitly
// C++20: [=] does NOT capture this -- must write [=, this] or [=, *this]

auto f1 = [=, this]  { return value; };   // captures this pointer explicitly
auto f2 = [=, *this] { return value; };   // copies *this explicitly
```

---

## Self-Assessment

### Q1: Show a bug where `[this]` captures `this` by pointer and the object is destroyed before the lambda runs

The `report` callback is stored past the lifetime of its originating object - a classic source of hard-to-diagnose crashes:

```cpp
#include <iostream>
#include <functional>
#include <vector>

class Sensor {
    std::string name;
    double reading = 0.0;
public:
    Sensor(std::string n, double r) : name(std::move(n)), reading(r) {}

    std::function<void()> get_reporter() {
        // BUG: captures this by pointer
        return [this] {
            std::cout << name << ": " << reading << "\n";
        };
    }
};

int main() {
    std::function<void()> report;

    {
        Sensor s("Temperature", 23.5);
        report = s.get_reporter();
        report();   // OK here: s is still alive
    } // s destroyed here!

    report();  // UNDEFINED BEHAVIOR: this points to destroyed Sensor
    // May print garbage, crash, or appear to work

    // Also dangerous with containers:
    std::vector<std::function<void()>> callbacks;
    {
        std::vector<Sensor> sensors;
        sensors.emplace_back("Pressure", 101.3);
        callbacks.push_back(sensors[0].get_reporter());
    } // sensors destroyed
    // callbacks[0]();  // UB!
}
```

**How it works:**

- `[this]` captures the raw pointer to `s`.
- When `s` goes out of scope, the pointer dangles.
- The lambda still holds the old address and accesses freed memory.

### Q2: Fix it using `[*this]` (C++17) to copy the object into the lambda's closure

With `[*this]` the lambda carries its own private copy of the `Sensor` - it no longer matters what happens to the original:

```cpp
#include <iostream>
#include <functional>

class Sensor {
    std::string name;
    double reading = 0.0;
public:
    Sensor(std::string n, double r) : name(std::move(n)), reading(r) {}

    std::function<void()> get_reporter() {
        // FIX: [*this] copies the entire Sensor into the lambda
        return [*this] {
            std::cout << name << ": " << reading << "\n";
        };
    }

    // Mutable version -- the copy can be modified without affecting the original
    std::function<void()> get_incrementer() {
        return [*this]() mutable {
            reading += 1.0;
            std::cout << name << " (copy): " << reading << "\n";
        };
    }
};

int main() {
    std::function<void()> report;

    {
        Sensor s("Temperature", 23.5);
        report = s.get_reporter();
    } // s destroyed, but lambda has its own copy

    report();   // OK! Prints "Temperature: 23.5" from the lambda's copy

    // Mutable copy example:
    Sensor s2("Pressure", 101.3);
    auto inc = s2.get_incrementer();
    inc();   // "Pressure (copy): 102.3"
    inc();   // "Pressure (copy): 103.3"
    std::cout << "Original reading unchanged: ";
    // s2.reading is still 101.3 (lambda has its own copy)
}
```

**How it works:**

- `[*this]` copy-constructs the entire object inside the lambda's closure.
- The lambda is fully self-contained - safe regardless of the original object's lifetime.
- With `mutable`, the lambda can modify its copy without affecting the original.

### Q3: Explain the size and performance implications of capturing `*this` vs `this`

**Answer:**

| Aspect | `[this]` | `[*this]` |
| --- | --- | --- |
| Closure size | +8 bytes (one pointer) | +`sizeof(YourClass)` bytes |
| Copy cost | Trivial | Full copy (copy constructor called) |
| Lifetime safety | Dangerous if object dies | Safe (independent copy) |
| Access to original | Yes (shares state) | No (separate copy) |
| Mutation | Affects the original object | Only affects the copy |

For small objects the copy is cheap; for large ones it can be significant:

```cpp
#include <iostream>

struct Small {
    int x = 1;
    auto f() { return [*this] { return x; }; }   // closure ~= 4 bytes extra
};

struct Large {
    int data[1000];   // 4000 bytes
    auto f() { return [*this] { return data[0]; }; }  // closure ~= 4000 bytes extra!
};

int main() {
    // [*this] is cheap for small objects:
    Small s;
    auto sl = s.f();   // copies 4 bytes

    // [*this] is expensive for large objects:
    Large l{};
    auto ll = l.f();   // copies 4000 bytes!

    // For large objects, consider:
    // 1. Use [this] if lifetime is guaranteed
    // 2. Use [self = std::make_shared<Large>(*this)] for shared ownership
    // 3. Capture only the needed members: [x = data[0]] { return x; }

    std::cout << "sizeof Small closure: ~" << sizeof(sl) << "\n";
    std::cout << "sizeof Large closure: ~" << sizeof(ll) << "\n";
}
```

**Guidelines:**

- Use `[*this]` for **small objects** or when the lambda **outlives the object** (callbacks, async tasks).
- Use `[this]` for **short-lived lambdas** within the same scope where lifetime is guaranteed (algorithms, ranges).
- For large objects, capture only the needed members individually instead of the whole object.

---

## Notes

- `[*this]` invokes the **copy constructor** - if copying is expensive or deleted, use `[this]` or shared pointers.
- C++20 deprecates implicit `this` capture by `[=]` - always be explicit: `[=, this]` or `[=, *this]`.
- `[*this]` with `mutable` lets you modify the lambda's copy - the original object is not affected.
- In coroutines, `[*this]` is essential because the coroutine frame may outlive the originating object.
