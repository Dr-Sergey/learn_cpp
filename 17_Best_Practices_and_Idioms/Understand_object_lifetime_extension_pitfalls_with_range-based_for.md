# Understand object lifetime extension pitfalls with range-based for

**Category:** Best Practices and Idioms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/range-for>  

---

## Topic Overview

Range-based for loops internally bind the range expression to `auto&& __range`. Temporaries returned by **simple** expressions have their lifetime extended to cover the loop. However, temporaries created in **chained expressions** (like `get_obj().get_container()`) are destroyed before the loop body runs - and that spells dangling reference.

### The Dangling Reference Problem (pre-C++23)

Here is how the compiler desugars a range-based for and where the lifetime trap hides:

```cpp
for (auto& x : get_wrapper().data())       // PRE-C++23: DANGER!
                |              |
                |              +- returns ref to member of temporary
                +- temporary wrapper created HERE
                   destroyed BEFORE loop body!
                   data() reference is DANGLING

C++23 fix (P2718R0): ALL temporaries in the range-init
are lifetime-extended to cover the entire loop.
```

The reason this trips people up is that simple cases - like `for (auto x : get_vec())` - work fine, so it feels safe. The danger only appears when you chain a method call on top of the temporary, returning a reference into it.

### Range-for Internal Expansion

To understand why the lifetime matters, look at what the compiler actually generates. The range-for loop:

```cpp
// for (auto& x : expr) { body; }
// becomes:
{
    auto&& __range = expr;           // lifetime of temporaries in expr?
    auto __begin = std::begin(__range);
    auto __end   = std::end(__range);
    for (; __begin != __end; ++__begin) {
        auto& x = *__begin;
        body;
    }
}
```

The temporary created by `expr` only lives until the end of the full expression that initializes `__range`. Pre-C++23, that means it is gone before `__begin` is even computed.

---

## Self-Assessment

### Q1: Show the dangling reference pitfall with chained calls in range-for

The dangerous pattern is calling a method on a temporary to get a reference, then iterating over that reference. The code below shows the problem and two safe alternatives.

```cpp
#include <iostream>
#include <string>
#include <vector>

class Database {
    std::vector<std::string> rows_ = {"Alice", "Bob", "Charlie"};
public:
    const std::vector<std::string>& rows() const { return rows_; }
};

Database get_database() {
    return Database{};  // returns a temporary
}

int main() {
    // DANGEROUS (pre-C++23):
    // for (const auto& name : get_database().rows()) {
    //     std::cout << name << '\n';  // UB! temporary Database is DEAD
    // }
    // Explanation:
    //   auto&& __range = get_database().rows();
    //   get_database() creates temporary Database
    //   .rows() returns const& to its member
    //   temporary Database is destroyed at end of full-expression
    //   __range is now a dangling reference!

    // FIX 1: Store in a variable
    auto db = get_database();
    for (const auto& name : db.rows()) {
        std::cout << name << '\n';  // OK: db lives through the loop
    }

    // FIX 2: Copy the container
    for (const auto& name : std::vector<std::string>(get_database().rows())) {
        std::cout << name << '\n';  // OK: owns the data
    }
}
// Expected output:
// Alice
// Bob
// Charlie
// Alice
// Bob
// Charlie
```

Fix 1 is usually the right move - just name the temporary before the loop. Fix 2 copies the data so the loop owns it outright.

### Q2: Explain C++23 lifetime extension rules (P2718R0) and remaining edge cases

C++23 (P2718R0) extends the lifetime of **all temporaries** in the range-for init expression to cover the entire loop. That directly patches the pre-C++23 trap.

```cpp
#include <iostream>
#include <string>
#include <vector>

class Wrapper {
    std::vector<int> data_ = {1, 2, 3, 4, 5};
public:
    const std::vector<int>& data() const { return data_; }
};

Wrapper make_wrapper() { return Wrapper{}; }

int main() {
    // C++23: SAFE! Temporary Wrapper lifetime extended
    for (int x : make_wrapper().data()) {
        std::cout << x << ' ';
    }
    std::cout << '\n';

    // Still applies to deeper chains:
    // for (auto& x : a().b().c().d()) { ... }
    // All temporaries (a(), b(), c()) extended in C++23
}
// Expected output:
// 1 2 3 4 5
```

C++23 makes this pattern safe, but two situations are still not covered:

```cpp
// Case 1: User stores reference manually (not in range-for init)
const auto& ref = make_wrapper().data();  // STILL dangling!
// The fix only applies to range-for, not general reference binding

// Case 2: Coroutine suspension
// If a coroutine suspends mid-loop, temporaries may be destroyed
// at suspension point (implementation-defined)
```

If you are on a pre-C++23 compiler, or writing coroutines, the old rule applies: name the temporary first.

### Q3: Show the C++23 fix in action with a practical example

Here is a more realistic scenario: a config loader that returns a temporary object whose data you want to iterate over. This is exactly the kind of code that silently went wrong before C++23.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <sstream>

class Config {
    std::vector<std::string> entries_;
public:
    Config(std::initializer_list<std::string> entries) : entries_(entries) {}
    const std::vector<std::string>& entries() const { return entries_; }
};

class ConfigLoader {
public:
    Config load() const {
        return Config{"host=localhost", "port=8080", "debug=true"};
    }
};

ConfigLoader get_loader() { return ConfigLoader{}; }

int main() {
    // Pre-C++23: THREE temporaries, all dangling
    // ConfigLoader temp1 = get_loader();
    // Config temp2 = temp1.load();
    // const vector<string>& ref = temp2.entries();
    // temp1 and temp2 destroyed -> ref dangles

    // C++23: All temporaries extended!
    // Equivalent safe code for pre-C++23:
    auto config = get_loader().load();
    for (const auto& entry : config.entries()) {
        auto pos = entry.find('=');
        std::cout << "Key: " << entry.substr(0, pos)
                  << ", Value: " << entry.substr(pos + 1) << '\n';
    }
}
// Expected output:
// Key: host, Value: localhost
// Key: port, Value: 8080
// Key: debug, Value: true
```

The pre-C++23 workaround shown here - storing the result of `get_loader().load()` into a named variable before the loop - is the pattern to reach for when you cannot rely on C++23.

---

## Notes

- P2718R0 was adopted for C++23. Check your compiler version: GCC 13+, Clang 16+, MSVC 17.7+.
- The rule of thumb pre-C++23: if the range expression contains a `.` (member access on a temporary), store it first.
- Range-for with simple temporaries (`for (auto x : get_vec())`) was always safe - the vector itself gets lifetime-extended.
- Use `-Wdangling` (GCC/Clang) to catch some of these issues at compile time.
