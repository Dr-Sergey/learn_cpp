# Know the Complete Catalog of Undefined Behavior in C++

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20, C++23  
**Reference:** [cppreference - Undefined Behavior](https://en.cppreference.com/w/cpp/language/ub)  

---

## Topic Overview

Undefined behavior (UB) in C++ is any construct for which the standard imposes **no requirements whatsoever**. Once UB is triggered, the program ceases to have any guaranteed behavior - past, present, or future. The standard lists over **200 distinct UB scenarios** across its normative text, and a working knowledge of their categories is essential for writing robust systems code.

UB is not simply "crashing." The compiler is free to assume UB never happens, and this assumption feeds directly into optimization passes. Code that appears correct under testing may silently break when compiled at higher optimization levels, on a different platform, or with a newer compiler version. The reason this is so dangerous is that UB can make code *look* like it works - the behavior just happens to be what you wanted under specific conditions, then silently changes when any condition shifts. Understanding the full catalog is the first step toward prevention.

The major categories of UB can be organized as follows. Each category has distinct detection strategies and mitigations:

| Category | Examples | Typical Consequence |
| --- | --- | --- |
| **Memory** | Null dereference, OOB access, use-after-free, double-free | Crash, data corruption, exploitable vulnerability |
| **Arithmetic** | Signed integer overflow, division by zero, shift past width | Wrong results, infinite loops |
| **Type System** | Strict aliasing violation, invalid `reinterpret_cast` | Miscompiled code, stale reads |
| **Lifetime** | Dangling reference, accessing destroyed object | Stale data, crash |
| **Concurrency** | Data race on non-atomic variable | Torn reads, heisenbugs |
| **Language Rules** | ODR violation, modifying `const` object, infinite loop w/o side-effects | Anything |

Sanitizers cover memory and some arithmetic UB well; type-system UB often requires careful code review or static analysis; concurrency UB needs ThreadSanitizer or formal verification.

---

## Self-Assessment

### Q1: Identify all UB in the following memory-related code

Memory UB is the most exploitable category - buffer overflows and use-after-free are the root cause of a huge fraction of real-world security vulnerabilities. Work through each annotated case and make sure you understand not just what the UB is but why the language rule exists:

```cpp
#include <cstdlib>
#include <cstring>
#include <iostream>
#include <memory>
#include <vector>

struct Sensor {
    int id;
    double readings[4];
};

void memory_ub_catalog() {
    // --- UB 1: Null pointer dereference ---
    int* p = nullptr;
    // *p = 42;  // UB: dereference of null pointer

    // --- UB 2: Out-of-bounds access ---
    int arr[5] = {1, 2, 3, 4, 5};
    // int val = arr[5];  // UB: index == size

    // --- UB 3: Use-after-free ---
    int* heap = new int(10);
    delete heap;
    // std::cout << *heap;  // UB: reading freed memory

    // --- UB 4: Double free ---
    int* q = new int(20);
    delete q;
    // delete q;  // UB: double free

    // --- UB 5: Misaligned access ---
    alignas(16) char buf[32];
    // Sensor* s = reinterpret_cast<Sensor*>(buf + 1);
    // s->id = 1;  // UB: misaligned pointer dereference

    // --- UB 6: Buffer overflow via memcpy ---
    char dst[4];
    const char* src = "Hello, World!";
    // std::memcpy(dst, src, 14);  // UB: writes past dst

    // --- UB 7: Pointer arithmetic past one-past-end ---
    int data[3] = {10, 20, 30};
    int* end = data + 3;       // OK: one-past-end
    // int* bad = data + 4;    // UB: more than one past end
    // int* neg = data - 1;    // UB: before array start
}

// --- UB 8: Returning reference to local ---
// int& dangling_ref() {
//     int local = 42;
//     return local;  // UB: dangling reference
// }

int main() {
    memory_ub_catalog();
}
```

Notice UB 7 carefully: even forming the invalid pointer is UB, not just dereferencing it. The standard requires that pointer arithmetic stay within one-past-the-end of the array - you cannot go further even if you never dereference.

**Answer:** Eight distinct UB instances are annotated. Each violates a different memory-safety rule. Key: pointer arithmetic UB (item 7) is often overlooked - even forming the invalid pointer is UB, not just dereferencing it.

---

### Q2: Identify arithmetic and type-system UB in this code

Arithmetic UB is the category where compilers most aggressively exploit UB for optimization. Signed overflow in particular enables loop transformations that unsigned arithmetic cannot. The type-system UB cases are subtler but often more dangerous because they produce silently wrong results rather than crashes:

```cpp
#include <climits>
#include <cstdint>
#include <cstring>
#include <iostream>

void arithmetic_ub() {
    // --- Signed integer overflow ---
    int a = INT_MAX;
    // int b = a + 1;  // UB: signed overflow

    // --- Division by zero ---
    int divisor = 0;
    // int c = 42 / divisor;  // UB: integer division by zero

    // --- Shift UB ---
    int x = 1;
    // int y = x << 32;   // UB if int is 32 bits: shift >= width
    // int z = x << -1;   // UB: negative shift count
    // int w = (-1) << 1; // UB before C++20; impl-defined since C++20

    // --- Unsigned is NOT UB (wraps modulo 2^N) ---
    unsigned u = UINT_MAX;
    unsigned v = u + 1;  // Well-defined: v == 0
    (void)v;
}

void type_system_ub() {
    // --- Strict aliasing violation ---
    float f = 3.14f;
    // int* ip = reinterpret_cast<int*>(&f);
    // int bits = *ip;  // UB: violates strict aliasing

    // Safe alternative:
    int bits_safe;
    std::memcpy(&bits_safe, &f, sizeof(float));  // OK

    // --- Accessing inactive union member (C++) ---
    union U { int i; float f; };
    U u;
    u.i = 42;
    // float val = u.f;  // UB in C++; defined in C99+

    // --- Modifying a const object ---
    const int ci = 100;
    // int* mp = const_cast<int*>(&ci);
    // *mp = 200;  // UB: modifying an object defined as const
}

int main() {
    arithmetic_ub();
    type_system_ub();
}
```

The unsigned arithmetic note is worth remembering: unsigned overflow is not UB - it wraps modulo 2^N by design. Only signed overflow is UB. This asymmetry surprises many people.

**Answer:**

| UB Type | Line | Rule Violated |
| --- | --- | --- |
| Signed overflow | `a + 1` | [expr.add] / [basic.fundamental] |
| Division by zero | `42 / divisor` | [expr.mul] §7.6.5/4 |
| Shift >= width | `x << 32` | [expr.shift] §7.6.7/1 |
| Negative shift | `x << -1` | [expr.shift] §7.6.7/1 |
| Strict aliasing | `*ip` | [basic.lval] §6.7.2/11 |
| Inactive union read | `u.f` | [class.union] §11.5/1 |
| Modify const | `*mp = 200` | [dcl.type.cv] §9.2.9.1/4 |

---

### Q3: Identify concurrency and lifetime UB

These two categories are especially nasty in combination: a data race can produce a dangling pointer, which then causes a use-after-free. Each of the examples below is a distinct failure mode worth recognizing on its own:

```cpp
#include <atomic>
#include <iostream>
#include <string>
#include <thread>
#include <vector>

// --- Data race UB ---
int shared_counter = 0;  // non-atomic

void data_race_example() {
    auto increment = [&]() {
        for (int i = 0; i < 100000; ++i) {
            ++shared_counter;  // UB: data race on non-atomic
        }
    };

    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    // Fix: use std::atomic<int> or a mutex
}

// --- Dangling reference UB ---
struct Widget {
    std::string name;
    const std::string& get_name() const { return name; }
};

const std::string& get_widget_name() {
    Widget w{"Gadget"};
    return w.get_name();
    // UB: returning reference to member of local object
}

// --- Lifetime UB: accessing object before construction completes ---
struct Base {
    int value;
    Base() : value(42) {
        // Calling virtual function in constructor is NOT UB,
        // but accesses the Base vtable - often a logic bug.
    }
    virtual void process() { std::cout << "Base\n"; }
};

// --- Use-after-scope UB ---
int* use_after_scope() {
    int local = 99;
    return &local;  // returning pointer to local: UB on dereference
}

// --- Infinite loop without side-effects (C++11+) ---
// void infinite() {
//     while (true) { }  // UB if thread has no observable side-effects
// }
// Compiler may assume this never happens and optimize away the loop.

int main() {
    data_race_example();
    // const std::string& ref = get_widget_name();
    // std::cout << ref;  // UB: dangling reference
}
```

The infinite loop case is one of the subtler entries in the UB catalog. C++11 added the "forward progress" guarantee, which means the compiler may assume every loop eventually terminates. An infinite loop with no side effects (no I/O, no synchronization, no volatile access) can legally be removed entirely by the optimizer.

**Answer:** The data race on `shared_counter` is the classic concurrency UB. The dangling reference in `get_widget_name()` and `use_after_scope()` are lifetime UB. The infinite loop without side-effects is a subtle C++11+ UB that allows compilers to assume forward progress.

---

## Notes

- The C++ standard does not have a single list of all UB. It is spread across normative text with phrases like "the behavior is undefined."
- **Annex J** (informative) in the C standard catalogs UB; C++ has no equivalent annex.
- UB is not "implementation-defined" - the compiler owes you **nothing**, not even a crash.
- Memory UB is the most exploitable (security vulnerabilities). Type-system UB is the most insidious (silent miscompilation).
- Sanitizers (`-fsanitize=undefined,address,thread`) catch many but not all categories - strict aliasing violations are notably hard to detect at runtime.
- In safety-critical code (MISRA, AUTOSAR), every known UB category has a corresponding rule that forbids the construct entirely.
