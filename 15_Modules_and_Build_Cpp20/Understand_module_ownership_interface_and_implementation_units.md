# Understand Module Ownership: Interface and Implementation Units

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** [cppreference - Modules](https://en.cppreference.com/w/cpp/language/modules)  

---

## Topic Overview

C++20 modules introduce a fundamentally different compilation model. A **module** is composed of one or more **module units**, each being a translation unit that declares `module M;` (or `export module M;`). The two primary kinds are the **module interface unit** (MIU) and the **module implementation unit**. A module must have exactly one **primary module interface unit** - the unit with `export module M;` and no module partition designation. It may additionally have **interface partitions** (`export module M:part;`) and **implementation partitions** (`module M:part;`).

The distinction between interface and implementation units drives the ownership and visibility model. Every declaration that appears in a module unit **belongs** to that module (has module ownership). An exported declaration (`export`) is both **visible** and **reachable** to importers. A non-exported declaration in an interface unit is **reachable** but not **visible** - importers can use it indirectly (e.g., as a return type) but cannot name it directly. Declarations in implementation units are neither visible nor reachable to importers.

The reason this distinction matters so much is linkage. Non-exported declarations in a module get **module linkage**, which means they're accessible across all translation units of the same module but invisible everywhere else. This is a new linkage kind distinct from internal (`static`) and external linkage, and it's what allows a large module to be split across many files while still sharing internal helpers freely.

| Unit Kind | Declaration | `export module M;` | Visible to Importer | Reachable to Importer |
| --- | --- | --- | --- | --- |
| Primary interface | `export class Foo {};` | Yes | Yes | Yes |
| Primary interface | `class Bar {};` (no export) | Yes | No | Yes |
| Implementation unit | `class Impl {};` | No (`module M;`) | No | No |
| Interface partition | `export void f();` | `export module M:part;` | Yes (if re-exported) | Yes |
| Implementation partition | `void g() {}` | `module M:part;` | No | No |

---

## Self-Assessment

### Q1: Given this module structure, which declarations are visible vs only reachable to an importer of `math`

This example has three entities you need to track: `Vec3` (exported), `Transform` (in the interface but not exported), and `normalize` (in the implementation unit). Watch what each one means for the consumer:

```cpp
// ---------- math.cppm (primary module interface unit) ----------
export module math;

export struct Vec3 {
    float x, y, z;
};

struct Transform {               // not exported - reachable only
    float matrix[16];
};

export Vec3 apply(Vec3 v, Transform t);   // Transform is reachable via signature

// ---------- math_impl.cpp (module implementation unit) ----------
module math;

// Helper with module linkage - invisible and unreachable to importers
Vec3 normalize(Vec3 v) {
    float len = /*...*/;
    return {v.x / len, v.y / len, v.z / len};
}

Vec3 apply(Vec3 v, Transform t) {
    auto n = normalize(v);
    // ... apply transform
    return n;
}

// ---------- main.cpp ----------
import math;

int main() {
    Vec3 v{1, 0, 0};
    // Transform t{};          // ERROR: Transform is reachable but NOT visible
    auto result = apply(v, {}); // OK: aggregate-init of reachable type
    // normalize(v);           // ERROR: not reachable (in implementation unit)
}
```

`Vec3` and `apply` are **visible** - exported, directly nameable. `Transform` is **reachable** - its members and layout are known to the compiler (you can pass `{}` for it), but you cannot name the type `Transform` in `main.cpp`. `normalize` is in the implementation unit, so it is neither visible nor reachable.

---

### Q2: What is the correct way to split a module into interface partitions and re-export them from the primary interface

When a module grows large, you split it into partitions. Each interface partition declares itself with `export module M:part;`. The critical rule is that the primary interface unit must `export import` every interface partition it wants importers to see:

```cpp
// ---------- graphics:shapes.cppm (interface partition) ----------
export module graphics:shapes;

export struct Circle {
    float radius;
    float area() const { return 3.14159f * radius * radius; }
};

export struct Rect {
    float w, h;
    float area() const { return w * h; }
};

// ---------- graphics:render.cppm (interface partition) ----------
export module graphics:render;

import :shapes;                  // partition import (within same module)

export void draw(const Circle& c);
export void draw(const Rect& r);

// ---------- graphics.cppm (primary module interface - must re-export) ----------
export module graphics;

export import :shapes;           // re-export partition interface
export import :render;           // re-export partition interface

// ---------- main.cpp ----------
import graphics;

int main() {
    Circle c{5.0f};
    draw(c);                     // OK - both Circle and draw are visible
}
```

The key rule here: the primary interface unit must `export import` every interface partition. A non-exported `import :part;` makes the partition reachable but not visible to importers of the top-level module. Failing to re-export an interface partition is ill-formed - the partition's exported declarations would be unreachable, which the standard does not allow.

---

### Q3: What happens when an implementation partition imports another partition, and how does module linkage apply

Implementation partitions (declared with `module M:part;`, no `export`) let you organize private implementation code into separate files. They can import other partitions freely. Here's how the pieces fit together and where module linkage shows up:

```cpp
// ---------- engine:detail.cppm (implementation partition) ----------
module engine:detail;            // no 'export' - implementation partition

struct InternalState {           // module linkage: visible within 'engine' only
    int frame_count = 0;
    bool dirty = true;
};

void reset(InternalState& s) {
    s.frame_count = 0;
    s.dirty = true;
}

// ---------- engine:core.cppm (interface partition) ----------
export module engine:core;

import :detail;                  // OK: same module can see implementation partitions

export class Engine {
    InternalState state_;        // OK: InternalState has module linkage
public:
    export void tick();
    // InternalState is reachable but not visible to importers of engine
};

// ---------- engine.cppm (primary module interface) ----------
export module engine;
export import :core;

// ---------- main.cpp ----------
import engine;

int main() {
    Engine e;
    e.tick();                    // OK
    // e.state_;                 // ERROR: InternalState not visible
    // InternalState s;          // ERROR: not visible (module linkage, not exported)
}
```

The key insight is that module linkage entities can be shared across all translation units of the same module - unlike `static` or anonymous-namespace entities, which are TU-local. This makes module linkage ideal for implementation details that multiple module units need to share internally without exposing anything to the outside world.

| Linkage Kind | Scope | Use Case |
| --- | --- | --- |
| External | Global across all TUs | Exported module APIs |
| Module | All TUs of the owning module | Shared implementation details |
| Internal (`static`) | Single TU only | TU-private helpers |
| No linkage | Block scope | Local variables |

---

## Notes

- A module must have **exactly one** primary module interface unit (`export module M;`).
- Interface partitions use `export module M:part;`; implementation partitions use `module M:part;`.
- The primary interface must `export import` all interface partitions - this is mandatory.
- Non-exported declarations in interface units create **reachable-but-not-visible** types - importers can use them indirectly but cannot name them.
- Module linkage is the new default for non-exported, non-`static` entities in module units - it replaces the old practice of anonymous namespaces when cross-TU sharing within the module is needed.
- Implementation units (`module M;` with no partition) cannot be imported - they contribute definitions but no interface.
- Avoid placing definitions of exported inline functions in implementation units; the definition must be reachable from the interface (use the interface unit or an interface partition).
- ODR violations are greatly reduced: each module produces a single BMI (Binary Module Interface), eliminating multiple-inclusion issues.
