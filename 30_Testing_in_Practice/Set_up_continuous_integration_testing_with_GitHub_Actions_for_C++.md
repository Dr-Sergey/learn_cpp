# Set up continuous integration testing with GitHub Actions for C++

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Continuous Integration (CI)** automatically builds and tests your code on every push and pull request, so regressions are caught before they reach the main branch rather than after. GitHub Actions is currently the most popular CI platform for GitHub-hosted projects, and it has excellent support for C++ workflows including matrix builds, caching, and integration with coverage and sanitizer tools.

The key insight behind a well-designed C++ CI pipeline is layering. You want cheap checks (basic build) to run first and block the expensive checks (sanitizers, coverage) from even starting if the basics are broken. This is what the `needs:` keyword in GitHub Actions does for you - it sequences jobs so you're not wasting 20 minutes of sanitizer time on a build that has a compilation error.

### CI Pipeline Stages

Here's the overall shape of a mature C++ CI pipeline. Each column represents a separate job that can run in parallel once its dependencies complete:

```cpp
Push/PR -> Build -> Unit Tests -> Integration Tests -> Static Analysis -> Deploy
   |          |          |              |                    |
   |       Matrix:    CTest         Label-based          clang-tidy
   |     GCC/Clang   GTest        (longer timeout)       cppcheck
   |     C++17/20/23  sanitizers      coverage
   v           v         v               v                   v
           Fail fast on first error, report all results
```

---

## Self-Assessment

### Q1: Create a comprehensive C++ CI workflow

**Answer:**

The workflow below covers the most important concerns: a matrix build across OS/compiler/build-type combinations, a sanitizer job that only runs if the basic build passes, and a coverage job. The `fail-fast: false` setting in the matrix is important - it means all matrix configurations finish even when one fails, so you get complete information rather than having to re-run the whole pipeline to find additional failures.

```yaml
# === .github/workflows/ci.yml ===
name: C++ CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    strategy:
      fail-fast: false  # Don't cancel other jobs on first failure
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        compiler: [gcc, clang]
        build_type: [Debug, Release]
        exclude:

          - os: windows-latest

            compiler: gcc      # Use MSVC on Windows instead

          - os: macos-latest

            compiler: gcc
        include:

          - os: windows-latest

            compiler: msvc
            build_type: Debug

          - os: windows-latest

            compiler: msvc
            build_type: Release

    runs-on: ${{ matrix.os }}
    name: ${{ matrix.os }}-${{ matrix.compiler }}-${{ matrix.build_type }}

    steps:

      - uses: actions/checkout@v4

        with:
          submodules: recursive

      - name: Install dependencies (Ubuntu)

        if: runner.os == 'Linux'
        run: |
          sudo apt-get update
          sudo apt-get install -y ninja-build
          if [ "${{ matrix.compiler }}" = "clang" ]; then
            sudo apt-get install -y clang-18 lld-18
          fi

      - name: Set compiler (GCC)

        if: matrix.compiler == 'gcc'
        run: |
          echo "CC=gcc-13" >> $GITHUB_ENV
          echo "CXX=g++-13" >> $GITHUB_ENV

      - name: Set compiler (Clang)

        if: matrix.compiler == 'clang'
        run: |
          echo "CC=clang-18" >> $GITHUB_ENV
          echo "CXX=clang++-18" >> $GITHUB_ENV

      - name: Configure

        run: |
          cmake -B build -G Ninja \
            -DCMAKE_BUILD_TYPE=${{ matrix.build_type }} \
            -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
            -DBUILD_TESTING=ON

      - name: Build

        run: cmake --build build --parallel

      - name: Run tests

        run: ctest --test-dir build --output-on-failure --parallel 4
        timeout-minutes: 10

      - name: Upload test results

        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.compiler }}-${{ matrix.build_type }}
          path: build/Testing/

  sanitizers:
    runs-on: ubuntu-latest
    needs: build-and-test  # Only run if basic tests pass
    strategy:
      matrix:
        sanitizer: [address, undefined, thread]
    name: sanitizer-${{ matrix.sanitizer }}

    steps:

      - uses: actions/checkout@v4

        with:
          submodules: recursive

      - name: Configure with sanitizer

        run: |
          cmake -B build -G Ninja \
            -DCMAKE_CXX_COMPILER=clang++-18 \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="-fsanitize=${{ matrix.sanitizer }} -fno-omit-frame-pointer"

      - name: Build

        run: cmake --build build --parallel

      - name: Run tests with sanitizer

        run: ctest --test-dir build --output-on-failure
        timeout-minutes: 20

  coverage:
    runs-on: ubuntu-latest
    needs: build-and-test

    steps:

      - uses: actions/checkout@v4

        with:
          submodules: recursive

      - name: Install lcov

        run: sudo apt-get install -y lcov

      - name: Configure with coverage

        run: |
          cmake -B build -G Ninja \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="--coverage -fprofile-arcs -ftest-coverage"

      - name: Build and test

        run: |
          cmake --build build --parallel
          ctest --test-dir build --output-on-failure

      - name: Generate coverage report

        run: |
          lcov --capture --directory build --output-file coverage.info
          lcov --remove coverage.info '/usr/*' '*/test/*' --output-file coverage.info
          lcov --list coverage.info

      - name: Check coverage threshold

        run: |
          COVERAGE=$(lcov --summary coverage.info 2>&1 | grep 'lines' | grep -oP '[\d.]+%' | head -1 | tr -d '%')
          echo "Line coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below 80% threshold"
            exit 1
          fi
```

Notice the `timeout-minutes: 10` on the test step and `timeout-minutes: 20` on the sanitizer step. Always set these - without them, a deadlock or infinite loop in your tests will consume the full GitHub Actions job timeout (6 hours by default), blocking other PRs and burning through your CI minutes.

### Q2: Cache dependencies for faster builds

**Answer:**

Without caching, every CI run downloads googletest, rebuilds all your source, and recompiles your dependencies from scratch. That's wasteful. GitHub Actions caching works by hashing a key file - if the hash hasn't changed, the cache is restored and you skip the download/rebuild. The two most impactful things to cache for C++ are the build directory (especially when using `FetchContent`) and `ccache` for incremental compilation.

```yaml
# === Add to the build job steps (before Configure) ===

      - name: Cache CMake build

        uses: actions/cache@v4
        with:
          path: build
          key: ${{ runner.os }}-${{ matrix.compiler }}-${{ matrix.build_type }}-${{ hashFiles('CMakeLists.txt', 'cmake/**') }}
          restore-keys: |
            ${{ runner.os }}-${{ matrix.compiler }}-${{ matrix.build_type }}-

      - name: Cache vcpkg packages

        uses: actions/cache@v4
        with:
          path: ~/.cache/vcpkg
          key: vcpkg-${{ runner.os }}-${{ hashFiles('vcpkg.json') }}

      - name: Cache Conan packages

        uses: actions/cache@v4
        with:
          path: ~/.conan2/p
          key: conan-${{ runner.os }}-${{ hashFiles('conanfile.txt') }}

      # === ccache for incremental compilation ===

      - name: Install ccache

        run: sudo apt-get install -y ccache

      - name: Cache ccache

        uses: actions/cache@v4
        with:
          path: ~/.ccache
          key: ccache-${{ runner.os }}-${{ matrix.compiler }}-${{ github.sha }}
          restore-keys: ccache-${{ runner.os }}-${{ matrix.compiler }}-

      - name: Configure with ccache

        run: |
          cmake -B build -G Ninja \
            -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
            -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}
```

The `restore-keys` fallback is worth understanding. When the exact cache key doesn't match (e.g., after a `CMakeLists.txt` change), GitHub Actions tries the fallback keys in order. Using a prefix without the hash as the fallback means you'll get a partial cache hit from the previous build of the same compiler/OS combination - much faster than a cold start.

### Q3: Add branch protection and PR status checks

**Answer:**

The point of branch protection is that required status checks must pass before a PR can merge. The trick is to configure a single summary job (`all-checks`) as the required check, rather than listing every individual matrix combination - otherwise adding a new OS to the matrix requires updating the branch protection rules too.

```yaml
# === Separate static analysis job ===
  static-analysis:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Configure

        run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

      - name: clang-tidy (changed files only)

        run: |
          # Only analyze files changed in this PR
          git diff --name-only origin/main -- '*.cpp' '*.h' | \
            xargs -r clang-tidy-18 -p build --warnings-as-errors='*'

      - name: cppcheck

        run: |
          cppcheck --enable=all --std=c++20 \
            --error-exitcode=1 \
            --suppress=missingIncludeSystem \
            --project=build/compile_commands.json

  # === Required status check summary ===
  all-checks:
    needs: [build-and-test, sanitizers, coverage, static-analysis]
    runs-on: ubuntu-latest
    if: always()
    steps:

      - name: Check all jobs

        run: |
          if [ "${{ needs.build-and-test.result }}" != "success" ] || \
             [ "${{ needs.sanitizers.result }}" != "success" ] || \
             [ "${{ needs.coverage.result }}" != "success" ]; then
            echo "One or more required checks failed"
            exit 1
          fi
```

Running clang-tidy only on changed files in PRs (rather than the whole codebase) keeps analysis time fast. You get full analysis on pushes to main where runtime cost is less of a concern. The `CMAKE_EXPORT_COMPILE_COMMANDS=ON` flag is required for clang-tidy to understand your include paths and preprocessor definitions correctly.

Branch protection settings to configure (Settings -> Branches -> main):

```cpp
Branch Protection Rules (Settings > Branches > main):

- Require status checks: "all-checks"
- Require up-to-date branches
- Require linear history (optional: enforce rebase)
- Include administrators
```

---

## Notes

- `fail-fast: false` in matrix builds ensures all configurations are tested even if one fails.
- Use `needs:` to sequence expensive jobs (sanitizers) after basic build succeeds.
- Cache `build/`, `~/.ccache`, and package manager caches for 2-5x faster builds.
- Run clang-tidy only on changed files in PRs - full analysis on main branch pushes.
- Set `timeout-minutes` on test steps to catch infinite loops and deadlocks.
- Use a summary job (`all-checks`) as the single required status check for branch protection.
- Sanitizer builds should use `-O1` not `-O0` - some bugs only manifest with optimization.
- Coverage thresholds prevent test erosion: fail the PR if coverage drops below threshold.
