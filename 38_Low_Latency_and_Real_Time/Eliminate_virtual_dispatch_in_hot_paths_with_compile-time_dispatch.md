# Eliminate Virtual Dispatch in Hot Paths with Compile-Time Dispatch

**Category:** Low Latency and Real Time  
**Standard:** C++20, C++23  
**Reference:** [P0847R7 - Deducing this](https://wg21.link/p0847), [CppCon - CRTP and Mixin Idioms](https://www.youtube.com/results?search_query=crtp+cppcon)  

---

## Topic Overview

A virtual function call involves loading the vptr, fetching the vtable address from memory, indexing into the vtable, and performing an indirect branch to the target function. This sequence costs 3-25ns depending on whether the vtable and target code are in cache. More importantly, **indirect branches are poorly predicted** by the CPU - a misprediction costs 15-20 cycles on modern x86. In a tight hot loop processing millions of events, this overhead is significant and unnecessary when the concrete type is known at compile time.

The reason indirect branches are poorly predicted is that the CPU's branch predictor is trained on which target address a branch has historically jumped to. For a virtual dispatch, the target depends on the dynamic type of the object. If your loop processes a mixture of derived types in random order, the predictor can't learn a useful pattern - every call is effectively a coin flip. The 15-20 cycle penalty per misprediction is the CPU flushing its in-flight instructions and starting over from the correct address.

The primary alternatives to virtual dispatch are: **CRTP (Curiously Recurring Template Pattern)** for static polymorphism, **`std::variant` with `std::visit`** for closed type sets, **C++23 deducing-this** for simplified CRTP, and **template policy/strategy** patterns where the policy is a template parameter. Each eliminates the indirect branch, enabling the compiler to inline the call and apply further optimizations.

The compiler can sometimes **devirtualize** virtual calls when it can prove the dynamic type - this happens with `final` classes, `final` methods, and when the object is constructed locally. However, devirtualization is an optimization, not a guarantee - compile-time dispatch is the only way to ensure zero virtual overhead.

| Dispatch Method | Overhead | Type Erasure | Open/Closed | Compiler Inlining |
| --- | --- | --- | --- | --- |
| Virtual call | 3-25 ns | Full (any derived) | Open | Only with devirtualization |
| CRTP | 0 (inlined) | None | Open | Always |
| `std::variant` + `visit` | 0-2 ns (jump table) | Fixed set | Closed | Usually |
| Deducing-this (C++23) | 0 (inlined) | None | Open | Always |
| Template policy | 0 (inlined) | None | Open | Always |
| `if constexpr` | 0 | None | Compile-time | Always |

Here's the key structural difference between the two dispatch mechanisms. Virtual dispatch always goes through an indirect branch. Compile-time dispatch becomes a direct inlined call - from the CPU's perspective it's as if you wrote the function body right there:

```cpp
VIRTUAL DISPATCH:                    CRTP / COMPILE-TIME DISPATCH:
obj->process(data)                   obj.process(data)
    │                                    │
    ▼                                    ▼
load vptr ──► vtable[N] ──►         direct call (inlined) ──►
indirect branch (stall)              no branch, no cache miss
    │
    ▼ (15-20 cycle mispredict penalty)
target function
```

---

## Self-Assessment

### Q1: Implement the same handler interface using virtual dispatch, CRTP, and `std::variant`, then benchmark all three in a tight loop

Let's see the three approaches side by side and let the benchmark numbers speak. The important thing to pay attention to is not just the handler implementation but how each approach dispatches the call at the call site.

```cpp
#include <chrono>
#include <cstdio>
#include <cstdint>
#include <variant>
#include <memory>
#include <vector>

struct MarketData {
    uint64_t sequence;
    double price;
    int32_t qty;
};

// Approach 1: Virtual dispatch
struct IHandler {
    virtual void on_data(const MarketData& md) noexcept = 0;
    virtual ~IHandler() = default;
};

struct VirtualHandler final : IHandler {
    double total = 0;
    void on_data(const MarketData& md) noexcept override {
        total += md.price * md.qty;
    }
};

// Approach 2: CRTP
template <typename Derived>
struct CRTPBase {
    void on_data(const MarketData& md) noexcept {
        static_cast<Derived*>(this)->on_data_impl(md);
    }
};

struct CRTPHandler : CRTPBase<CRTPHandler> {
    double total = 0;
    void on_data_impl(const MarketData& md) noexcept {
        total += md.price * md.qty;
    }
};

// Approach 3: std::variant
struct VariantHandlerA {
    double total = 0;
    void on_data(const MarketData& md) noexcept {
        total += md.price * md.qty;
    }
};

struct VariantHandlerB {
    double total = 0;
    void on_data(const MarketData& md) noexcept {
        total += md.price * md.qty * 1.001;  // slightly different
    }
};

using HandlerVariant = std::variant<VariantHandlerA, VariantHandlerB>;

void dispatch_variant(HandlerVariant& v, const MarketData& md) noexcept {
    std::visit([&md](auto& handler) { handler.on_data(md); }, v);
}

// Benchmark
template <typename Fn>
double benchmark(Fn fn, const char* label, int N) {
    MarketData md{0, 100.5, 200};
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        md.sequence = i;
        fn(md);
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    double ns_per_op = std::chrono::duration<double, std::nano>(t1 - t0).count() / N;
    std::printf("%-25s %6.2f ns/op\n", label, ns_per_op);
    return ns_per_op;
}

int main() {
    constexpr int N = 10'000'000;

    // Virtual
    std::unique_ptr<IHandler> virt = std::make_unique<VirtualHandler>();
    benchmark([&](const MarketData& md) { virt->on_data(md); },
              "Virtual dispatch", N);

    // CRTP
    CRTPHandler crtp;
    benchmark([&](const MarketData& md) { crtp.on_data(md); },
              "CRTP (compile-time)", N);

    // Variant
    HandlerVariant var{VariantHandlerA{}};
    benchmark([&](const MarketData& md) { dispatch_variant(var, md); },
              "std::variant + visit", N);
}
```

With optimization enabled, CRTP typically gets completely inlined down to a single multiply-add instruction. Virtual dispatch can't do that because the compiler doesn't know at compile time which function it'll actually call. The variant approach sits in between - it uses a jump table (one indirect branch total, not per-member-access).

### Q2: Use C++23 deducing-this to simplify CRTP, implementing a composable handler chain with zero virtual overhead

CRTP's main weakness is the syntax: the base class must be templated on the derived class, which creates an awkward circular dependency. C++23's deducing-this feature (`this auto& self`) solves this - the base can be a plain struct, and the compiler figures out the concrete type automatically at each call site.

```cpp
#include <cstdio>
#include <cstdint>
#include <utility>
#include <cstddef>
#include <tuple>

struct MarketData {
    uint64_t seq;
    double price;
    int32_t qty;
};

// C++23 deducing-this: no CRTP template parameter needed
struct HandlerBase {
    // Deducing-this: 'self' is the most-derived type
    void on_data(this auto& self, const MarketData& md) noexcept {
        self.process(md);  // statically dispatched — no vtable
    }
};

struct PriceAggregator : HandlerBase {
    double total_notional = 0;
    void process(const MarketData& md) noexcept {
        total_notional += md.price * md.qty;
    }
};

struct TradeCounter : HandlerBase {
    uint64_t count = 0;
    void process(const MarketData& /*md*/) noexcept {
        ++count;
    }
};

// Composable handler chain — all compile-time dispatched
template <typename... Handlers>
struct HandlerChain {
    std::tuple<Handlers...> handlers;

    explicit HandlerChain(Handlers... hs) : handlers(std::move(hs)...) {}

    void on_data(const MarketData& md) noexcept {
        std::apply([&md](auto&... h) {
            (h.on_data(md), ...);  // fold expression — calls each handler
        }, handlers);
    }

    template <std::size_t I>
    auto& get() { return std::get<I>(handlers); }
};

int main() {
    HandlerChain chain{PriceAggregator{}, TradeCounter{}};

    for (int i = 0; i < 1'000'000; ++i) {
        MarketData md{static_cast<uint64_t>(i), 99.5 + i * 0.001, 100};
        chain.on_data(md);  // zero virtual dispatch, fully inlined
    }

    std::printf("Total notional: %.2f\n", chain.get<0>().total_notional);
    std::printf("Trade count:    %lu\n",
                (unsigned long)chain.get<1>().count);
}
```

The fold expression `(h.on_data(md), ...);` expands at compile time into individual calls to each handler in the chain. The compiler sees all of them, can inline all of them, and can even reorder or fuse them for efficiency. You get the composability of a polymorphic handler chain with zero runtime overhead.

### Q3: Apply the template policy pattern to a matching engine, replacing virtual `OrderBook` strategies with compile-time policies

The policy pattern is the most explicit form of compile-time dispatch. You parameterize a class on its behavior, and the behavior is resolved entirely at compile time. Different instantiations of the same template produce different, independently optimizable binaries - no shared vtable, no indirection.

```cpp
#include <cstdio>
#include <cstdint>
#include <vector>
#include <algorithm>
#include <chrono>

struct Order {
    uint64_t id;
    double price;
    int32_t qty;
    bool is_buy;
};

// Policy: Price-Time Priority (FIFO)
struct PriceTimePriority {
    static bool has_priority(const Order& a, const Order& b) noexcept {
        if (a.is_buy)
            return a.price > b.price || (a.price == b.price && a.id < b.id);
        else
            return a.price < b.price || (a.price == b.price && a.id < b.id);
    }

    static bool can_match(const Order& buy, const Order& sell) noexcept {
        return buy.price >= sell.price;
    }
};

// Policy: Pro-Rata Allocation
struct ProRataAllocation {
    static bool has_priority(const Order& a, const Order& b) noexcept {
        // Pro-rata: larger orders get priority (simplified)
        return a.qty > b.qty || (a.qty == b.qty && a.id < b.id);
    }

    static bool can_match(const Order& buy, const Order& sell) noexcept {
        return buy.price >= sell.price;
    }
};

// Matching Engine: policy is compile-time
template <typename MatchPolicy>
class MatchingEngine {
    std::vector<Order> bids_;
    std::vector<Order> asks_;
    uint64_t matches_ = 0;

public:
    void add_order(Order order) {
        auto& book = order.is_buy ? bids_ : asks_;
        auto it = std::lower_bound(book.begin(), book.end(), order,
            [](const Order& a, const Order& b) {
                return MatchPolicy::has_priority(a, b);
            });
        book.insert(it, order);
    }

    int try_match() noexcept {
        int matched = 0;
        while (!bids_.empty() && !asks_.empty()) {
            if (!MatchPolicy::can_match(bids_.front(), asks_.front()))
                break;

            int fill_qty = std::min(bids_.front().qty, asks_.front().qty);
            bids_.front().qty -= fill_qty;
            asks_.front().qty -= fill_qty;

            if (bids_.front().qty == 0) bids_.erase(bids_.begin());
            if (!asks_.empty() && asks_.front().qty == 0) asks_.erase(asks_.begin());

            ++matches_;
            ++matched;
        }
        return matched;
    }

    uint64_t total_matches() const { return matches_; }
};

template <typename Policy>
double benchmark_engine(const char* label) {
    MatchingEngine<Policy> engine;
    constexpr int N = 100'000;

    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        engine.add_order({static_cast<uint64_t>(i), 100.0 + (i % 100) * 0.01,
                          100 + (i % 50), i % 2 == 0});
        engine.try_match();
    }
    auto t1 = std::chrono::high_resolution_clock::now();

    double ns = std::chrono::duration<double, std::nano>(t1 - t0).count() / N;
    std::printf("%-25s %7.1f ns/order  matches: %lu\n",
                label, ns, (unsigned long)engine.total_matches());
    return ns;
}

int main() {
    benchmark_engine<PriceTimePriority>("Price-Time (FIFO)");
    benchmark_engine<ProRataAllocation>("Pro-Rata");
}
```

Notice that `MatchingEngine<PriceTimePriority>` and `MatchingEngine<ProRataAllocation>` are completely separate types - the compiler generates independent code for each, with the policy's static functions inlined into the engine's methods. Switching strategies at runtime would require a virtual dispatch; with this pattern you switch by picking a different template instantiation at compile time.

---

## Notes

- **`final`** keyword helps the compiler devirtualize: `struct Handler final : IBase` allows devirtualization when the compiler sees the concrete type.
- **`std::variant` dispatch** generates a jump table - typically one indirect branch vs N vtable lookups in a chain. For 2-3 types, it's near-zero overhead.
- **CRTP overhead is exactly zero** - the compiler inlines everything, producing the same code as a direct function call.
- **Deducing-this** (C++23, P0847) eliminates CRTP's syntactic burden - the base class doesn't need a template parameter, yet `self` deduces the derived type.
- Use `[[gnu::always_inline]]` or `__forceinline` on policy methods if the compiler refuses to inline at the default optimization level.
- Verify devirtualization in assembly: look for `call` to a direct address vs `call [rax]` (indirect). `g++ -O2 -S -fverbose-asm` or Compiler Explorer.
