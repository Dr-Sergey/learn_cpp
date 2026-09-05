# Use CFI (Control Flow Integrity) to prevent vtable hijacking attacks

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/ControlFlowIntegrity.html>  

---

## Topic Overview

CFI (Control Flow Integrity) is a security mechanism that restricts which targets indirect calls and jumps can reach. In C++, the primary threat is **vtable hijacking**: an attacker corrupts a vtable pointer (via buffer overflow, use-after-free, etc.) to redirect virtual function calls to attacker-controlled code. CFI validates that every virtual call dispatches to a function that is a legitimate implementation of the called virtual method.

The reason this matters so much in C++ specifically is that virtual dispatch is essentially a table of function pointers that gets looked up at runtime. If an attacker can point the table pointer somewhere else, they control where "virtual function calls" actually go. CFI puts a guard right before each lookup.

### Vtable Hijacking Attack

Here is the control flow picture for normal dispatch, a corrupted dispatch, and what CFI does about it. Think of this as a decision tree the CPU walks through at each virtual call:

```cpp
Normal virtual dispatch:
  obj.vptr -> [vtable for Derived] -> Derived::process()

After vtable pointer corruption:
  obj.vptr -> [attacker-controlled memory] -> shellcode / gadget chain

CFI intercepts:
  obj.vptr -> [attacker-controlled memory]
  CFI check: is target in set of valid implementations of process()?
  -> NO -> abort!
```

The key insight: without CFI, the CPU happily follows any pointer wherever it points. With CFI, the compiler has already baked in a whitelist of valid destinations, and anything outside that list trips a trap.

### How Clang CFI Works

CFI is a two-phase operation - analysis happens at compile/link time, the actual check happens at runtime just before each virtual dispatch:

```cpp
Compile-time:

  1. Compiler builds a set of valid targets for each virtual call site
  2. Each class hierarchy's vtable is assigned a type identifier
  3. At each virtual call, compiler inserts a check instruction

Runtime:
  Before: obj->virtual_method(args)
  After:  assert(obj->vptr is in valid_vtable_set)
          obj->virtual_method(args)

  If vptr was corrupted -> check fails -> trap -> SIGILL
```

The reason CFI requires Link-Time Optimization (`-flto`) is that the compiler needs to see the entire program at once to build the complete set of valid vtables for each hierarchy. Without that whole-program view, it can't know which derived classes exist.

### CFI Scheme Comparison

CFI is not one monolithic check - it is a family of checks you can mix and match. The most important for C++ is `cfi-vcall`, which covers the virtual dispatch threat:

| Scheme | Flag | What it checks |
| --- | --- | --- |
| `cfi-vcall` | `-fsanitize=cfi-vcall` | Virtual calls - most important |
| `cfi-nvcall` | `-fsanitize=cfi-nvcall` | Non-virtual member function calls |
| `cfi-derived-cast` | `-fsanitize=cfi-derived-cast` | `static_cast` to derived class |
| `cfi-unrelated-cast` | `-fsanitize=cfi-unrelated-cast` | `reinterpret_cast` between unrelated types |
| `cfi-icall` | `-fsanitize=cfi-icall` | Indirect calls through function pointers |
| All | `-fsanitize=cfi` | All of the above |

### Core Example

This is the minimal form: compile a class hierarchy with CFI and let normal virtual dispatch proceed. The interesting part is what the compiler inserts invisibly before `obj->process()`:

```cpp
#include <iostream>
#include <memory>

// Compile: clang++ -std=c++20 -flto -fvisibility=hidden -fsanitize=cfi -fno-sanitize-recover=all cfi.cpp

struct Base {
    virtual void process() { std::cout << "Base::process\n"; }
    virtual ~Base() = default;
};

struct Derived : Base {
    void process() override { std::cout << "Derived::process\n"; }
};

int main() {
    auto obj = std::make_unique<Derived>();
    obj->process();  // CFI validates: vptr points to vtable for Derived -> OK
    // Output: Derived::process
}
```

Legitimate code runs without any observable difference. The protection is invisible when everything is correct - it only fires when something has gone wrong.

---

## Self-Assessment

### Q1: Compile with -fsanitize=cfi-vcall and explain how it validates virtual dispatch targets at runtime

**Answer:**

The code below walks through what CFI actually does at each dispatch site. Notice especially the pseudo-code comments explaining the generated check - that is the piece that normally lives in compiler output, not in your source:

```cpp
#include <iostream>
#include <memory>
#include <cstdint>
#include <cstring>

// Build commands
// Clang CFI REQUIRES:
//   -flto           (Link-Time Optimization - needed for cross-TU type info)
//   -fvisibility=hidden  (ensures vtable type info is available)
//   -fsanitize=cfi-vcall (enable virtual call checks)
//
// Full command:
//   clang++ -std=c++20 -flto -fvisibility=hidden \
//           -fsanitize=cfi-vcall -fno-sanitize-recover=all \
//           -O2 cfi_demo.cpp -o cfi_demo

struct __attribute__((visibility("default"))) Animal {
    virtual void speak() = 0;
    virtual ~Animal() = default;
};

struct __attribute__((visibility("default"))) Dog : Animal {
    void speak() override { std::cout << "Woof!\n"; }
};

struct __attribute__((visibility("default"))) Cat : Animal {
    void speak() override { std::cout << "Meow!\n"; }
};

struct __attribute__((visibility("default"))) Unrelated {
    virtual void hack() { std::cout << "HACKED!\n"; }
    virtual ~Unrelated() = default;
};

// How CFI-vcall validation works
//
// At compile time, for the call animal->speak():
//   1. Compiler identifies that 'animal' is Animal*
//   2. Builds the set: {Dog::vtable, Cat::vtable}
//      (all classes that derive from Animal and override speak())
//   3. Generates a type check before the virtual call:
//
//   Pseudo-code of generated check:
//     void* vptr = *(void**)animal;           // read vtable pointer
//     type_id = hash(vptr);                    // compute type identifier
//     if (type_id NOT IN valid_set_for_Animal) {
//         __cfi_check_fail();                  // trap (SIGILL by default)
//     }
//     // proceed with virtual dispatch
//     animal->speak();
//
// Implementation details:
//   - Uses bit-set checking (very fast: ~2 instructions)
//   - Valid vtables are placed in specific memory ranges at link time
//   - Check: (vptr - base) < range_size && bit_test(bitset, index)

void normal_dispatch() {
    std::unique_ptr<Animal> dog = std::make_unique<Dog>();
    std::unique_ptr<Animal> cat = std::make_unique<Cat>();

    dog->speak();  // CFI: vptr -> Dog::vtable -> in valid set -> OK
    cat->speak();  // CFI: vptr -> Cat::vtable -> in valid set -> OK
    // Output:
    // Woof!
    // Meow!
}

void simulated_attack() {
    // Simulate vtable corruption
    Dog real_dog;
    Unrelated bad_obj;

    // Attacker corrupts the vtable pointer:
    // *(void**)&real_dog = *(void**)&bad_obj;
    // ^ This would make real_dog.speak() call Unrelated::hack()!
    //
    // WITH CFI:
    //   Before calling speak(), CFI checks if vptr points to a valid Animal vtable.
    //   Unrelated::vtable is NOT in the valid set for Animal.
    //   -> SIGILL (trap) -> program aborts BEFORE the malicious code runs.
    //
    // WITHOUT CFI:
    //   The corrupted vptr is followed blindly.
    //   -> Calls Unrelated::hack() or attacker's shellcode.

    std::cout << "Vtable corruption would be caught by CFI before dispatch.\n";
}

int main() {
    normal_dispatch();
    simulated_attack();
}
```

**Explanation:** CFI-vcall inserts a validation check before every virtual function call. At link time (requires LTO), the compiler knows all legitimate vtables for each class hierarchy. At runtime, before dispatching through the vtable pointer, CFI verifies the pointer targets a legitimate vtable for that hierarchy. If the vtable pointer was corrupted to point elsewhere, the check fails and the process is terminated (SIGILL trap) before the attacker's code executes.

### Q2: Show how CFI prevents an attacker from overwriting a vtable pointer to redirect control flow

**Answer:**

This is a more concrete attack scenario. Notice the step-by-step comment walking through exactly what the attacker achieves without CFI, and what CFI stops:

```cpp
#include <iostream>
#include <cstring>
#include <cstdint>
#include <memory>

// Attack scenario: vtable pointer overwrite

struct PaymentProcessor {
    virtual void process_payment(int amount) {
        std::cout << "Processing $" << amount << " payment\n";
    }
    virtual ~PaymentProcessor() = default;
};

struct SecureProcessor : PaymentProcessor {
    void process_payment(int amount) override {
        std::cout << "Secure payment: $" << amount << " (verified)\n";
    }
};

// Attacker's goal: redirect process_payment to malicious code
struct MaliciousClass {
    virtual void steal_credentials() {
        std::cout << "STOLEN: user credentials exfiltrated!\n";
    }
    virtual ~MaliciousClass() = default;
};

// The attack (simplified)
void demonstrate_attack() {
    auto processor = std::make_unique<SecureProcessor>();

    // Normal operation:
    processor->process_payment(100);
    // Output: Secure payment: $100 (verified)

    // Attacker has a buffer overflow that reaches the vtable pointer.
    // They overwrite it to point to MaliciousClass's vtable.
    //
    // Step by step:
    //   1. Object in memory: [vptr | fields...]
    //   2. vptr normally -> SecureProcessor::vtable
    //   3. Buffer overflow overwrites vptr -> MaliciousClass::vtable
    //   4. Next virtual call: processor->process_payment(100)
    //      -> dispatches through MaliciousClass::vtable
    //      -> calls steal_credentials() instead of process_payment()!

    // WITHOUT CFI: attacker succeeds, credentials stolen
    // WITH CFI:
    //   Before virtual dispatch, CFI checks:
    //     Is processor->vptr in {PaymentProcessor::vtable, SecureProcessor::vtable}?
    //     MaliciousClass::vtable is NOT in this set.
    //     -> CFI violation detected -> SIGILL -> program terminates
    //     -> Attack BLOCKED, no credentials stolen

    std::cout << "Without CFI: attacker redirects virtual call to malicious code\n";
    std::cout << "With CFI:    SIGILL trap before malicious code executes\n";
}

// Real-world vtable attack vectors
//
// 1. Use-after-free:
//    delete obj;
//    // ... allocator reuses memory ...
//    new(obj_memory) MaliciousClass();  // place attacker vtable
//    obj->virtual_method();             // dispatches to attacker
//    -> CFI blocks: attacker's vtable not in valid set
//
// 2. Type confusion:
//    Base* b = static_cast<Base*>(reinterpret_cast<void*>(user_data));
//    b->method();  // user_data is not actually a Base!
//    -> cfi-unrelated-cast blocks the cast itself
//
// 3. Stack buffer overflow:
//    struct { char buf[16]; Base* ptr; } locals;
//    overflow(locals.buf);  // overwrites ptr
//    locals.ptr->method();  // corrupted ptr
//    -> CFI blocks: corrupted ptr's vptr not in valid set

int main() {
    demonstrate_attack();
    // Output:
    // Secure payment: $100 (verified)
    // Without CFI: attacker redirects virtual call to malicious code
    // With CFI:    SIGILL trap before malicious code executes
}
```

**Explanation:** The attacker corrupts an object's vtable pointer (via buffer overflow, use-after-free, or type confusion) so that the next virtual function call dispatches through a malicious vtable. CFI prevents this by validating that the vtable pointer belongs to the set of legitimate vtables for the declared type hierarchy. Since the attacker's fake vtable is not in this set, CFI triggers a trap before any malicious code executes.

### Q3: Explain the performance cost of CFI and strategies to apply it selectively to sensitive code

**Answer:**

The reason CFI's overhead is so low is that the check itself compiles down to two or three CPU instructions on top of the pointer load that was already happening anyway. The branch is almost never taken (an invalid vtable is the rare exception), so the CPU branch predictor handles it efficiently:

```cpp
// Performance Cost of CFI
//
// Benchmark data (Clang, from Google Chrome and Android):
//
//   CFI scheme        Overhead    Instructions added per call
//   cfi-vcall         < 1%        2-3 instructions (bit test + branch)
//   cfi-icall         1-2%        Similar, for function pointers
//   cfi-derived-cast  < 0.5%      Only at cast sites
//   cfi-nvcall        < 0.5%      Rare in practice
//   Full CFI          1-3%        All combined
//
// Cost breakdown:
//   - Code size: +5-10% (LTO + check instructions)
//   - Build time: +20-50% (LTO required - whole-program analysis)
//   - Runtime: +1-3% (extra instructions at dispatch sites)
//   - Memory: +small (vtable metadata)
//
// The check itself is very fast:
//   1. Load vtable pointer (already needed for virtual call)
//   2. Compute offset from base (subtract)
//   3. Check bounds (compare)
//   4. Test bit in bitset (single instruction)
//   5. Branch to trap if invalid (rarely taken -> predictor happy)

// Selective Application Strategies
//
// Strategy 1: Per-file CFI
//   # Compile only security-critical files with CFI:
//   clang++ -flto -fvisibility=hidden -fsanitize=cfi auth.cpp payment.cpp -c
//   clang++ -flto other.cpp utils.cpp -c
//   clang++ -flto -fsanitize=cfi *.o -o app
//
// Strategy 2: Disable CFI for specific functions
//   __attribute__((no_sanitize("cfi")))
//   void performance_critical_function() { ... }
//
// Strategy 3: Disable CFI for specific hot loops
//   [[clang::no_sanitize("cfi-vcall")]]
//   void hot_path(Base* objects[], int n) {
//       for (int i = 0; i < n; ++i)
//           objects[i]->process();  // No CFI check here
//   }
//
// Strategy 4: CFI in diagnostics mode (testing before enforcement)
//   -fsanitize=cfi -fsanitize-recover=all -fno-sanitize-trap=cfi
//   -> Logs violations but doesn't abort -> find false positives first
//
// Strategy 5: Combine with other mitigations
//   # Critical binary: full CFI
//   clang++ -flto -fvisibility=hidden -fsanitize=cfi \
//           -fstack-protector-strong -fPIE -pie app.cpp
//
//   # Performance-sensitive binary: partial CFI + other mitigations
//   clang++ -flto -fvisibility=hidden -fsanitize=cfi-vcall \
//           -fcf-protection=full app.cpp  # Intel CET for the rest

#include <iostream>

// Example: selective CFI with attribute
struct HotPath {
    virtual void tick() { /* called millions of times per frame */ }
};

// Exempt from CFI for performance
// __attribute__((no_sanitize("cfi-vcall")))
void game_loop(HotPath* objects[], int count) {
    for (int i = 0; i < count; ++i) {
        objects[i]->tick();  // CFI disabled for this callsite
    }
}

// Always CFI-protected
void authenticate(auto* auth_service) {
    auth_service->verify();  // CFI check here - security critical
}

int main() {
    std::cout << "CFI performance: ~1-3% overhead for full protection.\n";
    std::cout << "Build requirement: LTO (-flto) + visibility (-fvisibility=hidden)\n";
    std::cout << "Apply selectively with [[no_sanitize(\"cfi\")]] on hot paths.\n";
    std::cout << "Test in diagnostics mode first: -fsanitize-recover=all\n";
}
```

**Explanation:** Full CFI adds ~1-3% runtime overhead and requires LTO (Link-Time Optimization), which increases build time by 20-50%. For selective application: (1) compile only security-critical modules with CFI, (2) use `no_sanitize("cfi")` attributes on performance-critical hot paths, (3) start with diagnostics mode (`-fsanitize-recover=all`) to find false positives before enforcing. Google deploys CFI in Chrome and Android - the overhead is acceptable for production.

---

## Notes

- **CFI requires Clang + LTO** - GCC does not support Clang-style CFI (it has `-fcf-protection` for Intel CET instead).
- **`-fvisibility=hidden`** is required so that vtable type information is available across translation units.
- **Google Chrome** uses CFI in production, proving it's viable at scale.
- **Android** requires CFI for all new native code since Android 9.
- **False positives:** Custom allocators, `dlopen`-loaded plugins, and C++/C boundary code may need `no_sanitize` attributes.
- **Intel CET** (`-fcf-protection=full`) provides hardware-assisted shadow stack and indirect-branch tracking - complementary to software CFI.
