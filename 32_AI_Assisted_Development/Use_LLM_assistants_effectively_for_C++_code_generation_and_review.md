# Use LLM assistants effectively for C++ code generation and review

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

LLM assistants can dramatically accelerate C++ development when used strategically. They excel at **boilerplate generation**, **code review**, **explaining complex code**, and **exploring design alternatives**. They struggle with **subtle undefined behavior**, **complex template errors**, **memory model correctness**, and **ABI compatibility**. Knowing when to trust and when to verify is the key skill - and the default posture should always be "verify, then trust" rather than the other way around.

### LLM Strengths and Weaknesses for C++

The trust levels below are about how much scrutiny to apply, not whether to use the output at all. Even "High" trust means you still read the code before you ship it.

| Task | LLM Effectiveness | Trust Level |
| --- | --- | --- |
| Boilerplate (RAII, operators) | Excellent | High (still verify) |
| Code explanation | Excellent | High |
| Algorithm implementation | Good | Medium (test thoroughly) |
| Design pattern application | Good | Medium |
| Concurrency code | Weak | Low (verify with TSan) |
| Template metaprogramming | Weak | Low (compile and test) |
| UB detection | Medium | Low (use sanitizers) |
| Platform-specific (SIMD, syscalls) | Medium | Low (test on target) |

---

## Self-Assessment

### Q1: Effective workflows for C++ code generation

**Answer:**

The most important thing to understand about LLM code generation is that the workflow matters as much as the prompt. A single "write me X" prompt gives you one shot at a correct result. A structured generate-review-refine loop catches problems before they compound. The three workflows below are the ones that produce reliable results in practice.

```cpp
=== WORKFLOW 1: GENERATE + REVIEW + REFINE ===

Step 1 - Generate with constraints:
  "Implement a connection pool for PostgreSQL using libpq.
   C++20, max 16 connections, thread-safe, RAII.
   Return std::expected<Connection, Error>."

Step 2 - Self-review prompt:
  "Review the code you just generated. Check for:

   - Thread safety: are all shared resources protected?
   - Exception safety: what happens if libpq throws?
   - Resource leaks: is every PGconn* freed on all paths?
   - Performance: is there unnecessary lock contention?"

Step 3 - Address findings:
  "Fix issue #2: the PGconn* leaks if connection
   validation fails after acquisition. Use a scope guard."


=== WORKFLOW 2: EXPLAIN BEFORE GENERATE ===

  "I need to choose between:
   a) std::shared_mutex for read-heavy concurrent map
   b) Lock-free robin hood hash map
   c) Per-shard std::unordered_map with striped locks

   My workload: 95% reads, 5% writes, 4 threads, 100K entries.
   Which approach and why? Then implement your recommendation."


=== WORKFLOW 3: TEST-FIRST ===

  "Write GoogleTest test cases for a CircularBuffer<T, N>:

   - push/pop FIFO ordering
   - full buffer overwrites oldest
   - empty buffer pop returns nullopt
   - capacity() returns N
   - works with move-only types

   Then implement CircularBuffer to pass these tests."
```

Workflow 2 is underused. Asking LLMs to reason about design tradeoffs before writing code produces better implementations because the model is thinking about the problem constraints, not just the first pattern that matches "concurrent map." You also end up with the reasoning documented in the conversation, which helps with code review later.

The following code review example shows how to use an LLM to spot real bugs in existing code. The `SessionManager` below has subtle issues that are worth walking through explicitly.

```cpp
// === Code review prompt that catches real bugs ===
// Given this code, ask the LLM:

class SessionManager {
    std::unordered_map<int, std::shared_ptr<Session>> sessions_;
    std::mutex mtx_;
public:
    void add(int id, std::shared_ptr<Session> s) {
        std::lock_guard lock(mtx_);
        sessions_[id] = std::move(s);
    }

    std::shared_ptr<Session> get(int id) {
        std::lock_guard lock(mtx_);
        auto it = sessions_.find(id);
        return it != sessions_.end() ? it->second : nullptr;
    }

    void remove_expired() {
        std::lock_guard lock(mtx_);
        for (auto it = sessions_.begin(); it != sessions_.end(); ) {
            if (it->second->expired())
                it = sessions_.erase(it);
            else
                ++it;
        }
    }
};

// Prompt: "Review for thread safety, performance, and correctness.
//  Specifically: is the lock granularity correct? Any performance
//  issues with the mutex? What about the shared_ptr ref counting?"

// Good LLM response would identify:
// 1. remove_expired() holds lock while calling expired() - if
//    expired() is slow, all other threads are blocked
// 2. get() returns shared_ptr while lock is held - OK, but
//    the Session could expire right after return
// 3. Consider shared_mutex for read-heavy workload
// 4. remove_expired() could be called from a background thread
//    on a snapshot to reduce lock contention
```

A good LLM will identify all four issues above. Issue 1 is the most important: `remove_expired()` holds the mutex while potentially doing slow work inside `expired()`. If `expired()` involves any I/O or blocking, every other thread that needs `mtx_` is stuck. The fix is to collect keys to remove while holding the lock, then do the expensive check without the lock, or to work on a snapshot.

### Q2: Code review with LLMs - what to check

**Answer:**

Unstructured review prompts get unstructured results. The prompt template below forces the LLM to check specific categories, rate severity, and show fixes - which gives you something actionable rather than a list of general observations.

```text
=== STRUCTURED CODE REVIEW PROMPT ===

"Review this C++ code for the following categories.
For each issue, rate severity (Critical/Warning/Info)
and show the fix:

1. CORRECTNESS
   - Undefined behavior (lifetime, aliasing, overflow)
   - Logic errors
   - Off-by-one errors
   - Integer truncation/promotion issues

2. SAFETY
   - Resource leaks
   - Exception safety guarantee (basic/strong/nothrow)
   - Null pointer dereference paths
   - Buffer overflows

3. CONCURRENCY (if applicable)
   - Data races
   - Deadlock potential
   - Memory ordering correctness
   - TOCTOU (time-of-check-time-of-use)

4. PERFORMANCE
   - Unnecessary copies
   - Missing move semantics
   - Suboptimal container choice
   - Cache-unfriendly access patterns

5. STYLE / MAINTAINABILITY
   - Missing const
   - Missing [[nodiscard]]
   - Unclear ownership semantics
   - Magic numbers

[paste code here]"


=== DIFFERENTIAL REVIEW (code change) ===

"Here's a diff for a pull request. Review the CHANGES ONLY
(not the entire file). Focus on:

- Does the change introduce bugs?
- Are edge cases handled?
- Is the change consistent with surrounding code?
- Any ABI break if this is a public library?

[paste diff]"
```

The differential review prompt is worth highlighting separately. When reviewing a pull request, you want the LLM focused on what changed, not re-reviewing stable code. Pasting the diff rather than the full file also keeps the context window focused on what matters.

### Q3: Knowing when NOT to trust LLM output

**Answer:**

There are five areas where LLM output is consistently unreliable for C++. The reason is that these areas require reasoning about subtle invariants and platform-specific behavior that LLMs have seen inconsistently in training data. The rule is simple: generate freely, but always run the verification step before trusting the output in these categories.

```cpp
=== HIGH-RISK AREAS: ALWAYS VERIFY MANUALLY ===

1. LOCK-FREE / ATOMIC CODE:

   LLMs frequently get memory ordering wrong.
   -> Always use ThreadSanitizer (TSan)
   -> Run stress tests (millions of iterations)
   -> Verify on ARM (weaker memory model than x86)

2. TEMPLATE METAPROGRAMMING:

   Complex SFINAE/concept interactions often wrong.
   -> Must compile-test with multiple compilers
   -> Verify error messages for invalid types

3. UNDEFINED BEHAVIOR:

   LLMs may generate code that works on one platform
   but is technically UB (strict aliasing, signed overflow).
   -> Always compile with -fsanitize=undefined
   -> Test with optimization levels -O0 and -O3

4. PLATFORM-SPECIFIC CODE:

   System calls, SIMD intrinsics, compiler extensions.
   -> Test on the actual target platform
   -> Cross-reference with man pages / Intel intrinsics guide

5. SECURITY-SENSITIVE CODE:

   Crypto, input validation, privilege handling.
   -> Never trust AI-generated crypto
   -> Use established libraries (OpenSSL, libsodium)
   -> Fuzz-test all input parsing

=== VERIFICATION PIPELINE ===

Generate -> Compile (-Wall -Wextra -Werror)
         -> Static analysis (clang-tidy, cppcheck)
         -> Sanitizers (ASan + UBSan + TSan)
         -> Unit tests (generated or manual)
         -> Code review (human for high-risk areas)
```

The memory ordering point deserves a special note. x86 has a relatively strong memory model where many `memory_order_relaxed` bugs go undetected in testing because the hardware enforces stronger ordering than the C++ standard requires. When you run the same code on ARM, the weaker hardware memory model exposes the latent bug. This is one reason "tested on my machine" is not sufficient for lock-free code.

---

## Notes

- Use LLMs for the first draft, not the final version - always review and test before committing.
- "Explain-then-implement" prompts produce better results than direct "write X" prompts - the explanation phase forces the model to reason about your specific constraints.
- For complex code, generate tests first then implementation - the tests constrain the LLM to a specific contract and make it much harder to generate plausible-but-wrong code.
- LLMs are excellent at code explanation - use them to understand unfamiliar codebases quickly.
- Always verify with sanitizers (`-fsanitize=address,undefined,thread`) - they catch entire categories of AI-generated bugs that unit tests miss.
- Few-shot examples from your codebase teach style better than verbal descriptions - paste two examples of how your code does something, then ask for a third.
- The best use of LLMs is pair programming: you design, LLM implements, you verify.
