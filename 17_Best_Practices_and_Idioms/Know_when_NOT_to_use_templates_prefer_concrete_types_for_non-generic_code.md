# Know when NOT to use templates: prefer concrete types for non-generic code

**Category:** Best Practices and Idioms  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rt-concepts>  

---

## Topic Overview

Templates are powerful but come with real costs: longer compile times, harder-to-read error messages, code bloat, and reduced debuggability. The rule of thumb is simple - if a function only ever works with one type, don't template it. Templates exist to raise the level of abstraction, not to make every function look like it came out of a math textbook.

### Template Decision Guide

Use this flow when you're tempted to reach for a template:

```cpp
Does the function need to work with multiple types?
    | No  -> Use concrete types
    | Yes
    v
Will it be called with 2-3 specific types?
    | Yes -> Consider overloads or std::variant
    | No (truly generic)
    v
Can you constrain with concepts?
    | Yes -> Template + concept (GOOD)
    | No  -> Unconstrained template (BAD)
```

---

## Self-Assessment

### Q1: Show how over-templating hides intent and creates poor error messages

Here is a side-by-side comparison. The over-templated version accepts literally anything, which sounds flexible but is actually harmful - the errors surface deep inside the instantiation rather than at the call site.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <concepts>

// OVER-TEMPLATED: this function only ever processes strings
template<typename Container, typename Transformer, typename Output>
void process_data(const Container& c, Transformer t, Output& out) {
    for (const auto& item : c)
        out.push_back(t(item));
}
// Problem 1: What types does this accept? ANY. Including nonsensical ones.
// Problem 2: Error message when misused:
//   error: no match for call to '(main()::<lambda(int)>)(const std::string&)'
//   in instantiation of 'void process_data(const Container&, ...' [long traces]

// CONCRETE: clear intent, good errors
void process_strings(const std::vector<std::string>& input,
                     std::vector<std::string>& output) {
    for (const auto& s : input)
        output.push_back(s + "!");
}
// Error if misused: "cannot convert 'int' to 'const std::string&'"
// One line, immediately obvious.

// CONSTRAINED TEMPLATE (when generic IS needed):
template<std::ranges::input_range R>
    requires std::integral<std::ranges::range_value_t<R>>
int sum(const R& range) {
    int total = 0;
    for (auto v : range) total += v;
    return total;
}
// Error when misused with strings:
// "constraints not satisfied: std::integral<std::string> is false"

int main() {
    std::vector<std::string> in = {"hello", "world"};
    std::vector<std::string> out;
    process_strings(in, out);
    for (const auto& s : out) std::cout << s << ' ';
    std::cout << '\n';

    std::cout << sum(std::vector{1, 2, 3, 4, 5}) << '\n';
}
// Expected output:
// hello! world!
// 15
```

The constrained template gives you a short, readable error message at the call site. The unconstrained one gives you a wall of template backtrace pointing into the middle of the function body.

### Q2: Explain why unconstrained templates ("concept-less templates") are harmful

An unconstrained template is a silent trap. It will accept anything at the call site, and only blow up later - inside the function body - when the compiler tries to instantiate it.

**Problems with unconstrained templates:**

```cpp
// Unconstrained: accepts ANYTHING
template<typename T>
void process(T x) {
    x.foo();      // fails inside the template body
    x + 1;        // fails inside the template body
    std::cout << x; // fails inside the template body
}

process(42);  // error deep in process() body, not at call site
```

What goes wrong:

1. **Errors appear at instantiation**, not at the call site - often 50+ lines of template backtrace
2. **No documentation of requirements** - must read the entire body to understand valid types
3. **IDE autocomplete breaks** - no way to know valid operations on `T`
4. **Overload resolution surprises** - template matches everything, catching unintended types

The reason this trips people up is that templates feel like a natural "make it work with anything" tool, but "anything" is almost never what you actually want. The fix is concepts, which let you say exactly what you mean.

**The fix (C++20 concepts):**

```cpp
template<typename T>
    requires requires(T x) { x.foo(); }
void process(T x) {
    x.foo();
}

process(42);  // clear error: "constraints not satisfied"
```

Now the error fires at the call site, with a message that actually tells you what requirement `42` failed to meet.

### Q3: When `std::function` is cleaner than a template parameter

There is a real tension between template callbacks (zero overhead, one instantiation per type) and `std::function` (type-erased, single instantiation, slightly heavier). Here is when each makes sense.

```cpp
#include <functional>
#include <iostream>
#include <vector>

// TEMPLATE: each callback type creates a new instantiation
template<typename F>
void for_each_template(const std::vector<int>& v, F&& f) {
    for (int x : v) f(x);
}
// Instantiated separately for: lambda, function pointer, functor
// = code bloat in binary

// std::function: ONE instantiation, type-erased
void for_each_erased(const std::vector<int>& v,
                     std::function<void(int)> f) {
    for (int x : v) f(x);
}

// When std::function is better:
class EventSystem {
    // Can't use template here - must store different callback types
    std::vector<std::function<void(int)>> handlers_;  // type-erased
public:
    void on_event(std::function<void(int)> handler) {
        handlers_.push_back(std::move(handler));
    }
    void fire(int event) {
        for (auto& h : handlers_) h(event);
    }
};

int main() {
    EventSystem events;

    // Lambda
    events.on_event([](int e) { std::cout << "Lambda: " << e << '\n'; });

    // Function pointer
    struct Logger {
        static void log(int e) { std::cout << "Logger: " << e << '\n'; }
    };
    events.on_event(Logger::log);

    events.fire(42);
}
// Expected output:
// Lambda: 42
// Logger: 42
```

The event system example is the key case where `std::function` wins outright - you cannot store a heterogeneous collection of typed lambdas without type erasure. If you tried to template the `handlers_` vector, you would need all handlers to have the exact same type.

**When to use each:**

| Scenario | Use | Why |
| --- | --- | --- |
| Hot loop, performance-critical | Template | Zero overhead, inlinable |
| Stored callbacks (event systems) | `std::function` | Must store heterogeneous callables |
| Virtual interface | Neither - use virtual functions | Runtime polymorphism |
| Header-only library API | Template + concept | Maximum flexibility |
| Stable ABI boundary | `std::function` | Templates don't cross ABI |

---

## Notes

- `std::function` has ~40 byte overhead and a virtual call - don't use in tight loops.
- C++23 `std::move_only_function` avoids the copyability requirement.
- Unconstrained templates are informally called "template tarpits" - they suck everything in.
- C++ Core Guideline T.1: "Use templates to raise the level of abstraction." Not to obfuscate.
