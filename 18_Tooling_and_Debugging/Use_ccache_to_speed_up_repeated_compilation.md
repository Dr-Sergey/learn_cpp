# Use ccache to speed up repeated compilation

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://ccache.dev>  

---

## Topic Overview

ccache is a compiler cache that sits between your build system and the actual compiler. The first time you compile a file, ccache stores the resulting `.o` file in a local cache indexed by a hash of the preprocessed source and compiler flags. The next time you compile the same file with the same flags, ccache returns the cached object file instead of invoking the compiler - turning a 2-second compilation into a 10-millisecond cache lookup.

The practical payoff is enormous. If you delete your build directory and rebuild from scratch (something CI systems do constantly), a fully warm ccache can turn a 10-minute build into a 30-second build:

```cpp
Without ccache:          With ccache:
  source.cpp               source.cpp
      │                        │
  preprocessor             preprocessor
      │                        │
  compiler (~2s)           hash lookup
      │                    ┌───┴───┐
  source.o                miss      hit
                          │         │
                       compiler   return cached .o
                       (~2s)      (~0.01s)
```

---

## Self-Assessment

### Q1: Set up ccache with CMake

The cleanest way to integrate ccache with CMake is through the `CMAKE_CXX_COMPILER_LAUNCHER` variable. CMake will prepend ccache to every compiler invocation, so your build files stay unchanged:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

# Method 1: CMAKE_CXX_COMPILER_LAUNCHER (recommended)
find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
    set(CMAKE_C_COMPILER_LAUNCHER ccache)
    set(CMAKE_CXX_COMPILER_LAUNCHER ccache)
    message(STATUS "ccache found: ${CCACHE_FOUND}")
endif()

set(CMAKE_CXX_STANDARD 20)
add_executable(myapp main.cpp utils.cpp parser.cpp)
```

You can also enable it without changing the CMakeLists.txt, which is handy when working with a project you do not own:

```bash
# Method 2: Command-line override
cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

# Method 3: Environment variable
export CMAKE_CXX_COMPILER_LAUNCHER=ccache
cmake -B build && cmake --build build

# Method 4: Symlink (legacy, not recommended)
#   ln -s /usr/bin/ccache /usr/local/bin/g++
```

The symlink approach is the old way of using ccache and it works, but the CMake launcher approach is much cleaner because it is explicit and does not interfere with anything else that uses the compiler binary.

### Q2: Measure cache-hit speedup

The best way to see what ccache is doing is to clear the cache, build once to populate it, then rebuild from a clean directory and watch all the hits come in:

```bash
# Step 1: Clear ccache and build from scratch
ccache -C              # clear cache
ccache -z              # zero statistics

cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
time cmake --build build --parallel $(nproc)
# -> real 0m45.3s (first build, all cache MISSES)

ccache -s | head -5
# cache hit (direct):     0
# cache hit (preprocessed): 0
# cache miss:             127
# Misses: 127

# Step 2: Clean build directory, rebuild (cache already populated)
rm -rf build
cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
time cmake --build build --parallel $(nproc)
# -> real 0m3.2s (~14x faster! all cache HITS)

ccache -s | head -5
# cache hit (direct):     127
# cache hit (preprocessed): 0
# cache miss:             127
# Hits: 127, Misses: 127 (50% overall, 100% on second build)
```

The speedup scales with project size:

| Project Size | First Build | Cached Rebuild | Speedup |
| --- | --- | --- | --- |
| Small (10 TUs) | 5s | 0.5s | 10x |
| Medium (100 TUs) | 45s | 3s | 15x |
| Large (1000+ TUs) | 10min | 30s | 20x |

### Q3: When ccache invalidates its cache

ccache builds its cache key by hashing the preprocessed source plus the compiler flags. Understanding what changes the key tells you when you will get a cache miss and when you will get a hit.

| Change | Cache Effect | Reason |
| --- | --- | --- |
| Source file modified | Miss | Different preprocessed output |
| Header file modified | Miss | Included content changed |
| Compiler flags changed | Miss | Different hash (e.g., -O2 -> -O3) |
| Different compiler version | Miss | Compiler binary hash differs |
| Only whitespace/comments | **Hit** | Preprocessor strips them |
| Different build directory | **Hit** | ccache normalizes paths |
| `#define` value changed | Miss | Preprocessed output differs |
| Same code, different file name | **Hit** (with `sloppiness`) | Hash is on content, not path |

The whitespace and comment behavior is particularly useful: you can reformat your code, add documentation comments, or change blank lines without invalidating a single cache entry. ccache operates on the preprocessed output, not the raw source.

```bash
# Configure ccache for best results:
ccache --set-config=max_size=10G       # increase cache size
ccache --set-config=compression=true    # save disk space
ccache --set-config=sloppiness=include_file_mtime  # ignore mtime

# Useful ccache commands:
ccache -s       # show statistics
ccache -C       # clear cache
ccache -z       # zero statistics
ccache -p       # show configuration
```

---

## Notes

- ccache works with GCC, Clang, and MSVC (via ccache-msvc).
- Combine with Ninja for the fastest builds overall: `cmake -G Ninja -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`.
- ccache and precompiled headers (PCH) can conflict; use `sloppiness=pch_defines,time_macros` to smooth this over.
- You can share a ccache across CI runners with a remote storage backend such as Redis or an HTTP cache server.
- sccache (Mozilla) is an alternative with native S3/GCS remote caching support, which is worth considering for cloud-based CI.
