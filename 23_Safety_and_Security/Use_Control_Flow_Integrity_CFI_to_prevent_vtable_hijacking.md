# Use Control Flow Integrity (CFI) to prevent vtable hijacking

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/ControlFlowIntegrity.html>  

---

## Topic Overview

This topic focuses on the **practical mechanics** of Clang's CFI implementation - what exactly happens at each virtual dispatch site, how CFI violations are triggered, and how to integrate CFI checks into existing codebases with minimal disruption. Where the companion file covers the attack model and performance, this file drills into the low-level check mechanism and diagnostic workflow.

Understanding the exact check sequence matters when you're debugging a false positive or trying to understand why a particular piece of code trips CFI. The instructions added are cheap, but knowing where they live helps you reason about when they fire.

### What CFI Checks at Each Virtual Dispatch

Here is the side-by-side view of what the compiler generates without and with CFI. The three extra steps in the instrumented version are the entire cost of the protection:

```cpp
Source code:      base_ptr->virtual_method(args);

Without CFI (normal virtual dispatch):

  1. Load vptr:     void** vptr = *(void**)base_ptr;
  2. Load entry:    void* fn = vptr[method_index];
  3. Call:          fn(base_ptr, args);

With CFI (instrumented):

  1. Load vptr:     void** vptr = *(void**)base_ptr;
  2. CFI check:

     a. offset = (uintptr_t)vptr - (uintptr_t)__cfi_typeid_base;
     b. if (offset >= valid_range)  -> trap;
     c. if (!bitset[offset / alignment])  -> trap;

  3. Load entry:    void* fn = vptr[method_index];   // only reached if check passes
  4. Call:          fn(base_ptr, args);

Total overhead: 2-3 extra instructions (subtract, compare, bit-test)
```

Notice that step 1 (loading the vtable pointer) was already there. CFI only adds steps 2a through 2c. The bitset lookup is a single CPU instruction, and on non-attack paths the branch at 2b/2c is never taken, so the branch predictor costs nothing.

### CFI vs Other Mitigations

Different tools take different approaches to the same problem. This table helps you pick the right tool for your platform and threat model:

| Feature | Clang CFI | Intel CET IBT | GCC VTV | `-fvtable-verify` |
| --- | --- | --- | --- | --- |
| Mechanism | Bit-set check | Hardware indirect-branch tracking | Hash table lookup | Deprecated |
| Requires LTO | Yes | No | No | No |
| Coverage | vcall, icall, casts | All indirect branches | Virtual calls only | Virtual calls only |
| Overhead | 1-3% | < 1% (hardware) | 2-5% | 2-5% |
| Production use | Chrome, Android | Linux kernel 6.2+ | Experimental | Deprecated |

The big tradeoff is Clang CFI vs Intel CET: CFI is software-only and works on any CPU, but requires LTO and is Clang-specific. CET is hardware-accelerated and requires no LTO, but needs a 2020+ Intel or AMD processor.

### Core Example

The compile command is the important part here. All three flags together are required - omitting any one of them either disables CFI or silently makes it ineffective:

```cpp
#include <iostream>

// Compile: clang++ -std=c++20 -flto -fvisibility=hidden -fsanitize=cfi-vcall -fno-sanitize-recover=all demo.cpp

struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double r;
    Circle(double radius) : r(radius) {}
    double area() const override { return 3.14159 * r * r; }
};

struct Square : Shape {
    double s;
    Square(double side) : s(side) {}
    double area() const override { return s * s; }
};

void print_area(const Shape& shape) {
    // CFI checks shape's vtable pointer before dispatching area()
    std::cout << "Area: " << shape.area() << "\n";
}

int main() {
    Circle c(5.0);
    Square s(4.0);
    print_area(c);  // CFI: Circle vtable in {Circle, Square} -> OK
    print_area(s);  // CFI: Square vtable in {Circle, Square} -> OK
}
```

---

## Self-Assessment

### Q1: Compile with -fsanitize=cfi-vcall and explain what it checks at each virtual dispatch

**Answer:**

Pay attention to the "What CFI checks at each dispatch" comment block - it spells out the exact memory operations and conditions that CFI evaluates for every virtual call. This is the mental model you need for diagnosing false positives:

```cpp
#include <iostream>
#include <memory>

// Build command
// clang++ -std=c++20 -flto -fvisibility=hidden \
//         -fsanitize=cfi-vcall \
//         -fno-sanitize-recover=all \
//         -O1 -g cfi_check.cpp -o cfi_check
//
// REQUIRED flags:
//   -flto               LTO is MANDATORY (CFI needs cross-TU type info)
//   -fvisibility=hidden Ensures vtable type metadata is emitted
//   -fsanitize=cfi-vcall Enable virtual call checks
//   -fno-sanitize-recover=all  Abort on violation (don't just log)

struct __attribute__((visibility("default"))) Logger {
    virtual void log(const char* msg) = 0;
    virtual ~Logger() = default;
};

struct __attribute__((visibility("default"))) FileLogger : Logger {
    void log(const char* msg) override {
        std::cout << "[FILE] " << msg << "\n";
    }
};

struct __attribute__((visibility("default"))) ConsoleLogger : Logger {
    void log(const char* msg) override {
        std::cout << "[CONSOLE] " << msg << "\n";
    }
};

// What CFI checks at each dispatch
//
// For: logger->log("hello")
//
// Step 1: Extract vtable pointer
//   void** vptr = *(void**)logger;
//
// Step 2: Validate vtable pointer against type ID
//   The compiler assigned type IDs to Logger's hierarchy at link time.
//   All valid vtables (FileLogger, ConsoleLogger) are in a contiguous
//   memory range with a bitset indicating valid positions.
//
//   Check:
//     uintptr_t offset = (uintptr_t)vptr - type_base_address;
//     if (offset >= type_range_size)       -> FAIL (out of valid range)
//     if (offset % alignment != 0)         -> FAIL (misaligned)
//     if (!type_bitset[offset / alignment]) -> FAIL (not a known vtable)
//
// Step 3: On success, proceed with normal virtual dispatch
//   void* fn = vptr[slot_for_log];
//   fn(logger, "hello");
//
// Step 4: On failure
//   Default: ud2 instruction -> SIGILL
//   With -fno-sanitize-trap: calls __cfi_check_fail(data, addr)
//   With -fsanitize-recover: logs and continues (testing mode)

void use_logger(Logger& logger) {
    logger.log("CFI validates before this call dispatches");
}

int main() {
    FileLogger fl;
    ConsoleLogger cl;

    use_logger(fl);  // CFI check passes -> dispatches to FileLogger::log
    use_logger(cl);  // CFI check passes -> dispatches to ConsoleLogger::log

    // Output:
    // [FILE] CFI validates before this call dispatches
    // [CONSOLE] CFI validates before this call dispatches

    std::cout << "\nAll virtual calls passed CFI validation.\n";
}
```

**Explanation:** At each virtual call site, CFI adds 2-3 instructions: it loads the vtable pointer, computes its offset from the base of the valid vtable range for the class hierarchy, and tests a bit in a precomputed bitset. If the vtable pointer doesn't correspond to a known legitimate vtable (e.g., it was corrupted by an attacker), the check fails and the process is terminated via an illegal instruction trap (SIGILL) before any attacker-controlled code runs.

### Q2: Show a CFI violation: casting a void* to a class pointer and calling a virtual function

**Answer:**

This example shows two distinct CFI schemes catching the same type-confusion attack at different points - `cfi-unrelated-cast` catches the bad cast itself, while `cfi-vcall` catches the resulting virtual call. Both protect you; layering them means the attack is stopped as early as possible:

```cpp
#include <iostream>
#include <cstring>
#include <cstdint>

// Compile: clang++ -std=c++20 -flto -fvisibility=hidden \
//   -fsanitize=cfi-unrelated-cast,cfi-vcall -fno-sanitize-recover=all cfi_cast.cpp

struct __attribute__((visibility("default"))) Trusted {
    virtual void execute() {
        std::cout << "Trusted::execute - legitimate operation\n";
    }
    virtual ~Trusted() = default;
};

struct __attribute__((visibility("default"))) Unrelated {
    int data[4] = {1, 2, 3, 4};
    // NOT derived from Trusted - no virtual functions matching Trusted's layout
};

// CFI Violation: void* -> Trusted*
void demonstrate_violation() {
    Unrelated obj;

    // Type confusion attack: treat arbitrary memory as a Trusted object
    void* raw = &obj;

    // WITHOUT CFI: this "works" - reads garbage as vtable pointer
    // Trusted* fake = reinterpret_cast<Trusted*>(raw);
    // fake->execute();  // UB: reads obj.data[0] as vptr, dispatches to random address

    // WITH cfi-unrelated-cast:
    //   The reinterpret_cast itself is caught!
    //   CFI checks: does 'raw' point to an object whose vtable is valid for Trusted?
    //   Answer: NO (Unrelated has no relationship to Trusted)
    //   -> SIGILL trap at the CAST, before the virtual call even happens.

    // WITH cfi-vcall only (no cast check):
    //   The cast succeeds, but the virtual call is caught:
    //   CFI checks: does fake->vptr point to a valid Trusted vtable?
    //   obj.data[0] = 1 -> address 0x1 is NOT a valid vtable
    //   -> SIGILL trap at the virtual call site

    std::cout << "With CFI, the type-confused cast or call would trap.\n";
    std::cout << "cfi-unrelated-cast: catches the bad cast itself\n";
    std::cout << "cfi-vcall: catches the virtual call through invalid vtable\n";
}

// Common patterns that trigger CFI
//
// 1. Deserialization:
//    void* obj = deserialize(network_data);
//    static_cast<Handler*>(obj)->handle();
//    -> cfi-derived-cast: checks that obj is really a Handler
//
// 2. Plugin loading:
//    void* sym = dlsym(handle, "create");
//    auto factory = reinterpret_cast<Base*(*)()>(sym);
//    Base* plugin = factory();
//    -> cfi-icall: checks that 'sym' has the expected function signature
//
// 3. C callback wrappers:
//    void c_callback(void* user_data) {
//        auto* obj = static_cast<Handler*>(user_data);
//        obj->on_event();  // If user_data is wrong type -> CFI trap
//    }

int main() {
    // Normal usage - CFI passes
    Trusted t;
    t.execute();
    // Output: Trusted::execute - legitimate operation

    demonstrate_violation();
    // Output (without triggering actual violation):
    // With CFI, the type-confused cast or call would trap.
    // cfi-unrelated-cast: catches the bad cast itself
    // cfi-vcall: catches the virtual call through invalid vtable
}
```

**Explanation:** When a `void*` is cast to a class pointer and a virtual function is called, CFI validates the cast (with `cfi-unrelated-cast`) and/or the virtual dispatch (with `cfi-vcall`). If the memory doesn't contain a valid vtable for the target type, CFI triggers a trap. This catches type confusion attacks where an attacker provides controlled memory disguised as a polymorphic object.

### Q3: Explain the performance cost of CFI checks and how to apply them selectively

**Answer:**

The three-phase rollout strategy in this example is the recommended way to adopt CFI in an existing production codebase without surprises. Start in recovery mode to surface false positives, then enforce, then switch to the minimal trap mode for production:

```cpp
// Detailed Performance Analysis
//
// Measurement source: Google (Chrome, Android), LLVM benchmarks
//
// Build-time cost:
//   - LTO required -> 20-50% longer link time
//   - More memory during link (whole-program in memory)
//   - Mitigation: use ThinLTO instead of full LTO:
//     clang++ -flto=thin -fsanitize=cfi ...
//     ThinLTO: 80% of LTO benefit, 30% of link time cost
//
// Runtime cost per virtual call:
//   Instructions:  2-3 (load, subtract, bittest, conditional trap)
//   Cycles:        1-2 (modern CPU branch prediction handles this well)
//   Cache impact:  Minimal (bitset is small, stays in L1)
//
// Aggregate benchmarks:
//   SPEC CPU 2017:   +1.5% geomean (CFI-vcall only)
//   Chromium:        +0.5-1.5% (real-world browser benchmark)
//   Android system:  < 2% (full CFI on all native code)

// Selective Application Strategies

// Strategy 1: Per-module CFI
// Build only critical modules with CFI:
//
// CMakeLists.txt:
//   # Security-critical modules
//   set_source_files_properties(
//       auth.cpp payment.cpp crypto.cpp
//       PROPERTIES COMPILE_FLAGS "-fsanitize=cfi -fvisibility=hidden"
//   )
//   # Performance-critical modules: no CFI
//   set_source_files_properties(
//       render.cpp physics.cpp audio.cpp
//       PROPERTIES COMPILE_FLAGS ""
//   )
//   # Must still link with LTO and -fsanitize=cfi

// Strategy 2: Function-level opt-out
#include <iostream>

struct Renderer {
    virtual void render_frame() {
        // Called 60+ times per second, millions of objects
        std::cout << "Rendering frame\n";
    }
    virtual ~Renderer() = default;
};

// CFI disabled for verified hot path
// __attribute__((no_sanitize("cfi-vcall")))
void render_loop(Renderer* renderers[], int n) {
    for (int i = 0; i < n; ++i) {
        renderers[i]->render_frame();  // No CFI check - maximum speed
    }
}

// CFI enabled for security-critical path (default)
struct Authenticator {
    virtual bool verify(const char* token) {
        std::cout << "Verifying token\n";
        return true; // simplified
    }
    virtual ~Authenticator() = default;
};

void authenticate_request(Authenticator& auth, const char* token) {
    auth.verify(token);  // CFI check here - protecting auth pipeline
}

// Strategy 3: Diagnostics mode for rollout
//
// Phase 1: Discovery
//   -fsanitize=cfi -fsanitize-recover=all -fno-sanitize-trap=cfi
//   -> Logs violations but continues running
//   -> Find and fix all false positives
//
// Phase 2: Enforcement
//   -fsanitize=cfi -fno-sanitize-recover=all
//   -> Abort on any violation
//
// Phase 3: Trap mode (production)
//   -fsanitize=cfi -fsanitize-trap=cfi
//   -> Minimal overhead, no recovery handler, just SIGILL

int main() {
    std::cout << "CFI selective application strategies:\n";
    std::cout << "1. Per-module (CMake source properties)\n";
    std::cout << "2. Per-function (__attribute__((no_sanitize(\"cfi\"))))\n";
    std::cout << "3. Phased rollout (recover -> enforce -> trap)\n";
    std::cout << "4. ThinLTO for faster builds (-flto=thin)\n";
}
```

**Explanation:** CFI-vcall adds 1-3% runtime overhead and 20-50% build time (due to LTO). For selective application: use ThinLTO instead of full LTO to reduce build time, apply `no_sanitize("cfi")` attributes to verified hot paths, compile only security-critical modules with CFI flags, and use a phased rollout (diagnostics mode - enforcement - trap mode) to find false positives before enforcing in production.

---

## Notes

- **`-flto=thin`** (ThinLTO) is recommended over full LTO - provides CFI with significantly faster link times.
- **`cfi-icall`** is particularly important for C FFI boundaries where function pointers from external code may be corrupted.
- **Clang-only:** GCC does not implement the same CFI scheme. GCC provides `-fcf-protection` for Intel CET hardware support.
- **`-fsanitize-blacklist=file.txt`** can exclude specific functions/files from CFI to handle third-party code.
- **Interaction with dlopen:** Dynamically loaded libraries need `cfi-icall`-compatible type info. Use `-fsanitize-cfi-cross-dso` for cross-DSO CFI.
- CFI is deployed in production by Google (Chrome, Android), Microsoft (Edge), and Apple (WebKit).
