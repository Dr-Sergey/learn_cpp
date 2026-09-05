# Abstract platform-specific atomic and memory model differences

**Category:** Cross Platform Development  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/memory_order>  

---

## Topic Overview

### The C++ Memory Model vs Hardware Reality

The C++ memory model is an abstraction. Real CPUs have different hardware memory models, and understanding the gap between the two is essential for writing correct concurrent code that runs everywhere:

| Architecture | Model | Key characteristic |
| --- | --- | --- |
| **x86/x86-64** | TSO (Total Store Ordering) | Strong - stores are ordered, loads are ordered, only store-load reordering possible |
| **ARM (v7/v8)** | Weak | Loads and stores can be reordered freely |
| **RISC-V** | Weak (RVWMO) | Similar to ARM - requires explicit fences |
| **POWER/PowerPC** | Very Weak | Even dependent loads can be reordered |

### Why This Matters for Cross-Platform Code

Here is the kind of bug that bites people who only test on x86: code that uses `relaxed` ordering often "works" on x86 because the hardware already enforces stronger guarantees than you asked for. Ship the same binary to an ARM device, and suddenly you have a data race.

The following example shows exactly this scenario. On x86 the TSO model happens to give you the ordering you need even with `relaxed`, but on ARM a weak memory model allows the CPU to reorder operations freely, meaning the consumer might read stale data:

```cpp
// This WORKS on x86 but is BROKEN on ARM!
std::atomic<bool> ready{false};
int data = 0;

// Thread 1
void producer() {
    data = 42;
    ready.store(true, std::memory_order_relaxed);  // No fence!
}

// Thread 2
void consumer() {
    while (!ready.load(std::memory_order_relaxed)) {}
    int x = data;  // On ARM: might read 0 (stale data)!
                    // On x86: always reads 42 (TSO guarantees it)
}
```

The reason this trips people up is that x86's TSO model masks the mistake: you get the ordering for free from the hardware without the C++ standard promising it. On ARM, `relaxed` truly means "no ordering" and the CPU may reorder aggressively.

The fix is to use the proper memory ordering that the C++ standard actually guarantees across all architectures:

```cpp
// CORRECT - portable across all architectures
void producer() {
    data = 42;
    ready.store(true, std::memory_order_release);  // Release fence
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {}  // Acquire fence
    int x = data;  // Guaranteed to see 42 on ALL platforms
}
```

The release/acquire pair forms a synchronization edge: everything the producer wrote before the `release` store is visible to the consumer after the matching `acquire` load. This is the fundamental producer-consumer building block.

### Memory Order Mapping to Hardware

It helps to see what each C++ memory order actually compiles to on different hardware. The key takeaway is that on x86, most orderings translate to a plain `MOV` instruction because TSO already provides them. On ARM, every ordering beyond `relaxed` requires a real hardware fence instruction, which has an associated cost:

| C++ memory order | x86 | ARM v8 | RISC-V |
| --- | --- | --- | --- |
| `relaxed` | MOV (plain) | LDR/STR (plain) | LW/SW (plain) |
| `acquire` (load) | MOV (free - TSO gives this) | LDAR | LW; FENCE R,RW |
| `release` (store) | MOV (free - TSO gives this) | STLR | FENCE RW,W; SW |
| `acq_rel` | MOV (free) | LDAR/STLR | Full FENCE |
| `seq_cst` (load) | MOV (free) | LDAR | FENCE RW,RW; LW |
| `seq_cst` (store) | MOV + MFENCE (or XCHG) | STLR | SW; FENCE RW,RW |

Key insight: on x86, most orderings are **free** because the hardware already provides TSO. On ARM, every ordering beyond `relaxed` requires a hardware fence instruction.

### Abstraction Pattern: Lock-Free Queue

A well-designed lock-free data structure uses the minimum necessary memory ordering at each operation. The SPSC (single-producer, single-consumer) queue below carefully places `acquire` and `release` exactly where synchronization is needed - and no more. Notice that the `push` side reads the head with `acquire` (to sync with the consumer's release of slots), and writes the tail with `release` (to publish new data). The `pop` side does the mirror image:

```cpp
#include <atomic>
#include <array>
#include <optional>

template<typename T, size_t N>
class SPSCQueue {  // Single-producer, single-consumer
    std::array<T, N> buffer_;
    // Use alignas to prevent false sharing
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};

public:
    bool push(const T& value) {
        size_t tail = tail_.load(std::memory_order_relaxed);  // Only producer writes tail
        size_t next = (tail + 1) % N;

        if (next == head_.load(std::memory_order_acquire)) {  // Acquire: sync with consumer
            return false;  // Full
        }

        buffer_[tail] = value;
        tail_.store(next, std::memory_order_release);  // Release: publish the data
        return true;
    }

    std::optional<T> pop() {
        size_t head = head_.load(std::memory_order_relaxed);  // Only consumer writes head

        if (head == tail_.load(std::memory_order_acquire)) {  // Acquire: sync with producer
            return std::nullopt;  // Empty
        }

        T value = buffer_[head];
        head_.store((head + 1) % N, std::memory_order_release);  // Release: free the slot
        return value;
    }
};
// This works identically on x86, ARM, and RISC-V
```

This queue works correctly on all three architectures without any platform-specific code, because it uses the C++ memory model correctly rather than relying on hardware accidents.

### Testing on Weak Memory Architectures

Testing only on x86 gives a false sense of security. You have two practical tools for catching weak-memory bugs even before you have ARM hardware:

```bash
# Cross-compile for ARM and run under QEMU
aarch64-linux-gnu-g++ -std=c++20 -O2 -pthread test.cpp -o test_arm
qemu-aarch64 -L /usr/aarch64-linux-gnu test_arm

# Use ThreadSanitizer (catches ordering bugs even on x86)
g++ -std=c++20 -O2 -fsanitize=thread test.cpp -o test_tsan
./test_tsan
```

ThreadSanitizer is particularly valuable because it detects data races that would be hidden by x86's strong ordering but would fail on ARM. Run it routinely, not just when you suspect a problem.

### Common Cross-Platform Atomic Pitfalls

There are four recurring mistakes in cross-platform atomic code. Each one is easy to miss because it may compile and run correctly on x86, only to break on a different architecture or with a different compiler:

```cpp
// Pitfall 1: Assuming atomic operations are always lock-free
std::atomic<LargeStruct> a;  // May use a mutex internally!
static_assert(std::atomic<LargeStruct>::is_always_lock_free,
              "Need lock-free atomics for performance");

// Pitfall 2: Different alignment requirements
struct Data {
    std::atomic<int64_t> value;  // Must be 8-byte aligned
    // On some 32-bit ARM, misaligned atomic<int64_t> is not lock-free
};
static_assert(alignof(std::atomic<int64_t>) >= 8);

// Pitfall 3: Using volatile instead of atomic
volatile int flag = 0;  // NOT atomic! No ordering guarantees!
std::atomic<int> proper_flag{0};  // Use this instead

// Pitfall 4: Relying on memory_order_relaxed for synchronization
std::atomic<int> sync_flag{0};
sync_flag.store(1, std::memory_order_relaxed);  // No synchronization!
// Use release/acquire for producer-consumer patterns
```

Pitfall 3 is worth highlighting: `volatile` prevents compiler reordering of accesses to that variable, but it does not prevent CPU reordering, it does not provide atomicity, and it does not synchronize between threads. It is not a substitute for `std::atomic`.

---

## Self-Assessment

### Q1: Why does relaxed ordering work "by accident" on x86 but fail on ARM

x86 uses TSO (Total Store Ordering), which guarantees:

- Loads are not reordered with other loads
- Stores are not reordered with other stores
- Loads are not reordered with earlier stores

The only reordering x86 allows is a later load before an earlier store. This means `memory_order_acquire` and `memory_order_release` are effectively free on x86 - the hardware already provides them. Code using `relaxed` ordering often accidentally works because TSO provides stronger guarantees.

ARM uses a weak memory model where loads and stores can be reordered in any direction. `relaxed` really means "no ordering" - the CPU will reorder aggressively for performance. You must use explicit acquire/release to get the same behavior x86 gives you for free.

### Q2: How do you verify your lock-free code is correct on weak architectures

1. **ThreadSanitizer** (`-fsanitize=thread`) - detects data races, works on x86
2. **ARM cross-compilation + QEMU** - actually runs on a weak memory model
3. **CDSChecker** or **GenMC** - model checkers that explore all possible memory orderings
4. **Stress testing** with many threads - increases the chance of exposing reordering bugs

### Q3: What is the cost of `seq_cst` vs `relaxed` on each platform

On x86: `seq_cst` stores require an `MFENCE` instruction (~20-40 cycles). `relaxed` and `acquire`/`release` are free.

On ARM: `seq_cst` uses `LDAR`/`STLR` + full barrier. `acquire`/`release` uses `LDAR`/`STLR` (~5-10 cycles). `relaxed` is free. The relative cost is smaller on ARM because even weaker orderings need instructions.

Rule of thumb: use `acquire`/`release` for producer-consumer patterns and `seq_cst` only when you need total ordering across multiple atomics.

---

## Notes

- Always develop with ThreadSanitizer enabled - it catches bugs invisible on x86.
- Default to `memory_order_seq_cst` (the default) unless you have measured a performance need for weaker ordering.
- `std::atomic_ref` (C++20) lets you do atomic operations on non-atomic variables - same ordering rules apply.
- The `std::hardware_destructive_interference_size` constant helps avoid false sharing portably.
- When in doubt, use `seq_cst` - it is correct on all platforms and usually fast enough.
