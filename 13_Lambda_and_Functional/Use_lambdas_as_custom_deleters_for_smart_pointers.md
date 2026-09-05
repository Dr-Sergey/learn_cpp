# Use lambdas as custom deleters for smart pointers

**Category:** Lambda and Functional  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unique_ptr>  

---

## Topic Overview

Smart pointers can use **custom deleters** to manage resources beyond heap memory (file handles, sockets, C library allocations). Lambdas are ideal deleters because they're inline-friendly and - when stateless - add **zero size overhead** to `unique_ptr`. This is one of those features where using it correctly costs nothing compared to the safer code you get in return.

### Deleter Options

The choice of deleter type has real consequences for the size and efficiency of your `unique_ptr`. The table summarizes what you're buying with each option:

| Deleter Type | `sizeof(unique_ptr)` | Inlinable | Syntax |
| --- | --- | --- | --- |
| Default (`delete`) | 8 bytes (pointer only) | Yes | `unique_ptr<T>` |
| **Stateless lambda** | **8 bytes** (EBO!) | **Yes** | `unique_ptr<T, decltype(lambda)>` |
| Struct functor (empty) | 8 bytes (EBO) | Yes | `unique_ptr<T, Deleter>` |
| Function pointer | **16 bytes** (ptr + fptr) | No | `unique_ptr<T, void(*)(T*)>` |
| `std::function` | ~40+ bytes | No | Don't use as deleter |

The key insight is that a stateless lambda is an empty class, and `unique_ptr` uses Empty Base Optimization (EBO) to store it - so you pay nothing extra compared to a raw pointer. A function pointer, by contrast, stores an actual pointer value in the `unique_ptr`, doubling its size.

---

## Self-Assessment

### Q1: Create a `unique_ptr` with a lambda deleter for a C-style resource

The common real-world case is wrapping C APIs that give you resources via `create_*` functions and expect you to call `destroy_*` when done. A lambda deleter lets you express that cleanup contract right at the point of construction:

```cpp
#include <iostream>
#include <memory>
#include <cstdio>

// Simulating C-style resources (like SDL_Window, FILE*, etc.)
struct CHandle {
    int id;
};

CHandle* create_handle(int id) {
    std::cout << "Created handle " << id << "\n";
    return new CHandle{id};
}

void destroy_handle(CHandle* h) {
    std::cout << "Destroyed handle " << h->id << "\n";
    delete h;
}

int main() {
    // Lambda deleter for a C-style resource
    auto deleter = [](CHandle* h) {
        if (h) destroy_handle(h);
    };

    // unique_ptr with lambda deleter type
    std::unique_ptr<CHandle, decltype(deleter)> handle(
        create_handle(42), deleter
    );

    std::cout << "Using handle: " << handle->id << "\n";

    // FILE* example
    auto file_deleter = [](FILE* f) {
        if (f) {
            std::cout << "Closing file\n";
            std::fclose(f);
        }
    };

    {
        std::unique_ptr<FILE, decltype(file_deleter)> file(
            std::fopen("test_output.txt", "w"), file_deleter
        );
        if (file)
            std::fputs("Hello from RAII!\n", file.get());
        // file closed automatically when unique_ptr goes out of scope
    }

    // C++20: stateless lambda can be default-constructed
    // std::unique_ptr<CHandle, decltype(deleter)> h2(create_handle(99));
    // works in C++20 because stateless lambda is default-constructible
}
// Expected output:
//   Created handle 42
//   Using handle: 42
//   Closing file
//   Destroyed handle 42
```

The `if (h)` guard in the deleter is good practice - `unique_ptr` won't call a deleter on a null pointer by default, but being explicit costs nothing and protects you in edge cases.

---

### Q2: Show that a stateless lambda deleter has zero size overhead due to EBO

This example makes the size difference concrete. If you've ever wondered why the `std::function` approach is discouraged for deleters, this comparison tells the whole story:

```cpp
#include <iostream>
#include <memory>
#include <functional>

void free_func_deleter(int* p) { delete p; }

int main() {
    // 1. Default deleter
    std::unique_ptr<int> default_ptr(new int{1});
    std::cout << "default_delete:    sizeof = " << sizeof(default_ptr) << "\n";

    // 2. Stateless lambda deleter -> EBO: zero size!
    auto lambda_del = [](int* p) { delete p; };
    std::unique_ptr<int, decltype(lambda_del)> lambda_ptr(new int{2}, lambda_del);
    std::cout << "lambda deleter:    sizeof = " << sizeof(lambda_ptr) << "\n";

    // 3. Function pointer deleter -> stores pointer (8 bytes extra)
    std::unique_ptr<int, void(*)(int*)> fptr_ptr(new int{3}, free_func_deleter);
    std::cout << "func ptr deleter:  sizeof = " << sizeof(fptr_ptr) << "\n";

    // 4. std::function deleter -> large overhead
    std::unique_ptr<int, std::function<void(int*)>> func_ptr(
        new int{4}, [](int* p) { delete p; }
    );
    std::cout << "std::function del: sizeof = " << sizeof(func_ptr) << "\n";

    // The lambda has sizeof == 1 (empty class), but due to EBO
    // (Empty Base Optimization), unique_ptr doesn't add any bytes!
    std::cout << "\nsizeof(lambda_del) = " << sizeof(lambda_del) << "\n";
    std::cout << "sizeof(int*)       = " << sizeof(int*) << "\n";
}
// Expected output (64-bit):
//   default_delete:    sizeof = 8
//   lambda deleter:    sizeof = 8    <- same as raw pointer!
//   func ptr deleter:  sizeof = 16   <- 8 (ptr) + 8 (func ptr)
//   std::function del: sizeof = 40   <- large (or more)
//
//   sizeof(lambda_del) = 1
//   sizeof(int*)       = 8
```

**EBO (Empty Base Optimization):** `unique_ptr` stores the deleter as a compressed pair with the pointer. An empty (stateless) deleter class takes zero additional bytes because the compiler can overlap it with the pointer storage. The lambda's `sizeof` is `1` (the minimum for any class), but that `1` disappears inside the compressed pair.

---

### Q3: Compare lambda deleter vs function pointer deleter vs struct deleter

Here you can see all three approaches next to each other, including their runtime behavior. The output order is worth noting: destructors run in reverse order of construction, which is why the deleters fire bottom-to-top:

```cpp
#include <iostream>
#include <memory>

// Struct deleter
struct StructDeleter {
    void operator()(int* p) const {
        std::cout << "struct delete\n";
        delete p;
    }
};

// Function pointer deleter
void func_delete(int* p) {
    std::cout << "func delete\n";
    delete p;
}

int main() {
    // Lambda deleter
    auto lam = [](int* p) { std::cout << "lambda delete\n"; delete p; };

    {
        std::unique_ptr<int, decltype(lam)> p1(new int{1}, lam);
        std::unique_ptr<int, StructDeleter> p2(new int{2});
        std::unique_ptr<int, void(*)(int*)> p3(new int{3}, func_delete);

        std::cout << "sizeof lambda:  " << sizeof(p1) << "\n";
        std::cout << "sizeof struct:  " << sizeof(p2) << "\n";
        std::cout << "sizeof funcptr: " << sizeof(p3) << "\n";
    }
}
// Expected output (64-bit):
//   sizeof lambda:  8
//   sizeof struct:  8
//   sizeof funcptr: 16
//   lambda delete
//   func delete
//   struct delete
```

The summary table makes the trade-offs explicit. For most use cases, lambda and struct functors are equivalent - the main difference is that lambdas are inline and local, while structs can be defined once and reused across translation units:

| Criterion | Lambda | Struct Functor | Function Pointer |
| --- | --- | --- | --- |
| **Size** | 8 bytes (EBO) | 8 bytes (EBO) | 16 bytes |
| **Inlinable** | Yes (type known) | Yes (type known) | No (indirect call) |
| **Syntax** | Inline, concise | Separate struct | Separate function |
| **Stateful** | Via captures | Via data members | No |
| **Reusable** | Not across TUs | Yes | Yes |
| **C++20 default-ctor** | Yes (stateless) | Yes | N/A |

---

## Notes

- **`shared_ptr` is different:** It type-erases the deleter, so lambda/struct/function pointer all have the same `sizeof(shared_ptr)`. The deleter is stored in the control block, not the `shared_ptr` object itself.
- **Stateful lambda deleters** (with captures) are NOT zero-size - they add the capture size to `unique_ptr`. A capturing lambda deleter that stores an allocator pointer, for example, will add one pointer-worth of size.
- **C++20 tip:** Stateless lambdas are default-constructible, so `unique_ptr<T, decltype(lam)> p(raw);` works without passing the lambda instance - the type alone is enough.
- **`+lambda` for function pointer:** `unique_ptr<T, void(*)(T*)> p(raw, +lambda);` - but this loses inlineability since you now have an indirect call through a function pointer.
- **Best practice:** Use stateless lambdas or struct functors for zero-overhead RAII wrappers. Avoid `std::function` as a deleter - its overhead is large and its type erasure buys you nothing here.
