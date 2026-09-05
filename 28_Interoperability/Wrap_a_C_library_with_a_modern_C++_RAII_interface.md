# Wrap a C library with a modern C++ RAII interface

**Category:** Interoperability  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unique_ptr>  

---

## Topic Overview

C libraries expose resources through paired `create`/`destroy` functions and return error codes rather than throwing exceptions. Wrapping them in C++ RAII classes gives you automatic cleanup, exception safety, and a modern API that works with standard algorithms, ranges, and `std::span`.

The underlying pattern is always the same: you take C's imperative "allocate, use, free" and turn it into a C++ object whose constructor does the "allocate", whose methods do the "use", and whose destructor handles the "free" automatically - even when exceptions are involved.

### RAII Wrapper Strategy

Here is the mental model for mapping a C API to a C++ wrapper. Each C function gets a C++ counterpart that replaces raw pointers with smart pointers and error codes with exceptions.

```cpp
C library API                 C++ RAII wrapper
────────────────────          ────────────────────────────
EVP_MD_CTX_new()    ──►      unique_ptr<EVP_MD_CTX, Deleter>
EVP_MD_CTX_free()   ──►      (automatic in destructor)
EVP_DigestInit()    ──►      DigestContext::DigestContext(algo)
EVP_DigestUpdate()  ──►      DigestContext::update(span<const byte>)
EVP_DigestFinal()   ──►      DigestContext::finalize() -> vector<byte>
Error codes         ──►      Exceptions (std::system_error)
int* + size_t       ──►      std::span<int>
```

### Wrapper Patterns Cheat Sheet

This table covers the most common C patterns and their idiomatic C++ replacements.

| C Pattern | C++ RAII Pattern |
| --- | --- |
| `T* create() / destroy(T*)` | `unique_ptr<T, void(*)(T*)>` |
| `int func(...) -> error code` | Throw `std::system_error` on failure |
| `void* opaque handle` | Class with private `unique_ptr<void, Deleter>` |
| `T* buf + size_t len` | `std::span<T>` parameter |
| `callback + void* user_data` | `std::function` or lambda stored in class |
| `global init/shutdown` | Static count + constructor/destructor |

---

## Self-Assessment

### Q1: Wrap OpenSSL's EVP_MD_CTX in a unique_ptr with a custom deleter using EVP_MD_CTX_free

**Answer:**

The key design insight here is that a stateless deleter struct costs exactly zero bytes when stored in a `unique_ptr`. Contrast that with storing a `std::function` or a raw function pointer as the deleter - those add a pointer's worth of overhead per object. For wrappers that need to be zero-cost, always prefer a stateless struct.

<!-- compile: needs third-party library header `openssl/evp.h` -->
```cpp
#include <openssl/evp.h>
#include <memory>
#include <vector>
#include <span>
#include <string>
#include <stdexcept>
#include <iomanip>
#include <sstream>
#include <cstdint>

// Custom deleter for EVP_MD_CTX
struct EvpMdCtxDeleter {
    void operator()(EVP_MD_CTX* ctx) const noexcept {
        if (ctx) EVP_MD_CTX_free(ctx);
    }
};

using EvpMdCtxPtr = std::unique_ptr<EVP_MD_CTX, EvpMdCtxDeleter>;

// RAII wrapper class
class DigestContext {
    EvpMdCtxPtr ctx_;

    static EvpMdCtxPtr make_context() {
        EvpMdCtxPtr ctx(EVP_MD_CTX_new());
        if (!ctx) throw std::runtime_error("EVP_MD_CTX_new failed");
        return ctx;
    }

public:
    explicit DigestContext(const EVP_MD* algorithm)
        : ctx_(make_context())
    {
        if (EVP_DigestInit_ex(ctx_.get(), algorithm, nullptr) != 1)
            throw std::runtime_error("EVP_DigestInit_ex failed");
    }

    // Non-copyable, movable
    DigestContext(const DigestContext&) = delete;
    DigestContext& operator=(const DigestContext&) = delete;
    DigestContext(DigestContext&&) noexcept = default;
    DigestContext& operator=(DigestContext&&) noexcept = default;
    ~DigestContext() = default;  // unique_ptr handles cleanup

    void update(std::span<const uint8_t> data) {
        if (EVP_DigestUpdate(ctx_.get(), data.data(), data.size()) != 1)
            throw std::runtime_error("EVP_DigestUpdate failed");
    }

    // Convenience: accept string_view
    void update(std::string_view sv) {
        update(std::span<const uint8_t>(
            reinterpret_cast<const uint8_t*>(sv.data()), sv.size()));
    }

    std::vector<uint8_t> finalize() {
        std::vector<uint8_t> digest(EVP_MD_size(EVP_MD_CTX_get0_md(ctx_.get())));
        unsigned int len = 0;
        if (EVP_DigestFinal_ex(ctx_.get(), digest.data(), &len) != 1)
            throw std::runtime_error("EVP_DigestFinal_ex failed");
        digest.resize(len);
        return digest;
    }

    // Helper: compute hash in one call
    static std::string hex_digest(const EVP_MD* algo, std::string_view input) {
        DigestContext ctx(algo);
        ctx.update(input);
        auto digest = ctx.finalize();
        std::ostringstream oss;
        for (auto b : digest)
            oss << std::hex << std::setfill('0') << std::setw(2) << (int)b;
        return oss.str();
    }
};

// Usage:
int main() {
    // One-shot hashing
    auto sha256 = DigestContext::hex_digest(EVP_sha256(), "Hello, World!");
    // sha256 = "dffd6021bb2bd5b0af676290809ec3a53191dd81c7f70a4b28688a362182986f"

    // Incremental hashing
    DigestContext ctx(EVP_sha256());
    ctx.update("Hello, ");
    ctx.update("World!");
    auto digest = ctx.finalize();
    // Same result as one-shot

    return 0;
}
// Compile: g++ -std=c++20 -lssl -lcrypto main.cpp
```

A few design decisions here are worth calling out. The factory method `make_context()` throws before handing the raw pointer to `unique_ptr`, which means the `unique_ptr` only ever holds a valid, initialized pointer - you never have to check for null inside the class methods. And because `~DigestContext() = default`, the destructor is just `unique_ptr`'s destructor, which calls `EvpMdCtxDeleter::operator()`. No explicit cleanup code is needed anywhere.

**Key design decisions:**

- `struct EvpMdCtxDeleter` is stateless - zero overhead vs raw pointer.
- `unique_ptr` ensures `EVP_MD_CTX_free` is called even if exceptions are thrown.
- Factory method `make_context()` throws on allocation failure before storing into `unique_ptr`.

### Q2: Provide a C++ exception-throwing wrapper around C functions that return error codes

**Answer:**

C libraries that use error codes (SQLite, zlib, libcurl, etc.) follow a consistent pattern: every function returns an integer where 0 means success and negative values mean failure. The standard C++ way to wrap these is a small helper that converts the return code into an exception, and a custom `std::error_category` that carries the C library's error messages through the standard error machinery.

The reason to use `std::system_error` and `std::error_category` rather than just throwing `std::runtime_error` is that callers can inspect the error code programmatically - `e.code().value()` gives back the original C error code, which can be useful for logging or for deciding whether to retry.

```cpp
#include <cstdio>
#include <cstring>
#include <cerrno>
#include <system_error>
#include <memory>
#include <span>
#include <string>
#include <vector>

// Simulated C library (like SQLite, zlib, etc.)
extern "C" {
    typedef struct db_connection db_connection;
    typedef struct db_result db_result;

    // C API: returns 0 on success, negative error code on failure
    int db_open(const char* path, db_connection** out);
    int db_execute(db_connection* conn, const char* sql);
    int db_query(db_connection* conn, const char* sql, db_result** out);
    int db_result_row_count(db_result* res);
    const char* db_result_get(db_result* res, int row, int col);
    void db_result_free(db_result* res);
    void db_close(db_connection* conn);
    const char* db_error_message(int code);
}

// Error category for the C library
class DbErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "db"; }

    std::string message(int code) const override {
        const char* msg = db_error_message(code);
        return msg ? msg : "Unknown database error";
    }

    static const DbErrorCategory& instance() {
        static const DbErrorCategory cat;
        return cat;
    }
};

// Helper: check C return code, throw on error
inline void db_check(int code, const char* operation) {
    if (code != 0) {
        throw std::system_error(
            std::error_code(-code, DbErrorCategory::instance()),
            operation
        );
    }
}

// RAII Wrappers
struct ResultDeleter {
    void operator()(db_result* r) const noexcept { if (r) db_result_free(r); }
};

struct ConnectionDeleter {
    void operator()(db_connection* c) const noexcept { if (c) db_close(c); }
};

class Database {
    std::unique_ptr<db_connection, ConnectionDeleter> conn_;

public:
    explicit Database(const std::string& path) {
        db_connection* raw = nullptr;
        db_check(db_open(path.c_str(), &raw), "db_open");
        conn_.reset(raw);
    }

    void execute(const std::string& sql) {
        db_check(db_execute(conn_.get(), sql.c_str()), "db_execute");
    }

    class QueryResult {
        std::unique_ptr<db_result, ResultDeleter> res_;
    public:
        explicit QueryResult(db_result* r) : res_(r) {}
        int row_count() const { return db_result_row_count(res_.get()); }
        std::string get(int row, int col) const {
            const char* val = db_result_get(res_.get(), row, col);
            return val ? val : "";
        }
    };

    QueryResult query(const std::string& sql) {
        db_result* raw = nullptr;
        db_check(db_query(conn_.get(), sql.c_str(), &raw), "db_query");
        return QueryResult(raw);
    }
};

// Usage:
int main() {
    try {
        Database db("test.db");
        db.execute("CREATE TABLE IF NOT EXISTS users(id INTEGER, name TEXT)");
        db.execute("INSERT INTO users VALUES(1, 'Alice')");

        auto result = db.query("SELECT * FROM users");
        for (int i = 0; i < result.row_count(); ++i) {
            printf("Row %d: id=%s, name=%s\n",
                   i, result.get(i, 0).c_str(), result.get(i, 1).c_str());
        }
    } catch (const std::system_error& e) {
        // e.code() has the original C error code
        // e.what() has "operation: error message"
        fprintf(stderr, "Database error [%d]: %s\n",
                e.code().value(), e.what());
    }
    // db, result auto-cleaned up - even if exception thrown
    return 0;
}
```

Notice the exception safety: if `db_query` fails after `db_open` has already succeeded, the `Database` destructor still runs and closes the connection cleanly. That is RAII doing its job.

**Pattern highlights:**

- Custom `std::error_category` preserves the C library's error codes in `std::error_code`.
- `db_check()` is a thin translation layer - one call per C function wraps all error handling.
- Both `Database` and `QueryResult` use `unique_ptr` with custom deleters.
- Exception safety: if `db_query` fails after `db_open` succeeded, the connection is still cleaned up.

### Q3: Use std::span to bridge C array+length parameters with C++ range-based code

**Answer:**

`std::span` is the ideal glue between C's "pointer plus size" convention and C++'s range-based algorithms. It is nothing more than a pointer and a size bundled together with a nice interface - no allocation, no copying, zero overhead. What it gives you is the ability to pass C data directly into `std::ranges::sort`, range-based for loops, `std::accumulate`, and anything else that works with contiguous ranges.

```cpp
#include <cstddef>
#include <cstdlib>
#include <cstring>
#include <span>
#include <numeric>
#include <algorithm>
#include <vector>
#include <ranges>
#include <iostream>

// Simulated C library: image processing
extern "C" {
    struct c_image {
        unsigned char* pixels;  // raw pixel buffer
        int width;
        int height;
        int channels;           // 1=gray, 3=RGB, 4=RGBA
    };

    c_image* c_image_load(const char* path);
    void c_image_free(c_image* img);
    void c_image_apply_filter(unsigned char* pixels, size_t count,
                               const float* kernel, size_t kernel_size);
}

// C++ wrapper using std::span
class Image {
    struct Deleter {
        void operator()(c_image* img) const noexcept {
            if (img) c_image_free(img);
        }
    };
    std::unique_ptr<c_image, Deleter> img_;

public:
    explicit Image(const char* path) : img_(c_image_load(path)) {
        if (!img_) throw std::runtime_error("Failed to load image");
    }

    int width()    const { return img_->width; }
    int height()   const { return img_->height; }
    int channels() const { return img_->channels; }
    size_t pixel_count() const {
        return static_cast<size_t>(img_->width) * img_->height * img_->channels;
    }

    // Bridge: C raw pointer+size -> std::span
    std::span<unsigned char> pixels() {
        return { img_->pixels, pixel_count() };
    }
    std::span<const unsigned char> pixels() const {
        return { img_->pixels, pixel_count() };
    }

    // Now we can use C++ algorithms on C data!
    void invert() {
        for (auto& p : pixels()) {
            p = 255 - p;
        }
    }

    double average_brightness() const {
        auto px = pixels();
        double sum = std::accumulate(px.begin(), px.end(), 0.0);
        return sum / px.size();
    }

    // Apply filter: wraps C function with span-based API
    void apply_filter(std::span<const float> kernel) {
        c_image_apply_filter(
            img_->pixels, pixel_count(),
            kernel.data(), kernel.size()
        );
    }

    // Use ranges to process channels separately
    void boost_red(unsigned char amount) {
        if (channels() < 3) return;
        auto px = pixels();
        // Every 3rd byte starting at 0 is the red channel (RGB layout)
        for (size_t i = 0; i < px.size(); i += channels()) {
            px[i] = static_cast<unsigned char>(
                std::min(255, px[i] + amount));
        }
    }
};

// Generic span-based utilities
// Works with ANY contiguous data - C arrays, vectors, std::array
struct Stats {
    double mean;
    double min_val;
    double max_val;
};

Stats compute_stats(std::span<const double> data) {
    double sum = std::accumulate(data.begin(), data.end(), 0.0);
    auto [mn, mx] = std::ranges::minmax(data);
    return { sum / data.size(), mn, mx };
}

int main() {
    // Works with C array
    double c_array[] = {1.0, 2.0, 3.0, 4.0, 5.0};
    auto s1 = compute_stats(c_array);  // implicit span from C array

    // Works with vector
    std::vector<double> vec = {10.0, 20.0, 30.0};
    auto s2 = compute_stats(vec);      // implicit span from vector

    // Works with malloc'd C buffer
    double* c_buf = static_cast<double*>(malloc(3 * sizeof(double)));
    c_buf[0] = 100; c_buf[1] = 200; c_buf[2] = 300;
    auto s3 = compute_stats(std::span{c_buf, 3});  // explicit span from raw pointer
    free(c_buf);

    std::cout << "C array mean: " << s1.mean << "\n";    // 3.0
    std::cout << "Vector mean: " << s2.mean << "\n";      // 20.0
    std::cout << "C buffer mean: " << s3.mean << "\n";    // 200.0

    return 0;
}
```

The `compute_stats` function works equally well on a C array, a `std::vector`, and a `malloc`'d buffer - because `std::span` accepts all of them. The caller does not need to know which kind of memory backs the span. This is exactly the kind of interface you want at the C/C++ boundary.

**Why `std::span` is the ideal C/C++ bridge:**

```cpp
C world                      std::span                    C++ world
────────────                ──────────────               ─────────────
int* ptr + size_t n   ──►  span<int>{ptr, n}     ──►   range-for loops
const float* + len    ──►  span<const float>     ──►   std::accumulate
malloc'd buffer       ──►  span<uint8_t>         ──►   std::ranges::sort
char buf[256]         ──►  span<char, 256>       ──►   compile-time size
```

- **Zero copy, zero overhead** - span is just a pointer plus a size (16 bytes total).
- **Implicit conversion** from `std::vector`, `std::array`, and C arrays.
- **Bounds checking** available via `.at()` or `gsl::span` in debug builds.
- **Works with ranges** - `std::ranges::sort(my_span)` sorts C data in-place.

---

## Notes

- Prefer a stateless `struct Deleter` over `std::function` for custom deleters - stateless deleters have zero overhead, while `std::function` adds a pointer-sized member to the `unique_ptr`.
- Use `std::unique_ptr<T, Deleter>` for exclusive ownership, `std::shared_ptr<T>` only when the resource truly needs shared ownership.
- For C libraries requiring global initialization (like `curl_global_init`), use a reference-counted singleton that initializes in its constructor and shuts down in its destructor.
- `std::span` does not own the data it points at - always ensure the underlying C buffer outlives every span that refers to it.
- Combine the patterns together: RAII handle plus span access plus exception wrapper gives you a complete modern C++ interface over any C library.
- Always make wrappers non-copyable unless the C library explicitly supports reference counting on its handles.
