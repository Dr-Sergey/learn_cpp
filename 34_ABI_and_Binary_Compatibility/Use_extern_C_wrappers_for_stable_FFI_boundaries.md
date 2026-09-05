# Use extern "C" Wrappers for Stable FFI Boundaries

**Category:** ABI and Binary Compatibility  
**Standard:** C++11 and later  
**Reference:** https://en.cppreference.com/w/cpp/language/language_linkage  

---

## Topic Overview

`extern "C"` is the fundamental mechanism for creating stable Foreign Function Interface (FFI) boundaries in C++. It disables C++ name mangling, uses C calling conventions, and produces symbols that any language can call - Python (via ctypes/cffi), Rust (via FFI), Java (via JNI), C#, Go, and of course C itself. For any library intended to be consumed across language boundaries or across different C++ compilers, an `extern "C"` wrapper layer is essential.

The C ABI is the **universal interop standard** of systems programming. Unlike C++ ABI (which varies between compilers and even compiler versions), the C ABI on a given platform is stable, well-documented, and universally supported. Every operating system API (POSIX, Win32, Linux syscalls) uses C ABI. Building your library's public interface on top of C ABI guarantees maximum compatibility.

| Feature                    | C++ ABI                      | C ABI (extern "C")          |
| --- | --- | --- |
| Name mangling             | Compiler-specific            | No mangling (or minimal)    |
| Function overloading      | Supported (encoded in name)  | Not supported               |
| Exceptions                | Compiler-specific            | Must not cross boundary     |
| Classes/methods           | Encoded in mangled name      | No direct support           |
| Template instantiations   | Mangled with type params     | Must be wrapped manually    |
| Cross-compiler compat     | Not guaranteed               | Guaranteed on same platform |
| Cross-language compat     | Almost impossible             | Universal                   |

Here are the non-negotiable constraints you have to work within at an `extern "C"` boundary. Each one exists for a concrete reason:

1. **No name mangling** - function names are literal symbols.
2. **No overloading** - each function name must be unique.
3. **No exceptions across boundary** - must catch and convert to error codes.
4. **No C++ types in signatures** - use opaque pointers, plain C types, or POD structs.
5. **No RTTI or virtual dispatch** - handled internally, exposed as function pointers if needed.

---

## Self-Assessment

### Q1: Design a complete extern "C" wrapper for a C++ library with opaque handle types and proper lifecycle management

The pattern here is: keep all C++ objects completely hidden behind an opaque `typedef struct X* handle` type. Callers never see the real type, so you can change the internals without touching the public API. Every function that could throw must wrap in a try/catch and return an error code - nothing escapes. Let's walk through a database connection library as the example.

```cpp
// === database.h (C++ internal implementation) ===
#include <string>
#include <vector>
#include <memory>
#include <stdexcept>
#include <cstdio>

namespace db {

class Connection {
public:
    explicit Connection(const std::string& connstr)
        : connstr_(connstr), connected_(false) {}

    void connect() {
        // Simulate connection
        connected_ = true;
    }

    void disconnect() { connected_ = false; }
    bool is_connected() const { return connected_; }

    std::vector<std::string> query(const std::string& sql) {
        if (!connected_) throw std::runtime_error("not connected");
        // Simplified - real implementation would return rows
        return {"row1", "row2", "row3"};
    }

    int execute(const std::string& sql) {
        if (!connected_) throw std::runtime_error("not connected");
        return 1;  // rows affected
    }

private:
    std::string connstr_;
    bool connected_;
};

}  // namespace db


// === database_c_api.h (PUBLIC C HEADER - ships to all FFI consumers) ===
#ifndef DATABASE_C_API_H
#define DATABASE_C_API_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

// Opaque handle - hides C++ internals completely
typedef struct db_connection_t* db_connection_handle;

// Error codes - stable, well-defined contract
typedef enum {
    DB_OK              = 0,
    DB_ERR_NULL_PARAM  = -1,
    DB_ERR_NOT_CONNECTED = -2,
    DB_ERR_QUERY_FAILED  = -3,
    DB_ERR_OUT_OF_MEMORY = -4,
    DB_ERR_UNKNOWN       = -99
} db_error_t;

// Lifecycle management
db_error_t db_connection_create(const char* connstr, db_connection_handle* out_handle);
db_error_t db_connection_destroy(db_connection_handle handle);

// Operations
db_error_t db_connect(db_connection_handle handle);
db_error_t db_disconnect(db_connection_handle handle);
int        db_is_connected(db_connection_handle handle);

// Query - caller provides buffer, library fills it
db_error_t db_execute(db_connection_handle handle, const char* sql,
                      int* out_rows_affected);

// For result sets: iterator pattern
typedef struct db_result_t* db_result_handle;
db_error_t db_query(db_connection_handle handle, const char* sql,
                    db_result_handle* out_result);
db_error_t db_result_next(db_result_handle result, int* has_next);
db_error_t db_result_get_string(db_result_handle result, int column,
                                const char** out_value);
db_error_t db_result_destroy(db_result_handle result);

// Last error message - thread-safe via thread-local storage
const char* db_last_error();

#ifdef __cplusplus
}
#endif

#endif  // DATABASE_C_API_H


// === database_c_api.cpp (IMPLEMENTATION) ===
// #include "database_c_api.h"
// #include "database.h"
#include <cstring>

// Thread-local error message buffer
static thread_local char tls_error_buf[512] = {};

static void set_error(const char* msg) {
    std::strncpy(tls_error_buf, msg, sizeof(tls_error_buf) - 1);
    tls_error_buf[sizeof(tls_error_buf) - 1] = '\0';
}

// Hidden struct wrapping the C++ object
struct db_connection_t {
    db::Connection impl;
    explicit db_connection_t(const std::string& s) : impl(s) {}
};

struct db_result_t {
    std::vector<std::string> rows;
    size_t current_row = 0;
};

extern "C" {

db_error_t db_connection_create(const char* connstr,
                                db_connection_handle* out_handle) {
    if (!connstr || !out_handle) return DB_ERR_NULL_PARAM;
    try {
        *out_handle = new db_connection_t(connstr);
        return DB_OK;
    } catch (const std::bad_alloc&) {
        set_error("out of memory");
        return DB_ERR_OUT_OF_MEMORY;
    } catch (const std::exception& e) {
        set_error(e.what());
        return DB_ERR_UNKNOWN;
    }
}

db_error_t db_connection_destroy(db_connection_handle handle) {
    delete handle;  // safe even if null
    return DB_OK;
}

db_error_t db_connect(db_connection_handle handle) {
    if (!handle) return DB_ERR_NULL_PARAM;
    try {
        handle->impl.connect();
        return DB_OK;
    } catch (const std::exception& e) {
        set_error(e.what());
        return DB_ERR_UNKNOWN;
    }
}

db_error_t db_disconnect(db_connection_handle handle) {
    if (!handle) return DB_ERR_NULL_PARAM;
    handle->impl.disconnect();
    return DB_OK;
}

int db_is_connected(db_connection_handle handle) {
    if (!handle) return 0;
    return handle->impl.is_connected() ? 1 : 0;
}

db_error_t db_execute(db_connection_handle handle, const char* sql,
                      int* out_rows_affected) {
    if (!handle || !sql) return DB_ERR_NULL_PARAM;
    try {
        int rows = handle->impl.execute(sql);
        if (out_rows_affected) *out_rows_affected = rows;
        return DB_OK;
    } catch (const std::runtime_error& e) {
        set_error(e.what());
        return DB_ERR_NOT_CONNECTED;
    } catch (const std::exception& e) {
        set_error(e.what());
        return DB_ERR_UNKNOWN;
    }
}

db_error_t db_query(db_connection_handle handle, const char* sql,
                    db_result_handle* out_result) {
    if (!handle || !sql || !out_result) return DB_ERR_NULL_PARAM;
    try {
        auto result = new db_result_t();
        result->rows = handle->impl.query(sql);
        *out_result = result;
        return DB_OK;
    } catch (const std::exception& e) {
        set_error(e.what());
        return DB_ERR_QUERY_FAILED;
    }
}

db_error_t db_result_next(db_result_handle result, int* has_next) {
    if (!result || !has_next) return DB_ERR_NULL_PARAM;
    *has_next = (result->current_row < result->rows.size()) ? 1 : 0;
    if (*has_next) result->current_row++;
    return DB_OK;
}

db_error_t db_result_get_string(db_result_handle result, int column,
                                const char** out_value) {
    if (!result || !out_value) return DB_ERR_NULL_PARAM;
    if (result->current_row == 0 || result->current_row > result->rows.size())
        return DB_ERR_QUERY_FAILED;
    *out_value = result->rows[result->current_row - 1].c_str();
    return DB_OK;
}

db_error_t db_result_destroy(db_result_handle result) {
    delete result;
    return DB_OK;
}

const char* db_last_error() {
    return tls_error_buf;
}

}  // extern "C"
```

Notice that every fallible function ends in a `catch (const std::exception& e)` net. A C caller has no exception handling machinery - if a C++ exception propagates through an `extern "C"` frame the behavior is undefined on every platform, and in practice it usually crashes the process. The thread-local error buffer gives callers a way to get a human-readable message after any failure, which mirrors the `errno`/`strerror` pattern from the C standard library.

### Q2: Implement callback patterns across FFI boundaries with proper exception safety and lifetime management

Passing callbacks across language boundaries adds another layer of complexity: the callback is foreign code that could do anything, including re-entering your library. The safe approach is to copy the subscriber list under a lock, release the lock, then invoke the callbacks without holding the lock. That way a callback that unregisters itself (or registers a new callback) doesn't deadlock.

```cpp
#include <cstdio>
#include <cstring>
#include <functional>
#include <mutex>
#include <vector>

// === C API for event system with callbacks ===

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    EVENT_CONNECT    = 1,
    EVENT_DISCONNECT = 2,
    EVENT_DATA       = 3,
    EVENT_ERROR      = 4
} event_type_t;

typedef struct {
    event_type_t type;
    const void*  data;
    int          data_len;
    int          error_code;
} event_t;

// Callback signature: return 0 to continue, non-zero to unregister
typedef int (*event_callback_fn)(const event_t* event, void* user_data);

typedef struct event_system_t* event_system_handle;

int  event_system_create(event_system_handle* out);
void event_system_destroy(event_system_handle sys);

// Register callback with user_data pointer for context
int  event_system_register(event_system_handle sys,
                           event_type_t type,
                           event_callback_fn callback,
                           void* user_data);

int  event_system_fire(event_system_handle sys, const event_t* event);

#ifdef __cplusplus
}
#endif


// === Implementation ===

struct Subscription {
    event_type_t type;
    event_callback_fn callback;
    void* user_data;
};

struct event_system_t {
    std::mutex mtx;
    std::vector<Subscription> subs;
};

extern "C" {

int event_system_create(event_system_handle* out) {
    if (!out) return -1;
    try {
        *out = new event_system_t();
        return 0;
    } catch (...) {
        return -1;
    }
}

void event_system_destroy(event_system_handle sys) {
    delete sys;
}

int event_system_register(event_system_handle sys, event_type_t type,
                          event_callback_fn callback, void* user_data) {
    if (!sys || !callback) return -1;
    try {
        std::lock_guard<std::mutex> lock(sys->mtx);
        sys->subs.push_back({type, callback, user_data});
        return 0;
    } catch (...) {
        return -1;
    }
}

int event_system_fire(event_system_handle sys, const event_t* event) {
    if (!sys || !event) return -1;

    // Copy subscribers under lock, then invoke without lock
    // (callbacks might re-enter the event system)
    std::vector<Subscription> snapshot;
    {
        std::lock_guard<std::mutex> lock(sys->mtx);
        snapshot = sys->subs;
    }

    std::vector<size_t> to_remove;
    for (size_t i = 0; i < snapshot.size(); ++i) {
        if (snapshot[i].type == event->type) {
            // CRITICAL: callback is foreign code - must not let
            // exceptions propagate from C callback
            int result = snapshot[i].callback(event, snapshot[i].user_data);
            if (result != 0) {
                to_remove.push_back(i);
            }
        }
    }

    // Remove unsubscribed callbacks
    if (!to_remove.empty()) {
        std::lock_guard<std::mutex> lock(sys->mtx);
        for (auto it = to_remove.rbegin(); it != to_remove.rend(); ++it) {
            if (*it < sys->subs.size()) {
                sys->subs.erase(sys->subs.begin()
                    + static_cast<std::ptrdiff_t>(*it));
            }
        }
    }

    return 0;
}

}  // extern "C"


// === Usage example (from C or any FFI consumer) ===

int my_handler(const event_t* event, void* user_data) {
    const char* name = static_cast<const char*>(user_data);
    std::printf("[%s] Event type=%d, data_len=%d\n",
                name, event->type, event->data_len);
    return 0;  // keep subscribed
}

int main() {
    event_system_handle sys = nullptr;
    event_system_create(&sys);

    const char* ctx = "MyApp";
    event_system_register(sys, EVENT_DATA, my_handler,
                          const_cast<char*>(ctx));

    event_t ev{EVENT_DATA, "hello", 5, 0};
    event_system_fire(sys, &ev);

    event_system_destroy(sys);
    return 0;
}
```

The `void* user_data` parameter on the callback is mandatory in a well-designed C API. Without it, the caller has no way to pass context to the callback without global state - and global state makes thread safety a nightmare.

### Q3: Handle thread safety at FFI boundaries and document the threading contract for your C API

Documenting the threading model is just as important as implementing it. A C consumer in Python or Rust cannot inspect your header to figure out which functions are thread-safe - you have to tell them explicitly. The pattern below uses a reader-writer lock for the cache data and atomic counters for the statistics, giving you concurrent reads without sacrificing write correctness.

```cpp
#include <cstdio>
#include <atomic>
#include <mutex>
#include <shared_mutex>

// === Thread-safety documentation contract ===
//
// Thread Safety Guarantees:
//
// | Function                  | Thread Safety       | Notes                    |
// |--------------------------|---------------------|--------------------------|
// | cache_create()           | NOT thread-safe     | Call once during init     |
// | cache_destroy()          | NOT thread-safe     | Call once during shutdown |
// | cache_get()              | Thread-safe (shared)| Multiple concurrent reads |
// | cache_put()              | Thread-safe (excl)  | Serialized with reads     |
// | cache_remove()           | Thread-safe (excl)  | Serialized with reads     |
// | cache_stats()            | Thread-safe (atomic)| Lock-free                |

#ifdef __cplusplus
extern "C" {
#endif

typedef struct cache_t* cache_handle;

// NOT thread-safe - call from single thread during initialization
int  cache_create(int max_entries, cache_handle* out);
void cache_destroy(cache_handle c);

// Thread-safe operations
int  cache_get(cache_handle c, const char* key,
               char* value_buf, int buf_size);
int  cache_put(cache_handle c, const char* key, const char* value);
int  cache_remove(cache_handle c, const char* key);

typedef struct {
    int64_t hits;
    int64_t misses;
    int64_t evictions;
    int     current_size;
} cache_stats_t;

// Lock-free statistics query
int cache_stats(cache_handle c, cache_stats_t* out);

#ifdef __cplusplus
}
#endif

// === Implementation with proper synchronization ===

#include <unordered_map>
#include <string>

struct cache_t {
    std::shared_mutex rw_mutex;             // reader-writer lock
    std::unordered_map<std::string, std::string> data;
    int max_entries;

    // Atomic counters for lock-free stats
    std::atomic<int64_t> hits{0};
    std::atomic<int64_t> misses{0};
    std::atomic<int64_t> evictions{0};
};

extern "C" {

int cache_create(int max_entries, cache_handle* out) {
    if (!out || max_entries <= 0) return -1;
    try {
        auto c = new cache_t();
        c->max_entries = max_entries;
        *out = c;
        return 0;
    } catch (...) {
        return -1;
    }
}

void cache_destroy(cache_handle c) {
    delete c;
}

int cache_get(cache_handle c, const char* key,
              char* value_buf, int buf_size) {
    if (!c || !key || !value_buf || buf_size <= 0) return -1;

    std::shared_lock lock(c->rw_mutex);  // shared read lock
    auto it = c->data.find(key);
    if (it == c->data.end()) {
        c->misses.fetch_add(1, std::memory_order_relaxed);
        return -2;  // not found
    }

    c->hits.fetch_add(1, std::memory_order_relaxed);
    int copy_len = static_cast<int>(it->second.size());
    if (copy_len >= buf_size) copy_len = buf_size - 1;
    std::memcpy(value_buf, it->second.c_str(), copy_len);
    value_buf[copy_len] = '\0';
    return copy_len;
}

int cache_put(cache_handle c, const char* key, const char* value) {
    if (!c || !key || !value) return -1;

    std::unique_lock lock(c->rw_mutex);  // exclusive write lock

    if (static_cast<int>(c->data.size()) >= c->max_entries
        && c->data.find(key) == c->data.end()) {
        // Simple eviction: remove first entry
        auto first = c->data.begin();
        if (first != c->data.end()) {
            c->data.erase(first);
            c->evictions.fetch_add(1, std::memory_order_relaxed);
        }
    }

    c->data[key] = value;
    return 0;
}

int cache_remove(cache_handle c, const char* key) {
    if (!c || !key) return -1;
    std::unique_lock lock(c->rw_mutex);
    return c->data.erase(key) > 0 ? 0 : -2;
}

int cache_stats(cache_handle c, cache_stats_t* out) {
    if (!c || !out) return -1;
    // All reads are atomic - no lock needed
    out->hits = c->hits.load(std::memory_order_relaxed);
    out->misses = c->misses.load(std::memory_order_relaxed);
    out->evictions = c->evictions.load(std::memory_order_relaxed);
    // current_size needs lock for consistency
    std::shared_lock lock(c->rw_mutex);
    out->current_size = static_cast<int>(c->data.size());
    return 0;
}

}  // extern "C"

int main() {
    cache_handle c = nullptr;
    cache_create(100, &c);

    cache_put(c, "key1", "value1");
    cache_put(c, "key2", "value2");

    char buf[64];
    if (cache_get(c, "key1", buf, sizeof(buf)) >= 0) {
        std::printf("Got: %s\n", buf);
    }

    cache_stats_t stats;
    cache_stats(c, &stats);
    std::printf("Hits: %lld, Misses: %lld\n",
                static_cast<long long>(stats.hits),
                static_cast<long long>(stats.misses));

    cache_destroy(c);
    return 0;
}
```

The `cache_stats` function is worth noting: the hit/miss/eviction counters use `memory_order_relaxed` atomics because slightly stale statistics are acceptable, but `current_size` takes a shared lock because an inconsistent size count would be actively misleading.

---

## Notes

- **Never let C++ exceptions cross `extern "C"` boundaries** - this is undefined behavior on every platform. Catch all exceptions and convert to error codes.
- Use **opaque handles** (`typedef struct X* handle_type`) instead of `void*` - this gives type safety even in C and prevents accidentally passing wrong handle types.
- **Thread-local storage** for error messages (`thread_local char[]`) is the standard pattern for providing detailed error information without thread-safety issues (see `errno`, `strerror`).
- Callbacks across FFI must use **C calling convention** (`__cdecl` on Windows) - never pass `std::function` or lambda captures across the boundary.
- Always include a **void* user_data** parameter in callback signatures - this allows callers to pass context without global state.
- Document **thread-safety guarantees** for every function in your C API - consumers in other languages cannot inspect your implementation.
- The **create/destroy** pattern maps naturally to RAII in C++ consumers and resource management in any other language.
