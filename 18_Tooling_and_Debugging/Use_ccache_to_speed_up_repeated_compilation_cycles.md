# Use ccache to speed up repeated compilation cycles

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://ccache.dev>  

---

## Topic Overview

ccache stores a mapping from a hash of your preprocessed source and compiler flags to the compiled object file. When you ask for the same compilation again - same source, same flags, same compiler - it returns the cached object in milliseconds rather than invoking the compiler at all.

Understanding what goes into the cache key explains when you get a hit and when you do not:

```cpp
Cache key = hash of:

  1. Preprocessed source code (after #include expansion)
  2. Compiler binary (path + content hash)
  3. Compiler flags (-O2, -std=c++20, etc.)
  4. Target architecture

Cache value = compiled object file (.o)
```

The reason ccache hashes the preprocessed source rather than the raw source file is that it accounts for header changes automatically. If you change a header that `main.cpp` includes, the preprocessed output of `main.cpp` changes, the hash changes, and ccache correctly triggers a miss.

---

## Self-Assessment

### Q1: CMake integration with `CMAKE_CXX_COMPILER_LAUNCHER`

Setting `CMAKE_CXX_COMPILER_LAUNCHER` tells CMake to prepend a wrapper program to every compiler invocation. Setting it to `ccache` means every `g++` or `clang++` call goes through ccache first:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(FastBuild LANGUAGES CXX)

# Auto-detect ccache
find_program(CCACHE ccache)
if(CCACHE)
    message(STATUS "Using ccache: ${CCACHE}")
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE})
    set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE})
else()
    message(STATUS "ccache not found; building without cache")
endif()

set(CMAKE_CXX_STANDARD 20)
add_executable(app main.cpp engine.cpp renderer.cpp physics.cpp)
```

```bash
# Build with ccache:
cmake -B build && cmake --build build -j$(nproc)

# Or override from command line:
cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
```

### Q2: Cache hit/miss statistics after a clean + rebuild cycle

The best demonstration of ccache's value is a clean rebuild. Delete your build directory, rebuild, and watch every compilation become a cache hit:

```bash
# 1. Clear everything
ccache -C && ccache -z

# 2. First build (all misses)
cmake --build build --clean-first -j$(nproc)
ccache -s
```

After the first build you will see all misses, which is expected:

```text
cache directory                     /home/user/.cache/ccache
primary config                      /home/user/.config/ccache/ccache.conf
cache hit (direct)                     0
cache hit (preprocessed)               0
cache miss                            42
called for preprocessing               0
files in cache                        42
cache size                          12.3 MB
```

```bash
# 3. Clean and rebuild (all hits!)
cmake --build build --clean-first -j$(nproc)
ccache -s
```

The second build is dramatically faster, and the statistics show why:

```text
cache hit (direct)                    42   # <-- all hits!
cache hit (preprocessed)               0
cache miss                            42   # from first build
files in cache                        42
cache size                          12.3 MB
```

It is worth understanding the two types of cache hit. A **direct** hit means ccache found a match without even running the preprocessor - it recognized the source file and compiler flags immediately. A **preprocessed** hit means ccache had to run the preprocessor first to compute the hash, then found a match. Direct hits are faster because they skip preprocessing entirely.

### Q3: What ccache caches and when it invalidates

Here is a concrete picture of how ccache decides whether to recompile or return a cached result:

```cpp
Source file:  main.cpp
  │
  ├─ #include <vector>    ──┐
  ├─ #include "config.h"  ─┤  All expanded into
  └─ your code            ─┤  one preprocessed blob
                            │
                     hash(blob + flags + compiler)
                            │
                     cache lookup
                     ┌────┴────┐
                   HIT         MISS
                   │             │
               return .o    compile -> store .o
```

The specific events that trigger a miss or preserve a hit:

| Event | Result | Explanation |
| --- | --- | --- |
| Edit `main.cpp` | Miss | Source content changed |
| Edit `config.h` | Miss | Included header changed |
| Change `-O2` to `-O3` | Miss | Flags are part of hash |
| Upgrade GCC 13 -> 14 | Miss | Compiler binary changed |
| Switch branches | Depends | Hit if same code, miss if different |
| `git stash pop` (restore) | **Hit** | Same code as before = same hash |
| Add a comment | **Hit** | Preprocessor removes comments |
| Change `__TIME__` macro | Miss | Preprocessed output differs |

The comment behavior is worth highlighting: adding or removing comments, reformatting whitespace, and touching documentation strings in the source file all produce cache hits because the preprocessor strips those before hashing. You can annotate your code freely without breaking the cache.

---

## Notes

- ccache's default cache size is 5 GB; increase it with `ccache --set-config=max_size=20G` for larger projects.
- Use `ccache -p` to see the full configuration, including where the cache directory is.
- ccache supports remote caching via Redis or HTTP for sharing a warm cache across multiple CI machines.
- sccache is an alternative with native cloud storage support (S3, GCS, Azure) if you need a distributed cache.
- Combine ccache with Ninja for the fastest possible builds: the two complement each other well.
