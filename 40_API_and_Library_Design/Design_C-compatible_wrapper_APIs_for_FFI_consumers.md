# Design C-compatible wrapper APIs for FFI consumers

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cpl-c-style-programming>  

---

## Topic Overview

If you want your C++ library to be callable from Python, Rust, C#, Go, or any other language, you need a C ABI boundary. The C ABI is the universal language that foreign function interfaces (FFIs) speak. That means `extern "C"` functions with no C++ types in their signatures - no `std::string`, no exceptions, no RAII handles, no references. Just pointers, integers, and C structs.

This is a topic where the rules are strict and the consequences of breaking them are severe (typically a crash or silent memory corruption), so it pays to understand the pattern thoroughly before reaching for it.

### The Wrapper Pattern

The standard approach is to write a C header (`mylib.h`) that an FFI consumer can read directly, and a separate C++ file that implements those C functions by delegating to your internal C++ classes. Opaque pointer handles stand in for C++ objects. Error codes stand in for exceptions.

Here is a complete example of a connection library with a C-compatible interface:

```cpp
// mylib.h - C header consumed by FFI users
#ifndef MYLIB_H
#define MYLIB_H

#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct mylib_connection mylib_connection;  // Opaque handle

typedef enum {
    MYLIB_OK = 0,
    MYLIB_ERR_INVALID_ARG = 1,
    MYLIB_ERR_CONNECT = 2,
    MYLIB_ERR_TIMEOUT = 3,
} mylib_error;

// Lifecycle
mylib_error mylib_connection_create(const char* host, uint16_t port,
                                     mylib_connection** out);
void mylib_connection_destroy(mylib_connection* conn);

// Operations
mylib_error mylib_connection_send(mylib_connection* conn,
                                   const uint8_t* data, size_t len);
mylib_error mylib_connection_recv(mylib_connection* conn,
                                   uint8_t* buf, size_t buflen, size_t* received);

// Version
const char* mylib_version(void);

#ifdef __cplusplus
}
#endif
#endif

// mylib_wrapper.cpp - C++ implementation
#include "mylib.h"
#include "connection.hpp"  // Internal C++ class

struct mylib_connection {
    mylib::Connection impl;
};

extern "C" {

mylib_error mylib_connection_create(const char* host, uint16_t port,
                                     mylib_connection** out) {
    if (!host || !out) return MYLIB_ERR_INVALID_ARG;
    try {
        auto* conn = new mylib_connection{mylib::Connection(host, port)};
        *out = conn;
        return MYLIB_OK;
    } catch (const std::exception&) {
        return MYLIB_ERR_CONNECT;
    }
}

void mylib_connection_destroy(mylib_connection* conn) {
    delete conn;
}

const char* mylib_version(void) {
    return "1.0.0";
}

}
```

A few things to notice here. The `#ifdef __cplusplus` guard in the header means the same file works as both a C header and a C++ header. The `mylib_connection` struct is declared but never defined in the header - that is the opaque pointer trick. The `try/catch` in `mylib_connection_create` is mandatory; no exception can be allowed to escape through the `extern "C"` boundary.

---

## Self-Assessment

### Q1: Why use opaque pointers instead of exposing struct layout

Opaque pointers (`typedef struct X X;` without a body definition) hide the C++ internal layout from FFI consumers entirely. This gives you ABI stability - you can change the internal struct freely, add members, reorganize it, even swap out the underlying C++ class, without breaking binary compatibility with existing callers. It also prevents FFI users from accessing or accidentally corrupting internal members by poking at raw memory.

### Q2: Why must exceptions never cross the extern "C" boundary

The C ABI has no exception mechanism whatsoever. When a C++ exception is thrown, the runtime unwinds the stack using C++ RTTI and metadata that C code knows nothing about. Letting a C++ exception propagate through a C stack frame - or into Python's or Rust's runtime - causes undefined behavior, which in practice almost always means a crash or a corrupted process state. Every single `extern "C"` function must wrap its body in `try { ... } catch (...) { return ERROR_CODE; }` without exception.

### Q3: Show how Python calls this API via ctypes

Once you have a clean C interface, any FFI can use it. Here is Python doing it with `ctypes`. The key step is telling ctypes the argument and return types so it can marshal values correctly:

```python
import ctypes

lib = ctypes.CDLL("./libmylib.so")

# Define argument/return types
lib.mylib_connection_create.argtypes = [ctypes.c_char_p, ctypes.c_uint16,
                                         ctypes.POINTER(ctypes.c_void_p)]
lib.mylib_connection_create.restype = ctypes.c_int

conn = ctypes.c_void_p()
rc = lib.mylib_connection_create(b"localhost", 8080, ctypes.byref(conn))
if rc != 0:
    print(f"Error: {rc}")
#
lib.mylib_connection_destroy(conn)
```

Notice that the Python side treats the connection object as a plain `c_void_p` - it has no idea what is inside. That is the opaque pointer doing its job on the other side of the boundary.

---

## Notes

- Follow a consistent naming convention: `libname_type_verb` (e.g., `mylib_connection_create`). This makes the API scannable and avoids name collisions in the global C namespace.
- Always provide a `_destroy` function for every `_create` - the caller owns the object and must be able to release it.
- Return error codes, never throw across the boundary. This is non-negotiable.
- Strings must be null-terminated C strings (`const char*`). Do not pass `std::string` or any C++ type.
- For buffer output, use the `(buf, buflen, actual_len)` triple pattern - the caller provides the buffer and its size, and you write the actual bytes written into `actual_len`.
