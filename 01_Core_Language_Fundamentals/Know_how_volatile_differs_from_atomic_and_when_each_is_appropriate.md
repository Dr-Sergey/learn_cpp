# Know how volatile differs from atomic and when each is appropriate

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/cv>  

---

## Topic Overview

### volatile vs. atomic - Fundamentally Different

This is one of the most misunderstood pairs in all of C++. Both keywords stop the compiler from doing *something*, so people lump them together - but they solve genuinely different problems and are almost never interchangeable:

| Feature | `volatile` | `std::atomic<T>` |
| --- | --- | --- |
| **Purpose** | Prevent compiler from optimizing away memory accesses | Thread-safe access to shared data |
| **Atomicity** | No guarantee | Guaranteed (lock-free or with internal lock) |
| **Memory ordering** | No ordering between threads | Sequential consistency by default |
| **Prevents reordering** | Only compiler reordering of volatile accesses | Both compiler and CPU reordering |
| **Thread safety** | Not safe for inter-thread communication | Designed for multi-threaded code |
| **Use case** | Hardware registers, signal handlers | Shared flags, counters, lock-free data structures |

The one-line summary: `volatile` is for memory that changes *outside your program* (hardware); `std::atomic` is for memory shared *between threads*.

### What `volatile` Actually Does

`volatile` is a promise *to the compiler*: "this variable can change at any moment for reasons you can't see - so never cache it in a register, always go to memory."

```cpp
volatile int* hardware_register = reinterpret_cast<volatile int*>(0x40021000);

// Without volatile: compiler may optimize to a single read
// With volatile: compiler reads from memory address EVERY time
int status = *hardware_register;     // Forced memory read #1
int status2 = *hardware_register;    // Forced memory read #2 (may differ!)
```

And here's the crucial part - the list of things `volatile` does **not** do, which is where every misuse comes from:

- It does *not* stop the CPU from reordering instructions.
- It does *not* make reads/writes atomic (a 64-bit write might happen as two 32-bit writes).
- It does *not* create happens-before relationships between threads.
- It does *not* prevent data races - using `volatile` for threading is outright UB.

### What `std::atomic` Does

`std::atomic` is the tool that actually covers what people *wish* `volatile` did. It gives you:

1. **Atomicity** - operations are indivisible.
2. **Memory ordering** - it establishes happens-before relationships.
3. **Visibility** - a write in one thread is guaranteed visible to a read in another.

```cpp
#include <atomic>
std::atomic<int> counter{0};

// Thread 1:
counter.fetch_add(1);  // Atomic increment - indivisible, visible to all threads

// Thread 2:
int val = counter.load();  // Guaranteed to see Thread 1's write (or later)
```

### C++20: Deprecated `volatile` Compound Operations

C++20 cleaned house by deprecating `volatile` operations that never had coherent meaning - chiefly compound assignments, which imply an atomic read-modify-write that `volatile` was never able to deliver:

```cpp
volatile int v = 0;
v++;        // DEPRECATED in C++20 (compound assignment on volatile)
v += 5;     // DEPRECATED in C++20
int x = v;  // OK - simple read
v = 42;     // OK - simple write
```

---

## Self-Assessment

### Q1: Explain that `volatile` prevents optimization of reads/writes but does not provide atomicity or ordering

The first block shows what `volatile` *does* (forces every read); the two comments after it show the two things it conspicuously *doesn't* (atomicity, ordering):

```cpp
#include <iostream>

volatile int sensor_value = 0;

void read_sensor() {
    // Without volatile: compiler could optimize this to a single read or even
    // replace with a constant if it proves sensor_value isn't written in this function
    int a = sensor_value;   // Forced read from memory
    int b = sensor_value;   // Another forced read from memory (may differ!)
    int c = sensor_value;   // Yet another forced read

    // With a non-volatile int, the compiler could do:
    // int a = sensor_value;  // One read
    // int b = a;             // Reuse cached value
    // int c = a;             // Reuse cached value
    std::cout << a << " " << b << " " << c << "\n";
}

// But volatile provides NO atomicity:
volatile long long big_val = 0;
// On a 32-bit system, reading big_val may require two 32-bit loads.
// Another thread could modify big_val between the two loads -> torn read!

// And volatile provides NO ordering:
volatile int x = 0, y = 0;
// Thread 1:          Thread 2:
// x = 1;             if (y == 1)
// y = 1;                 assert(x == 1); // MAY FAIL! CPU can reorder
```

**Key takeaways:**

- `volatile` is purely a deal with the **compiler** - it forces memory traffic but emits no CPU fence instructions.
- Modern CPUs reorder loads and stores in hardware, and `volatile` does nothing to stop that.
- A 64-bit `volatile` on a 32-bit platform can be **torn** - read or written in two separate halves.
- Bottom line: `volatile` alone is *never* enough for safe communication between threads.

### Q2: Show a case where `volatile` is appropriate: memory-mapped hardware registers

Now the legitimate use. On bare metal, a hardware register's bits change because the *device* changed them, not because your code did - so the compiler absolutely must re-read it each time:

<!-- compile: norun: bare-metal register code; main waits for Ctrl-C -->
```cpp
#include <cstdint>

// Memory-mapped I/O for an embedded UART peripheral
struct UART_Registers {
    volatile uint32_t DR;     // Data register - changes when data arrives
    volatile uint32_t SR;     // Status register - hardware updates these bits
    volatile uint32_t CR;     // Control register - writing changes hardware behavior
    volatile uint32_t BRR;    // Baud rate register
};

// Map to hardware address (platform-specific)
auto* const uart1 = reinterpret_cast<UART_Registers*>(0x40011000);

void uart_send_byte(uint8_t byte) {
    // Wait for transmit buffer empty - SR changes by HARDWARE
    // Without volatile, compiler would read SR once and loop forever
    while (!(uart1->SR & (1 << 7))) {
        // busy-wait: SR is volatile, so it's re-read every iteration
    }
    uart1->DR = byte;  // Write to data register triggers hardware transmission
}

uint8_t uart_receive_byte() {
    // Wait for receive buffer not empty
    while (!(uart1->SR & (1 << 5))) {
        // busy-wait: must re-read SR from hardware every time
    }
    return static_cast<uint8_t>(uart1->DR);  // Read clears the RXNE flag in hardware
}

// Another valid use: signal handlers (limited)
#include <csignal>

volatile std::sig_atomic_t signal_received = 0;

void signal_handler(int sig) {
    signal_received = 1;  // volatile ensures the write isn't optimized away
}

int main() {
    std::signal(SIGINT, signal_handler);
    while (!signal_received) {
        // volatile ensures signal_received is re-read each iteration
    }
    return 0;
}
```

**Why `volatile` is the right call here:**

- The registers change *independently of your program*, so caching them would be a bug - `volatile` forbids the caching.
- This is single-threaded bare-metal code, so there's no atomicity question to answer.
- For signal handlers, `volatile sig_atomic_t` is precisely the type the standard mandates.

### Q3: Show a case where `volatile` is wrong but `std::atomic` is correct

This side-by-side is the heart of the topic. The `wrong` namespace uses `volatile` for thread communication and has two distinct bugs; the `correct` namespace fixes both with `std::atomic` and acquire/release ordering:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

// ===== WRONG: using volatile for inter-thread communication =====
namespace wrong {
    volatile bool stop_flag = false;  // volatile: NO atomicity, NO ordering!
    volatile int shared_data = 0;

    void producer() {
        shared_data = 42;     // Write data
        stop_flag = true;     // Signal "data is ready"
        // BUG: CPU may reorder these writes! Consumer may see stop_flag=true
        // but shared_data still 0
    }

    void consumer() {
        while (!stop_flag) {} // Spin-wait
        // BUG: even if we see stop_flag==true, shared_data may not be 42 yet
        std::cout << "wrong: " << shared_data << "\n";  // May print 0!
    }
}

// ===== CORRECT: using std::atomic =====
namespace correct {
    std::atomic<bool> stop_flag{false};
    int shared_data = 0;  // Protected by the atomic flag (acquire-release)

    void producer() {
        shared_data = 42;                          // Write data
        stop_flag.store(true, std::memory_order_release);  // Release: all prior writes visible
    }

    void consumer() {
        while (!stop_flag.load(std::memory_order_acquire)) {} // Acquire: sees all writes before release
        std::cout << "correct: " << shared_data << "\n";  // GUARANTEED to print 42
    }
}

// ===== WRONG: using volatile for a shared counter =====
namespace counter_wrong {
    volatile int count = 0;

    void increment() {
        for (int i = 0; i < 100000; ++i)
            count++;  // NOT atomic! Read-modify-write is 3 steps: load, add, store
                      // Two threads can load the same value -> lost updates
    }
}

// ===== CORRECT: using atomic for a shared counter =====
namespace counter_correct {
    std::atomic<int> count{0};

    void increment() {
        for (int i = 0; i < 100000; ++i)
            count.fetch_add(1, std::memory_order_relaxed);  // Atomic RMW operation
    }
}

int main() {
    // Test the correct flag pattern:
    {
        std::thread t1(correct::producer);
        std::thread t2(correct::consumer);
        t1.join(); t2.join();
        // Always prints: correct: 42
    }

    // Test the correct counter:
    {
        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i)
            threads.emplace_back(counter_correct::increment);
        for (auto& t : threads) t.join();
        std::cout << "Atomic counter: " << counter_correct::count << "\n";
        // Always prints: 400000
    }

    return 0;
}
```

**Why `volatile` fails for threading:**

1. **No atomicity:** `count++` on a `volatile int` is still load -> add -> store. Two threads can interleave those steps and clobber each other's updates.
2. **No memory ordering:** nothing stops the CPU from moving `shared_data = 42` *after* `stop_flag = true`, so the consumer can see the flag but read stale data.
3. **No guaranteed visibility:** on some architectures a non-atomic write can sit in a store buffer and never reach another core.

**Why `std::atomic` works:**

1. `fetch_add` is a single indivisible read-modify-write.
2. `store(release)` paired with `load(acquire)` builds a happens-before relationship across threads.
3. Everything written *before* a release store is guaranteed visible *after* the matching acquire load - which is what makes the plain `shared_data` safe.

---

## Additional Examples

### When You Need Both volatile AND atomic

Occasionally - and it really is rare - embedded code needs both qualities at once, like a memory-mapped register that's also touched concurrently:

```cpp
// C++11 doesn't support volatile atomic directly.
// C++20 deprecated volatile operations on atomics.
// For hardware registers that need both:
#include <atomic>

// Use volatile pointer to atomic:
volatile std::atomic<uint32_t>* hw_semaphore =
    reinterpret_cast<volatile std::atomic<uint32_t>*>(0x40002000);

// Or better: use atomic_ref (C++20) with a volatile access:
```

### Summary Decision Table

When in doubt, this table answers the question directly:

| Scenario | Use `volatile`? | Use `std::atomic`? |
| --- | --- | --- |
| Memory-mapped hardware register | Yes | No |
| Signal handler flag | `volatile sig_atomic_t` | No |
| Shared flag between threads | No | Yes |
| Shared counter between threads | No | Yes |
| Lock-free queue | No | Yes |
| DMA buffer | Yes | Depends |

---

## Notes

- **`volatile` is not thread-safe.** This is the single biggest misconception in C++.
- If you're coming from Java or C#, beware: *their* `volatile` provides atomicity and ordering. C++ `volatile` does not.
- C++20 deprecated compound operations on `volatile` (like `volatile_var++`).
- For threading, always reach for `std::atomic`, `std::mutex`, or another synchronization primitive.
- For hardware, use `volatile` and nothing else - `std::atomic` may insert fences you don't want on a single-threaded device.
