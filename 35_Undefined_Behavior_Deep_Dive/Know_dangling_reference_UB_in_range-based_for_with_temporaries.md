# Know dangling reference UB in range-based for with temporaries

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/range-for>  

---

## Topic Overview

### The Hidden Trap

Range-based `for` loops have a subtle lifetime bug that catches even experienced C++ developers. When the range expression creates a **temporary** that is used indirectly, the temporary can be destroyed before the loop body executes. The compiler will not warn you about this in most cases, so it is genuinely dangerous.

Here is what the dangerous pattern looks like:

```cpp
#include <vector>
#include <string>

struct Config {
    std::vector<std::string> get_items() const {
        return {"alpha", "beta", "gamma"};
    }
};

Config make_config() { return Config{}; }

// DANGLING REFERENCE - UB!
for (const auto& item : make_config().get_items()) {
    std::cout << item << "\n";  // Reading destroyed memory!
}
```

### Why This Happens

The reason this trips people up is that range-based `for` is syntactic sugar, and the desugaring reveals the problem. The compiler transforms:

```cpp
for (auto&& elem : range_expr) { body; }
```

into approximately this:

```cpp
{
    auto&& __range = range_expr;   // (1) Bind the range
    auto __begin = begin(__range);  // (2) Get iterators
    auto __end = end(__range);
    for (; __begin != __end; ++__begin) {
        auto&& elem = *__begin;    // (3) Dereference
        body;
    }
}
```

The problem is step (1). The lifetime extension rules say: when you bind a reference to a temporary, the temporary's lifetime is extended to match the reference. But **only for the direct temporary**:

```cpp
auto&& __range = make_config().get_items();
//                ^^^^^^^^^^^^^ ^^^^^^^^^^^
//                temporary #1   temporary #2
//                (Config)       (vector<string>)
```

- `make_config()` creates a temporary `Config` (#1)
- `.get_items()` creates a temporary `vector<string>` (#2)
- `__range` binds to temporary #2, extending its lifetime (OK)
- But temporary #1 (the `Config`) is **not** directly bound - its lifetime is NOT extended
- Temporary #1 is destroyed at the end of the full expression (the semicolon of `auto&& __range = ...;`)

Wait - actually the issue is slightly different. Let me show the real problem:

```cpp
// The REAL issue: intermediate temporary is destroyed
auto&& __range = make_config().get_items();
// make_config() returns a temporary Config
// .get_items() is called on that temporary, returning a vector BY VALUE
// The temporary Config is destroyed at the ;
// BUT the returned vector is a NEW temporary, bound to __range - VALID (OK)

// So this case is actually FINE. Let's see the real trap:

const std::vector<int>& get_data(const Config& c) { return c.data; }

// DANGLING! The returned reference refers to a member of the temporary Config
for (auto&& elem : get_data(make_config())) {
    // make_config() temporary is destroyed, get_data returned a reference
    // into that destroyed temporary
}
```

The **real** trap is when the range expression returns a **reference** to a subobject of a temporary. Here is the canonical example that bites people most often:

```cpp
struct Wrapper {
    std::vector<int> items;
    const std::vector<int>& get_items() const { return items; }  // Returns reference!
};

// UB - get_items() returns reference to member of temporary Wrapper
for (auto x : Wrapper{{1,2,3}}.get_items()) {
    std::cout << x;  // Dangling reference!
}

// SAFE - get_items() returns by value (copy)
struct SafeWrapper {
    std::vector<int> items;
    std::vector<int> get_items() const { return items; }  // Returns copy!
};

for (auto x : SafeWrapper{{1,2,3}}.get_items()) {
    std::cout << x;  // OK - the returned copy is lifetime-extended
}
```

The difference between `Wrapper` and `SafeWrapper` is one word in the return type, but the lifetime consequences are completely different.

### Common Dangerous Patterns

You will encounter variations of this trap in real code. Watch for all of these:

```cpp
// Pattern 1: Chained method returning reference
for (auto& c : get_string().substr(0, 5)) { }
// std::string::substr returns by value in C++17 -> SAFE
// But std::string_view::substr returns string_view -> could dangle!

// Pattern 2: std::string to string_view in range
std::string make_str() { return "hello world"; }
for (char c : std::string_view{make_str()}) {  // DANGLING!
    // make_str() temporary destroyed, string_view points to freed memory
}

// Pattern 3: Returning reference from function
const auto& get_ref(const std::vector<int>& v) { return v; }
for (auto x : get_ref(std::vector<int>{1,2,3})) {  // DANGLING!
    // The temporary vector is destroyed after get_ref returns
}
```

Pattern 2 with `string_view` is especially nasty because `string_view` is designed to be lightweight and non-owning, which makes it easy to forget that it has no lifetime of its own.

### C++23 Improvement

C++23 (P2718) extends the lifetime of temporaries in range-based `for` init expressions:

```cpp
// C++23: temporaries in the range-expression are lifetime-extended
// for the entire loop
for (auto x : make_config().get_items()) {
    // In C++23: make_config() temporary lives for the entire loop (OK)
}
```

This fix addresses many (but not all) dangling reference cases in range-based for.

### Safe Alternatives

Here are the four patterns you should reach for when dealing with temporaries in range-for:

```cpp
// Fix 1: Store the temporary in a named variable
auto config = make_config();
for (const auto& item : config.get_items()) {
    // config lives for the scope - safe
}

// Fix 2: Use init-statement in range-for (C++20)
for (auto config = make_config(); const auto& item : config.get_items()) {
    // config lives for the loop - safe (and scoped!)
}

// Fix 3: Return by value instead of reference
struct BetterWrapper {
    std::vector<int> items;
    std::vector<int> get_items() const { return items; }  // Copy, not reference
};

// Fix 4: Use auto (copy) instead of auto& when the range might dangle
for (auto item : make_config().get_items()) {  // Copies each element
    // Even if the container is destroyed, we have copies
}
```

Fix 2 is the most elegant for C++20 and later: the init-statement scopes the temporary exactly to the loop, making the lifetime relationship explicit in the code.

---

## Self-Assessment

### Q1: Which of these loops has UB

Look at the return type of `get()` versus `copy()` - that is the only thing that differs between the safe and unsafe versions:

```cpp
struct Data {
    std::vector<int> nums;
    const std::vector<int>& get() const { return nums; }
    std::vector<int> copy() const { return nums; }
};

Data make_data() { return Data{{1,2,3}}; }

// Loop A
for (auto x : make_data().copy()) { }

// Loop B
for (auto x : make_data().get()) { }

// Loop C
for (auto data = make_data(); auto x : data.get()) { }
```

**Answer:**

- **Loop A**: Safe. `copy()` returns by value. The returned `vector` is a temporary bound to the range reference - lifetime extended.
- **Loop B**: **UB**. `get()` returns a reference to the `nums` member of the temporary `Data`. The `Data` temporary is destroyed before the loop begins.
- **Loop C**: Safe. The `Data` object is stored in `data`, which lives for the entire loop.

### Q2: How does C++23 fix this

C++23 adopted P2718, which extends the lifetime of **all** temporaries created in the range-expression to cover the entire loop body. In C++23, Loop B above becomes safe because the temporary `Data` object lives until the loop ends.

However, this only applies to the range-expression directly. Other patterns (like creating a `string_view` from a temporary in an initializer) can still dangle.

### Q3: What is the safest general pattern for range-based for with expressions

Use the C++20 init-statement form:

```cpp
for (auto&& container = make_data(); const auto& item : container.get()) {
    // container is alive for the entire loop
}
```

This works in all C++ versions from C++20 onward and doesn't require knowing whether `get()` returns by value or reference.

---

## Notes

- This is one of the most common sources of subtle UB in modern C++ - it compiles without warnings
- Clang-Tidy check `bugprone-dangling-handle` catches some cases
- GCC 13+ warns about some dangling range-for patterns with `-Wdangling-reference`
- The rule of thumb: if you call a method on a temporary in a range-for, make sure the method returns **by value**, not by reference
- Always prefer the init-statement form (`for (auto x = expr; auto& e : x)`) when in doubt
