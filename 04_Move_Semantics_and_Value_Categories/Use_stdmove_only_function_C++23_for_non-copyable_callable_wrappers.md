# Use std::move_only_function (C++23) for Non-Copyable Callable Wrappers

**Category:** Move Semantics and Value Categories  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/move_only_function>  

---

## Topic Overview

### The Problem with `std::function`

`std::function` **requires its stored callable to be copyable**. This means you cannot store:

- Lambdas capturing `std::unique_ptr` (move-only)
- Lambdas capturing any move-only type
- Move-only function objects

The reason is that `std::function` itself needs to be copyable, and copying it means copying the stored callable. If the callable isn't copyable, the whole thing falls apart at compile time.

```cpp
auto ptr = std::make_unique<int>(42);
// std::function<void()> f = [p = std::move(ptr)]{ *p = 10; };  // ERROR!
// std::function requires CopyConstructible callable
```

### `std::move_only_function` (C++23)

`std::move_only_function` drops the copy requirement - it only requires the callable to be **move-constructible**. In exchange, the wrapper itself is no longer copyable. That's the deal: you give up copyability, you gain the ability to capture anything movable.

| Feature | `std::function` | `std::move_only_function` |
| --- | :---: | :---: |
| Callable must be copyable | **Yes** | No |
| Callable must be movable | Yes | **Yes** |
| Wrapper is copyable | Yes | **No** |
| Wrapper is movable | Yes | **Yes** |
| `const` qualifier support | No (always mutable) | **Yes** (via signature) |
| Has `target()` / `target_type()` | Yes | **No** |
| Small-buffer optimization | Implementation-defined | Implementation-defined |
| Header | `<functional>` | `<functional>` |

### const-Correctness Advantage

`std::function<void()>::operator()` is `const` even though it may call a non-const callable - this is a known design flaw. `std::move_only_function` fixes it by encoding const directly in the signature:

```cpp
std::move_only_function<void()>        f1;  // operator() is non-const
std::move_only_function<void() const>  f2;  // operator() is const
```

---

## Self-Assessment

### Q1: Store a lambda that captures a `unique_ptr` in a `std::move_only_function<void()>`

This is the simplest use case: you have a lambda that owns a resource via `unique_ptr` and you need to store it somewhere. `std::function` refuses at compile time; `std::move_only_function` handles it without complaint.

```cpp
#include <functional>
#include <memory>
#include <iostream>

class Logger {
    std::move_only_function<void(std::string_view)> sink_;

public:
    // Accept any callable - including move-only ones
    Logger(std::move_only_function<void(std::string_view)> sink)
        : sink_(std::move(sink)) {}

    void log(std::string_view msg) {
        if (sink_) sink_(msg);
    }
};

int main() {
    // Lambda capturing a unique_ptr - move-only!
    auto file_handle = std::make_unique<std::ostream*>(&std::cout);

    std::move_only_function<void()> greet = [fh = std::move(file_handle)]() {
        **fh << "Hello from move_only_function!\n";
    };

    greet();  // Output: Hello from move_only_function!

    // More practical: callback with unique ownership
    auto buffer = std::make_unique<std::string>("buffered data");

    std::move_only_function<void()> flush = [buf = std::move(buffer)]() {
        std::cout << "Flushing: " << *buf << "\n";
    };

    flush();  // Output: Flushing: buffered data

    // Use in a class
    auto resource = std::make_unique<int>(99);
    Logger logger([res = std::move(resource)](std::string_view msg) {
        std::cout << "[Resource " << *res << "] " << msg << "\n";
    });

    logger.log("system started");   // Output: [Resource 99] system started
    logger.log("processing...");    // Output: [Resource 99] processing...

    // move_only_function is itself move-only
    auto moved_flush = std::move(greet);
    // greet is now empty
    std::cout << "greet empty after move: " << (!greet ? "yes" : "no") << "\n";
    // Output: greet empty after move: yes

    return 0;
}
```

**Expected output:**

```text
Hello from move_only_function!
Flushing: buffered data
[Resource 99] system started
[Resource 99] processing...
greet empty after move: yes
```

### Q2: Show that `std::function` rejects this while `move_only_function` accepts it

The compile errors are commented out so you can see exactly what the problem is, then see the working alternatives immediately after:

```cpp
#include <functional>
#include <memory>
#include <iostream>
#include <vector>

// A move-only callable object
struct UniqueTask {
    std::unique_ptr<int> data;

    UniqueTask(int v) : data(std::make_unique<int>(v)) {}

    // NOT copyable
    UniqueTask(const UniqueTask&) = delete;
    UniqueTask& operator=(const UniqueTask&) = delete;

    // Movable
    UniqueTask(UniqueTask&&) = default;
    UniqueTask& operator=(UniqueTask&&) = default;

    void operator()() const {
        std::cout << "Task result: " << *data << "\n";
    }
};

int main() {
    std::cout << "=== std::function vs std::move_only_function ===\n\n";

    // --- std::function REJECTS move-only callables ---
    // Uncomment any of these to see the compile error:

    // ERROR 1: Lambda capturing unique_ptr
    // auto ptr = std::make_unique<int>(42);
    // std::function<void()> f1 = [p = std::move(ptr)]{ std::cout << *p; };
    //   -> error: use of deleted copy constructor

    // ERROR 2: Move-only functor
    // std::function<void()> f2 = UniqueTask(10);
    //   -> error: call to deleted copy constructor of 'UniqueTask'

    // --- std::move_only_function ACCEPTS them ---

    // OK 1: Lambda capturing unique_ptr
    auto ptr = std::make_unique<int>(42);
    std::move_only_function<void()> mof1 = [p = std::move(ptr)] {
        std::cout << "Lambda with unique_ptr: " << *p << "\n";
    };
    mof1();  // Output: Lambda with unique_ptr: 42

    // OK 2: Move-only functor
    std::move_only_function<void()> mof2 = UniqueTask(100);
    mof2();  // Output: Task result: 100

    // Practical use: task queue with move-only tasks
    std::vector<std::move_only_function<void()>> task_queue;

    for (int i = 0; i < 3; ++i) {
        auto resource = std::make_unique<int>(i * 10);
        task_queue.push_back([r = std::move(resource)] {
            std::cout << "Processing resource: " << *r << "\n";
        });
    }

    // std::vector<std::function<void()>> would FAIL for the above

    std::cout << "\nExecuting task queue:\n";
    for (auto& task : task_queue) {
        task();
    }

    // const-correctness: move_only_function respects const
    int counter = 0;
    std::move_only_function<void()> mutating = [&counter] { ++counter; };
    // std::move_only_function<void() const> const_fn = [&counter] { ++counter; };
    // Would also work - lambdas with captures by ref are const-callable

    mutating();
    std::cout << "\nCounter after mutating call: " << counter << "\n";

    return 0;
}
```

**Expected output:**

```text
=== std::function vs std::move_only_function ===

Lambda with unique_ptr: 42
Task result: 100

Executing task queue:
Processing resource: 0
Processing resource: 10
Processing resource: 20

Counter after mutating call: 1
```

### Q3: Compare the size and overhead of `move_only_function` vs `function` for simple function pointers

`move_only_function` is typically smaller than `std::function` because it doesn't need to store a copy-function pointer in its internal vtable. The invocation overhead is usually identical since both go through an indirect call.

```cpp
#include <functional>
#include <iostream>
#include <chrono>
#include <memory>

void free_function() {}

int add(int a, int b) { return a + b; }

int main() {
    std::cout << "=== Size Comparison ===\n\n";

    std::cout << "sizeof(void*)                              = " << sizeof(void*) << "\n";
    std::cout << "sizeof(std::function<void()>)              = " << sizeof(std::function<void()>) << "\n";
    std::cout << "sizeof(std::move_only_function<void()>)    = " << sizeof(std::move_only_function<void()>) << "\n";
    std::cout << "sizeof(std::function<int(int,int)>)        = " << sizeof(std::function<int(int,int)>) << "\n";
    std::cout << "sizeof(std::move_only_function<int(int,int)>) = " << sizeof(std::move_only_function<int(int,int)>) << "\n";

    // Typical results (MSVC/x64):
    //   std::function<void()>:           64 bytes (stores type-erased callable + vtable + SBO buffer)
    //   std::move_only_function<void()>: 32-48 bytes (no copy vtable needed -> smaller)

    std::cout << "\n=== Overhead Analysis ===\n\n";

    std::cout << "std::function overhead:\n";
    std::cout << "  - Must store: callable + invoke ptr + copy ptr + destroy ptr\n";
    std::cout << "  - Copy pointer adds ~8 bytes to type-erased vtable\n";
    std::cout << "  - target_type() requires RTTI storage\n\n";

    std::cout << "std::move_only_function overhead:\n";
    std::cout << "  - Stores: callable + invoke ptr + destroy ptr\n";
    std::cout << "  - No copy pointer needed -> smaller vtable\n";
    std::cout << "  - No target_type() -> no RTTI needed\n\n";

    // Performance benchmark: invoke overhead
    constexpr int N = 100'000'000;

    std::function<int(int,int)> std_fn = add;
    std::move_only_function<int(int,int)> mof_fn = add;

    volatile int sink = 0;

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        sink = std_fn(i, 1);
    }
    auto t2 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        sink = mof_fn(i, 1);
    }
    auto t3 = std::chrono::high_resolution_clock::now();

    auto ms_fn = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    auto ms_mof = std::chrono::duration_cast<std::chrono::milliseconds>(t3 - t2).count();

    std::cout << "=== Invocation benchmark (" << N << " calls) ===\n";
    std::cout << "std::function:           " << ms_fn << " ms\n";
    std::cout << "std::move_only_function: " << ms_mof << " ms\n";
    std::cout << "(Invocation cost is typically similar - both use indirect call)\n";

    std::cout << "\n=== When to use which ===\n";
    std::cout << "+------------------------------+-------------------+------------------------+\n";
    std::cout << "| Scenario                     | std::function     | move_only_function     |\n";
    std::cout << "+------------------------------+-------------------+------------------------+\n";
    std::cout << "| Copyable callable needed     | YES               | No                     |\n";
    std::cout << "| Move-only callable (unique_ptr) | No             | YES                    |\n";
    std::cout << "| Callback stored in container | OK                | OK (container movable) |\n";
    std::cout << "| Thread pool task queue       | Only if copyable  | YES (preferred)        |\n";
    std::cout << "| const-correct invocation     | No (always const) | YES (via signature)    |\n";
    std::cout << "+------------------------------+-------------------+------------------------+\n";

    return 0;
}
```

---

## Notes

- `std::move_only_function` (C++23) is the move-only sibling of `std::function`.
- Use it when your callable captures move-only types (`unique_ptr`, `jthread`, etc.).
- It is typically smaller than `std::function` (no copy vtable entry, no RTTI).
- It properly encodes `const`/`noexcept` qualifiers in the signature - a fix for `std::function`'s const-correctness bug.
- `std::move_only_function` itself is **not copyable**, only movable.
- For C++26, `std::copyable_function` will be the "fixed" copyable version (const-correct).
