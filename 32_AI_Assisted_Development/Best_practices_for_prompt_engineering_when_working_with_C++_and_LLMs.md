# Best practices for prompt engineering when working with C++ and LLMs

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

Getting useful C++ code from an LLM is not just about describing what you want - it requires feeding the model the context it needs to produce correct, safe, and idiomatic output. C++ prompts need more precision than prompts for other languages because so much correctness depends on details that are easy to omit: which standard version you are targeting, whether exceptions are enabled, what your memory model expectations are, and what platform you are building for.

The core discipline is providing **context** (C++ standard, compiler, platform), **constraints** (no exceptions, embedded target, no Boost), and **examples** (expected input/output, interface signatures you want the code to fit). Without those anchors, the model will fill in defaults that may not match your environment at all.

The table below illustrates how dramatically prompt quality affects the result. This is not a minor tuning effect - the difference between "Poor" and "Excellent" is often the difference between code that needs to be thrown away and code that is production-ready.

### Prompt Quality Spectrum

| Quality | Prompt Example | Likely Result |
| --- | --- | --- |
| **Poor** | "Write a thread pool" | Generic, may have UB |
| **Medium** | "Write a C++17 thread pool with task queuing" | Decent but may miss edge cases |
| **Good** | "C++20 jthread pool, 8 workers, std::function tasks, graceful shutdown, exception-safe, no boost" | Production-quality |
| **Excellent** | Above + "Show RAII cleanup, handle task exceptions, include unit test with 3 scenarios" | Complete, testable |

---

## Self-Assessment

### Q1: Structure effective C++ prompts with context and constraints

**Answer:**

A well-structured C++ prompt has six parts: a context block, a task description, constraints, an interface spec, quality requirements, and a verification request. The parts can be short - the context block is often just two or three lines - but omitting any of them usually degrades the output. Here is a full template:

```cpp
=== PROMPT TEMPLATE FOR C++ CODE GENERATION ===

1. CONTEXT BLOCK (always include):

   "Using C++20 (GCC 13 / Clang 17), targeting Linux x86-64.
    Project uses CMake, GoogleTest, no Boost.
    Coding style: snake_case, .hpp/.cpp split."

2. TASK DESCRIPTION:

   "Implement a lock-free single-producer single-consumer
    ring buffer for inter-thread communication."

3. CONSTRAINTS:

   "- No dynamic allocation after construction
    - Must be header-only
    - Use std::atomic with acquire/release memory ordering
    - Power-of-two capacity only (for fast modulo)
    - No exceptions"

4. INTERFACE SPEC:

   "template<typename T, size_t Capacity>
    class SPSCQueue {
        bool try_push(const T& item);
        bool try_push(T&& item);
        std::optional<T> try_pop();
        size_t size() const;
        bool empty() const;
    };"

5. QUALITY REQUIREMENTS:

   "- Include static_assert for power-of-two check
    - Add [[nodiscard]] where appropriate
    - Show cache-line padding to prevent false sharing
    - Include a usage example with producer/consumer threads"

6. VERIFICATION:

   "Write a GoogleTest fixture that tests:
    - Single-threaded push/pop correctness
    - Full buffer returns false on push
    - Empty buffer returns nullopt on pop
    - Concurrent producer/consumer with 1M items"
```

Beyond the full template, a few prompt patterns are useful to keep in your back pocket for recurring C++ tasks:

```cpp
=== PROMPT PATTERNS FOR COMMON C++ TASKS ===

PATTERN: "Explain then implement"
"First explain the trade-offs between std::variant visitor
and dynamic polymorphism for a state machine with 8 states.
Then implement the variant approach in C++17."

PATTERN: "Fix with explanation"
"This code has undefined behavior. Find it, explain why,
and fix it. Show the fix with a comment explaining the UB:
[paste code]"

PATTERN: "Incremental refinement"
"I have this basic implementation:
[paste code]
Refactor to:
1. Replace raw pointers with unique_ptr
2. Add move semantics
3. Make it exception-safe (strong guarantee)
Show each step separately."

PATTERN: "Constrain the output"
"Generate ONLY the .hpp header. Do not generate main().
Do not use std::endl (use '\n'). Do not use 'using namespace std'."
```

The "constrain the output" pattern is often overlooked but very useful. LLMs will add `main()`, `using namespace std`, and `std::endl` by default because those patterns are everywhere in textbook code. Being explicit that you do not want them saves you from editing them out.

### Q2: C++-specific prompt techniques for complex topics

**Answer:**

Complex C++ topics need more structured prompts because the model needs more guidance about what aspect of the problem matters most. Here are prompts tuned for four particularly tricky areas:

```cpp
=== TEMPLATE METAPROGRAMMING PROMPTS ===

BAD: "Write a compile-time sort"

GOOD: "Using C++20 concepts and consteval, implement a
compile-time sorting algorithm for std::array<int, N>.
Use insertion sort (simple, adequate for small N).
The result should be usable in static_assert:
  static_assert(ct_sort(std::array{3,1,2}) == std::array{1,2,3});
Show the assembly output comment to prove it's compile-time."


=== UNDEFINED BEHAVIOR ANALYSIS PROMPT ===

"Analyze this code for ALL categories of undefined behavior:

- Lifetime issues (dangling references, use-after-free)
- Data races (shared mutable state without synchronization)
- Signed integer overflow
- Strict aliasing violations
- Null pointer dereference
- Out-of-bounds access

For each issue found, cite the standard clause and show the fix.
[paste code]"


=== MEMORY MODEL PROMPTS ===

"Explain the memory ordering in this lock-free code.
For each atomic operation, explain:

1. Why this specific memory_order was chosen
2. What happens-before relationship it establishes
3. What could go wrong with a weaker ordering
4. Would this be correct on ARM vs x86?

[paste code]"


=== PERFORMANCE OPTIMIZATION PROMPT ===

"Profile analysis shows this function takes 40% of CPU time.
Target: process 1M items in <10ms on x86-64 (AVX2 available).
Current: 45ms for 1M items.

Optimize considering:
1. Cache-friendly data layout (current struct is 96 bytes)
2. SIMD opportunities (independent per-element ops)
3. Branch prediction (90% take the if-branch)
4. Compiler auto-vectorization hints

Show before/after with expected speedup rationale.
[paste hot function + data structures]"
```

The memory model prompt is worth highlighting because this is one of the areas where LLMs are weakest. Asking it to reason about `happens-before` relationships and ARM vs x86 behavior forces it to be explicit about assumptions that are often left implicit in generated code. You may still get wrong answers on subtle memory ordering questions, but asking for the reasoning at least makes the errors visible.

### Q3: Anti-patterns and verification strategies

**Answer:**

Knowing what not to do in a prompt is just as important as knowing what to include. These anti-patterns consistently produce poor C++ output:

```cpp
=== PROMPT ANTI-PATTERNS TO AVOID ===

1. VAGUE REQUESTS:

   BAD: "Make this faster"
   GOOD: "Reduce latency of process() from 5ms to <1ms.
          Profile shows 80% in the inner loop. Target: x86-64
          with AVX2. Current data is AoS, consider SoA."

2. MISSING STANDARD VERSION:

   BAD: "Use modern C++ features"
   GOOD: "Use C++20 features: concepts, ranges, std::format,
          designated initializers. Cannot use C++23."

3. NO ERROR HANDLING SPEC:

   BAD: "Handle errors properly"
   GOOD: "Use std::expected<T, Error> for recoverable errors.
          Use assertions for programming errors. No exceptions
          (embedded target, -fno-exceptions)."

4. COPY-PASTE WITHOUT CONTEXT:

   BAD: [paste 500 lines with no explanation]
   GOOD: "Here's a Connection class (key parts shown).
          The issue is in the send() method.
          Full class context: [focused excerpt]"


=== VERIFICATION CHECKLIST FOR AI-GENERATED C++ CODE ===

[ ] Compiles with -Wall -Wextra -Werror -Wpedantic
[ ] No undefined behavior (run with ASan, UBSan, TSan)
[ ] No memory leaks (run with Valgrind or ASan)
[ ] Exception safety guarantees stated and correct
[ ] Move semantics correct (no use-after-move)
[ ] Thread safety documented and correct
[ ] No raw new/delete (uses smart pointers)
[ ] RAII for all resources
[ ] const-correct (const references, const methods)
[ ] No implicit conversions (explicit constructors)
[ ] Tests provided and they pass
[ ] No 'using namespace std' in headers
[ ] Correct include guards or #pragma once
```

The verification checklist is meant to be applied to every piece of AI-generated code before you merge it. The sanitizer step (`-fsanitize=address,undefined,thread`) is especially important - AI output looks correct at a glance more often than it actually is correct under the sanitizers.

---

## Notes

- Always specify the **C++ standard version** - C++17 vs C++20 changes everything, including which features exist and which idioms are idiomatic.
- Include **compiler and platform** for platform-specific code; MSVC, GCC, and Clang all have different extension support.
- For concurrent code, specify **memory model** expectations explicitly - relaxed? acquire/release? sequential consistency?
- Ask the LLM to **explain trade-offs** before implementing - this surfaces design issues before you are committed to a direction.
- **Iterative refinement** works better than one giant prompt: generate -> review -> refine on the specific thing that is wrong.
- Always verify AI output with sanitizers (`-fsanitize=address,undefined,thread`).
- LLMs are weakest on: template metaprogramming edge cases, memory model subtleties, and ABI compatibility questions.
