# Understand Sink Parameters and By-Value vs Const-Reference Trade-offs

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-in-out>  

---

## Topic Overview

### What Is a Sink Parameter

A **sink parameter** is a function parameter where the function **takes ownership** (stores/consumes the value). The question is: what's the best way to accept it?

The answer depends on how cheap the type's move is and how often the function is called. For most everyday types like `std::string` and `std::vector`, taking by value and then moving from the parameter is the sweet spot - simple code with near-optimal performance.

| Approach | Lvalue cost | Rvalue cost | Code complexity |
| --- | :---: | :---: | :---: |
| `f(const T&)` + assign | 1 copy | 1 copy (!) | Simple |
| `f(T)` + move | 1 copy + 1 move | 1 move | Simple |
| `f(const T&)` + `f(T&&)` overloads | 1 copy | 1 move | 2x functions |

### The Recommendation

For types where **move is cheap** (strings, vectors, unique_ptr):

- **By-value + move** is the best default - simple code, near-optimal performance
- Two overloads only needed for hot paths where the extra move matters

---

## Self-Assessment

### Q1: Show that a constructor storing a string should take by value + move

Notice the asymmetry in the cost column: passing an rvalue costs only a move (cheap), and passing an lvalue costs one copy plus one move. That extra move compared to the two-overload approach is essentially free for types like `std::string` - it is just a pointer swap:

```cpp
#include <iostream>
#include <string>
#include <utility>

struct Person {
    std::string name_;
    std::string email_;

    // BY VALUE + MOVE -- the recommended sink idiom
    Person(std::string name, std::string email)
        : name_(std::move(name))    // Move from the parameter
        , email_(std::move(email))
    {
        std::cout << "  Constructed: " << name_ << ", " << email_ << "\n";
    }
};

int main() {
    std::cout << "=== By-Value Sink Constructor ===\n\n";

    // Case 1: Passing rvalues (temporaries) -- cost: 1 move each
    std::cout << "1. Both rvalues (1 move per param):\n";
    Person p1(std::string("Alice"), std::string("alice@example.com"));

    // Case 2: Passing lvalues -- cost: 1 copy + 1 move each
    std::cout << "\n2. Both lvalues (1 copy + 1 move per param):\n";
    std::string name = "Bob";
    std::string email = "bob@example.com";
    Person p2(name, email);
    std::cout << "  Originals preserved: " << name << ", " << email << "\n";

    // Case 3: Mixed -- each param independently optimal
    std::cout << "\n3. Mixed (lvalue name, rvalue email):\n";
    Person p3(name, std::string("bob2@example.com"));

    // Why this works for both cases:
    // - Rvalue arg -> move-constructed into parameter -> moved into member: 2 moves
    // - Lvalue arg -> copy-constructed into parameter -> moved into member: 1 copy + 1 move
    // The "extra" move is nearly free for types like string/vector

    return 0;
}
```

The key insight from the comments: whether you pass an lvalue or an rvalue, the member always ends up initialized via a single move from the parameter. The cost difference between the two cases is paid at the parameter construction step, not inside the constructor body.

### Q2: Compare the two-overload approach vs single by-value sink

Here all three approaches are shown together so you can see the trade-offs at a glance. The by-value approach has one extra move per call; the two-overload approach eliminates it but requires writing twice as much function code:

```cpp
#include <iostream>
#include <string>

// APPROACH 1: By-value sink (simple, slightly suboptimal)
class Widget_ByValue {
    std::string name_;
public:
    void setName(std::string name) {  // One function handles both
        name_ = std::move(name);
    }
    // Lvalue cost: 1 copy (into param) + 1 move (into member) = copy + move
    // Rvalue cost: 1 move (into param) + 1 move (into member) = 2 moves
};

// APPROACH 2: Two overloads (optimal, more code)
class Widget_TwoOverloads {
    std::string name_;
public:
    void setName(const std::string& name) {  // For lvalues
        name_ = name;  // 1 copy directly into member
    }
    void setName(std::string&& name) {       // For rvalues
        name_ = std::move(name);  // 1 move directly into member
    }
    // Lvalue cost: 1 copy (direct) -- no extra move!
    // Rvalue cost: 1 move (direct) -- no extra move!
};

// APPROACH 3: Forwarding reference (optimal, complex)
class Widget_Forward {
    std::string name_;
public:
    template <typename T>
        requires std::constructible_from<std::string, T>
    void setName(T&& name) {
        name_ = std::forward<T>(name);
    }
    // Same as two overloads cost-wise, but single template
};

int main() {
    std::cout << "=== Comparison ===\n\n";

    std::cout << "Cost analysis per call:\n\n";
    std::cout << "                     Lvalue arg        Rvalue arg\n";
    std::cout << "  ──────────────────────────────────────────────\n";
    std::cout << "  By-value sink:     1 copy + 1 move   2 moves\n";
    std::cout << "  Two overloads:     1 copy            1 move\n";
    std::cout << "  Forwarding ref:    1 copy            1 move\n";
    std::cout << "\n";
    std::cout << "  Extra cost of by-value:  1 extra move per call\n";
    std::cout << "  (Moves are O(1) for string/vector -> negligible)\n";
    std::cout << "\n";

    std::cout << "Recommendation:\n";
    std::cout << "  - Default: by-value sink (simple, good enough)\n";
    std::cout << "  - Hot path: two overloads or forwarding reference\n";
    std::cout << "  - Generic library code: forwarding reference\n";
    std::cout << "  - unique_ptr params: rvalue ref (can't copy anyway)\n";

    // Special case: assignment optimization
    // When assigning (not constructing), by-value loses buffer reuse:
    std::cout << "\n=== Assignment optimization loss ===\n";
    std::cout << "  With two overloads (lvalue path):\n";
    std::cout << "    name_ = name;  // Can reuse name_'s existing buffer!\n";
    std::cout << "  With by-value:\n";
    std::cout << "    name_ = std::move(name);  // name was freshly copied\n";
    std::cout << "    // Lost chance to reuse name_'s buffer\n";

    return 0;
}
```

The "assignment optimization loss" section at the bottom is an easy thing to miss: when the target member already has allocated capacity, assigning directly from a `const string&` can reuse that capacity. The by-value path always discards the old buffer and adopts the parameter's buffer instead - a subtle extra allocation for setter-style code called in a loop.

### Q3: Explain when by-value sink is suboptimal: types where copy is expensive but never elided

The by-value idiom shines when move is cheap. It breaks down for two kinds of types: trivially copyable types where move equals copy (so you pay two copies instead of one), and repeated-setter scenarios where buffer reuse would save allocations. This example quantifies both situations:

```cpp
#include <iostream>
#include <string>
#include <array>
#include <chrono>

// CASE 1: Large trivially-copyable types -- move == copy (both expensive)
struct Matrix4x4 {
    std::array<double, 16> data{};
    // sizeof = 128 bytes
    // Move = Copy = memcpy 128 bytes (no pointers to steal)
    // By-value sink: 2 memcpys instead of 1 -> 2x cost!
};

// CASE 2: Types with expensive copy AND expensive move
struct AudioBuffer {
    float* samples;
    size_t size;
    // Move: steal pointer (cheap)
    // Copy: allocate + copy all samples (expensive)
    // By-value rvalue: move + move = 2 cheap ops
    // By-value lvalue: copy + move = 1 expensive + 1 cheap (but const-ref avoids extra move)
};

// CASE 3: Frequently-called setter with buffer reuse opportunity
class Logger {
    std::string buffer_;  // May have pre-allocated capacity
public:
    // SUBOPTIMAL by-value: always allocates new string for lvalue path
    void log_byval(std::string msg) {
        buffer_ = std::move(msg);
        // buffer_ gets msg's newly allocated storage
        // Old buffer_ storage is wasted (freed)
    }

    // OPTIMAL const-ref: can reuse buffer_'s existing allocation!
    void log_optimal(const std::string& msg) {
        buffer_ = msg;
        // If buffer_.capacity() >= msg.size(), no allocation needed!
    }
};

int main() {
    std::cout << "=== When By-Value Sink Is Suboptimal ===\n\n";

    std::cout << "1. Large trivially-copyable types:\n";
    std::cout << "   sizeof(Matrix4x4) = " << sizeof(Matrix4x4) << " bytes\n";
    std::cout << "   Move = Copy = memcpy " << sizeof(Matrix4x4) << " bytes\n";
    std::cout << "   -> By-value adds an EXTRA full copy\n";
    std::cout << "   -> Use const reference instead\n";

    std::cout << "\n2. Repeated assignment with buffer reuse:\n";
    Logger log;
    std::string msg = "Some log message that's moderately long";
    // First call: allocates in both cases
    log.log_optimal(msg);
    // Subsequent calls: log_optimal reuses buffer; log_byval always reallocates

    std::cout << "   const-ref setter: reuses existing buffer allocation\n";
    std::cout << "   by-value setter:  always discards old buffer, uses copy's buffer\n";

    std::cout << "\n=== When to AVOID by-value sink ===\n";
    std::cout << "  1. Large trivially-copyable types (move == copy)\n";
    std::cout << "  2. Hot-loop setters where buffer reuse matters\n";
    std::cout << "  3. Performance-critical tight loops (extra move adds up)\n";
    std::cout << "  4. Types with no-op move (e.g., std::array<T, N>)\n";

    std::cout << "\n=== When by-value sink IS fine ===\n";
    std::cout << "  1. Constructors (no existing buffer to reuse)\n";
    std::cout << "  2. Types where move is O(1): string, vector, unique_ptr\n";
    std::cout << "  3. Functions called infrequently\n";
    std::cout << "  4. When code simplicity > micro-optimization\n";

    return 0;
}
```

The `std::array<T, N>` note in the output is worth remembering: unlike `std::vector`, `std::array` has no heap buffer to steal. Its move constructor copies element-by-element just like the copy constructor. Any type built on `std::array` will have the same property.

---

## Notes

- **Default to by-value sink** for constructors + types with cheap moves (string, vector, unique_ptr).
- Two overloads (`const T&` + `T&&`) save one move per call - only worth it on hot paths.
- By-value sink **loses buffer reuse** in repeated assignments - prefer `const T&` for setters called in loops.
- For move-only types (`unique_ptr`), always take by rvalue reference (`T&&`), never by value.
- The extra move is typically ~5 ns on modern hardware - profile before optimizing.
