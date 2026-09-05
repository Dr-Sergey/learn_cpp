# Use include-what-you-use (IWYU) to enforce minimal includes

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://include-what-you-use.org>  

---

## Topic Overview

Include-what-you-use (IWYU) is a static analysis tool built on Clang that answers two questions about every source file: which headers are you including but not actually using, and which headers are you relying on without including them directly? The first category should be removed. The second category is a latent bug waiting to surface when you upgrade your compiler or switch standard library implementations.

```cpp
Before IWYU:                       After IWYU:
#include <algorithm>               #include <algorithm>  // for std::sort
#include <iostream>                #include <vector>     // for std::vector
#include <vector>                  #include <string>     // for std::string
#include <map>         // UNUSED!
#include <functional>  // UNUSED!
// std::string used but not        // <iostream> removed (unused)
// directly included!              // <map>, <functional> removed
```

The goal is that every symbol you use in a translation unit is backed by a header you include explicitly. This makes your code portable across compilers, robust against library implementation changes, and faster to compile because the compiler only processes headers that are actually needed.

---

## Self-Assessment

### Q1: Run IWYU and remove transitive includes

Here is a file that looks like it works but is actually relying on implementation-specific behavior - specifically, the fact that `<algorithm>` on GCC happens to pull in headers for `std::string` and `std::vector`.

```cpp
// bad_includes.cpp - relies on transitive includes
#include <algorithm>   // pulls in <utility> on some implementations
#include <map>         // not used at all!
#include <string>
#include <vector>

// Uses std::string but doesn't include <string>
// Works on GCC because <algorithm> transitively includes it.
void greet(const std::string& name) {
    // Uses std::vector but doesn't include <vector>
    // Works because <algorithm> pulls it in on this compiler.
    std::vector<int> data = {3, 1, 2};
    std::sort(data.begin(), data.end());
}

int main() {
    greet("World");
}
```

Running IWYU produces a clear prescription:

```bash
# Run IWYU:
$ include-what-you-use -std=c++20 bad_includes.cpp

# IWYU output:
# bad_includes.cpp should add these lines:
#   #include <string>   // for std::string
#   #include <vector>   // for std::vector
#
# bad_includes.cpp should remove these lines:
#   - #include <map>    // lines 2-2
#
# The full include-list for bad_includes.cpp:
#   #include <algorithm>  // for sort
#   #include <string>     // for string
#   #include <vector>     // for vector

# Auto-apply fixes:
$ include-what-you-use -std=c++20 bad_includes.cpp 2>&1 | fix_includes.py
```

After applying the fixes, the file is explicit about everything it uses:

```cpp
// bad_includes.cpp - after IWYU
#include <algorithm>   // for std::sort
#include <string>      // for std::string
#include <vector>      // for std::vector

void greet(const std::string& name) {
    std::vector<int> data = {3, 1, 2};
    std::sort(data.begin(), data.end());
}

int main() {
    greet("World");
}
```

Notice that the comments now document why each header is there. This is a convention IWYU encourages and one that makes the include block informative rather than mysterious.

### Q2: Why transitive includes are fragile

The reason transitive includes are a problem is that they are not part of the C++ standard. Standard headers are only required to declare the symbols they document. Whether `<algorithm>` happens to include `<string>` internally is an implementation detail that can change at any time.

```cpp
// This code compiles on GCC 12 but FAILS on GCC 13:
#include <algorithm>
// GCC 12: <algorithm> includes <string> internally
// GCC 13: <algorithm> no longer includes <string>

void process(const std::string& s) {  // ERROR on GCC 13!
    // std::string not declared
}
```

The problem is not just theoretical - GCC 13 actually removed several transitive includes that projects had been relying on for years:

```cpp
Transitive include fragility:

GCC 12:                              GCC 13:
<algorithm>                          <algorithm>
  └─> <utility>                        └─> <utility>
        └─> <string>  works!               (no <string>)  breaks!

The fix: always #include what you directly use.
```

Real-world examples of broken transitive includes:

- GCC 13 removed `<string>`, `<cstdint>`, `<algorithm>` transitive includes from many headers.
- libc++ (Clang) has always been stricter than libstdc++ about transitive includes.
- Upgrading compilers without IWYU frequently breaks builds.

The pattern of "it works on my machine" followed by "CI is broken after the compiler upgrade" is almost always a transitive include problem. Running IWYU in CI prevents this class of regression entirely.

### Q3: Configure IWYU mappings for custom libraries

For your own libraries, IWYU needs to know which internal headers correspond to which public API headers. Without this information, IWYU might suggest that users include internal implementation files directly, which is not what you want. Mapping files solve this.

```yaml
# my_library.imp - IWYU mapping file
[
  # Map internal headers to public headers:
  { include: ["\"mylib/internal/impl.h\"", private,
              "\"mylib/mylib.h\"", public] },

  # Map multiple internals to one public header:
  { include: ["\"mylib/internal/parser_impl.h\"", private,
              "\"mylib/parser.h\"", public] },
  { include: ["\"mylib/internal/lexer_impl.h\"", private,
              "\"mylib/parser.h\"", public] },

  # Symbol-based mapping:
  { symbol: ["mylib::Widget", private,
             "\"mylib/widget.h\"", public] },

  # Reference an existing mapping file:
  { ref: "third_party/boost.imp" }
]
```

Once you have a mapping file, you can use it from the command line or integrate it directly into your CMake build:

```bash
# Use mapping file:
$ include-what-you-use -Xiwyu --mapping_file=my_library.imp -std=c++20 main.cpp

# CMake integration:
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_INCLUDE_WHAT_YOU_USE
    include-what-you-use
    -Xiwyu --mapping_file=${CMAKE_SOURCE_DIR}/my_library.imp
    -Xiwyu --no_fwd_decls  # optional: disable forward declaration suggestions
)

add_executable(myapp main.cpp)
```

Setting `CMAKE_CXX_INCLUDE_WHAT_YOU_USE` causes CMake to run IWYU as a linting step alongside every compilation, which means you get include hygiene feedback continuously as you build rather than in a separate pass.

---

## Notes

- IWYU uses Clang's frontend under the hood, so you pass it Clang-compatible flags rather than GCC flags.
- `--no_fwd_decls` disables forward declaration suggestions, which can sometimes be overly aggressive about replacing full includes with declarations.
- Mapping files handle the distinction between internal implementation headers and the public API headers that library consumers should include.
- Running IWYU in CI is the best way to prevent transitive include regressions from sneaking in over time.
- IWYU may not fully support the very latest C++23 or C++26 features yet - check the project's release notes before relying on it for cutting-edge code.
