# Use std::out_ptr and std::inout_ptr (C++23) for Smart Pointer Interop with C APIs

**Category:** Memory and Ownership  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/memory/out_ptr_t>  

---

## Topic Overview

### The Problem: C APIs That Allocate via `T**`

Many C APIs allocate resources and return them through an output pointer parameter. That is a common pattern: you hand the API a pointer-to-pointer, and it fills in the actual address. The interface typically looks like this:

```c
// C-style API: creates a resource, stores pointer in *out
int create_handle(Handle** out);   // Allocate
int resize_buffer(Buffer** inout); // Reallocate
void destroy_handle(Handle* h);    // Free
```

Before C++23, bridging these with smart pointers was manual and error-prone.

### `std::out_ptr` and `std::inout_ptr` (C++23)

C++23 gives you two adapters that do the release-call-rewrap dance automatically. Here is what each one is for:

| Adapter | Use Case | On construction | On destruction |
| --- | --- | --- | --- |
| `out_ptr(sp)` | C func that **allocates** new resource | Releases current (if any) | Stores new pointer into smart ptr |
| `inout_ptr(sp)` | C func that **reallocates** existing | Releases + passes old ptr | Stores new pointer into smart ptr |

With these adapters the call site becomes a single, exception-safe expression:

```cpp
// C++23 - clean, exception-safe
std::unique_ptr<Handle, HandleDeleter> h;
create_handle(std::out_ptr(h));       // h now owns the new handle
resize_buffer(std::inout_ptr(h));     // h frees old, takes new
```

---

## Self-Assessment

### Q1: Use `std::out_ptr(up)` to pass the address of a `unique_ptr` to a C function that allocates via `void**`

Here we simulate a C API that allocates a resource through an output pointer, then show how `std::out_ptr` wraps the smart pointer cleanly. The `#else` branch shows the equivalent manual steps for pre-C++23 code so you can see exactly what the adapter replaces.

```cpp
#include <iostream>
#include <memory>
#include <cstdlib>
#include <cstring>

// Simulate a C API that allocates a resource
struct CResource {
    int id;
    char name[32];
};

extern "C" {
    // C API: allocates and returns via output pointer
    int c_create_resource(CResource** out, int id, const char* name) {
        *out = static_cast<CResource*>(std::malloc(sizeof(CResource)));
        if (!*out) return -1;
        (*out)->id = id;
        std::strncpy((*out)->name, name, 31);
        (*out)->name[31] = '\0';
        return 0;  // success
    }

    void c_destroy_resource(CResource* r) {
        std::cout << "  C API: freeing resource " << r->id << "\n";
        std::free(r);
    }
}

// Custom deleter for the C resource
struct CResourceDeleter {
    void operator()(CResource* r) const {
        if (r) c_destroy_resource(r);
    }
};

using ResourcePtr = std::unique_ptr<CResource, CResourceDeleter>;

int main() {
    std::cout << "=== std::out_ptr with C API ===\n\n";

#if __cplusplus >= 202302L
    // C++23: clean and safe
    ResourcePtr res;
    int err = c_create_resource(std::out_ptr(res), 42, "Sensor");
    if (err == 0) {
        std::cout << "Created: id=" << res->id << " name=" << res->name << "\n";
    }
    // res automatically freed via CResourceDeleter
#else
    // Pre-C++23: manual release/re-wrap pattern
    std::cout << "Pre-C++23 manual pattern:\n";
    CResource* raw = nullptr;
    int err = c_create_resource(&raw, 42, "Sensor");
    ResourcePtr res(raw);  // Wrap immediately
    if (err == 0) {
        std::cout << "Created: id=" << res->id << " name=" << res->name << "\n";
    }

    // Simulating what std::out_ptr does internally:
    // 1. Releases any existing resource
    // 2. Provides T** to the C function
    // 3. On scope exit, wraps the new pointer into the smart ptr
    std::cout << "\nManual out_ptr simulation:\n";
    ResourcePtr res2;
    {
        CResource* temp = nullptr;
        c_create_resource(&temp, 99, "Camera");
        res2.reset(temp);  // Manual equivalent of out_ptr
    }
    std::cout << "Created: id=" << res2->id << " name=" << res2->name << "\n";
#endif

    std::cout << "\nDestruction:\n";
    return 0;
}
```

When the scope ends, `res` goes out of scope and `CResourceDeleter` calls `c_destroy_resource` automatically - no manual cleanup needed.

### Q2: Show that `std::inout_ptr` handles functions that may reallocate an existing resource

`inout_ptr` is for the trickier case: the C function takes the old pointer, frees it itself, then writes a new pointer. The adapter has to release ownership from the smart pointer before the call so the C API can take over, then rewrap the new result afterward. Watch how this compares to the manual pre-C++23 approach in the `#else` branch.

```cpp
#include <iostream>
#include <memory>
#include <cstdlib>
#include <cstring>

struct Buffer {
    char* data;
    size_t size;
};

extern "C" {
    int c_create_buffer(Buffer** out, size_t sz) {
        *out = static_cast<Buffer*>(std::malloc(sizeof(Buffer)));
        (*out)->data = static_cast<char*>(std::malloc(sz));
        (*out)->size = sz;
        std::memset((*out)->data, 0, sz);
        std::cout << "  C API: created buffer size=" << sz << "\n";
        return 0;
    }

    // Reallocates: frees old buffer, creates new one
    int c_resize_buffer(Buffer** inout, size_t new_sz) {
        if (*inout) {
            std::cout << "  C API: freeing old buffer size=" << (*inout)->size << "\n";
            std::free((*inout)->data);
            std::free(*inout);
        }
        *inout = static_cast<Buffer*>(std::malloc(sizeof(Buffer)));
        (*inout)->data = static_cast<char*>(std::malloc(new_sz));
        (*inout)->size = new_sz;
        std::memset((*inout)->data, 0, new_sz);
        std::cout << "  C API: created new buffer size=" << new_sz << "\n";
        return 0;
    }

    void c_destroy_buffer(Buffer* b) {
        if (b) {
            std::cout << "  C API: destroying buffer size=" << b->size << "\n";
            std::free(b->data);
            std::free(b);
        }
    }
}

struct BufferDeleter {
    void operator()(Buffer* b) const { c_destroy_buffer(b); }
};

using BufPtr = std::unique_ptr<Buffer, BufferDeleter>;

int main() {
    std::cout << "=== std::inout_ptr for reallocation ===\n\n";

#if __cplusplus >= 202302L
    BufPtr buf;
    c_create_buffer(std::out_ptr(buf), 100);

    // inout_ptr: releases old resource, passes to C, wraps new resource
    c_resize_buffer(std::inout_ptr(buf), 500);
#else
    // Pre-C++23 manual equivalent
    std::cout << "Pre-C++23 manual inout_ptr simulation:\n\n";

    BufPtr buf;
    {
        Buffer* raw = nullptr;
        c_create_buffer(&raw, 100);
        buf.reset(raw);
    }
    std::cout << "Buffer size: " << buf->size << "\n\n";

    // Manual inout_ptr: release + call + re-wrap
    {
        Buffer* raw = buf.release();  // Release ownership
        c_resize_buffer(&raw, 500);   // C API reallocates
        buf.reset(raw);               // Re-wrap new pointer
    }
    std::cout << "Buffer size after resize: " << buf->size << "\n";
#endif

    std::cout << "\nDestruction:\n";
    return 0;
}
```

The three-step manual pattern - `release`, call, `reset` - is exactly what `inout_ptr` encapsulates, and it does so in a way that stays safe even if the C function throws (or if some cleanup between the steps would throw).

### Q3: Explain why manually releasing and re-wrapping is error-prone compared to these adapters

This example catalogs the specific bugs that appear when you try to do the `out_ptr`/`inout_ptr` job by hand. Each comment block isolates one failure mode so the pattern is easy to recognize in real code.

```cpp
#include <iostream>
#include <memory>
#include <cstdlib>

struct Handle { int id; };

void destroy(Handle* h) { std::free(h); }
int create(Handle** out) {
    *out = static_cast<Handle*>(std::malloc(sizeof(Handle)));
    (*out)->id = 1;
    return 0;
}

using HPtr = std::unique_ptr<Handle, decltype(&destroy)>;

int main() {
    std::cout << "=== Error-prone manual patterns ===\n\n";

    // Bug 1: Forgot to wrap the raw pointer
    {
        Handle* raw = nullptr;
        create(&raw);
        // Oops - forgot to wrap in unique_ptr
        // raw leaks if we return or throw here!
        std::free(raw);  // Manual cleanup - brittle
    }

    // Bug 2: Exception between release and reset
    {
        HPtr h(nullptr, destroy);
        Handle* raw = nullptr;
        create(&raw);
        h.reset(raw);

        // Now "resize" - manual release + re-wrap
        raw = h.release();  // h no longer owns it
        // If the next line throws, raw leaks!
        // some_c_function_that_might_fail(&raw);  // THROWS -> leak!
        h.reset(raw);
    }

    // Bug 3: Double-free if you forget to release before reset
    {
        HPtr h(nullptr, destroy);
        Handle* raw = nullptr;
        create(&raw);
        h.reset(raw);

        // Wrong: didn't release before passing to C API
        // create(&raw);     // Creates new handle
        // h.reset(raw);     // Old handle already freed by reset? No - leaked!
    }

    std::cout << "=== How out_ptr/inout_ptr fix these ===\n\n";
    std::cout << "out_ptr:    Atomically: release old -> get T** -> wrap new\n";
    std::cout << "inout_ptr:  Atomically: release old -> pass old to C -> wrap new\n";
    std::cout << "\nBenefits:\n";
    std::cout << "1. No naked pointer window (exception-safe)\n";
    std::cout << "2. Can't forget to wrap (automatic)\n";
    std::cout << "3. Can't double-free (release is part of the operation)\n";
    std::cout << "4. Single expression instead of 3 separate steps\n";
    std::cout << "5. Works with unique_ptr AND shared_ptr\n";

    return 0;
}
```

The core issue is that the manual approach has a window between `release()` and `reset()` where the raw pointer is owned by nobody. If anything goes wrong in that window the resource leaks. The adapters eliminate that window entirely.

---

## Notes

- `std::out_ptr` (C++23): wraps smart pointer for C APIs that allocate via `T**` output parameters.
- `std::inout_ptr` (C++23): handles C APIs that reallocate - releases old resource and wraps new one.
- Both are exception-safe - no naked pointer window where a throw could cause a leak.
- Works with `unique_ptr`, `shared_ptr`, and any type supporting `release()`/`reset()`.
- Before C++23, use the manual release-call-reset pattern, being careful about exception safety.
