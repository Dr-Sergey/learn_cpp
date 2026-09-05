# Design callback and extension interfaces

**Category:** API and Library Design  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function>  

---

## Topic Overview

Libraries often need to let users inject their own behavior - think event handlers, sort comparators, pipeline stages, or plugin hooks. There are three main tools for this in C++, and they have genuinely different tradeoffs. Understanding when to reach for each one is the point of this topic.

### Comparison

The table below summarizes the three approaches at a glance. The core tension is between flexibility (can store heterogeneous callables, can be put in a container) and performance (can the call be inlined, does it allocate).

| Approach | Overhead | Inlinable | Copyable | Type-erased |
| --- | --- | --- | --- | --- |
| `std::function` | Heap alloc possible | No | Yes | Yes |
| Template parameter | Zero | Yes | Depends | No |
| Type-erased interface | Virtual call | No | Depends | Yes |

### std::function - Simple, Dynamic

`std::function` is the go-to choice when you need to store callbacks of different types in the same container, or when the callback is going to be stored and called later rather than immediately. The cost is an indirect function call and, for large captures, a potential heap allocation.

```cpp
#include <functional>
#include <iostream>
#include <vector>

class EventSystem {
    std::vector<std::function<void(int)>> handlers_;
public:
    void on_event(std::function<void(int)> handler) {
        handlers_.push_back(std::move(handler));
    }
    void fire(int event_id) {
        for (auto& h : handlers_) h(event_id);
    }
};

// Usage:
EventSystem sys;
sys.on_event([](int id) { std::cout << "Event: " << id << "\n"; });
sys.fire(42);
```

This pattern is easy to read and works with any callable - lambdas, function pointers, functors. The `vector<std::function<void(int)>>` lets you register any number of handlers of different types, which is hard to do with templates alone.

### Template - Zero Overhead, Compile-Time

When you know the callback type at compile time and you care about performance - especially in hot loops - a template parameter is the right choice. The compiler knows the exact callable type and can inline the call entirely, producing the same code as if you had written the logic inline.

```cpp
#include <concepts>
#include <iostream>

template<typename Handler>
requires std::invocable<Handler, int>
class EventProcessor {
    Handler handler_;
public:
    explicit EventProcessor(Handler h) : handler_(std::move(h)) {}
    void process(int event) { handler_(event); }
};

// Usage - handler is inlined
auto proc = EventProcessor([](int id) { /* inlined */ });
proc.process(42);
```

The `requires std::invocable<Handler, int>` constraint (C++20) gives a clear error message if the user passes something that is not callable with an `int`. Without constraints, the error would come from deep inside the template instantiation and be much harder to read.

### Type Erasure - Runtime Polymorphism Without Inheritance

The third approach is a manual type erasure pattern. This is what `std::function` does internally, but you can build it yourself when you need custom behavior - a different storage strategy, reference semantics, or a different interface shape. It gives you runtime polymorphism without forcing users to inherit from a base class.

```cpp
#include <memory>
#include <iostream>

class AnyHandler {
    struct Concept {
        virtual void call(int) = 0;
        virtual ~Concept() = default;
    };
    template<typename T>
    struct Model : Concept {
        T handler;
        Model(T h) : handler(std::move(h)) {}
        void call(int v) override { handler(v); }
    };
    std::unique_ptr<Concept> impl_;
public:
    template<typename T>
    AnyHandler(T handler) : impl_(std::make_unique<Model<T>>(std::move(handler))) {}
    void operator()(int v) { impl_->call(v); }
};
```

The internal `Concept`/`Model` split is the heart of it: `Concept` defines the interface (a pure virtual base), and `Model<T>` wraps whatever concrete type the user provides. The user just constructs an `AnyHandler` from any callable - the template constructor captures the type and stores it in a `Model`. From the outside, `AnyHandler` looks like a uniform type.

---

## Self-Assessment

### Q1: When to choose each approach

- **`std::function`**: when callbacks need to be stored in a container, when you need to handle heterogeneous callable types, or when simplicity matters more than raw performance.
- **Template parameter**: on hot paths, in library internals, or anywhere the compiler needs a chance to inline. This is the choice when you know the callback type at the point of use.
- **Type erasure**: when you need runtime polymorphism but do not want to force users to inherit from an abstract base class. Also useful when you want control over the storage strategy that `std::function` does not give you.

### Q2: What is `std::move_only_function` (C++23) and when to prefer it

`std::move_only_function` does not require the wrapped callable to be copyable. This matters when a callback captures a `unique_ptr` or other move-only resource - `std::function` cannot hold it because `std::function` requires its stored callable to be copyable. Use `std::move_only_function` whenever the callback owns a resource and does not need to be copied itself.

### Q3: Show the performance difference

To make the tradeoff concrete, here are rough per-call costs on modern hardware:

```cpp
std::function call:     ~5-10 ns (indirect call + possible heap)
Template/inlined call:  ~0-1 ns (direct call, often inlined)
Virtual type erasure:   ~3-5 ns (indirect call, no heap per call)
```

In a tight inner loop called millions of times, the difference between zero-overhead template dispatch and `std::function` is real and measurable. For an event system called a few hundred times per frame, `std::function` is perfectly fine.

---

## Notes

- `std::function` can allocate on the heap for large captures - use `std::function_ref` (C++26) for non-owning references when you only need to call the callback immediately and not store it.
- Templates are best for library-internal callbacks; `std::function` is the right default for stored callbacks in public APIs.
- `std::move_only_function` (C++23) replaces `std::function` when copyability is not needed - prefer it when the callable captures move-only resources.
