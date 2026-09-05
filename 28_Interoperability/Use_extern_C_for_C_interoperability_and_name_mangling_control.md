# Use extern C for C interoperability and name mangling control

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/language_linkage>  

---

## Topic Overview

This topic focuses on the **practical patterns** surrounding `extern "C"`: designing dual-language headers, controlling symbol visibility, and understanding how the linker resolves symbols across C and C++ translation units.

The fundamental problem `extern "C"` solves is name mangling. In C++, the compiler encodes a function's full signature (parameter types, namespace, class membership) into its symbol name. This is what enables overloading - two functions named `process` with different parameter types get different symbol names. In C, there is no overloading, so the symbol name is just the function name, unchanged. When you try to link C code against a C++ library, the linker can't match the C object file's `process` against the C++ object file's `_Z7processi`. `extern "C"` tells the C++ compiler: "emit a plain, unmangled symbol for this function."

### Symbol Resolution Pipeline

The diagram below shows exactly what goes wrong without `extern "C"` and what goes right with it.

```cpp
C++ Translation Unit               C Translation Unit
┌─────────────────────┐           ┌─────────────────────┐
│ extern "C"          │           │                     │
│ int process(int x); │           │ int process(int x); │
│        │            │           │        │            │
│   Compiler (g++)    │           │   Compiler (gcc)    │
│        │            │           │        │            │
│   Symbol: process   │           │   Symbol: process   │
└────────┬────────────┘           └────────┬────────────┘
         │                                 │
         └──────────┬──────────────────────┘
                    │ Linker
                    ▼
              Symbol match: "process" - linked successfully
```

Without `extern "C"`, the C++ side would emit `_Z7processi` instead of `process`, and the linker would report an undefined reference error.

### extern "C" Rules Summary

Here is what you can and cannot put behind an `extern "C"` declaration. The restrictions are driven by the fact that C doesn't have overloading, templates, namespaces, or exceptions - so features that depend on those concepts can't be given C linkage.

| What you CAN do | What you CANNOT do |
| --- | --- |
| Free functions | Overloaded functions (same name) |
| Function pointers | Templates |
| Global variables | Class member functions |
| Entire blocks `extern "C" { }` | Namespaced linkage (name appears global) |
| `noexcept` functions | Let exceptions propagate |

---

## Self-Assessment

### Q1: Wrap a C++ function with extern "C" and verify the unmangled symbol name with nm

**Answer:**

The example below is a real-world-style audio processing library. The header uses the `#ifdef __cplusplus` guard pattern so it can be included by both C and C++ consumers without modification - this is the standard idiom for dual-language headers.

```cpp
// ═══════════ audio.h — dual C/C++ header ═══════════
#ifndef AUDIO_H
#define AUDIO_H

#ifdef __cplusplus
extern "C" {
#endif

typedef struct AudioBuffer {
    float* samples;
    int    num_samples;
    int    sample_rate;
} AudioBuffer;

// Lifecycle
AudioBuffer* audio_create(int num_samples, int sample_rate);
void         audio_destroy(AudioBuffer* buf);

// Processing
void audio_normalize(AudioBuffer* buf);
void audio_apply_gain(AudioBuffer* buf, float gain_db);
float audio_peak_amplitude(const AudioBuffer* buf);

#ifdef __cplusplus
}
#endif

#endif
```

The C++ implementation uses `std::` functions internally - `extern "C"` doesn't restrict what you do inside the function body, only the name the linker sees.

```cpp
// ═══════════ audio.cpp — C++ implementation ═══════════
#include "audio.h"
#include <cmath>
#include <cstdlib>
#include <algorithm>

extern "C" {

AudioBuffer* audio_create(int num_samples, int sample_rate) {
    auto* buf = static_cast<AudioBuffer*>(std::malloc(sizeof(AudioBuffer)));
    if (!buf) return nullptr;
    buf->samples = static_cast<float*>(
        std::calloc(num_samples, sizeof(float)));
    if (!buf->samples) { std::free(buf); return nullptr; }
    buf->num_samples = num_samples;
    buf->sample_rate = sample_rate;
    return buf;
}

void audio_destroy(AudioBuffer* buf) {
    if (buf) {
        std::free(buf->samples);
        std::free(buf);
    }
}

void audio_normalize(AudioBuffer* buf) {
    float peak = audio_peak_amplitude(buf);
    if (peak > 0.0f) {
        float scale = 1.0f / peak;
        for (int i = 0; i < buf->num_samples; ++i)
            buf->samples[i] *= scale;
    }
}

void audio_apply_gain(AudioBuffer* buf, float gain_db) {
    float linear = std::pow(10.0f, gain_db / 20.0f);
    for (int i = 0; i < buf->num_samples; ++i)
        buf->samples[i] *= linear;
}

float audio_peak_amplitude(const AudioBuffer* buf) {
    float peak = 0.0f;
    for (int i = 0; i < buf->num_samples; ++i)
        peak = std::max(peak, std::abs(buf->samples[i]));
    return peak;
}

}  // extern "C"
```

After compiling, `nm` lets you verify directly that the symbols are unmangled. The ` T ` flag means the symbol lives in the text (code) section - all five functions appear with their plain names.

```bash
# Compile and verify symbols:
g++ -c audio.cpp -o audio.o
nm audio.o | grep " T "

# Output (all unmangled):
# 0000000000000000 T audio_apply_gain
# 0000000000000040 T audio_create
# 0000000000000090 T audio_destroy
# 00000000000000c0 T audio_normalize
# 0000000000000100 T audio_peak_amplitude

# Compare: without extern "C" they'd be:
# _Z16audio_apply_gainP11AudioBufferf
# _Z12audio_createii
# etc
```

### Q2: Explain C++ name mangling and why it prevents direct linking with C object files

**Answer:**

Name mangling encodes function signature information into the symbol name. This is **required** for C++ features that C doesn't have. The table below shows exactly how each feature contributes to the mangled symbol, using GCC's Itanium ABI format.

```cpp
Feature that needs mangling        Mangled symbol (GCC Itanium ABI)
──────────────────────────         ────────────────────────────────
Overloading:
  void print(int)                  _Z5printi
  void print(double)               _Z5printd
  void print(const char*)          _Z5printPKc

Namespaces:
  ns::foo()                        _ZN2ns3fooEv

Classes:
  MyClass::method(int)             _ZN7MyClass6methodEi

Templates:
  max<int>(int, int)               _Z3maxIiET_S0_S0_

const/volatile:
  void f(const int&)               _Z1fRKi
  void f(int&)                     _Z1fRi
```

Here is what actually goes wrong when you try to link C and C++ object files without `extern "C"`. The error message from the linker says "undefined reference to `process`" - which is confusing because you know you defined it. The real cause is that the linker is looking for the plain name `process` (from the C side) and finding only `_Z7processi` (from the C++ side).

```cpp
C translation unit (compiled with gcc):
  Declares: int process(int x);
  Expects symbol: process

C++ translation unit (compiled with g++):
  Defines: int process(int x) { return x * 2; }
  Emits symbol: _Z7processi

Linker: C object needs "process", C++ object has "_Z7processi"
-> UNDEFINED REFERENCE ERROR

Fix: extern "C" int process(int x) { return x * 2; }
  Emits symbol: process
-> Linker matches successfully
```

The reason MSVC mangling is different is that there is no standard for name mangling - every compiler vendor chose their own scheme. **MSVC uses different mangling** (`?process@@YAHH@Z`), so mangled symbols are not portable between compilers. `extern "C"` is the **only** way to guarantee cross-compiler symbol compatibility.

### Q3: Create a C-compatible header with #ifdef __cplusplus guards for dual C/C++ use

**Answer:**

This example shows the full pattern for wrapping a C++ data structure (an `unordered_map`) behind a C-compatible API. The key design decisions are: use an opaque handle so C code never sees the C++ internals, use `void` in empty parameter lists (required for C89/C90 compatibility), and use `<stddef.h>` rather than `<cstddef>` so the header works in both languages.

```cpp
// ═══════════ hashmap.h — C API for a C++ unordered_map ═══════════
#ifndef HASHMAP_H
#define HASHMAP_H

#include <stddef.h>  // size_t — use C header, not <cstddef>
#include <stdbool.h> // bool in C99+ — harmless in C++

#ifdef __cplusplus
extern "C" {
#endif

// Opaque handle — C can't see the internals
typedef struct HashMap HashMap;

// ── Lifecycle ──
HashMap* hashmap_create(void);           // void for C compatibility
void     hashmap_destroy(HashMap* map);

// ── Operations ──
bool   hashmap_insert(HashMap* map, const char* key, int value);
bool   hashmap_remove(HashMap* map, const char* key);
bool   hashmap_contains(HashMap* map, const char* key);
int    hashmap_get(HashMap* map, const char* key, int default_value);
size_t hashmap_size(const HashMap* map);

// ── Iteration (callback-based for C compatibility) ──
typedef void (*hashmap_visitor)(const char* key, int value, void* user_data);
void hashmap_for_each(const HashMap* map, hashmap_visitor fn, void* user_data);

#ifdef __cplusplus
}   // extern "C"
#endif

#endif // HASHMAP_H
```

The implementation is where the C++ lives. Notice that the `HashMap` struct is redefined in the `.cpp` file with its real internals - this is valid because the C header's `typedef struct HashMap HashMap` just forward-declares it as an opaque type. C code can hold `HashMap*` pointers and pass them around without ever needing to know what's inside.

```cpp
// ═══════════ hashmap.cpp — C++ implementation ═══════════
#include "hashmap.h"
#include <unordered_map>
#include <string>

// Internal C++ type — never exposed to C
struct HashMap {
    std::unordered_map<std::string, int> data;
};

extern "C" {

HashMap* hashmap_create(void) {
    try { return new HashMap{}; }
    catch (...) { return nullptr; }
}

void hashmap_destroy(HashMap* map) {
    delete map;
}

bool hashmap_insert(HashMap* map, const char* key, int value) {
    try {
        map->data[key] = value;
        return true;
    } catch (...) { return false; }
}

bool hashmap_remove(HashMap* map, const char* key) {
    return map->data.erase(key) > 0;
}

bool hashmap_contains(HashMap* map, const char* key) {
    return map->data.count(key) > 0;
}

int hashmap_get(HashMap* map, const char* key, int default_value) {
    auto it = map->data.find(key);
    return it != map->data.end() ? it->second : default_value;
}

size_t hashmap_size(const HashMap* map) {
    return map->data.size();
}

void hashmap_for_each(const HashMap* map, hashmap_visitor fn, void* user_data) {
    for (const auto& [key, value] : map->data) {
        fn(key.c_str(), value, user_data);
    }
}

}  // extern "C"
```

From the C consumer's perspective, this looks and feels exactly like a C library. The `void* user_data` parameter in the visitor callback is the C equivalent of a closure - it lets the caller thread state through to the callback without global variables.

```c
// ═══════════ main.c — Pure C consumer ═══════════
#include "hashmap.h"
#include <stdio.h>

void print_entry(const char* key, int value, void* data) {
    (void)data;
    printf("  %s = %d\n", key, value);
}

int main(void) {
    HashMap* map = hashmap_create();

    hashmap_insert(map, "apple", 3);
    hashmap_insert(map, "banana", 5);
    hashmap_insert(map, "cherry", 2);

    printf("Size: %zu\n", hashmap_size(map));
    printf("apple: %d\n", hashmap_get(map, "apple", -1));
    printf("missing: %d\n", hashmap_get(map, "missing", -1));

    printf("All entries:\n");
    hashmap_for_each(map, print_entry, NULL);

    hashmap_destroy(map);
    return 0;
}
/* Output:
   Size: 3
   apple: 3
   missing: -1
   All entries:
     cherry = 2
     banana = 5
     apple = 3
*/
```

---

## Notes

- Use `void` in empty parameter lists for C compatibility: `create(void)` not `create()` - in C, `create()` means "takes unspecified arguments," while `create(void)` means "takes no arguments."
- Use `<stddef.h>` not `<cstddef>`, and `<stdbool.h>` not `<stdbool>` - the C-style headers work in both languages, while the `c`-prefixed C++ versions are not valid C.
- The opaque handle + `extern "C"` functions pattern is the industry-standard way to wrap C++ classes for C consumption.
- Callback-based iteration with `void* user_data` is the C equivalent of passing a lambda - it lets callers thread arbitrary context into the callback without global variables.
- On Windows, add `__declspec(dllexport)` for DLL exports; on Linux, use `__attribute__((visibility("default")))` to control which symbols are exported from a shared library.
- `extern "C"` applies only to linkage - you can freely use C++ features (exceptions, STL, RAII) inside the function bodies.
