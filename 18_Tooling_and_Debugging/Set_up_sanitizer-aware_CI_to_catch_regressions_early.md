# Set up sanitizer-aware CI to catch regressions early

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

Sanitizers are **runtime bug detectors** built directly into Clang and GCC. When you compile with a sanitizer flag, the compiler inserts extra checks throughout your code that fire as soon as a bug is triggered at runtime. The key insight is that many memory bugs, data races, and undefined-behavior issues are completely silent in a normal build - they may not even crash. Sanitizers make them loud and immediately diagnosable.

Running sanitizers in CI means you catch these regressions the moment they are introduced, rather than weeks later when a mysterious crash appears in production.

| Sanitizer | Flag | Detects | Combinable? |
| --- | --- | --- | --- |
| ASAN | `-fsanitize=address` | Heap/stack overflow, use-after-free, leaks | + UBSAN |
| UBSAN | `-fsanitize=undefined` | Signed overflow, null deref, shift errors | + ASAN |
| TSAN | `-fsanitize=thread` | Data races, lock-order inversions | Alone only |
| MSAN | `-fsanitize=memory` | Uninitialized reads | Alone only |

The reason TSAN and MSAN must run alone is that they each need exclusive control over how shadow memory is laid out. You can think of it as three different programs all wanting to install different firmware in the same slot - they cannot coexist. The practical consequence is that your CI pipeline needs separate jobs:

```cpp
CI Pipeline:
  ├─ Job 1: ASAN + UBSAN   (catches memory + UB)
  ├─ Job 2: TSAN            (catches races)
  └─ Job 3: MSAN            (catches uninit reads)
       All must pass to merge!
```

---

## Self-Assessment

### Q1: Set up a matrix CI job with separate sanitizer builds

Here is a complete GitHub Actions workflow that runs all three sanitizer configurations in parallel. Each matrix entry compiles with different flags and sets different environment variables:

```yaml
# .github/workflows/sanitizers.yml
name: Sanitizer CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  sanitize:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - name: ASAN+UBSAN
            cxx_flags: "-fsanitize=address,undefined -fno-omit-frame-pointer -g"
            env_vars: "ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1"
          - name: TSAN
            cxx_flags: "-fsanitize=thread -g"
            env_vars: "TSAN_OPTIONS=halt_on_error=1:second_deadlock_stack=1"
          - name: MSAN
            cxx_flags: "-fsanitize=memory -fsanitize-memory-track-origins -fno-omit-frame-pointer -g"
            env_vars: "MSAN_OPTIONS=halt_on_error=1"

    name: ${{ matrix.name }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure
        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++-17 \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="${{ matrix.cxx_flags }}"

      - name: Build
        run: cmake --build build --parallel $(nproc)

      - name: Test
        env:
          ASAN_OPTIONS: "detect_leaks=1:halt_on_error=1"
          UBSAN_OPTIONS: "halt_on_error=1:print_stacktrace=1"
          TSAN_OPTIONS: "halt_on_error=1"
          MSAN_OPTIONS: "halt_on_error=1"
        run: ctest --test-dir build --output-on-failure --timeout 300
```

Notice that `Debug` build type is used. Optimizations in `Release` builds can rearrange or eliminate code in ways that actually hide bugs from sanitizers. Debug mode keeps the code shape closer to what you wrote.

### Q2: Why can't sanitizers all be combined

The reason trips people up because the answer is not "compatibility settings" - it is a fundamental conflict in how each sanitizer works internally. Each one needs to intercept memory allocations and maintain its own shadow memory, and those shadow layouts are physically incompatible.

| Conflict | Reason |
| --- | --- |
| ASAN + TSAN | Both replace `malloc`/`free` with different implementations |
| ASAN + MSAN | Both use shadow memory but with different layouts |
| TSAN + MSAN | Both instrument memory accesses differently |
| ASAN + UBSAN | Compatible - UBSAN just adds lightweight checks |

The shadow memory layouts are particularly illustrative. ASAN uses one shadow byte to represent eight bytes of application memory (a 1:8 ratio). TSAN uses four shadow words per eight bytes of application memory to track access history from multiple threads. Those two schemes cannot share the same address space:

```cpp
ASAN shadow memory layout:     TSAN shadow memory layout:
┌──────────┬──────────┐       ┌──────────┬──────────────────┐
│ App mem  │ Shadow   │       │ App mem  │ 4 shadow words   │
│ 8 bytes  │ 1 byte   │       │ 8 bytes  │ per 8 bytes      │
└──────────┴──────────┘       └──────────┴──────────────────┘
  1:8 ratio                     1:4 ratio (different!)
```

Additional reasons for separate jobs:

- Different sanitizers have **different performance overhead** (ASAN ~2x, TSAN ~5-15x, MSAN ~3x).
- MSAN requires **all** linked libraries to be instrumented (including libc++).
- Separate jobs can run in **parallel**, keeping total CI time manageable.

### Q3: CI configuration that blocks merge on sanitizer failures

The goal is to make sanitizer failures block the merge just like a failing test would. This requires two things: the sanitizer must actually exit with a non-zero code when it finds an issue, and GitHub branch protection must require those CI jobs to pass.

```yaml
# In the repo settings, configure branch protection:
# Settings > Branches > main > Require status checks:
#   ASAN+UBSAN
#   TSAN
#   MSAN

# The key configuration elements:

# 1. halt_on_error=1 ensures non-zero exit code:
env:
  ASAN_OPTIONS: "halt_on_error=1"
  UBSAN_OPTIONS: "halt_on_error=1:print_stacktrace=1"
  TSAN_OPTIONS: "halt_on_error=1"

# 2. CTest propagates the non-zero exit:
#    ctest returns non-zero if any test fails

# 3. GitHub Actions marks the job as failed
#    Branch protection rules block the merge
```

On the CMake side, it is cleaner to expose sanitizers as a CMake option rather than passing raw flags from the command line. This lets CI enable them without modifying the project's main configuration:

```cmake
# CMakeLists.txt
option(ENABLE_SANITIZERS "Enable sanitizer flags" OFF)

if(ENABLE_SANITIZERS)
    set(SANITIZER_FLAGS "-fsanitize=address,undefined -fno-omit-frame-pointer")
    add_compile_options(${SANITIZER_FLAGS})
    add_link_options(${SANITIZER_FLAGS})
endif()
```

---

## Notes

- Always use `-g` (debug info) with sanitizers for readable stack traces.
- Use `-fno-omit-frame-pointer` for complete call stacks.
- MSAN is Clang-only - GCC does not support it.
- Sanitizer builds should use `Debug` or `RelWithDebInfo`, not `Release`.
- Set `halt_on_error=1` so the CI job actually fails when issues are found.
- Consider adding `LSAN_OPTIONS=suppressions=lsan.supp` for known third-party leaks.
