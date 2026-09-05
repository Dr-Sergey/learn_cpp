# Use Sanitizers in CI to catch bugs that tests alone miss

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

Unit tests verify logic, but many critical bugs - buffer overflows, data races, undefined behavior - can hide behind **"passing"** tests. Sanitizers instrument your compiled code to detect these invisible bugs at runtime.

The reason tests alone miss these bugs is subtle. A heap buffer overflow might read a few bytes of padding that happen to be zero, and your test assertion passes. A use-after-free might access memory that has not been reclaimed yet, so the old value is still there. A data race might only manifest when thread scheduling happens to interleave in a particular way. None of these are detectable by checking return values - you need instrumentation built into the binary itself.

### Bugs Tests Alone Miss

| Bug Class | Why Tests Pass | Sanitizer That Catches It |
| --- | --- | --- |
| Heap buffer overflow | Read lands in allocated padding | ASAN |
| Use-after-free | Memory not yet reclaimed | ASAN |
| Signed integer overflow | Works on x86 (wraps) | UBSAN |
| Uninitialized read | Happens to be 0 | MSAN |
| Data race | Thread scheduling hides it | TSAN |
| Null dereference (optimized away) | Compiler removes check | UBSAN |
| Stack-use-after-scope | Old value still on stack | ASAN |

### CI Strategy: Separate Jobs per Sanitizer

The key constraint is that ASAN and TSAN cannot be combined in the same binary - they instrument memory at a low level and conflict with each other. ASAN and UBSAN are compatible and can share a job. MSAN is the most expensive to set up because it requires a specially instrumented libc++.

```bash
                git push
                   │
        ┌──────────┼──────────┬──────────┐
        ▼          ▼          ▼          ▼
    Release    ASAN+UBSAN    TSAN      MSAN
    -O2        -O1 -g        -O1 -g    -O1 -g
    fast       2× slower     10× slow  3× slow
    │          │             │          │
    ctest      ctest         ctest      ctest
    │          │             │          │
    ship     block         block      block
```

---

## Self-Assessment

### Q1: Run the full test suite with ASAN, UBSAN, and TSAN in separate CI jobs

The GitHub Actions matrix strategy below runs each sanitizer as its own independent job. The `fail-fast: false` setting is important: if ASAN finds something, you still want to see what TSAN finds, so you do not cancel the other jobs early.

**GitHub Actions matrix strategy:**

```yaml
# .github/workflows/sanitizers.yml
name: Sanitizer Matrix

on: [push, pull_request]

jobs:
  sanitizer-tests:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false  # Run ALL sanitizer jobs even if one fails
      matrix:
        include:
          - name: ASAN+UBSAN
            cxx_flags: "-fsanitize=address,undefined -fno-omit-frame-pointer -fno-sanitize-recover=all -g -O1"
            link_flags: "-fsanitize=address,undefined"
            env_opts: "ASAN_OPTIONS=abort_on_error=1:detect_leaks=1 UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1"
          - name: TSAN
            cxx_flags: "-fsanitize=thread -g -O1"
            link_flags: "-fsanitize=thread"
            env_opts: "TSAN_OPTIONS=halt_on_error=1:second_deadlock_stack=1"

    name: ${{ matrix.name }}
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++-17 \
            -DCMAKE_CXX_FLAGS="${{ matrix.cxx_flags }}" \
            -DCMAKE_EXE_LINKER_FLAGS="${{ matrix.link_flags }}"
      - name: Build
        run: cmake --build build -j$(nproc)
      - name: Test
        env:
          ASAN_OPTIONS: "abort_on_error=1:detect_leaks=1"
          UBSAN_OPTIONS: "halt_on_error=1:print_stacktrace=1"
          TSAN_OPTIONS: "halt_on_error=1"
        run: cd build && ctest --output-on-failure --timeout 300
```

The `-fno-sanitize-recover=all` flag is worth explaining: by default, UBSAN prints an error message and continues running. That means CI would not fail even when UB is detected. This flag makes every UBSAN finding abort the process immediately, so the test runner sees a crash and marks the job as failed.

### Q2: Show a test that passes without sanitizers but catches a bug with ASAN enabled

Each of these three bugs represents a class of error that passes a logic-based test but would be caught by ASAN. The comments explain both what the test sees without instrumentation and what ASAN reports when it runs the same code.

```cpp
#include <gtest/gtest.h>
#include <vector>
#include <cstring>

// Bug 1: Heap buffer overflow
std::string get_initials(const std::string& name) {
    std::string result;
    const char* p = name.c_str();
    for (size_t i = 0; i <= name.size(); ++i) {  // BUG: <= instead of <
        if (i == 0 || p[i - 1] == ' ') {
            result += p[i];  // Reads one byte past buffer on last iteration!
        }
    }
    return result;
}

TEST(Initials, BasicName) {
    EXPECT_EQ(get_initials("John Doe"), "JD");
    // WITHOUT sanitizer: PASSES (reads padding byte, happens to not be ' ')
    // WITH ASAN: ERROR: heap-buffer-overflow on address 0x602000000039
    //   READ of size 1 at 0x602000000039
    //   #0 get_initials(std::string const&) main.cpp:8
}

// Bug 2: Use-after-free
class EventQueue {
    std::vector<std::string> events_;
public:
    void add(const std::string& e) { events_.push_back(e); }

    const std::string& last() const { return events_.back(); }

    void process() {
        for (auto& e : events_) { /* handle */ }
        events_.clear();  // Frees all strings
    }
};

TEST(EventQueue, ProcessAndLast) {
    EventQueue q;
    q.add("click");
    const std::string& ref = q.last();
    q.process();  // Clears the vector - 'ref' is dangling!

    // WITHOUT sanitizer: might print "click" (memory not yet reused)
    // WITH ASAN: ERROR: heap-use-after-free
    EXPECT_FALSE(ref.empty());  // UB! Accessing freed memory
}

// Bug 3: Stack buffer overflow
void copy_name(char* dest, const char* src) {
    strcpy(dest, src);  // No bounds check!
}

TEST(CopyName, LongInput) {
    char buffer[8];
    copy_name(buffer, "Alexander");  // 10 bytes into 8-byte buffer

    // WITHOUT sanitizer: might work (overwrites stack, no immediate crash)
    // WITH ASAN: ERROR: stack-buffer-overflow
    EXPECT_STREQ(buffer, "Alexander");
}
```

### Q3: Configure CMake presets for sanitizer builds and add them to the CI matrix

Using CMake presets rather than custom CMake variables has a real practical benefit: any developer can reproduce exactly what CI runs with a single command. The preset handles all the flags, so there is no chance of getting them slightly wrong.

**CMakePresets.json:**

```json
{
    "version": 6,
    "configurePresets": [
        {
            "name": "default",
            "binaryDir": "${sourceDir}/build/default",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_CXX_FLAGS": "-O2"
            }
        },
        {
            "name": "asan",
            "binaryDir": "${sourceDir}/build/asan",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_COMPILER": "clang++",
                "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer -fno-sanitize-recover=all -g -O1",
                "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined"
            }
        },
        {
            "name": "tsan",
            "binaryDir": "${sourceDir}/build/tsan",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_COMPILER": "clang++",
                "CMAKE_CXX_FLAGS": "-fsanitize=thread -g -O1",
                "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=thread"
            }
        },
        {
            "name": "msan",
            "binaryDir": "${sourceDir}/build/msan",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_COMPILER": "clang++",
                "CMAKE_CXX_FLAGS": "-fsanitize=memory -fno-omit-frame-pointer -g -O1 -stdlib=libc++",
                "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=memory -stdlib=libc++"
            }
        }
    ],
    "buildPresets": [
        { "name": "default", "configurePreset": "default" },
        { "name": "asan",    "configurePreset": "asan" },
        { "name": "tsan",    "configurePreset": "tsan" },
        { "name": "msan",    "configurePreset": "msan" }
    ],
    "testPresets": [
        { "name": "default", "configurePreset": "default", "output": {"outputOnFailure": true} },
        { "name": "asan",    "configurePreset": "asan",    "output": {"outputOnFailure": true} },
        { "name": "tsan",    "configurePreset": "tsan",    "output": {"outputOnFailure": true} },
        { "name": "msan",    "configurePreset": "msan",    "output": {"outputOnFailure": true} }
    ]
}
```

**Using presets in CI:**

```yaml
jobs:
  sanitizer-matrix:
    strategy:
      matrix:
        preset: [asan, tsan]
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: cmake --preset ${{ matrix.preset }}
      - name: Build
        run: cmake --build --preset ${{ matrix.preset }} -j$(nproc)
      - name: Test
        run: ctest --preset ${{ matrix.preset }}
```

**Local developer workflow:**

```bash
# Quick sanitizer check before pushing
cmake --preset asan
cmake --build --preset asan -j$(nproc)
ctest --preset asan

# If a test fails, ASAN output looks like:
# =================================================================
# ==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000039
#     #0 0x4a3b2c in my_function src/parser.cpp:42
#     #1 0x4a3b2c in TEST_Parser_Basic_Test::TestBody() tests/parser_test.cpp:17
# =================================================================
```

---

## Notes

- Sanitizer findings are **deterministic** - same binary + same input = same report (unlike Valgrind)
- MSAN is the hardest to set up: requires libc++ built with MSAN instrumentation
- Add `detect_container_overflow=1` to ASAN_OPTIONS for catching std::vector OOB
- Use `symbolize=1` and ensure `llvm-symbolizer` is on PATH for readable stack traces
- Sanitizer runtime overhead makes them unsuitable for production - CI/dev builds only
- When a sanitizer finds a bug, add a **regression test** for that specific input
