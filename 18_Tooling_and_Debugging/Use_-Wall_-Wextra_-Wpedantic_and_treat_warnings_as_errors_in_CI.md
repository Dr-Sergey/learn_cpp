# Use -Wall, -Wextra, -Wpedantic and treat warnings as errors in CI

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html>  

---

## Topic Overview

Warning flags are one of the cheapest tools available to you - they catch bugs at compile time with zero runtime cost. The reason to use all three of `-Wall`, `-Wextra`, and `-Wpedantic` together is that they cover different ground, and each one catches real bugs that the others miss.

| Flag | Catches | Example |
| --- | --- | --- |
| `-Wall` | Common bugs | Unused variables, missing return, sign compare |
| `-Wextra` | Extra checks | Unused parameters, missing field initializers |
| `-Wpedantic` | Standard conformance | GNU extensions, trailing commas |
| `-Werror` | Treat all warnings as errors | Forces clean builds in CI |

One thing that surprises new users: `-Wall` does **not** enable all warnings. The name is historical. It enables a curated set of about 40 widely-agreed-upon checks, but there are many more that GCC/Clang know about:

```cpp
-Wall         ⊂  -Wextra       ⊂  -Wpedantic
(~40 warnings)  (~20 more)       (~10 more)
```

Adding `-Werror` in CI converts any remaining warning into a build failure, which forces the codebase to stay clean. You typically use `-Werror` only in CI (not developer builds), with targeted `-Wno-error=...` exclusions for specific warnings you want to see but not block on.

---

## Self-Assessment

### Q1: Enable all three flags and fix every warning

The best way to understand these flags is to see what each one flags and how to silence it correctly. Here is a file with one problem per warning group, shown before and after the fix:

```cpp
// warnings_demo.cpp
// Compile: g++ -std=c++20 -Wall -Wextra -Wpedantic -Werror warnings_demo.cpp

#include <iostream>

// ====== BEFORE (has warnings) ======

// -Wall: missing return value
// int get_value(int x) {
//     if (x > 0) return x;
//     // WARNING: control reaches end of non-void function
// }

// -Wextra: unused parameter
// void process(int data, int unused_param) {
//     std::cout << data << '\n';
//     // WARNING: unused parameter 'unused_param'
// }

// -Wpedantic: trailing comma in enum (GNU extension)
// enum Color { Red, Green, Blue, };
//                              ^-- WARNING: comma at end of enumerator list

// ====== AFTER (all warnings fixed) ======

int get_value(int x) {
    if (x > 0) return x;
    return 0;  // FIX: always return a value
}

void process(int data, [[maybe_unused]] int unused_param) {
    std::cout << data << '\n';
    // FIX: [[maybe_unused]] silences -Wextra warning
}

enum Color { Red, Green, Blue };  // FIX: no trailing comma

int main() {
    std::cout << get_value(42) << '\n';
    process(10, 0);
    Color c = Red;
    std::cout << "Color: " << c << '\n';
}
// Expected output:
// 42
// 10
// Color: 0
```

The missing-return-value warning is particularly important. Without it, a function that falls off the end of a non-void function exhibits undefined behavior - the compiler is allowed to assume that code path is never taken, which can lead to bizarre optimizations and crashes that appear nowhere near the actual bug.

### Q2: `-Werror` in CI with false-positive suppression workflow

In CI you want warnings to be hard errors. But there are legitimate situations where you need to silence specific warnings - for example, a third-party callback API that forces an unused parameter signature, or code that deliberately uses deprecated APIs during a migration. The key is to suppress surgically rather than globally:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

# Enable strict warnings
add_compile_options(
    -Wall -Wextra -Wpedantic
    -Werror                    # treat warnings as errors in CI
    -Wno-error=deprecated-declarations  # allow deprecated for migration
)

# For a specific target with a known third-party issue:
add_library(third_party_wrapper wrapper.cpp)
target_compile_options(third_party_wrapper PRIVATE
    -Wno-unused-parameter      # third-party callback signature
)
```

There are three clean techniques for suppressing a warning at the source level when you cannot change the problematic code:

```cpp
#include <iostream>

void callback(int event_id, void* /*user_data*/) {
    // Method 1: Comment out parameter name (preferred)
    std::cout << "Event: " << event_id << '\n';
}

void legacy_api(int x, [[maybe_unused]] int reserved) {
    // Method 2: [[maybe_unused]] attribute (C++17)
    std::cout << x << '\n';
}

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wdeprecated-declarations"
void use_old_api() {
    // Method 3: pragma to disable warning for a block
    // old_deprecated_function();
}
#pragma GCC diagnostic pop

int main() {
    callback(1, nullptr);
    legacy_api(42, 0);
}
// Expected output:
// Event: 1
// 42
```

Method 1 (commenting out the parameter name) is the most portable and the most obvious to a reader. Method 2 uses a standard C++17 attribute, which is the right tool when the parameter must remain named for documentation. Method 3 is the escape hatch when you need to suppress a warning in a block of code you do not own.

### Q3: Warnings caught by -Wextra but not -Wall, and -Wpedantic only

These are the ones people most often miss because they assume `-Wall` handles everything:

```cpp
#include <iostream>

// ====== -Wextra catches (but -Wall misses) ======

// 1. Missing field initializer:
struct Point { int x; int y; int z; };
Point p = {1, 2};  // -Wextra: missing initializer for 'Point::z'
                    // -Wall: silent!

// 2. Signed/unsigned comparison in certain contexts:
void compare(int a, unsigned b) {
    if (a == b) {}  // -Wextra may warn about signed/unsigned compare
                    // depending on context
}

// ====== -Wpedantic catches (but -Wall and -Wextra miss) ======

// 1. Zero-length arrays (GNU extension):
// struct Packet {
//     int header;
//     char data[0];  // -Wpedantic: ISO C++ forbids zero-size array
// };

// 2. Statement expressions (GNU extension):
// int x = ({ int y = 5; y * 2; });  // -Wpedantic: ISO C++ forbids

// 3. typeof (pre-C++23 GNU extension):
// typeof(42) val = 10;  // -Wpedantic: use decltype instead

int main() {
    std::cout << "p = {" << p.x << ", " << p.y << ", " << p.z << "}\n";
    compare(1, 1u);
}
// Compile: g++ -std=c++20 -Wall -Wextra -Wpedantic warnings.cpp
// Expected output:
// p = {1, 2, 0}
```

The missing-field-initializer case is particularly sneaky. The struct member `z` is zero-initialized when you write `{1, 2}` - that is well-defined in C++. But if you refactored and added `z` later without updating all initializers, `-Wextra` will remind you that you may have forgotten to set something intentionally. Without `-Wextra`, that refactor silently "works" while leaving a latent bug.

---

## Notes

- `-Wall` does **NOT** enable all warnings (misleading name). Use `-Wall -Wextra -Wpedantic`.
- Useful extras beyond the big three: `-Wconversion`, `-Wshadow`, `-Wnon-virtual-dtor`.
- MSVC equivalent: `/W4 /WX` (W4 = high warnings, WX = treat as errors).
- Recommended CMake: `set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wpedantic")`.
- In CI, use `-Werror` but allow specific suppressions for third-party code.
- Consider `-Wno-error=...` for warnings you want to see but not block on.
