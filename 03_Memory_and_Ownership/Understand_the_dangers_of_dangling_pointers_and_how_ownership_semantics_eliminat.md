# Understand the Dangers of Dangling Pointers and How Ownership Semantics Eliminate Them

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory>  

---

## Topic Overview

### What Is a Dangling Pointer

A **dangling pointer** points to memory that has been freed or to an object whose lifetime has ended. Dereferencing it is **undefined behavior** - the value you read might be garbage, it might be a partially overwritten object, or it might look correct today and silently corrupt memory tomorrow.

### Common Causes

| Cause | Example |
| --- | --- |
| Returning pointer to local | `int* f() { int x = 5; return &x; }` |
| `delete` then use | `delete p; p->method();` |
| Container invalidation | `v.push_back(x);` invalidates iterators |
| Scope mismatch | Object destroyed, pointer survives |
| Use-after-free | Free in one path, use in another |

### How Ownership Semantics Fix This

The whole idea is that if ownership is tracked automatically, the object stays alive exactly as long as it is needed. You cannot have a dangling pointer if the owner's destructor is what triggers the deletion.

| Ownership Model | Dangling Protection |
| --- | --- |
| `unique_ptr` | Single owner - destroyed with scope |
| `shared_ptr` | Reference counting - alive while any owner exists |
| `weak_ptr` | Detects expiry before access |
| Value semantics | No pointers at all - copies/moves |
| `std::optional` | "Nullable value" without pointer indirection |

---

## Self-Assessment

### Q1: Show a dangling pointer from returning a raw pointer to a local variable

The three "BAD" functions below each represent a different flavour of the same mistake. All of them compile without warnings in some configurations, which is exactly what makes them dangerous. The fixed versions show the right idiom for each case: return by value, return a smart pointer, or return an owning string.

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <vector>

// BUG: Returns pointer to local variable — instant dangling pointer
int* get_value_BAD() {
    int local = 42;
    return &local;  // WARNING: address of local variable returned!
    // local is destroyed here — pointer is dangling
}

// BUG: Pointer to temporary string data
const char* get_name_BAD() {
    std::string name = "Hello";
    return name.c_str();  // Dangling! string destroyed at function exit
}

// BUG: Pointer into vector that gets reallocated
int* get_element_BAD(std::vector<int>& v) {
    int* ptr = &v[0];
    v.push_back(999);  // May reallocate! ptr is now dangling
    return ptr;
}

// FIXED versions using ownership:

// Fix 1: Return by value (best when possible)
int get_value_GOOD() {
    int local = 42;
    return local;  // Copied — no pointer issues
}

// Fix 2: Return a smart pointer for heap allocation
std::unique_ptr<int> get_heap_value() {
    return std::make_unique<int>(42);  // Caller owns the memory
}

// Fix 3: Return by value (string copies/moves)
std::string get_name_GOOD() {
    std::string name = "Hello";
    return name;  // RVO or move — no dangling
}

int main() {
    std::cout << "=== Dangling pointer scenarios ===\n\n";

    // Demonstrating BAD patterns (commented out — would be UB):
    // int* p = get_value_BAD();
    // std::cout << *p;    // UB: dangling pointer dereference
    // const char* s = get_name_BAD();
    // std::cout << s;     // UB: string already destroyed

    // GOOD patterns:
    int val = get_value_GOOD();
    std::cout << "By value: " << val << "\n";

    auto ptr = get_heap_value();
    std::cout << "unique_ptr: " << *ptr << "\n";

    std::string name = get_name_GOOD();
    std::cout << "String: " << name << "\n";

    // Iterator invalidation example
    std::cout << "\n=== Iterator invalidation ===\n";
    std::vector<int> v = {1, 2, 3};
    // int& ref = v[0];
    // v.push_back(4);     // May invalidate ref!
    // std::cout << ref;    // Potential UB

    // Safe: use index instead of pointer/reference
    v.push_back(4);
    std::cout << "v[0] = " << v[0] << " (safe via index)\n";

    return 0;
}
// Expected output:
// === Dangling pointer scenarios ===
//
// By value: 42
// unique_ptr: 42
// String: Hello
//
// === Iterator invalidation ===
// v[0] = 1 (safe via index)
```

### Q2: Replace the dangling pointer with a `shared_ptr` return and verify lifetime extension

This example demonstrates the key promise of `shared_ptr`: the object stays alive as long as at least one `shared_ptr` holds a reference to it. Even after `pool` is destroyed and its internal reference drops away, `conn` still holds the connection open. The `use_count` output makes this visible.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Connection {
    std::string name;
    Connection(std::string n) : name(std::move(n)) {
        std::cout << "  [+] Connection '" << name << "' opened\n";
    }
    ~Connection() {
        std::cout << "  [-] Connection '" << name << "' closed\n";
    }
    void query(const char* q) const {
        std::cout << "  [?] " << name << ": " << q << "\n";
    }
};

// BAD: raw pointer — who owns this? When is it deleted?
// Connection* create_connection_BAD() {
//     return new Connection("DB");
//     // Caller might forget to delete -> leak
//     // Or delete too early -> dangling
//     // Or delete twice -> double-free
// }

// GOOD: shared_ptr — lifetime extends as long as any holder exists
std::shared_ptr<Connection> create_connection() {
    return std::make_shared<Connection>("DB");
}

class ConnectionPool {
    std::vector<std::shared_ptr<Connection>> pool_;
public:
    std::shared_ptr<Connection> get() {
        if (pool_.empty()) {
            pool_.push_back(create_connection());
        }
        return pool_.back();  // Shared ownership — can't dangle
    }
};

int main() {
    std::cout << "=== Shared ownership prevents dangling ===\n\n";

    std::shared_ptr<Connection> conn;

    {
        ConnectionPool pool;
        conn = pool.get();
        conn->query("SELECT 1");
        std::cout << "  use_count inside scope: " << conn.use_count() << "\n";
        // pool is destroyed here, but conn still holds a reference
    }

    std::cout << "\n  Pool destroyed, but connection still alive:\n";
    std::cout << "  use_count after scope: " << conn.use_count() << "\n";
    conn->query("SELECT 2");  // Still valid! shared_ptr extended lifetime

    conn.reset();  // Now the connection is truly destroyed
    std::cout << "\nDone.\n";

    return 0;
}
// Expected output:
// === Shared ownership prevents dangling ===
//
//   [+] Connection 'DB' opened
//   [?] DB: SELECT 1
//   use_count inside scope: 2
//
//   Pool destroyed, but connection still alive:
//   use_count after scope: 1
//   [?] DB: SELECT 2
//   [-] Connection 'DB' closed
//
// Done.
```

### Q3: Use AddressSanitizer to detect a dangling pointer use in an automated test

AddressSanitizer is the practical safety net for any codebase that still uses raw pointers or raw allocation. The test functions here are intentionally not executed (their dangerous lines are commented out), but they demonstrate exactly what ASan intercepts. The real takeaway is the compile flags and the `test_safe_patterns` function that shows what the safe equivalents look like.

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <cassert>

// Compile with:
//   g++ -fsanitize=address -g -O1 dangling_test.cpp -o dangling_test
//   clang++ -fsanitize=address -g -O1 dangling_test.cpp -o dangling_test
//
// ASan will detect:
// - Use after free (heap-use-after-free)
// - Stack use after return (stack-use-after-return, needs ASAN_OPTIONS)
// - Stack use after scope (stack-use-after-scope)
// - Double free
// - Memory leaks (leak sanitizer, part of ASan)

// Test 1: Detect use-after-free
void test_use_after_free() {
    int* p = new int(42);
    delete p;
    // ASan catches: *p is use-after-free
    // int val = *p;  // ASAN: heap-use-after-free
}

// Test 2: Detect stack-use-after-scope
void test_stack_scope() {
    int* p;
    {
        int local = 10;
        p = &local;
    }
    // local is out of scope — p is dangling
    // ASan catches: *p is stack-use-after-scope
    // int val = *p;  // ASAN: stack-use-after-scope
}

// Test 3: Detect vector iterator invalidation
void test_iterator_invalidation() {
    std::vector<int> v = {1, 2, 3};
    int* ptr = v.data();
    v.resize(10000);  // Reallocates, old buffer freed
    // ASan catches access to old buffer
    // int val = *ptr;  // ASAN: heap-use-after-free
}

// Safe patterns that DON'T trigger ASan:
void test_safe_patterns() {
    // unique_ptr: automatic cleanup
    auto p = std::make_unique<int>(42);
    assert(*p == 42);  // Always safe

    // shared_ptr: reference counted
    std::shared_ptr<int> sp;
    {
        auto local = std::make_shared<int>(10);
        sp = local;
    }
    assert(*sp == 10);  // Safe: shared_ptr extends lifetime

    // Value semantics: no pointers at all
    std::vector<int> orig = {1, 2, 3};
    std::vector<int> copy = orig;  // Deep copy
    orig.clear();
    assert(copy[0] == 1);  // Safe: independent copy

    std::cout << "All safe patterns passed!\n";
}

int main() {
    std::cout << "=== ASan dangling pointer detection ===\n\n";

    // Uncomment ONE of these to see ASan in action:
    // test_use_after_free();           // heap-use-after-free
    // test_stack_scope();              // stack-use-after-scope
    // test_iterator_invalidation();    // heap-use-after-free

    // These are always safe:
    test_safe_patterns();

    std::cout << "\n=== ASan flags summary ===\n";
    std::cout << "Compile:  -fsanitize=address -g\n";
    std::cout << "Runtime:  ASAN_OPTIONS=detect_stack_use_after_return=1\n";
    std::cout << "CI:       add ASan build to your test matrix\n";
    std::cout << "Coverage: use with unit tests for maximum detection\n";

    return 0;
}
```

---

## Notes

- **Dangling pointers are the #1 source of C++ security vulnerabilities** (use-after-free, buffer overflow).
- Ownership semantics (`unique_ptr`, `shared_ptr`) make dangling impossible by design.
- `weak_ptr::lock()` returns `nullptr` if the object was destroyed - safe expiry detection.
- AddressSanitizer has ~2x runtime overhead - suitable for testing, not production.
- Prefer **value semantics** (return by value, move semantics) over pointers when possible - eliminates the entire category of bugs.
