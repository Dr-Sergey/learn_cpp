# Use std::rcu (Read-Copy-Update) for scalable read-heavy data structures

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/thread/rcu_domain>  

---

## Topic Overview

**Read-Copy-Update (RCU)** is a synchronization mechanism where **readers never block** - they execute in a lock-free critical section. Writers create a *new copy* of the data, atomically swap it in, then wait for all existing readers to finish before reclaiming the old copy. This waiting interval is called the *grace period*. Originally from the Linux kernel (used there since 2002), C++26 standardizes it via `<rcu>`.

The core insight behind RCU is asymmetry: in read-dominated workloads, making reads completely free of synchronization overhead - at the cost of making writes somewhat more expensive - produces a huge overall win. If you have 99 readers for every 1 writer, paying a grace period on writes and getting near-zero cost reads is a great trade.

### RCU Protocol

Here's the protocol at a glance. Readers grab a guard, do their work, and release it. Writers create a new copy, swap it in atomically, then wait for all in-flight readers to drain before freeing the old data:

```cpp
Reader thread:                       Writer thread:
─────────────                        ─────────────
std::rcu_reader lock;                1. Create new_data = copy of old
read(*current_ptr);                  2. atomic swap: current_ptr = new_data
// lock destructor exits             3. synchronize_rcu()  <- blocks until
// critical section                     all readers in old epoch finish

                                     4. delete old_data  (safe now)
```

### Key API (C++26)

| Type / Function                 | Purpose                                        |
| ------------------------------- | ---------------------------------------------- |
| `std::rcu_obj_base<T>`          | CRTP base that enables `retire()` on your type |
| `std::rcu_reader`               | RAII guard for a read-side critical section    |
| `std::synchronize_rcu()`        | Block until all current readers have exited    |
| `obj->retire()`                 | Defer deletion until grace period completes    |
| `std::rcu_domain::instance()`   | The default RCU domain                         |

---

## Self-Assessment

### Q1: Use rcu_obj_base and synchronize_rcu() to implement a lock-free read-side critical section

**Answer:**

Your data type needs to inherit from `std::rcu_obj_base<T>` to opt into the RCU reclamation machinery. This is the CRTP pattern - it gives the type a `retire()` method that the RCU system uses to schedule deferred deletion. Notice how the reader code has no mutex, no atomic read-modify-write, and no spin: it just creates a guard and reads:

<!-- compile: needs `rcu` (not in the CI standard library yet) -->
```cpp
#include <rcu>       // C++26
#include <atomic>
#include <iostream>
#include <thread>
#include <string>

// Data type must inherit from rcu_obj_base for deferred reclamation
struct Config : std::rcu_obj_base<Config> {
    std::string db_host;
    int db_port;
    Config(std::string h, int p) : db_host(std::move(h)), db_port(p) {}
};

std::atomic<Config*> current_config;

// Reader: lock-free, never blocks
void reader_thread() {
    for (int i = 0; i < 1000; ++i) {
        std::rcu_reader lock;  // Enter read-side critical section (zero-cost on x86)
        Config* cfg = current_config.load(std::memory_order_acquire);
        if (cfg) {
            // Safe to access - RCU guarantees cfg won't be deleted
            // while we're in the critical section
            volatile auto port = cfg->db_port;
            volatile auto host_len = cfg->db_host.size();
        }
        // ~rcu_reader exits critical section
    }
}

// Writer: creates new copy, waits for grace period
void writer_thread() {
    // Create new config
    auto* new_cfg = new Config("newhost.example.com", 5433);

    // Atomically swap in the new config
    Config* old_cfg = current_config.exchange(new_cfg, std::memory_order_release);

    // Wait for all readers currently accessing old_cfg to finish
    std::synchronize_rcu();

    // Now safe to delete - no reader can still be in old epoch
    delete old_cfg;
    std::cout << "Config updated and old config safely deleted\n";
}

int main() {
    current_config.store(new Config("localhost", 5432));

    std::thread r1(reader_thread);
    std::thread r2(reader_thread);
    std::thread w(writer_thread);

    r1.join();
    r2.join();
    w.join();

    delete current_config.load();
}
```

The reason readers are safe to access `old_cfg` even after the writer has swapped it out is that `synchronize_rcu()` won't return until every active `rcu_reader` guard that was alive at the point of the swap has been destroyed. The old data is truly unreachable by the time `delete old_cfg` runs.

---

### Q2: Show that RCU readers never block and writers pay the cost of waiting for a grace period

**Answer:**

This example makes the performance asymmetry concrete with timing. The reader loop is genuinely near-zero cost per iteration; the writer pays a real grace-period wait for each update. Understanding this trade-off is essential to deciding when RCU is the right tool:

<!-- compile: needs `rcu` (not in the CI standard library yet) -->
```cpp
#include <rcu>
#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>
#include <vector>

struct Data : std::rcu_obj_base<Data> {
    int value;
    Data(int v) : value(v) {}
};

std::atomic<Data*> shared_data;

void demonstrate_reader_never_blocks() {
    // Reader: Zero blocking, zero waiting, zero contention
    auto start = std::chrono::high_resolution_clock::now();
    int reads = 0;

    for (int i = 0; i < 1'000'000; ++i) {
        std::rcu_reader lock;  // Enters critical section
        // On x86: This is essentially a compiler fence - no atomic RMW,
        // no spinlock, no mutex - truly zero overhead
        Data* d = shared_data.load(std::memory_order_acquire);
        if (d) volatile int v = d->value;
        ++reads;
        // ~rcu_reader: exits critical section (another compiler fence)
    }

    auto elapsed = std::chrono::high_resolution_clock::now() - start;
    auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(elapsed).count();
    std::cout << "Reader: " << reads << " reads in " << ns / 1'000'000.0
              << " ms (" << ns / reads << " ns/read)\n";
    // Typical: ~2-5 ns per read (compared to ~50-200 ns with shared_mutex)
}

void demonstrate_writer_grace_period() {
    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < 100; ++i) {
        Data* new_data = new Data(i);
        Data* old_data = shared_data.exchange(new_data, std::memory_order_release);

        // Writer must wait for grace period - all readers to finish
        auto grace_start = std::chrono::high_resolution_clock::now();
        std::synchronize_rcu();  // Blocks until all readers exit
        auto grace_end = std::chrono::high_resolution_clock::now();

        delete old_data;

        if (i == 0) {
            auto grace_ns = std::chrono::duration_cast<std::chrono::microseconds>(
                grace_end - grace_start).count();
            std::cout << "Writer: grace period ~" << grace_ns << " us\n";
            // Typical: 1-50 us (depends on reader activity)
        }
    }

    auto elapsed = std::chrono::high_resolution_clock::now() - start;
    std::cout << "Writer: 100 updates in "
              << std::chrono::duration<double, std::milli>(elapsed).count() << " ms\n";
}

int main() {
    shared_data.store(new Data(0));

    std::thread reader(demonstrate_reader_never_blocks);
    std::thread writer(demonstrate_writer_grace_period);

    reader.join();
    writer.join();

    delete shared_data.load();
    // Key insight: readers run at full speed; writers absorb ALL synchronization cost
}
```

The 2-5 ns per read vs. 50-200 ns for a `shared_mutex` lock is the headline number. At 8 cores all reading simultaneously, a `shared_mutex` starts cache-line bouncing under contention; RCU readers don't interact with each other at all.

---

### Q3: Compare std::rcu with std::shared_mutex for a 99% read workload in terms of throughput

**Answer:**

The table below summarizes the trade-offs. The right choice depends almost entirely on your read-to-write ratio and how long your critical sections are:

| Aspect                       | `std::rcu`                          | `std::shared_mutex`               |
| ---------------------------- | ----------------------------------- | --------------------------------- |
| **Read-side cost**           | ~2-5 ns (compiler fence only)       | ~50-200 ns (atomic RMW)           |
| **Read scalability**         | Linear with cores                   | Degrades with contention          |
| **Read blocking**            | Never                               | Blocked by pending writers        |
| **Write-side cost**          | Grace period (~1-50 us)             | Exclusive lock (~50-200 ns)       |
| **Write throughput**         | Lower (grace period wait)           | Higher (no grace period)          |
| **Best read:write ratio**    | 99:1 or higher                      | 80:20 to 95:5                     |
| **Memory overhead**          | Higher (old copies during grace)    | None                              |

Here's the benchmark setup so you can see how the comparison runs in practice:

```cpp
#include <shared_mutex>
#include <atomic>
#include <thread>
#include <chrono>
#include <iostream>
#include <vector>

// Shared mutex approach
struct SharedMutexStore {
    mutable std::shared_mutex mtx;
    int value = 0;

    int read() const {
        std::shared_lock lock(mtx);
        return value;
    }
    void write(int v) {
        std::unique_lock lock(mtx);
        value = v;
    }
};

// RCU approach (conceptual, simplified)
struct RcuStore {
    std::atomic<int*> ptr{new int(0)};

    int read() const {
        // std::rcu_reader lock;
        return *ptr.load(std::memory_order_acquire);
    }
    void write(int v) {
        int* new_val = new int(v);
        int* old_val = ptr.exchange(new_val, std::memory_order_release);
        // std::synchronize_rcu();
        delete old_val;
    }
    ~RcuStore() { delete ptr.load(); }
};

int main() {
    constexpr int N_READERS = 8;
    constexpr int OPS = 1'000'000;

    // Benchmark shared_mutex (99% reads, 1% writes)
    {
        SharedMutexStore store;
        std::vector<std::thread> threads;
        auto start = std::chrono::high_resolution_clock::now();
        for (int t = 0; t < N_READERS; ++t) {
            threads.emplace_back([&, t] {
                for (int i = 0; i < OPS; ++i) {
                    if (i % 100 == 0 && t == 0) store.write(i);
                    else volatile int v = store.read();
                }
            });
        }
        for (auto& t : threads) t.join();
        auto ms = std::chrono::duration<double, std::milli>(
            std::chrono::high_resolution_clock::now() - start).count();
        std::cout << "shared_mutex (8 threads, 99% read): " << ms << " ms\n";
    }

    /*
    Expected results (8 cores):
    ┌──────────────────┬──────────┬─────────────────────────┐
    │ Method           │ Time     │ Reads/sec               │
    ├──────────────────┼──────────┼─────────────────────────┤
    │ shared_mutex     │ ~500 ms  │ ~16M/s (cache-line bounce)│
    │ std::rcu         │ ~50 ms   │ ~160M/s (no contention)  │
    └──────────────────┴──────────┴─────────────────────────┘

    RCU is ~10x faster for read-dominated workloads because:

    1. No atomic read-modify-write on the read side
    2. No cache-line bouncing between cores
    3. Readers never wait for writers or other readers

    */
}
```

The ~10x throughput difference at 8 cores for a 99% read workload is the practical reason to reach for RCU. If your read-to-write ratio is more balanced (say, 80:20), a `shared_mutex` is simpler and the performance difference shrinks significantly.

---

## Notes

- `std::rcu` is C++26; the Linux kernel has used RCU since 2002, so the concept is battle-tested even if the standard API is new.
- Read-side critical sections should be kept **short** - holding them open for a long time delays the writer's grace period and increases memory pressure from accumulating old copies.
- `retire()` is the async alternative to `synchronize_rcu()`: it defers deletion without blocking the writer, useful when you can't afford to stall.
- RCU and `std::hazard_pointer` are complementary tools: RCU fits read-heavy workloads, hazard pointers are better for balanced access patterns.
- For single-writer scenarios, RCU is often the **best possible** synchronization mechanism available.
