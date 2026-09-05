# Use AI assistants for learning new C++ features with interactive examples

**Category:** AI Assisted Development  
**Standard:** C++23  

---

## Topic Overview

If you've ever tried to learn a new C++ feature from the standard's wording or from cppreference, you know how long it can take to get from "I've read the definition" to "I actually understand this." AI assistants short-circuit that process dramatically. They can explain concepts progressively (simple -> advanced), generate runnable examples, create exercises with solutions, compare old vs. new approaches, and answer follow-up questions instantly. This creates a personalized, interactive learning experience that's far more effective than reading documentation alone - because you can ask exactly the question that's confusing you rather than hunting for a tutorial that happens to address your specific stumbling point.

### Effective Learning Prompts

The right prompt structure matters a lot here. Here are the patterns that reliably produce useful responses.

| Learning Goal | Prompt Pattern | Example |
| --- | --- | --- |
| Understand a feature | "Explain X with 3 examples, easy to hard" | "Explain C++20 concepts" |
| Compare old vs new | "Show how X was done before and after Y" | "SFINAE vs concepts" |
| Apply to real code | "Refactor this code to use X" | "Use ranges here" |
| Edge cases | "What are the gotchas of X?" | "std::move gotchas" |
| Build intuition | "When should I use X vs Y?" | "coroutines vs threads" |
| Practice | "Give me 5 exercises on X with solutions" | "constexpr exercises" |

---

## Self-Assessment

### Q1: Learn a new feature progressively

**Answer:**

The "progressive steps" prompt pattern is one of the best for learning. By asking the AI to structure the explanation from motivation -> basics -> advanced -> exercises, you get a complete learning arc rather than a reference dump.

```cpp
=== PROMPT ===

"Teach me C++20 concepts in 4 progressive steps:

1. What they replace and why (motivation)
2. Basic syntax with simple examples
3. Composing concepts and requires expressions
4. Advanced: subsumption, constrained auto, concept maps

For each step, give a compilable example and an exercise."


=== AI RESPONSE (Step 1: Motivation) ===

"Concepts replace SFINAE-based constraints. Here's why:"
```

The contrast here is genuinely striking. The SFINAE version achieves the same constraint but produces error messages that are pages long. The concepts version produces an error that tells you exactly which constraint failed and why.

```cpp
// BEFORE (C++17 SFINAE): Cryptic error messages
template<typename T,
    typename = std::enable_if_t<
        std::is_integral_v<T> && sizeof(T) >= 4>>
void process(T value) { /* ... */ }

process(3.14);  // Error: no matching function 'process'
// Error message: pages of template instantiation failures

// AFTER (C++20 concepts): Clear constraint
template<typename T>
concept LargeInteger = std::integral<T> && (sizeof(T) >= 4);

void process(LargeInteger auto value) { /* ... */ }

process(3.14);  // Error: constraints not satisfied:
                // 'double' does not satisfy 'LargeInteger'
                // because: double is not integral

// Exercise: Write a concept 'Printable' that requires
// T to have operator<< with std::ostream.
```

```cpp
=== AI RESPONSE (Step 2: Basic syntax) ===
```

There are three syntactically distinct ways to apply a concept constraint to a template, and all three mean the same thing for single-concept cases. The choice between them is mostly stylistic.

```cpp
// Three ways to constrain templates:

// Way 1: requires clause
template<typename T>
    requires std::integral<T>
T double_it(T x) { return x * 2; }

// Way 2: Constrained template parameter
template<std::integral T>
T triple_it(T x) { return x * 3; }

// Way 3: Abbreviated function template
auto quadruple_it(std::integral auto x) { return x * 4; }

// All three are equivalent for single constraints.
// Use Way 1 for complex constraints, Way 3 for simple ones.

// Custom concept:
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

void compute(Numeric auto x) {
    // Works with int, double, float, etc.
}

// Exercise: Create a concept 'Container' that requires:
// - begin() and end() methods
// - a value_type alias
// - a size() method returning a size_t-like type
```

### Q2: Compare old vs new approaches

**Answer:**

Asking for multiple before/after comparisons in a single prompt is efficient for building a broad sense of how C++20 improves on C++17. Including "when to still use the old way" is important - AI that only shows the new way can give you the impression that everything should always be upgraded, which isn't true.

```cpp
=== PROMPT ===

"Show me 5 common C++ patterns that are better in C++20.
For each: old way, new way, and when to still use the old way."
```

Here are four of the five patterns, with honest notes about when the older approach is still appropriate.

```cpp
// === 1. Iteration: for loop vs ranges ===

// Old: manual iterator loop
std::vector<int> v{3,1,4,1,5};
std::vector<int> result;
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it > 2) result.push_back(*it * *it);
}
std::sort(result.begin(), result.end());

// New: ranges pipeline
auto result = v
    | std::views::filter([](int x) { return x > 2; })
    | std::views::transform([](int x) { return x * x; })
    | std::ranges::to<std::vector>();  // C++23
std::ranges::sort(result);

// Still use old when: custom iterator logic, performance-critical
// inner loops where you need iterator invalidation control.


// === 2. String formatting ===

// Old:
char buf[256];
snprintf(buf, sizeof(buf), "User %s (id=%d)", name, id);  // Unsafe
std::string msg = "User " + name + " (id=" + std::to_string(id) + ")";  // Slow

// New:
auto msg = std::format("User {} (id={})", name, id);  // Safe + fast

// Still use old when: embedded systems without stdlib, hot paths
// where format string parsing overhead matters.


// === 3. Variant visitation ===

// Old:
struct Visitor {
    void operator()(int x) const { /*...*/ }
    void operator()(const std::string& s) const { /*...*/ }
    void operator()(double d) const { /*...*/ }
};
std::visit(Visitor{}, my_variant);

// New: overload pattern (C++20 with CTAD)
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

std::visit(overloaded{
    [](int x) { /*...*/ },
    [](const std::string& s) { /*...*/ },
    [](double d) { /*...*/ }
}, my_variant);


// === 4. Compile-time branching ===

// Old: tag dispatch or SFINAE
template<typename T>
auto serialize(const T& val)
    -> std::enable_if_t<has_to_string_v<T>, std::string>;

// New: constexpr if + concepts
template<typename T>
std::string serialize(const T& val) {
    if constexpr (requires { val.to_string(); })
        return val.to_string();
    else if constexpr (std::formattable<T, char>)
        return std::format("{}", val);
    else
        static_assert(false, "No serialization for T");
}
```

The `serialize` example in pattern 4 is worth studying. The `requires { val.to_string(); }` inline requires expression is doing ad-hoc concept checking without needing a named concept - very useful for one-off constraints.

### Q3: Practice with exercises

**Answer:**

Asking for progressive exercises with solutions is one of the most effective ways to solidify understanding of a hard feature. Coroutines are a good candidate for this because they have a lot of moving parts (`promise_type`, `co_yield`, handles, suspension) that take practice to connect together.

```cpp
=== PROMPT ===

"Give me 3 progressive exercises on C++20 coroutines.
For each: problem statement, hints, and full solution.
Difficulty: beginner, intermediate, advanced."
```

Here's the beginner exercise. The reason coroutines are tricky at first is that the machinery (the `Generator` type with its `promise_type`) is completely separate from the coroutine body (the `fibonacci` function). Once you see how they connect, the rest makes sense.

```cpp
// === Exercise 1 (Beginner): Simple generator ===
// Problem: Write a generator that yields Fibonacci numbers.
// Hint: You need a Generator<T> type with promise_type.

// Solution:
#include <coroutine>

template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;
        Generator get_return_object() {
            return Generator{Handle::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    using Handle = std::coroutine_handle<promise_type>;

    bool next() {
        if (handle_ && !handle_.done()) {
            handle_.resume();
            return !handle_.done();
        }
        return false;
    }
    T value() const { return handle_.promise().current_value; }

    ~Generator() { if (handle_) handle_.destroy(); }
    Generator(Generator&& o) noexcept
        : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    Generator(Handle h) : handle_(h) {}
    Handle handle_;
};

// Fibonacci generator coroutine
Generator<uint64_t> fibonacci() {
    uint64_t a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// Usage:
auto fib = fibonacci();
for (int i = 0; i < 10 && fib.next(); ++i) {
    std::cout << fib.value() << ' ';
}
// Output: 0 1 1 2 3 5 8 13 21 34
```

The `co_yield a` line is the heart of it: every time the caller calls `fib.next()`, the coroutine resumes from just after the last `co_yield`, calculates the next value, and suspends again. The generator never runs past where the caller is ready to consume.

---

## Notes

- **"Explain like I know C++17 but not C++20"** is an effective prompt for learning new features - it tells the AI exactly what background to assume rather than starting from scratch or using vocabulary you don't know yet.
- Ask for **compilable examples** - AI sometimes generates code that doesn't actually compile, especially for newer features with finicky syntax like coroutines and modules. Always verify in a compiler.
- Use AI for **"what's new in C++23"** summaries with code examples for each feature - this is a fast way to survey what's available before diving into the ones that matter for your work.
- **Follow-up questions** are where AI learning really shines: "What happens if I remove the `const`?", "Why is this `noexcept`?", "What's the compile error if I use this with a `float`?" You can iterate much faster than with documentation.
- Ask AI to **generate Compiler Explorer links** for trying code instantly - some AI tools can produce godbolt.org links with the code pre-filled.
- AI is excellent at **explaining error messages** from new features like concepts and coroutines, which produce novel error formats that you won't have seen before.
- For deep learning, ask AI to **explain the standard wording** for a specific feature - having the precise formulation alongside a plain-language explanation is the best of both worlds.
