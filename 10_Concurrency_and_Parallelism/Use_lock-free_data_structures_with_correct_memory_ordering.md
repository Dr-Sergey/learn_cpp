# Use lock-free data structures with correct memory ordering

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/memory_order>  

---

## Topic Overview

Lock-free data structures avoid mutexes by using atomic operations with specific memory orderings. The key challenge - and where most lock-free bugs come from - is choosing the **minimum** memory ordering that preserves correctness. Weaker orderings allow more hardware optimization, but incorrect orderings cause subtle, hard-to-reproduce bugs that only surface on ARM or under heavy load.

### SPSC Ring Buffer Architecture

The single-producer single-consumer (SPSC) ring buffer is the simplest meaningful lock-free structure. One thread writes at `write_idx`, another reads at `read_idx`. Because there's exactly one writer and one reader, no CAS is needed - just the right memory ordering on the index stores and loads:

```cpp
Producer writes here                Consumer reads here
         |                                  |
+---+---+---+---+---+---+---+---+
|   | A | B | C |   |   |   |   |
+---+---+---+---+---+---+---+---+
     ^               ^
    read_idx        write_idx
    (consumer)      (producer)

Producer: write data[write_idx], then advance write_idx
Consumer: read data[read_idx], then advance read_idx

Only ONE writer and ONE reader -> no CAS needed!
Correct memory ordering makes this safe without any locks.
```

### Memory Ordering Rules for SPSC

Each row in this table reflects one synchronization relationship in the buffer. The pattern is always the same: release when publishing, acquire when consuming:

| Operation | Side | Required Order | Why |
| --- | --- | --- | --- |
| Write data to slot | Producer | Before `release` | Data must be visible before index advances |
| Advance write_idx | Producer | `release` | Publishes the data |
| Read write_idx | Consumer | `acquire` | Sees all data written before index |
| Read data from slot | Consumer | After `acquire` | Reads what producer wrote |
| Advance read_idx | Consumer | `release` | Tells producer slot is free |
| Read read_idx | Producer | `acquire` | Sees that consumer freed the slot |

---

## Self-Assessment

### Q1: Implement a lock-free SPSC (single producer single consumer) ring buffer

**Answer:**

Read the memory order annotations in the comments carefully - each one is there for a specific reason, and the wrong order at any point can silently corrupt data on weakly-ordered architectures:

```cpp
#include <atomic>
#include <array>
#include <optional>
#include <thread>
#include <iostream>
#include <cassert>

template<typename T, std::size_t N>
class SPSCRingBuffer {
    static_assert((N & (N - 1)) == 0, "N must be power of 2"); // for fast modulo

    std::array<T, N> buffer_;
    alignas(64) std::atomic<std::size_t> write_idx_{0}; // separate cache lines
    alignas(64) std::atomic<std::size_t> read_idx_{0};  // to avoid false sharing

public:
    bool try_push(const T& value) {
        const auto write = write_idx_.load(std::memory_order_relaxed);
        const auto read = read_idx_.load(std::memory_order_acquire);
        //                                              ^^^^^^^^
        // acquire: see consumer's advancement of read_idx
        // to know if there's free space

        if (write - read >= N)
            return false; // full

        buffer_[write & (N - 1)] = value;
        //                         ^ write data BEFORE advancing index

        write_idx_.store(write + 1, std::memory_order_release);
        //                                      ^^^^^^^
        // release: ensures buffer write is visible before index advance
        // Consumer's acquire load of write_idx will see this data
        return true;
    }

    std::optional<T> try_pop() {
        const auto read = read_idx_.load(std::memory_order_relaxed);
        const auto write = write_idx_.load(std::memory_order_acquire);
        //                                              ^^^^^^^^
        // acquire: see producer's data writes before this point

        if (read >= write)
            return std::nullopt; // empty

        T value = buffer_[read & (N - 1)];
        //        ^ read data BEFORE advancing index

        read_idx_.store(read + 1, std::memory_order_release);
        //                                     ^^^^^^^
        // release: tells producer this slot is now free
        return value;
    }

    std::size_t size() const {
        return write_idx_.load(std::memory_order_relaxed) -
               read_idx_.load(std::memory_order_relaxed);
    }
};

int main() {
    SPSCRingBuffer<int, 1024> buffer;
    constexpr int COUNT = 1'000'000;

    // Producer thread
    std::thread producer([&] {
        for (int i = 0; i < COUNT; ++i) {
            while (!buffer.try_push(i))
                ; // spin until space available
        }
    });

    // Consumer thread
    std::thread consumer([&] {
        for (int i = 0; i < COUNT; ++i) {
            std::optional<int> val;
            while (!(val = buffer.try_pop()))
                ; // spin until data available
            assert(*val == i); // verify FIFO order
        }
    });

    producer.join();
    consumer.join();
    std::cout << "SPSC: " << COUNT << " items transferred correctly\n";
    // Output: SPSC: 1000000 items transferred correctly
}
```

**Explanation:** The ring buffer uses two indices - one for the producer and one for the consumer. Each index is only written by one thread and read by the other. Acquire/release ordering ensures that when the consumer sees an advanced `write_idx`, it also sees the data that was written before the index was advanced.

### Q2: Explain why acquire/release on producer and consumer sides is sufficient for SPSC

**Answer:**

This is a conceptual question worth sitting with. The key insight is that SPSC only has two threads, and the only thing they need to agree on is the ordering *between those two threads* - not a global total order visible to all observers. That's exactly what acquire/release provides, and why `seq_cst` would be overkill:

```cpp
WHY acquire/release IS SUFFICIENT for SPSC:
============================================

In SPSC, there are exactly two synchronization relationships:

1. PRODUCER publishes data -> CONSUMER reads data

   Producer: write data, then store write_idx (RELEASE)
   Consumer: load write_idx (ACQUIRE), then read data

   The release-acquire pair creates a happens-before:
   data_write -> store(release) -> load(acquire) -> data_read
   ^                                               ^
   Consumer is GUARANTEED to see producer's data

2. CONSUMER frees slot -> PRODUCER reuses slot

   Consumer: read data, then store read_idx (RELEASE)
   Producer: load read_idx (ACQUIRE), then write data

   The release-acquire pair ensures:
   data_read -> store(release) -> load(acquire) -> data_write
   ^                                              ^
   Producer is GUARANTEED that consumer finished reading

WHY seq_cst IS NOT NEEDED:
  seq_cst provides a total order visible to ALL threads.
  SPSC has only TWO threads - acquire/release gives sufficient
  ordering between them. No third observer needs to agree on order.

WHY relaxed IS TOO WEAK:
  relaxed provides atomicity but NO ordering guarantees.
  The consumer might load write_idx (seeing new value) but read
  STALE buffer data (from before the producer's write).

  On x86: rarely manifests (strong memory model)
  On ARM: WILL break under load (weak memory model)

DIAGRAM:
  Producer                    Consumer
  --------                    --------
  buffer[i] = val
  write_idx.store(RELEASE)
  - - - - synchronization - - - - ->
                              write_idx.load(ACQUIRE)
                              val = buffer[i]  <- sees producer's write
```

### Q3: Show a failure mode when using relaxed ordering incorrectly in the SPSC buffer

**Answer:**

Here's the broken version. On x86 it may appear to work because x86's hardware memory model is already quite strong - but the code is still undefined behavior, and it *will* fail on ARM:

```cpp
#include <atomic>
#include <array>
#include <thread>
#include <iostream>
#include <cassert>

// === BROKEN SPSC: uses relaxed everywhere ===
template<typename T, std::size_t N>
class BrokenSPSC {
    std::array<T, N> buffer_;
    std::atomic<std::size_t> write_idx_{0};
    std::atomic<std::size_t> read_idx_{0};

public:
    bool try_push(const T& value) {
        auto write = write_idx_.load(std::memory_order_relaxed);
        auto read = read_idx_.load(std::memory_order_relaxed);
        if (write - read >= N) return false;

        buffer_[write & (N - 1)] = value;

        // BUG: relaxed does NOT guarantee that buffer write
        // is visible before write_idx advances
        write_idx_.store(write + 1, std::memory_order_relaxed);
        //                                      ^^^^^^^
        // On ARM: the store to write_idx might become visible to
        // the consumer BEFORE the store to buffer[write]!
        return true;
    }

    T try_pop() {
        auto read = read_idx_.load(std::memory_order_relaxed);
        auto write = write_idx_.load(std::memory_order_relaxed);
        //                                      ^^^^^^^
        // BUG: relaxed load of write_idx does NOT guarantee
        // that we see the buffer data written before it
        if (read >= write) return T{};

        T value = buffer_[read & (N - 1)];
        // On ARM: might read stale/uninitialized data!

        read_idx_.store(read + 1, std::memory_order_relaxed);
        return value;
    }
};

int main() {
    // On x86: this might APPEAR to work because x86 has
    // a strong memory model (stores are release, loads are acquire
    // by default in hardware). But it's still TECHNICALLY UB.
    //
    // On ARM/POWER: this WILL produce incorrect results:
    //   - Consumer reads buffer[i] before producer finishes writing it
    //   - Consumer sees garbage/stale data
    //   - Uninitialized reads, torn values, or values from wrong slots
    //
    // Example failure scenario on ARM:
    //
    // Producer:                    Consumer:
    // buffer[0] = 42              (not yet visible)
    // write_idx = 1               write_idx = 1 (visible!)
    //                             val = buffer[0] -> reads 0 or garbage!
    //                             (buffer write not yet visible)
    //
    // The fix: use RELEASE on index stores, ACQUIRE on index loads.
    // This creates the happens-before relationship that guarantees
    // data visibility.

    BrokenSPSC<int, 256> broken;
    std::cout << "On x86, this MAY appear to work (strong HW model)\n";
    std::cout << "On ARM, this WILL produce incorrect results\n";
    std::cout << "In both cases, it's UNDEFINED BEHAVIOR per the C++ standard\n";

    // Correct version: see Q1's implementation with acquire/release
}
```

**Explanation:** With `relaxed` ordering, atomicity is guaranteed (no torn reads/writes on the index), but *ordering* between the buffer data write and the index advance is not guaranteed. On weakly-ordered architectures (ARM, POWER), the consumer can observe the updated index before the buffer data, reading stale or uninitialized values.

---

## Notes

- **SPSC is the simplest lock-free pattern:** Only one reader and one writer, so no CAS retries are needed. Perfect for audio pipelines, logging, inter-thread communication.
- **Power-of-two size** enables `index & (N-1)` instead of `index % N` - a single AND instruction instead of an expensive division.
- **Cache line separation:** Align `write_idx` and `read_idx` to different cache lines (64 bytes) to avoid false sharing between producer and consumer threads.
- **x86 hides bugs:** x86's strong memory model makes relaxed-ordering bugs extremely rare on that architecture. Always test on ARM or with TSan.
- **MPMC (multi-producer multi-consumer)** ring buffers require CAS on indices and are significantly more complex.
- Compile with `-std=c++17 -O2 -pthread -fsanitize=thread`.
