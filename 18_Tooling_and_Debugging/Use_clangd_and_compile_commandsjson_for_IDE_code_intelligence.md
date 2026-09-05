# Use clangd and compile_commands.json for IDE code intelligence

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clangd.llvm.org>  

---

## Topic Overview

clangd is a language server that provides IDE features - code completion, inline diagnostics, go-to-definition, rename, and more - by running the actual Clang compiler on your code. The key difference from simpler syntax-highlighting tools is that clangd understands your code at the same depth as the compiler does. It knows the types, it resolves overloads, and it evaluates concept constraints. That means the errors it shows you are real compiler errors, not guesses.

To work correctly, clangd needs to know the exact compiler flags for each file. That information comes from a `compile_commands.json` file, which is a list of every compilation command in your project. CMake can generate this file automatically.

```cpp
Editor (VS Code/Vim/Emacs)
       │ LSP protocol
    clangd
       │ reads
 compile_commands.json
  [
    { "file": "main.cpp",
      "command": "g++ -std=c++20 -I/usr/include ...",
      "directory": "/project/build" }
  ]
```

---

## Self-Assessment

### Q1: Generate `compile_commands.json` with CMake

One line in your `CMakeLists.txt` is all it takes. When `CMAKE_EXPORT_COMPILE_COMMANDS` is on, CMake writes `compile_commands.json` into the build directory during configuration:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

# THIS IS THE KEY LINE:
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 20)
add_executable(app src/main.cpp src/engine.cpp src/renderer.cpp)
target_include_directories(app PRIVATE include)
```

```bash
# Generate compile_commands.json:
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# The file is in build/compile_commands.json
# Symlink it to project root for clangd to find:
ln -s build/compile_commands.json .
```

clangd looks for `compile_commands.json` by walking up from each source file, so symlinking it to the project root makes it visible to the whole project. The generated file has one entry per source file, each with the full compiler invocation:

```json
[
  {
    "directory": "/home/user/project/build",
    "command": "/usr/bin/g++ -std=c++20 -Iinclude -o CMakeFiles/app.dir/src/main.cpp.o -c /home/user/project/src/main.cpp",
    "file": "/home/user/project/src/main.cpp"
  },
  {
    "directory": "/home/user/project/build",
    "command": "/usr/bin/g++ -std=c++20 -Iinclude -o CMakeFiles/app.dir/src/engine.cpp.o -c /home/user/project/src/engine.cpp",
    "file": "/home/user/project/src/engine.cpp"
  }
]
```

Every include path, every `-D` flag, every `-std` flag - clangd sees exactly what the compiler would see.

### Q2: clangd template and concept diagnostics

This is where clangd really earns its place. Template errors in C++ are notoriously verbose and hard to read when they come from the command line. clangd shows them inline in your editor at the exact call site, with a readable explanation of why the constraint failed.

```cpp
// clangd shows this error INLINE in the editor:
#include <concepts>
#include <iostream>

template<std::integral T>  // concept constraint
T add(T a, T b) { return a + b; }

int main() {
    add(1, 2);       // OK: int satisfies std::integral
    add(1.5, 2.5);   // clangd: error! double does not satisfy std::integral
    //                // clangd message:
    //                // "no matching function for call to 'add'"
    //                // "candidate template ignored: constraints not satisfied"
    //                // "because 'std::integral<double>' evaluated to false"
}
```

Compare what basic syntax highlighting can do versus what clangd provides:

| Feature | Basic Syntax | clangd |
| --- | --- | --- |
| Syntax errors | Some | All (Clang-accurate) |
| Template errors | None | Full diagnostics |
| Concept violations | None | Detailed constraint info |
| Include errors | None | Missing header detection |
| Go-to-definition | Text match | Semantic (correct overload) |
| Auto-complete | Keyword-based | Context-aware (types, scopes) |
| Rename | Text replace | Semantic (all references) |

The go-to-definition difference is important in practice. A text-based tool takes you to the first file that contains the name. clangd takes you to the specific overload that the compiler actually selected for that call.

### Q3: Configure `.clangd` for custom include paths

Sometimes your project has unusual include paths, or you want clangd to apply different settings to test files versus production files. The `.clangd` config file handles this. Place it in the project root alongside your `compile_commands.json`:

```yaml
# .clangd (place in project root)

CompileFlags:
  Add:
    - "-std=c++20"
    - "-I/opt/custom-lib/include"
    - "-I../third_party/json/include"
    - "-Wall"
    - "-Wextra"

  Remove:
    - "-W*"           # remove all warning flags from compile_commands.json

  Compiler: clang++    # use clang++ for analysis even if project uses g++

Diagnostics:
  ClangTidy:
    Add:
      - modernize-*
      - performance-*
      - bugprone-*

    Remove:
      - modernize-use-trailing-return-type  # too noisy

Index:
  Background: Build    # index in background after build

# Separate settings for test files:
---
If:
  PathMatch: tests/.*
CompileFlags:
  Add:
    - "-I../third_party/catch2/include"
```

The conditional `If: PathMatch:` section at the bottom is what makes this powerful for larger projects. Test files often need extra include paths that do not belong in the production compile commands, and this lets you add them selectively.

Verify your configuration is taking effect with:

```bash
# Verify clangd sees your configuration:
clangd --check=src/main.cpp
# Shows: flags used, errors found, time taken
```

---

## Notes

- Always symlink `compile_commands.json` to the project root so clangd can find it regardless of which source file it is analyzing.
- For non-CMake builds, use Bear (`bear -- make`) to record compiler invocations and produce a `compile_commands.json`.
- In VS Code, install the "clangd" extension and disable the Microsoft C/C++ IntelliSense extension - running both simultaneously causes conflicts.
- clangd indexes the entire project in the background after the first open, which is what enables fast go-to-definition across the whole codebase.
- Use `clangd --query-driver=/usr/bin/g++*` when using GCC so clangd can detect the correct GCC system headers.
- The `.clangd` config file is YAML; use `---` as a separator between conditional sections (one per `If:` block).
