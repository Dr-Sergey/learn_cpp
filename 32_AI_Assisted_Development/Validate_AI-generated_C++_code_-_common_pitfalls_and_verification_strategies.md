# Validate AI-generated C++ code - common pitfalls and verification strategies

**Category:** AI Assisted Development  
**Standard:** C++11  

---

## Topic Overview

AI-generated C++ code frequently contains subtle bugs that compile and appear to work but have **undefined behavior**, **resource leaks**, **thread-safety issues**, or **incorrect template instantiations**. The reason this is dangerous is exactly because the code looks fine - it passes a quick read, it compiles cleanly, and it passes basic tests. The bugs only show up under load, with optimizations enabled, or on a different platform. A disciplined verification pipeline catches these before they reach production. The key insight: AI code is a **first draft** that requires the same rigor as any pull request.

### Common AI-Generated C++ Pitfalls

The table below shows the most common classes of bugs, how often they appear, and which tool catches them. Notice that the "very high" frequency items are all about concurrency - that's the area where AI is most consistently unreliable.

| Pitfall | Frequency | Detection | Example |
| --- | --- | --- | --- |
| **Use-after-move** | High | UBSan, code review | `v.push_back(std::move(x)); use(x);` |
| **Dangling reference** | High | ASan | Returning ref to local |
| **Missing thread safety** | Very high | TSan | Shared state without mutex |
| **Wrong memory ordering** | High | TSan + stress test | `memory_order_relaxed` where `acquire` needed |
| **Integer overflow** | Medium | UBSan | `int` instead of `size_t` |
| **Exception-unsafe code** | Medium | Code review | Leak if constructor throws |
| **Incorrect RAII** | Medium | ASan, Valgrind | Missing destructor, copy in move |
| **ABI incompatibility** | Low | Link errors | `std::string` across DLL boundary |

---

## Self-Assessment

### Q1: Build a verification pipeline for AI-generated code

**Answer:**

The pipeline below runs six stages in sequence, each catching different classes of bugs. The key design decision is that sanitizers require separate builds - ASan and TSan cannot run in the same binary, so you need at least two test runs.

```bash
#!/bin/bash
# === AI Code Verification Pipeline ===

# Stage 1: Compile with maximum warnings
echo "=== Stage 1: Strict Compilation ==="
g++ -std=c++20 -Wall -Wextra -Werror -Wpedantic \
    -Wconversion -Wsign-conversion -Wshadow \
    -Wold-style-cast -Wnon-virtual-dtor \
    -Wcast-align -Woverloaded-virtual \
    -c generated_code.cpp -o /dev/null

# Stage 2: Static analysis
echo "=== Stage 2: Static Analysis ==="
clang-tidy generated_code.cpp \
    -checks='-*,bugprone-*,cert-*,cppcoreguidelines-*,\
             misc-*,modernize-*,performance-*,readability-*' \
    -- -std=c++20

cppcheck --enable=all --std=c++20 \
         --suppress=missingInclude \
         --error-exitcode=1 generated_code.cpp

# Stage 3: Build with sanitizers
echo "=== Stage 3: AddressSanitizer ==="
g++ -std=c++20 -g -O1 \
    -fsanitize=address,undefined \
    -fno-omit-frame-pointer \
    generated_code.cpp test_generated.cpp -o test_asan
./test_asan

# Stage 4: ThreadSanitizer (separate build, incompatible with ASan)
echo "=== Stage 4: ThreadSanitizer ==="
g++ -std=c++20 -g -O1 \
    -fsanitize=thread \
    generated_code.cpp test_generated.cpp -o test_tsan
./test_tsan

# Stage 5: Memory leak check
echo "=== Stage 5: Memory Leak Check ==="
valgrind --leak-check=full --error-exitcode=1 ./test_asan

# Stage 6: Coverage (verify all paths tested)
echo "=== Stage 6: Coverage ==="
g++ -std=c++20 --coverage generated_code.cpp test_generated.cpp \
    -o test_cov
./test_cov
gcovr --fail-under-branch 80 --fail-under-line 90

echo "=== All checks passed ==="
```

Stage 3 using `-O1` rather than `-O0` is intentional. Some UB only manifests under optimization - the compiler is allowed to assume UB never happens and eliminate code based on that assumption. Running at `-O0` can hide the bug by disabling the optimization that triggers it.

### Q2: Spot and fix common AI code bugs

**Answer:**

The five bugs below are the most common patterns you'll see in AI-generated C++ code. Each one compiles without warnings in basic configurations, which is exactly why they're dangerous. Work through each one so you recognize the pattern on sight.

```cpp
// === BUG 1: Use-after-move (AI frequently generates this) ===
// BAD:
void process(std::vector<std::string> items) {
    for (auto& item : items) {
        database.insert(std::move(item));
        logger.log("Inserted: " + item);  // BUG: item is moved-from!
    }
}
// GOOD:
void process(std::vector<std::string> items) {
    for (auto& item : items) {
        logger.log("Inserting: " + item);  // Log BEFORE move
        database.insert(std::move(item));
    }
}


// === BUG 2: Dangling reference to temporary ===
// BAD:
const std::string& get_name(int id) {
    auto it = cache_.find(id);
    if (it != cache_.end())
        return it->second;
    return "Unknown";  // BUG: dangling ref to temporary!
}
// GOOD:
std::string get_name(int id) {  // Return by value
    auto it = cache_.find(id);
    if (it != cache_.end())
        return it->second;
    return "Unknown";
}


// === BUG 3: Thread-unsafe singleton ===
// BAD:
class Config {
    static Config* instance_;
public:
    static Config& get() {
        if (!instance_)              // BUG: data race
            instance_ = new Config;  // BUG: leak + race
        return *instance_;
    }
};
// GOOD: Meyer's singleton (thread-safe since C++11)
class Config {
public:
    static Config& get() {
        static Config instance;  // Thread-safe, no leak
        return instance;
    }
};


// === BUG 4: Exception-unsafe resource acquisition ===
// BAD:
void setup() {
    auto* conn = new Connection(host);  // Raw new
    auto* cache = new Cache(conn);      // If throws, conn leaks!
    // ...
}
// GOOD: RAII
void setup() {
    auto conn = std::make_unique<Connection>(host);
    auto cache = std::make_unique<Cache>(conn.get());
    // Both cleaned up on any exception
}


// === BUG 5: Integer overflow in size calculation ===
// BAD:
void* allocate_buffer(int width, int height, int channels) {
    int size = width * height * channels;  // BUG: overflow!
    return malloc(size);    // BUG: negative size if overflow
}
// GOOD:
void* allocate_buffer(size_t width, size_t height, size_t channels) {
    size_t size;
    if (__builtin_mul_overflow(width, height, &size) ||
        __builtin_mul_overflow(size, channels, &size)) {
        return nullptr;  // Overflow detected
    }
    return malloc(size);
}
```

Bug 2 (dangling reference to temporary) is the reason you should almost never return a `const std::string&` from a function that might construct a new string as a fallback. The reference is immediately dangling. The fix is simply to return by value - modern compilers will often elide the copy anyway.

Bug 3 is the classic double-checked locking pattern, which is famously broken without memory barriers. Meyer's singleton (a function-local static) is the correct C++11 idiom: the standard guarantees that local statics are initialized exactly once, thread-safely, the first time control reaches them.

### Q3: Automated verification with CI integration

**Answer:**

The CI workflow below runs the same verification stages as the manual pipeline but in GitHub Actions, which means every pull request gets checked automatically. The important detail is that ASan+UBSan and TSan are separate jobs because they cannot coexist in the same binary.

```yaml
# === .github/workflows/verify-ai-code.yml ===
name: Verify AI-Generated Code
on: [pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Strict compilation

        run: |
          g++-13 -std=c++20 -Wall -Wextra -Werror -Wpedantic \
            -Wconversion -Wsign-conversion -Wshadow \
            -c src/*.cpp

      - name: Clang-tidy

        run: |
          clang-tidy-17 src/*.cpp \
            -checks='-*,bugprone-*,cert-*,cppcoreguidelines-*,\
                     performance-*,modernize-*' \
            -warnings-as-errors='*' \
            -- -std=c++20

      - name: Build with ASan+UBSan

        run: |
          cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined"
          cmake --build build

      - name: Run tests with sanitizers

        run: ctest --test-dir build --output-on-failure

      - name: Build with TSan

        run: |
          cmake -B build-tsan -DCMAKE_CXX_FLAGS="-fsanitize=thread"
          cmake --build build-tsan

      - name: Run concurrent tests with TSan

        run: ctest --test-dir build-tsan --output-on-failure

      - name: Coverage check

        run: |
          cmake -B build-cov -DCMAKE_CXX_FLAGS="--coverage"
          cmake --build build-cov
          ctest --test-dir build-cov
          gcovr --fail-under-branch 80 --fail-under-line 90
```

In addition to runtime checks, you can add compile-time guards to your codebase that catch entire categories of AI mistakes before they even reach the test stage. The helpers below use concepts and `static_assert` to reject code that doesn't meet your type requirements.

```cpp
// === Compile-time verification helpers ===
// Add to your codebase to catch AI mistakes at compile time:

// Prevent accidental copies of move-only types
#define DECLARE_MOVE_ONLY(ClassName) \
    ClassName(const ClassName&) = delete; \
    ClassName& operator=(const ClassName&) = delete; \
    ClassName(ClassName&&) noexcept = default; \
    ClassName& operator=(ClassName&&) noexcept = default;

// Ensure size_t instead of int for sizes
template<typename T>
concept SafeSize = std::unsigned_integral<T>;

void allocate(SafeSize auto size) {  // Rejects int, accepts size_t
    // ...
}

// Static assert for common mistakes
template<typename T>
class ResourceWrapper {
    static_assert(!std::is_pointer_v<T>,
        "Use smart pointers, not raw pointers");
    static_assert(std::is_nothrow_move_constructible_v<T>,
        "Resource types must be nothrow-moveable");
};
```

The `DECLARE_MOVE_ONLY` macro is particularly useful as a defense against AI code that accidentally introduces a copy constructor where only move was intended. If AI generates a copy operation for a type that manages a unique resource, the macro turns it into a compile error immediately.

---

## Notes

- Treat every AI-generated line as untested code from a junior developer - review it with the same scrutiny you'd apply to any pull request.
- The most dangerous bugs are those that compile and work in simple tests but fail under load, with optimizations, or on a different architecture.
- Sanitizers are non-negotiable: ASan + UBSan + TSan catch the vast majority of AI-generated bugs that testing alone misses.
- Static analysis (clang-tidy) catches style issues and common patterns like missing `const`, `[[nodiscard]]`, and obvious resource leaks.
- Use-after-move and dangling references are the most common AI mistakes in C++ - train yourself to spot the pattern.
- AI often generates C-with-classes instead of modern C++ - review specifically for RAII, smart pointers, and exception safety.
- Add compile-time checks (concepts, `static_assert`) to catch categories of bugs automatically before they reach the test stage.
