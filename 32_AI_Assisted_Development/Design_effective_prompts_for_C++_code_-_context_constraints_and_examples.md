# Design effective prompts for C++ code - context, constraints, and examples

**Category:** AI Assisted Development  
**Standard:** C++20  

---

## Topic Overview

Effective C++ prompts follow a **CCoE** structure: **Context** (standard, compiler, project constraints), **Constraints** (what to avoid, resource limits), and **Examples** (expected input/output, interface signatures). This structured approach produces dramatically better results than freeform requests, and the reason is straightforward: C++ correctness depends on details that vary enormously between environments. Code for an embedded system with no exceptions is completely different from code for a Linux server with full C++20 support. The model cannot guess which world you live in, so you have to tell it.

The table below shows what happens when you leave each component out. Each omission has a predictable failure mode.

### Prompt Component Impact

| Component | Without It | With It |
| --- | --- | --- |
| **C++ version** | May use deprecated features | Targets exact standard |
| **Compiler** | Generic code | Compiler-specific attrs/intrinsics |
| **Error handling** | Random mix of exceptions/codes | Consistent strategy |
| **Interface spec** | AI invents its own API | Matches existing codebase |
| **Examples** | May misunderstand intent | Nails the format/behavior |
| **Anti-requirements** | Includes unnecessary deps | Clean, focused output |

---

## Self-Assessment

### Q1: Build prompts for different C++ development scenarios

**Answer:**

Different development scenarios call for different prompt shapes. A new class implementation needs a full context block and interface spec. A performance-critical function needs explicit latency targets. A template library needs cross-compiler compatibility requirements. Here are three concrete scenarios with their full prompt structures:

```cpp
=== SCENARIO: New class implementation ===

CONTEXT:
  "C++20, GCC 13, Linux, project uses:
   - std::expected for error handling (no exceptions)
   - RAII for all resources
   - GoogleTest for testing
   - snake_case naming convention"

TASK:
  "Implement a memory-mapped file wrapper class."

INTERFACE (show exactly what you want):
  "class MappedFile {
   public:
     static std::expected<MappedFile, std::error_code>
       open(const std::filesystem::path& path, AccessMode mode);

     std::span<const std::byte> data() const;
     std::span<std::byte> mutable_data();  // only if writable
     size_t size() const;

     // Move-only (owns mmap handle)
     MappedFile(MappedFile&&) noexcept;
     MappedFile& operator=(MappedFile&&) noexcept;
     ~MappedFile();  // munmap in destructor
   };"

CONSTRAINTS:
  "- No copy construction/assignment
   - Use mmap()/munmap() directly, no Boost
   - Handle files >4GB (use size_t, not int)
   - Return std::error_code on failure (not throw)"

EXAMPLE USAGE:
  "auto file = MappedFile::open("data.bin", AccessMode::ReadOnly);
   if (file) {
     auto bytes = file->data();
     process(bytes);
   } else {
     log_error(file.error());
   }"


=== SCENARIO: Performance-critical code ===

CONTEXT:
  "C++17, Clang 16, x86-64. Hot path in trading system.
   Latency budget: <100ns per call. Called 10M times/sec."

TASK:
  "Implement a lock-free timestamp cache that stores the
   last N timestamps and returns the median."

CONSTRAINTS:
  "- No heap allocation in the hot path
   - No mutexes or locks
   - Cache-friendly: all data in one cache line if possible
   - Use std::atomic with relaxed ordering where safe
   - Compile with -O3 -march=native"

BENCHMARK TARGET:
  "Include a benchmark using Google Benchmark that proves
   <100ns per median() call with N=16."


=== SCENARIO: Template library ===

CONTEXT:
  "Header-only C++20 library. Must work with GCC 12+,
   Clang 15+, MSVC 19.34+."

TASK:
  "Implement a type-safe printf replacement using
   variadic templates and concepts."

CONSTRAINTS:
  "- Compile-time format string validation (consteval)
   - Concepts to constrain printable types
   - No <format> dependency (not available on all targets)
   - Support: int, double, string_view, pointer, bool
   - Reject unsupported types at compile time"

EXPECTED:
  "safe_printf("Name: {}, Age: {}", name, age);  // OK
   safe_printf("Value: {}", my_mutex);  // Compile error"
```

Notice how the example usage is not just decoration - it is the most direct way to communicate intent. "Returns `std::expected`" in a constraints block is less clear than showing a call site where you use `.error()`. Show the AI what correct usage looks like and it will work backward to match it.

### Q2: Iterative prompt refinement workflow

**Answer:**

Complex code almost never comes out right in a single prompt. The iterative refinement workflow breaks the problem into four steps: explore the design space first, then scaffold the interface, then implement, then verify. Each step reviews the previous output before committing to the next one.

This matters more in C++ than in other languages because the cost of discovering a design mistake late is high. Getting the interface wrong before you implement means rewriting implementations. Getting the thread-safety model wrong before you verify means finding races in production.

```cpp
=== FOUR-STEP REFINEMENT WORKFLOW ===

STEP 1: EXPLORE (for unfamiliar topics)
  "What are the approaches for implementing a work-stealing
   thread pool in C++? Compare:
   - std::deque per thread with lock
   - Lock-free Chase-Lev deque
   - std::jthread with stop_token

   List trade-offs: complexity, performance, correctness."

  [Review response, choose approach]

STEP 2: SCAFFOLD
  "Implement a work-stealing thread pool with Chase-Lev deques.
   Show only the class declaration and key method signatures.
   Include comments for thread-safety guarantees.
   Don't implement the bodies yet."

  [Review interface, adjust before full implementation]

STEP 3: IMPLEMENT
  "Now implement the full class based on this interface:
   [paste refined interface from Step 2]
   Focus on correctness:
   - Memory ordering for the Chase-Lev deque
   - Proper shutdown sequence
   - Exception handling for tasks

   Add inline comments for non-obvious synchronization."

  [Review implementation]

STEP 4: VERIFY
  "Review this implementation for:
   1. Data races (any shared mutable state without sync?)
   2. ABA problems in the lock-free deque
   3. Memory ordering correctness (too weak? too strong?)
   4. Exception safety (what if a task throws?)
   5. Shutdown correctness (can tasks be lost?)

   For each issue found, show the fix."


=== REFINEMENT PROMPTS ===

"The code you generated uses std::mutex. Refactor to use
std::atomic<> with the minimum memory ordering that's correct.
Explain why each ordering was chosen."

"The solution allocates in the hot path (line 42, new Task{}).
Refactor to use a pre-allocated object pool instead.
Capacity fixed at construction time."

"Add [[nodiscard]], [[likely]]/[[unlikely]], and
__builtin_expect hints where they'd help. Annotate noexcept
on all functions that don't throw."
```

The refinement prompts at the bottom are useful for tightening up a first implementation that is correct but not optimal. "Minimum memory ordering" forces the AI to reason about which operations actually need synchronization, rather than defaulting to `memory_order_seq_cst` everywhere.

### Q3: Few-shot prompting for consistent code style

**Answer:**

The most effective way to teach an LLM your project's style is to show it an existing piece of code from the same codebase and ask it to follow the same patterns. This is called few-shot prompting, and it works better than describing the style in words. Descriptions leave room for interpretation; examples are unambiguous.

The key is to use a real, representative example - one that shows the patterns you care about (factory method, move semantics, error handling strategy, RAII discipline) all together in one place. Then you can say "follow the EXACT same patterns" and mean it:

```cpp
=== FEW-SHOT: teach the LLM your project style ===

"My project follows this pattern for RAII wrappers.
Here's an existing example:

class FileDescriptor {
public:
    static std::expected<FileDescriptor, std::error_code>
    open(const char* path, int flags) {
        int fd = ::open(path, flags);
        if (fd < 0)
            return std::unexpected(std::error_code(
                errno, std::system_category()));
        return FileDescriptor(fd);
    }

    FileDescriptor(FileDescriptor&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)) {}

    FileDescriptor& operator=(FileDescriptor&& other) noexcept {
        if (this != &other) {
            close();
            fd_ = std::exchange(other.fd_, -1);
        }
        return *this;
    }

    ~FileDescriptor() { close(); }

    int get() const noexcept { return fd_; }
    explicit operator bool() const noexcept { return fd_ >= 0; }

private:
    explicit FileDescriptor(int fd) : fd_(fd) {}
    void close() noexcept {
        if (fd_ >= 0) { ::close(fd_); fd_ = -1; }
    }

    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;

    int fd_ = -1;
};

Now write a similar RAII wrapper for a POSIX shared memory
region (shm_open/shm_unlink + mmap/munmap).
Follow the EXACT same patterns:
- Factory method returning std::expected
- Move-only
- Private constructor
- noexcept where possible
- std::exchange in move operations"
```

The example `FileDescriptor` class communicates several specific style choices in one go: `std::expected` for errors (not exceptions), `std::exchange` in the move constructor (not manual null-then-copy), private constructor to enforce the factory method, and a `close()` helper used by both destructor and move assignment. A well-chosen example like this is worth several paragraphs of style description.

---

## Notes

- **Context block** should be reusable across prompts - save it in a system prompt or template file that you paste at the start of every session.
- **Interface-first prompts** produce better results than "implement X" - you control the API shape and prevent the AI from inventing a different one.
- **Anti-requirements** ("no Boost", "no exceptions") prevent unwanted dependencies from appearing in generated code.
- **Few-shot examples** from your own codebase teach consistent style better than any written description.
- Iterative refinement (scaffold -> implement -> verify) beats one-shot for complex code - each step catches a different class of problem.
- For templates and concepts: show expected compile-time errors, not just successes. The model needs to know what "works correctly" means for invalid inputs.
- Always include **example usage** - it disambiguates intent better than any description and is the first thing you should add if the AI misunderstands the task.
