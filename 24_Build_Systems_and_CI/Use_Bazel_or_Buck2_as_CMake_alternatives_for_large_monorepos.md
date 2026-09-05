# Use Bazel or Buck2 as CMake alternatives for large monorepos

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://bazel.build>  

---

## Topic Overview

This file focuses on **monorepo-specific advantages** of Bazel and Buck2 versus CMake, covering reproducibility by design, the hermetic build model, and real-world performance comparisons for large-scale projects.

When a codebase grows into a true monorepo - multiple services, shared libraries, generated code, polyglot targets - CMake starts showing its limits. It was designed around a model where the build system trusts the developer to declare dependencies correctly and trusts the filesystem to reflect what is actually installed. In a large team, that trust breaks down quickly. Bazel and Buck2 take the opposite approach: they trust nothing, verify everything, and cache at the action level.

### Monorepo Build Challenges

Here is the kind of structure you are dealing with in a large monorepo, and the specific problems that arise when you try to manage it with CMake:

```cpp
// Large monorepo (1000+ targets):
// |-- services/auth/       (C++ microservice)
// |-- services/billing/    (C++ microservice)
// |-- libs/networking/     (shared C++ library)
// |-- libs/logging/        (shared C++ library)
// |-- tools/codegen/       (C++ code generator)
// `-- frontend/dashboard/  (TypeScript - Bazel handles polyglot!)

// Problems CMake struggles with:
//
//   1. Which targets changed? -> CMake re-configures everything
//   2. Can I cache across machines? -> No native remote cache
//   3. Is my build correct? -> Implicit deps can hide bugs
//   4. Can I parallelize across machines? -> No remote execution
```

---

## Self-Assessment

### Q1: Write a BUILD file that defines a cc_library and a cc_binary and builds them with bazel build

**Answer:**

In Bazel, every target is described in a `BUILD` file using a Python-like syntax called Starlark. The key discipline is that every source file and every dependency must be listed explicitly - no globs, no implicit include paths. Here is a realistic library definition:

```python
# libs/networking/BUILD
cc_library(
    name = "http_client",
    srcs = [
        "http_client.cpp",
        "connection_pool.cpp",
    ],
    hdrs = [
        "http_client.h",
        "connection_pool.h",
    ],
    deps = [
        "//libs/logging:logger",       # Internal dependency
        "@curl//:curl",                 # External C library
        "@abseil-cpp//absl/status",     # Abseil status
        "@abseil-cpp//absl/strings",
    ],
    copts = ["-std=c++20"],
    visibility = ["//services:__subpackages__"],  # Only services can use this
)
```

And a binary that depends on it, along with a test target:

```python
# services/auth/BUILD
cc_binary(
    name = "auth_service",
    srcs = ["main.cpp", "auth_handler.cpp"],
    hdrs = ["auth_handler.h"],
    deps = [
        "//libs/networking:http_client",
        "//libs/logging:logger",
        "@grpc//:grpc++",
        "@protobuf//:protobuf",
    ],
    copts = ["-std=c++20", "-Wall", "-Wextra"],
    linkopts = ["-lpthread"],
)

cc_test(
    name = "auth_handler_test",
    srcs = ["auth_handler_test.cpp"],
    deps = [
        ":auth_service_lib",   # Extract testable lib from binary
        "@googletest//:gtest_main",
    ],
    size = "small",
)
```

Once the BUILD files are written, the command-line interface is simple and consistent. Notice how `bazel query` can answer questions about the dependency graph that would be impossible to answer with CMake:

```bash
# Build one target:
bazel build //services/auth:auth_service

# Build everything:
bazel build //...

# Build only changed targets since last commit:
bazel build //... --test_tag_filters=-large

# Query what depends on a library:
bazel query 'rdeps(//..., //libs/networking:http_client)'
# Output:
# //services/auth:auth_service
# //services/billing:billing_service
# -> Shows impact of changing http_client
```

### Q2: Explain Bazel's hermetic build model and why it achieves reproducibility by construction

**Answer:**

The hermetic build model is Bazel's core insight. Instead of hoping that developers declare all their dependencies correctly, Bazel makes it impossible to use an undeclared dependency. The build runs inside a sandbox that only contains what was explicitly listed. If something is missing, you get a compile error, not a silent success that breaks later on a different machine.

There are three pillars to this:

```cpp
// Bazel's Hermetic Build Model

// Three pillars of hermeticity:

// 1. EXPLICIT DEPENDENCIES (no implicit includes)
//
//    cc_library(
//      name = "parser",
//      srcs = ["parser.cpp"],
//      hdrs = ["parser.h"],
//      deps = ["//utils:string_helpers"],
//    )
//
//    parser.cpp can ONLY #include:
//      - parser.h (own hdrs)
//      - utils/string_helpers.h (from deps)
//      - standard library headers
//      - NOTHING ELSE (sandbox enforced!)

// 2. SANDBOXED EXECUTION
//
//    Each build action runs in an isolated sandbox:
//
//    - Only declared inputs are visible
//    - Undeclared file system access fails
//    - Environment variables are cleared
//    - /tmp is per-action
//
//    -> Build result depends ONLY on declared inputs

// 3. CONTENT-ADDRESSED CACHING
//
//    action_hash = hash(
//      command_line,
//      input_file_hashes,
//      environment_variables,
//      toolchain_hash
//    )
//    -> Same inputs ALWAYS produce same outputs
//    -> Cache is correct by construction (no stale entries)

// Why CMake can't guarantee this:
//
// CMake:
//
//   - Uses file timestamps (not content hashes)
//   - Header scanning is approximate (may miss deps)
//   - No sandbox -> can accidentally use host headers
//   - No content-based cache -> stale builds possible
//
//   Example bug (can't happen in Bazel):
//
//     1. Developer adds #include "utils/helper.h" to parser.cpp
//     2. Forgets to add utils to CMake target_link_libraries
//     3. Build works because helper.h is in the include path
//     4. CI fails (or worse: links wrong version)
//     5. Debugging takes hours
```

### Q3: Compare Bazel and CMake for a 1000-target monorepo: incremental build time, remote caching

**Answer:**

The performance gap between CMake and Bazel widens as the project grows. The key difference is not raw compilation speed - both use the same compiler - but what gets rebuilt after a change, and whether results can be reused from a shared cache. Here is real-world data for a project of this scale:

```cpp
// Performance Comparison (Real-World Data)

// Project: 1000 C++ targets, 500K lines of code, 200 test targets
//
//                           |  CMake + Ninja      |  Bazel + Remote     |
//                           |  (local, sccache)   |  (remote exec+cache)|
// --------------------------|---------------------|---------------------|
// Full clean build           |  25 min (16 cores)  |  4 min (200 remote) |
// 1 header in lib changed    |  90 sec (recompile  |  5 sec (only        |
//   (20 dependents)          |   all dependents)   |   affected actions) |
// 1 .cpp file changed        |  15 sec             |  3 sec              |
// No-change rebuild          |  3 sec (Ninja)      |  0.5 sec (cached)   |
// New developer first build  |  25 min             |  4 min (cache warm) |
// CI (warm cache)            |  5 min (sccache)    |  30 sec (remcache)  |
//
// Space efficiency:
//   CMake: each developer has full build tree (~10 GB)
//   Bazel: shared remote cache, local only needs active targets
//
// Correctness:
//   CMake: missing deps may silently "work" until CI fails
//   Bazel: sandbox catches ALL missing deps immediately
//
// Scale ceiling:
//   CMake: practical limit ~5000 targets (configure slows down)
//   Bazel: tested at Google scale (millions of targets)
```

The remote caching story is especially compelling for a large team. When CI builds an artifact, every developer who checks out that commit gets it for free:

```bash
# Bazel remote caching performance

# Developer A pushes a change, CI builds it:
bazel build //...
# INFO: 1000 processes: 950 remote cache hit, 50 internal

# Developer B pulls the same commit:
bazel build //...
# INFO: 1000 processes: 1000 remote cache hit
# Build time: 2 seconds (all from cache!)

# CMake equivalent requires more setup
# cmake + sccache can cache compilation, but:
# - Linking is not cached
# - Test results are not cached
# - Configuration step is not cached
# - Code generation steps are not cached
# Bazel caches ALL of these at the action level
```

**Migration cost:** Converting a CMake project to Bazel requires writing BUILD files for every target. For a 1000-target project, this is roughly 2-4 weeks of effort. The ROI depends on team size and CI frequency.

---

## Notes

- **Buck2** (Meta) uses a similar model but has better support for multi-language builds (C++ + Rust + Python), which can be a deciding factor for polyglot monorepos.
- **Bazel's visibility system** controls which targets can depend on which - this prevents the "everybody depends on everything" problem that tends to develop in large codebases over time.
- **`bazel query`** and **`bazel cquery`** enable powerful dependency analysis: find affected targets, detect cycles, audit deps - none of this is possible with CMake without external tooling.
- **Hybrid approach:** Use `rules_foreign_cc` to wrap existing CMake projects inside Bazel BUILD files during migration, so you don't have to convert everything at once.
- **Turborepo / Nx** are similar content-addressed build systems for JavaScript/TypeScript monorepos, if the comparison to the JS ecosystem is helpful context.
