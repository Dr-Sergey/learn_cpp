# Use AI to analyze and fix compiler errors and warnings

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

C++ compiler errors have a well-earned reputation for being cryptic, and template errors in particular can produce hundreds of lines of output for a one-character mistake. The reason is that the compiler reports the problem from deep inside the instantiation chain - not at the call site where you actually made the mistake. AI assistants can decode long error messages, explain what the compiler is complaining about, identify the root cause (often far from where the error reports), and suggest the correct fix. This is one of the highest-value uses of AI for C++ development, because the alternative is staring at a wall of template instantiation notes and working backwards manually.

### Compiler Error Categories and AI Effectiveness

The "Difficulty" column reflects how hard these errors are to diagnose by eye. "AI Effectiveness" reflects how reliably AI can pinpoint the root cause and suggest a fix. The key insight is that very hard errors (template instantiation, SFINAE failures) are also the ones where AI adds the most value - because those are exactly the ones where a human would spend the most time.

| Error Category | Difficulty | AI Effectiveness | Example |
| --- | --- | --- | --- |
| Template instantiation errors | Very hard | Excellent | 200-line error for wrong template arg |
| Concept constraint failures | Hard | Good | `constraints not satisfied` |
| Overload resolution | Hard | Good | `no matching function for call to...` |
| SFINAE failures | Very hard | Good | Substitution failure, wrong overload selected |
| Linker errors | Medium | Good | Undefined reference, ODR violations |
| Warning interpretation | Easy | Excellent | `-Wconversion`, `-Wshadow` |
| MSVC vs GCC differences | Medium | Good | Conformance differences |

---

## Self-Assessment

### Q1: Decode template error messages with AI

**Answer:**

When pasting a compiler error into an AI prompt, always include the full error text - including all the "note:" lines. Those notes are where the root cause usually appears, and without them the AI is working with incomplete information.

```cpp
=== PROMPT: Template error ===

"Explain this GCC error and show how to fix it.
I'm trying to use std::sort with a custom comparator:

/usr/include/c++/13/bits/stl_algo.h:1950:22: error:
no match for call to
'(ProcessOrder) (Order&, Order&)'
 1950 |    if (__comp(*__first, *__next))
      |        ~~~~~~^~~~~~~~~~~~~~~~~~~
note: candidate: 'bool ProcessOrder::operator()
       (const Order&, const Order&)'
  declared as 'const'

My code:
struct ProcessOrder {
    bool operator()(const Order& a, const Order& b) const {
        return a.priority < b.priority;
    }
};

std::sort(orders.begin(), orders.end(), ProcessOrder{});"

=== AI ANALYSIS ===

"The error says the comparator's operator() is declared
const, but std::sort is calling it on a non-const copy.

Actually wait - looking more carefully, this error occurs
when the comparator is named the same as a function that's
being called. Let me re-read...

The real issue: 'ProcessOrder' might conflict with another
overload. But if the code you showed is exactly as written,
it should work. Make sure:

1. Order& (not const) in the sort's iterator dereference
2. The comparator accepts const Order& (which it does)
3. No other 'ProcessOrder' in scope

Most likely fix: ensure headers are included correctly
and there's no macro interfering."
```

Note that the AI's response here shows the limits of the tool - even when it doesn't immediately find the root cause, it gives you a useful checklist of what to verify. That's often enough to unblock you.

Here are three common template error patterns and their AI-assisted fixes. These are worth recognizing because you'll see them repeatedly.

```cpp
// ERROR: "incomplete type" in templates
// PROMPT: "Why does this fail?"
template<typename T>
class Container {
    T value_;  // Error if T is incomplete at this point
};

class Node {
    Container<Node> children_;  // ERROR: Node is incomplete here
};

// AI FIX: Use pointer or vector (they work with incomplete types)
class Node {
    std::vector<Node> children_;  // OK: vector works with incomplete T
};


// ERROR: "cannot deduce template argument"
// PROMPT: "Template deduction fails here, why?"
template<typename T>
T max_value(std::vector<T>& v) { return *std::max_element(v.begin(), v.end()); }

const auto& data = get_data();  // Returns const vector<int>&
auto m = max_value(data);  // ERROR: can't bind const ref to non-const param

// AI FIX: Accept const reference
template<typename T>
T max_value(const std::vector<T>& v) {
    return *std::max_element(v.begin(), v.end());
}


// ERROR: Concept constraint not satisfied
// PROMPT: "What does this concept error mean?"
// error: constraints not satisfied for 'void process(T)'
//   [with T = MyStruct]
// note: the expression 'std::is_trivially_copyable_v<T>'
//   evaluated to 'false'

// AI: MyStruct is not trivially copyable because it has
// a user-defined copy constructor or destructor.
// Fix: make MyStruct trivially copyable or relax the concept.
```

The const reference deduction error is a particularly common one - it trips people up because the original function looks like it should work for any `vector<T>`. The fix is just adding `const`, but without knowing the rule "template argument deduction doesn't allow implicit const conversions," you might spend a while guessing.

### Q2: Decode linker errors with AI

**Answer:**

Linker errors are a different beast from compiler errors - they happen after compilation succeeds, and the error message refers to symbols rather than source code. The key to diagnosing them is understanding ODR (One Definition Rule) and how C++ splits code across translation units.

```cpp
=== PROMPT: Multiple definition linker error ===

"Explain this linker error and show the fix:

/usr/bin/ld: build/handler.o: in function `Config::instance()':
handler.cpp:(.text+0x1a): multiple definition of
  `Config::instance()'; build/main.o:main.cpp:(.text+0x1a):
  first defined here
collect2: error: ld returned 1 exit status

Config::instance() is defined in config.hpp which is
included by both main.cpp and handler.cpp."

=== AI ANALYSIS ===

"ODR (One Definition Rule) violation: Config::instance()
is defined in a header that's included in multiple
translation units, creating duplicate definitions.

The function is defined in the header but is not:

- inline
- a template
- constexpr
- inside a class body

Fixes (choose one):

1. Mark as inline:

   inline Config& Config::instance() { ... }

2. Move definition to config.cpp

3. Define inside the class body (implicitly inline):

   class Config {
       static Config& instance() {
           static Config inst;
           return inst;
       }
   };"
```

The AI's list of "what makes a definition OK in a header" is worth memorizing: `inline`, template, `constexpr`, and inside-class-body are all valid; anything else needs to live in exactly one `.cpp` file.

Here are several more linker error patterns that AI handles well, each with its root cause explained.

```cpp
// === Common linker errors AI can diagnose ===

// 1. Undefined reference to vtable
// Cause: virtual function declared but not defined
class Base {
    virtual void process();  // Declared but never defined
    // Fix: define it, or make it pure virtual: = 0;
};

// 2. Undefined reference to static member
class Counter {
    static int count_;  // Declaration only
};
// Fix: add definition in .cpp:
int Counter::count_ = 0;
// Or in C++17: inline static int count_ = 0;

// 3. ABI mismatch between libraries
// undefined reference to 'std::__cxx11::basic_string<...>'
// Cause: mixing C++11 ABI and pre-C++11 ABI libraries
// Fix: rebuild all libraries with same ABI setting:
//   -D_GLIBCXX_USE_CXX11_ABI=1
```

The vtable error is a particularly confusing one - GCC reports it as "undefined reference to vtable for Foo" rather than "you forgot to define Base::process()", which would be much more helpful. Knowing the pattern lets you go straight to the fix.

### Q3: Systematic approach to using AI for warnings

**Answer:**

Compiler warnings are compiler errors waiting to happen. The important thing AI teaches here is the difference between "suppress the warning" (cast the value, add `[[maybe_unused]]`) and "fix the root cause" (use the right type, rename the variable). AI is good at distinguishing these.

```cpp
=== PROMPT TEMPLATE FOR WARNINGS ===

"We compile with -Wall -Wextra -Wpedantic -Wconversion.
Explain each warning below and show the safest fix
that preserves the intended behavior:

1. warning: implicit conversion loses integer precision:
   'size_t' to 'int' [-Wshorten-64-to-32]
   int count = vec.size();

2. warning: declaration shadows a local variable
   [-Wshadow]

3. warning: unused parameter 'config' [-Wunused-parameter]

4. warning: comparison of integer expressions of different
   signedness [-Wsign-compare]
   for (int i = 0; i < vec.size(); ++i)"


=== AI FIXES ===
```

Here are the fixes, with comments explaining why the naive suppression approach is often wrong.

```cpp
// Fix 1: Use correct type (not just cast!)
// BAD: int count = static_cast<int>(vec.size());  // Hides the real issue
// GOOD:
size_t count = vec.size();  // Use the correct type
// Or if int is truly needed:
if (vec.size() > static_cast<size_t>(std::numeric_limits<int>::max()))
    throw std::overflow_error("Too many elements");
int count = static_cast<int>(vec.size());

// Fix 2: Rename the shadowing variable
void process(int value) {
    if (condition) {
        int result = compute(value);  // Was: int value = compute(value);
    }
}

// Fix 3: Several options for unused parameters
void handler(Request& req, [[maybe_unused]] const Config& config) {
    // config will be used in future implementation
}
// Or remove the name:
void handler(Request& req, const Config& /*config*/) { }

// Fix 4: Use correct loop variable type
for (size_t i = 0; i < vec.size(); ++i)  // size_t, not int
// Or better, use range-for:
for (const auto& elem : vec)
// Or ranges:
for (auto [i, elem] : vec | std::views::enumerate) // C++23
```

Fix 1's "BAD" comment deserves extra attention. The `static_cast<int>(vec.size())` pattern doesn't fix anything - it just silences the warning while leaving the potential overflow in place. If `vec.size()` can be larger than `INT_MAX`, you now have silent undefined behavior instead of a loud warning. The fix is to use the right type.

---

## Notes

- **Always include the full error message** in the prompt, including all "note:" lines - those notes contain the root cause, and without them AI is working with incomplete information.
- For template errors, include the **template instantiation chain** (the "required from here" notes) - they tell AI which call site triggered the chain.
- AI is especially good at **MSVC vs GCC** differences - ask "why does this compile on GCC but not MSVC?" and you'll often get a clear explanation of the conformance issue.
- Linker errors often need **build system context** - include the CMakeLists.txt excerpt when the error is about library linking or undefined symbols.
- Don't suppress warnings with casts - ask AI for the **semantic fix** (correct types, proper logic) rather than the syntactic suppression. A suppressed warning is a bug with a silencer.
- Concept errors in C++20 are much clearer than SFINAE errors, but AI handles both well - the SFINAE case benefits more from AI assistance because the error messages are less human-readable.
- Keep a personal **error pattern library** of common C++ errors and their fixes - after you've seen an ODR violation or a missing vtable definition once, you can diagnose it instantly next time.
