# Prevent use-after-free with ownership discipline and lifetime analysis

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/analyzer/developer-docs/DebugChecks.html>  

---

## Topic Overview

Use-after-free (UAF) is one of the most exploitable vulnerability classes in C++. It occurs when a program accesses memory after it has been freed or deallocated. Attackers can reallocate the freed memory with controlled data, then trigger the stale pointer access to achieve code execution.

The reason UAF is so serious is the exploit model. It's not just a crash - it's a takeover opportunity.

### UAF Attack Model

Walk through the attack sequence step by step. The key moment is step 4, where the attacker gets to put their own data where your pointer still points:

```cpp
1. Object A allocated at address 0x1000
2. Pointer p -> 0x1000
3. Object A freed (0x1000 returned to allocator)
4. Attacker causes allocation B at 0x1000 (controlled content)
5. Code uses p -> reads attacker-controlled data from B!

   If p was a vtable pointer -> virtual call dispatches to attacker code
```

The vtable case is especially dangerous - a virtual call through a freed object is a direct path to arbitrary code execution.

### Prevention Strategies

There's no single magic fix, but a combination of strategies at different levels:

| Strategy | Level | Tool/Technique |
| --- | --- | --- |
| RAII + smart pointers | Design | `unique_ptr`, `shared_ptr` |
| Lifetime annotations | Static analysis | `gsl::owner<>`, `gsl::not_null<>` |
| Compiler warnings | Build | `-Wdangling`, `-Wdangling-pointer` |
| Clang lifetime analysis | Static analysis | `-Wlifetime` (experimental) |
| AddressSanitizer | Runtime | `-fsanitize=address` |
| MTE (ARM) | Hardware | Memory Tagging Extension |

The design-level fix (smart pointers) prevents the bug from being written. The tooling-level fixes (warnings, ASan) catch it after the fact. Use both.

### Core Example

Here's a concrete comparison between the raw-pointer pattern that creates UAF and the smart-pointer pattern that prevents it:

```cpp
#include <memory>
#include <string>
#include <iostream>
#include <vector>

// BAD: use-after-free with raw pointer
std::string* create_bad() {
    auto* s = new std::string("hello");
    delete s;
    // return s; // DANGLING POINTER - s points to freed memory!
    return nullptr;
}

// GOOD: unique_ptr prevents UAF
std::unique_ptr<std::string> create_good() {
    return std::make_unique<std::string>("hello");
    // Ownership transfers to caller - no dangling pointer possible
}

// Common UAF patterns
void dangling_reference() {
    std::vector<int> v{1, 2, 3};
    int& ref = v[0];
    v.push_back(4); // MAY reallocate -> ref is now dangling!
    // int x = ref;  // USE-AFTER-FREE (if reallocation occurred)
}

int main() {
    auto ptr = create_good();
    std::cout << *ptr << "\n"; // "hello" - safe
}
```

With `unique_ptr`, the memory is tied to the pointer's lifetime. When the `unique_ptr` is destroyed, the memory is freed. There's no way to accidentally have a raw pointer to the old address still in use.

---

## Self-Assessment

### Q1: Run the Clang lifetime safety analysis (-Wdangling, -Wdangling-pointer) on a codebase

**Answer:**

Static analysis and runtime detection complement each other here. Static analysis catches patterns the compiler can reason about at compile time; ASan catches everything else during test execution:

```cpp
// Compiler flags for lifetime safety
//
// GCC:
//   -Wdangling-pointer       Warn about pointers to locals that escape scope
//   -Wdangling-pointer=2     More aggressive (warn about members too)
//   -Wreturn-local-addr      Warn about returning address of local variable
//   -Wuse-after-free         Warn about use-after-free patterns
//   -Wuse-after-free=3       Most aggressive level
//
// Clang:
//   -Wdangling               Warn about dangling references/pointers
//   -Wdangling-assignment    Warn about assigning dangling pointer
//   -Wreturn-stack-address   Warn about returning stack address
//   -Wdangling-gsl           Warn about GSL owner/pointer violations
//
// Both:
//   -fsanitize=address       Runtime: detects UAF with 100% accuracy

// Examples each flag catches

#include <string>
#include <string_view>
#include <vector>
#include <iostream>

// Caught by -Wreturn-local-addr / -Wreturn-stack-address:
// int* bad_return() {
//     int local = 42;
//     return &local;  // WARNING: address of local variable returned
// }

// Caught by -Wdangling:
// std::string_view bad_view() {
//     std::string temp = "hello";
//     return temp;  // WARNING: returning view of local (dangling)
// }

// Caught by -Wdangling-pointer (GCC):
// int* bad_assign() {
//     int* p;
//     {
//         int local = 42;
//         p = &local;
//     } // local destroyed
//     return p;  // WARNING: dangling pointer to local
// }

// Caught by AddressSanitizer at RUNTIME:
void asan_demo() {
    std::vector<int> v{1, 2, 3};
    int* p = &v[0];
    v.push_back(4);  // may reallocate
    // int x = *p;   // ASan: heap-use-after-free
    (void)p;
}

int main() {
    asan_demo();
    std::cout << "Compiled with lifetime warnings enabled\n";
}

// Build commands
// Debug build (maximum safety):
// g++ -std=c++20 -Wall -Wextra -Wdangling-pointer=2 -Wuse-after-free=3 \
//     -fsanitize=address,undefined -fno-omit-frame-pointer -O1 code.cpp
//
// Clang:
// clang++ -std=c++20 -Wall -Wextra -Wdangling -Wdangling-gsl \
//         -fsanitize=address,undefined -fno-omit-frame-pointer -O1 code.cpp
//
// CI pipeline: run FULL test suite with ASan enabled
// ASan adds ~2x slowdown but catches ALL heap-use-after-free at runtime
```

Static analysis catches obvious patterns (returning local addresses, dangling references to temporaries), while AddressSanitizer catches runtime UAF that depends on execution paths (vector reallocation, conditional deletes, multithreaded access). Use both: compiler warnings in all builds, ASan in CI test runs.

### Q2: Show a use-after-free introduced by returning a reference to a member of a temporary

**Answer:**

This is one of the subtler UAF patterns. The reason it trips people up is that the code looks completely harmless at first glance - there's no `delete` in sight:

```cpp
#include <string>
#include <string_view>
#include <iostream>
#include <vector>

struct Config {
    std::string name;
    std::vector<int> values;

    const std::string& get_name() const { return name; }
    const int& first_value() const { return values[0]; }
};

Config make_config() {
    return Config{"production", {1, 2, 3}};
}

int main() {
    // BUG 1: Reference to member of temporary
    // const std::string& name = make_config().get_name();
    // The temporary Config is destroyed at the semicolon.
    // name is now a dangling reference to the destroyed string.
    // std::cout << name << "\n";  // USE-AFTER-FREE!

    // WHY: make_config() returns a temporary.
    //       .get_name() returns a reference to temporary.name
    //       Temporary dies at ;
    //       Reference outlives the temporary's member.

    // FIX 1: Bind the temporary to extend its lifetime
    const Config& cfg = make_config(); // temporary lifetime extended!
    std::cout << cfg.get_name() << "\n"; // "production" - safe

    // FIX 2: Copy the result
    std::string name_copy = make_config().get_name(); // copies the string
    std::cout << name_copy << "\n"; // safe

    // BUG 2: string_view to temporary string
    // std::string_view sv = std::string("temp");
    // std::string("temp") is destroyed at ;
    // sv now points to freed memory!
    // std::cout << sv << "\n";  // USE-AFTER-FREE!

    // Fix: keep the string alive
    std::string kept = "temp";
    std::string_view sv = kept;
    std::cout << sv << "\n"; // safe

    // BUG 3: Iterator invalidation (vector UAF)
    std::vector<std::string> names{"Alice", "Bob"};
    // for (const auto& name : names) {
    //     if (name == "Alice")
    //         names.push_back("Charlie"); // INVALIDATES iterators!
    //     // next iteration: USE-AFTER-FREE (if reallocation occurred)
    // }

    // Fix: don't modify container during range-for
    std::vector<std::string> to_add;
    for (const auto& n : names) {
        if (n == "Alice") to_add.push_back("Charlie");
    }
    names.insert(names.end(), to_add.begin(), to_add.end());

    // BUG 4: Lambda capturing reference to local
    // auto make_lambda() {
    //     int local = 42;
    //     return [&local]() { return local; }; // DANGLING capture!
    // }

    // Fix: capture by value
    // return [local]() { return local; }; // copy, not reference
}
```

Returning a reference to a member of a temporary is one of the subtlest UAF patterns. The temporary object (and all its members) is destroyed at the end of the full expression. Any reference to its internals becomes dangling immediately. Clang's `-Wdangling` catches many of these statically. The fix is to either bind the entire temporary to a `const` reference (extending its lifetime) or copy the value.

### Q3: Use lifetime profile annotations (gsl::owner<>, gsl::not_null<>) to document ownership

**Answer:**

Even when you can't immediately migrate legacy code to smart pointers, you can use GSL annotations to make ownership explicit and enable static analysis to check it. Think of `gsl::owner<>` as a comment that the compiler can reason about:

```cpp
#include <memory>
#include <iostream>
#include <stdexcept>
#include <cassert>

// GSL (Guidelines Support Library) provides annotation types
// that document ownership and nullability intent.
// Install: https://github.com/microsoft/GSL
// Or use the concepts without the library:

namespace gsl {
    // owner<T> - documents that this raw pointer OWNS the pointee
    // The holder is responsible for deleting it.
    template<typename T>
    using owner = T;

    // not_null<T> - documents that this pointer is never null
    template<typename T>
    class not_null {
        T ptr_;
    public:
        not_null(T p) : ptr_(p) {
            if (ptr_ == nullptr)
                throw std::logic_error("not_null violated");
        }
        T get() const { return ptr_; }
        operator T() const { return ptr_; }
        T operator->() const { return ptr_; }
        auto& operator*() const { return *ptr_; }
    };
}

// Usage: owner<> documents ownership transfer

class ResourceManager {
    // clang-tidy's cppcoreguidelines-owning-memory check
    // enforces that owner<> pointers are properly deleted

    gsl::owner<int*> data_; // I OWN this memory

public:
    ResourceManager() : data_(new int(42)) {}

    ~ResourceManager() {
        delete data_; // owner<> reminds us to delete
    }

    // Transfer ownership - caller now owns the pointer
    [[nodiscard]] gsl::owner<int*> release() {
        auto* p = data_;
        data_ = nullptr;
        return p; // caller MUST delete this
    }

    // Non-owning access - caller must NOT delete
    int* borrow() const { return data_; } // no owner<> = non-owning
};

// Usage: not_null<> documents nullability contract

void process(gsl::not_null<int*> p) {
    // No null check needed - guaranteed non-null by type system
    *p += 1;
    std::cout << "Value: " << *p << "\n";
}

// Better: combine with smart pointers
void process_unique(gsl::not_null<std::unique_ptr<int>> p) {
    std::cout << "Unique value: " << *p << "\n";
}

int main() {
    // owner<> example
    ResourceManager rm;
    int* borrowed = rm.borrow(); // non-owning - don't delete!
    std::cout << "Borrowed: " << *borrowed << "\n"; // 42

    gsl::owner<int*> owned = rm.release(); // I now own this
    std::cout << "Owned: " << *owned << "\n"; // 42
    delete owned; // MUST delete - I'm the owner

    // not_null<> example
    int x = 10;
    process(&x); // OK - &x is not null
    // Output: Value: 11

    // int* np = nullptr;
    // process(np); // THROWS: not_null violated

    // Practical benefit for static analysis
    // clang-tidy checks:
    //   cppcoreguidelines-owning-memory: enforces owner<> discipline
    //   cppcoreguidelines-pro-type-cstyle-cast: catches unsafe casts
    //
    // Example violations clang-tidy catches:
    //   gsl::owner<int*> p = new int(5);
    //   int* q = p;        // WARNING: assigning owner to non-owner
    //   delete q;          // WARNING: deleting non-owner pointer
    //   // CORRECT: gsl::owner<int*> q = p;

    // Best practice: prefer smart pointers over owner<>
    // owner<> is for legacy code that can't be migrated to unique_ptr.
    // For new code:
    auto safe = std::make_unique<int>(42);
    // No owner<> needed - unique_ptr encodes ownership in the type system
}
```

`gsl::owner<T>` is a type alias that documents "this raw pointer owns the memory." It enables clang-tidy's `cppcoreguidelines-owning-memory` check to verify that owned pointers are properly deleted and ownership is correctly transferred. `gsl::not_null<T>` wraps a pointer or smart pointer to enforce non-null at construction time, eliminating null-pointer dereference vulnerabilities. Both are documentation-and-analysis aids - prefer `unique_ptr`/`shared_ptr` for new code.

---

## Notes

- **UAF is CWE-416** and consistently ranks in the CWE Top 25. It's the number one vulnerability class in Chrome and Firefox.
- **AddressSanitizer** (`-fsanitize=address`) is the gold standard for detecting UAF at runtime. It inserts quarantine zones around freed memory.
- **`-Wdangling-gsl`** (Clang) specifically warns about GSL owner/pointer lifetime violations.
- **Prefer smart pointers:** `unique_ptr` eliminates UAF by design - when the owner is destroyed, the memory is freed and the pointer is nulled. No dangling reference possible.
- **`string_view` pitfall:** `string_view` is non-owning. Never store a `string_view` to a temporary `std::string`.
- Compile with `-std=c++20 -Wall -Wextra -Wdangling -fsanitize=address`.
