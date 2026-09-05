# Enable Control Flow Integrity (CFI) to prevent vtable hijacking

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/ControlFlowIntegrity.html>  

---

## Topic Overview

CFI ensures indirect calls (virtual calls, function pointers) only reach valid targets. This prevents vtable hijacking, ROP gadgets, and type confusion attacks. Complementary to #736 (more CFI details) and #656 (CFI + vtable hijacking attacks).

To understand why CFI matters, you need to understand what a vtable hijacking attack looks like. When an attacker corrupts a heap object's vtable pointer (for example, through a buffer overflow), they can redirect every virtual call on that object to arbitrary code. Without CFI, the CPU has no way to know whether a virtual dispatch is jumping to a legitimate overrider or to attacker-controlled shellcode.

```cpp
Vtable hijacking attack:

  Normal:                         Attack:
  obj.vptr -> [vtable]            obj.vptr -> [fake_vtable]  (overwritten)
               |                                |
           [Derived::f()]                  [shellcode / gadget]

  CFI protection:
  Before indirect call:

    1. Check: does target address belong to a valid function?
    2. Check: is the function the RIGHT TYPE for this call site?
    3. If not: trap / abort
```

CFI inserts those three checks at every indirect call site, at compile time, using type metadata embedded in the binary. The check is extremely fast - roughly 2-3 instructions - because it uses a compact bitset or range lookup rather than a hash table or dynamic type system.

| CFI scheme | Flag | Protects |
| --- | --- | --- |
| vcall | `-fsanitize=cfi-vcall` | Virtual function calls |
| nvcall | `-fsanitize=cfi-nvcall` | Non-virtual member calls |
| derived-cast | `-fsanitize=cfi-derived-cast` | `static_cast` to derived |
| unrelated-cast | `-fsanitize=cfi-unrelated-cast` | Casts between unrelated types |
| icall | `-fsanitize=cfi-icall` | Indirect calls via function pointer |
| All | `-fsanitize=cfi` | All of the above |

---

## Self-Assessment

### Q1: Compile with -fsanitize=cfi

This example sets up a class hierarchy and then shows what a type confusion attack would look like. The commented-out `reinterpret_cast` is the attack vector - CFI would trap on the call through the fake pointer before any harm is done.

```cpp
// Compile: clang++ -std=c++20 -flto -fvisibility=hidden -fsanitize=cfi cfi_demo.cpp
// Note: CFI requires LTO (-flto) and hidden visibility (-fvisibility=hidden)
#include <iostream>

class Base {
public:
    virtual void greet() { std::cout << "Base::greet\n"; }
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    void greet() override { std::cout << "Derived::greet\n"; }
};

class Unrelated {
public:
    virtual void hack() { std::cout << "Unrelated::hack\n"; }
};

int main() {
    // Normal virtual call - CFI allows this
    Base* b = new Derived();
    b->greet();  // OK: Derived IS-A Base

    // Type confusion attack - CFI catches this!
    // Attacker corrupts vptr to point to Unrelated's vtable
    Unrelated* u = new Unrelated();

    // Simulating vtable hijack (reinterpret_cast)
    // Base* fake = reinterpret_cast<Base*>(u);
    // fake->greet();  // CFI TRAP: target type doesn't match!

    // With CFI: program aborts before the call executes
    // Without CFI: calls Unrelated::hack() with wrong 'this' -> exploit

    std::cout << "\nCFI protects:\n";
    std::cout << "  1. Virtual calls (cfi-vcall): vtable pointer validation\n";
    std::cout << "  2. Indirect calls (cfi-icall): function pointer type check\n";
    std::cout << "  3. Bad casts (cfi-derived-cast): static_cast<Derived*>(base)\n";

    delete b;
    delete u;
}
```

The key line is the `reinterpret_cast` comment: even though both classes have virtual dispatch, `Unrelated` is not in the same class hierarchy as `Base`. CFI knows the complete type graph (because of LTO) and will abort the program before the bogus virtual call runs.

### Q2: How CFI validates function pointers

CFI uses type metadata embedded at compile time (via LTO) to validate targets:

**Step 1: Compile with LTO** - The compiler sees ALL code and can build a complete type graph.

**Step 2: Type metadata** - Each function/vtable gets a type identifier based on its signature.

**Step 3: Before indirect call** - The compiler inserts a check at every indirect call site:

```cpp
// Pseudocode of CFI check:
if (target_address NOT in valid_targets_for_type(expected_type))
    __cfi_check_fail();  // trap!
else
    call target_address;
```

**Implementation detail (Clang):**

- Uses bitset or range check per call site type.
- Valid targets stored in a compact data structure in the binary.
- Check is ~2-3 instructions: load type metadata, compare, branch.

**What CFI catches:**

- Corrupted vtable pointer - points to non-vtable memory.
- Type confusion - vtable for wrong class hierarchy.
- Corrupted function pointer - points to wrong function type.
- JIT-sprayed shellcode - not in valid function set.

**What CFI does NOT catch:**

- Calling a valid function of the right type but wrong instance.
- Data-only attacks (not involving indirect calls).
- Intra-function control flow (use CFG for that).

The "valid function of the right type but wrong instance" gap is worth understanding. If an attacker can redirect a call to a real, legitimate virtual function of the correct type - just the wrong object - CFI will not stop it. This is called a "same-type confusion" attack and is a known limitation of coarse-grained CFI. Fine-grained CFI variants address this but at higher cost.

### Q3: CFI overhead and production suitability

| Metric | Without CFI | With CFI |
| --- | --- | --- |
| Binary size | Baseline | +5-15% (type metadata) |
| Runtime overhead | Baseline | +1-5% (typical) |
| Virtual call cost | ~1 indirect jump | ~3 instructions + jump |
| Compilation time | Baseline | +20-50% (requires LTO) |

**Production deployment:**

```cpp
# Build command for production CFI:
clang++ -std=c++20 -O2 \
  -flto                          \ # Required: Link-Time Optimization
  -fvisibility=hidden            \ # Required: default hidden visibility
  -fsanitize=cfi                 \ # Enable all CFI checks
  -fno-sanitize-recover=cfi      \ # Abort on violation (don't continue)
  main.cpp -o app

# For diagnostics (development):
clang++ ... -fsanitize=cfi -fsanitize-recover=cfi \
  -fno-sanitize-trap=cfi         \ # Print diagnostic instead of trap
```

The `-fvisibility=hidden` requirement deserves a moment's attention. CFI needs to know the complete set of possible targets for every indirect call. If symbols are publicly visible (the default), a third-party shared library loaded at runtime could add new valid targets that the compiler never saw - breaking CFI's closed-world assumption. Hidden visibility ensures CFI's type graph is complete.

**When to enable CFI in production:**

- Security-critical applications (browsers, OS components).
- Code handling untrusted input (parsers, network servers).
- Already: Chrome, Android, Linux kernel use CFI.

**When to skip:**

- Performance-critical inner loops (rare case where 1-5% matters).
- Heavy use of plugins/dlopen (CFI requires LTO across boundaries).
- MSVC (CFI is Clang-specific; MSVC has Control Flow Guard /guard:cf).

---

## Notes

- Complementary to #736 and #656 (more CFI topics).
- CFI REQUIRES `-flto` (Link-Time Optimization) and `-fvisibility=hidden`.
- MSVC equivalent: `/guard:cf` (Control Flow Guard) - different mechanism, similar goal.
- Android enables CFI by default for system binaries since Android 9.
- Chrome uses CFI for all builds since 2018.
