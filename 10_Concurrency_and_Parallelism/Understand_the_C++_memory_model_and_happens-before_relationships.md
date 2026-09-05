# Understand the C++ memory model and happens-before relationships

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/memory_order>  

---

## Topic Overview

The **C++ memory model** (since C++11) defines how memory accesses behave in multi-threaded programs. It specifies when writes by one thread become visible to reads by other threads, and what constitutes undefined behavior. This is one of the genuinely hard corners of C++ - the reason it trips people up is that your intuition is built on single-threaded execution, where "what you write is immediately visible." In a multi-threaded world, that intuition is wrong. Compilers reorder instructions, CPUs execute out of order, and memory caches are not instantly synchronized. The memory model is C++'s formal way of telling you what you can rely on.

### Key Relationships

These three relationships build on each other. Think of them as layers of ordering guarantee:

```cpp
Sequenced-before (within one thread):
  A ; B  ->  A is sequenced-before B (program order)

Synchronizes-with (between threads):
  Thread 1: x.store(1, release)  -synchronizes-with->  Thread 2: x.load(acquire)

Happens-before (transitivity):
  If A happens-before B, and B happens-before C, then A happens-before C.

  A seq-before B, B sync-with C, C seq-before D
  => A happens-before D
```

The critical chain is: sequenced-before + synchronizes-with + sequenced-before = happens-before across threads. That transitivity is how you reason about visibility in concurrent code.

### Memory Orders Summary

If the table feels dense, here's the mental model: `relaxed` only guarantees the operation itself is atomic; `acquire`/`release` guarantee that memory before a release is visible after a matching acquire; `seq_cst` gives a single total order visible to all threads.

| Order | Guarantee | Cost (x86) | Cost (ARM) |
| --- | --- | --- | --- |
| `relaxed` | Atomicity only, no ordering | Free | Free |
| `acquire` | Reads after this see all writes before paired release | Free | `ldar` |
| `release` | Writes before this are visible after paired acquire | Free | `stlr` |
| `acq_rel` | Both acquire and release | Free | `ldar`+`stlr` |
| `seq_cst` | Total order across all threads | `MFENCE` or `LOCK XCHG` | `DMB ISH` |

### Data Race = Undefined Behavior

A data race is not just "probably wrong" - it is fully undefined behavior. The compiler is allowed to assume data races don't exist, and it will optimize based on that assumption. If you violate it, all bets are off.

```cpp
Two accesses to the same memory location form a DATA RACE if:

1. At least one is a WRITE
2. They are NOT ordered by happens-before
3. At least one is NOT atomic

Data race -> UNDEFINED BEHAVIOR (compiler may optimize as if single-threaded)
```

---

## Self-Assessment

### Q1: Explain what data race means in the C++ standard and why it is undefined behavior

**Answer:**

The example below shows a race, and then three safe alternatives. Pay attention to the comment explaining *why* the compiler is allowed to do unexpected things:

```cpp
#include <thread>
#include <iostream>

// === DATA RACE: undefined behavior ===
int shared = 0; // non-atomic

void writer() { shared = 42; }
void reader() { std::cout << shared << "\n"; }

int main() {
    std::thread t1(writer);
    std::thread t2(reader);
    t1.join();
    t2.join();
    // This is a DATA RACE:
    // - writer() writes to 'shared' (non-atomic)
    // - reader() reads from 'shared' (non-atomic)
    // - No happens-before relationship between them
    // - At least one is a write
    //
    // RESULT: Undefined Behavior. Possible outcomes:
    // 1. Prints 0 (old value)
    // 2. Prints 42 (new value)
    // 3. Prints garbage (torn read)
    // 4. Crashes
    // 5. The compiler REMOVES the read entirely (it's UB, anything goes)
    //
    // Why UB? Because the C++ abstract machine assumes:
    // "If there's no synchronization, no other thread modifies this variable."
    // The compiler optimizes based on this assumption.
    // Breaking the assumption invalidates all compiler reasoning.
}

// === NOT a data race (correctly synchronized): ===
// 1. Using atomic:
//    std::atomic<int> shared{0};  -> atomic operations, not a race
//
// 2. Using mutex:
//    std::mutex m;
//    { lock_guard l(m); shared = 42; }   -> happens-before via mutex
//    { lock_guard l(m); cout << shared; } -> ordered after the write
//
// 3. Using join:
//    t1.join();  -> thread completion happens-before join returns
//    cout << shared;  -> safe if only t1 wrote to shared
```

Here are three common data race patterns worth memorizing, because they appear in real code constantly:

```cpp
Pattern 1: Unprotected counter
  Thread A: counter++      Thread B: counter++
  -> Both read-modify-write without synchronization -> UB

Pattern 2: Flag without atomic
  Thread A: done = true    Thread B: while(!done) {}
  -> Compiler may hoist !done out of loop (assumes single-threaded)
  -> Infinite loop even after Thread A sets done!

Pattern 3: Publishing a pointer
  Thread A: data=42; ptr=&data;   Thread B: if(ptr) use(*ptr);
  -> Without release/acquire, Thread B might see ptr!=null
     but read stale data (data==0) - reordered stores
```

### Q2: Show a happens-before chain between a store and a load using acquire/release semantics

**Answer:**

The comments in this example lay out the full chain step by step. Read them alongside the code rather than after it - that's how the chain becomes concrete:

```cpp
#include <atomic>
#include <thread>
#include <cassert>
#include <iostream>

std::atomic<bool> ready{false};
int payload = 0; // non-atomic - safe due to happens-before

void producer() {
    // Step A: write payload (sequenced-before Step B)
    payload = 42;

    // Step B: release store on 'ready'
    ready.store(true, std::memory_order_release);
    //
    // Happens-before chain:
    // A seq-before B (within producer thread)
}

void consumer() {
    // Step C: acquire load on 'ready'
    while (!ready.load(std::memory_order_acquire))
        ;
    // B synchronizes-with C (release-acquire pair on 'ready')

    // Step D: read payload (sequenced-after Step C)
    assert(payload == 42); // GUARANTEED
    std::cout << "payload = " << payload << "\n";
    //
    // Full chain: A seq-before B sync-with C seq-before D
    // Therefore: A happens-before D
    // payload=42 is guaranteed visible at Step D
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    std::cout << "Happens-before chain verified\n";
    // Output:
    // payload = 42
    // Happens-before chain verified
}

// === VISUAL: the happens-before chain ===
//
// Thread 1 (producer)     Thread 2 (consumer)
// ------------------      ------------------
// payload = 42
//   | (sequenced-before)
//   v
// ready.store(release) --synchronizes-with--> ready.load(acquire)
//                                               | (sequenced-before)
//                                               v
//                                             assert(payload==42)
//
// Chain: payload=42 -> store(release) -> load(acquire) -> read payload
//        =========================================================
//                    HAPPENS-BEFORE (transitive)
```

### Q3: Explain why double-checked locking without atomics is broken and how to fix it

**Answer:**

Double-checked locking is a famous trap. The idea feels reasonable - check whether the singleton exists before taking the lock, and check again inside the lock. The problem is that object construction and pointer assignment are two separate steps, and the compiler or CPU is allowed to reorder them. Another thread can see the pointer as non-null before the object it points to has finished being constructed.

```cpp
#include <atomic>
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>

struct Expensive {
    int data[100];
    Expensive() { for (auto& d : data) d = 42; }
};

// === BROKEN: Double-checked locking without atomics ===
// (This was the standard pattern in pre-C++11 code - it's WRONG)
namespace broken {
    Expensive* instance = nullptr; // non-atomic pointer
    std::mutex mtx;

    Expensive* get() {
        if (!instance) {              // CHECK #1: no synchronization!
            std::lock_guard lock(mtx);
            if (!instance) {          // CHECK #2: under lock
                instance = new Expensive();
                // PROBLEM: compiler/CPU may reorder:
                // 1. Allocate memory
                // 2. Store pointer to instance <- other threads see non-null
                // 3. Construct object           <- but object isn't ready yet!
                //
                // Another thread sees instance != null at CHECK #1
                // but the object is half-constructed -> UB
            }
        }
        return instance;
    }
}

// === FIXED: using std::atomic with acquire/release ===
namespace fixed_atomic {
    std::atomic<Expensive*> instance{nullptr};
    std::mutex mtx;

    Expensive* get() {
        Expensive* tmp = instance.load(std::memory_order_acquire);
        // acquire: if we see non-null, we also see the complete construction

        if (!tmp) {
            std::lock_guard lock(mtx);
            tmp = instance.load(std::memory_order_relaxed); // under lock
            if (!tmp) {
                tmp = new Expensive();
                instance.store(tmp, std::memory_order_release);
                // release: construction happens-before this store
                // Other threads' acquire loads will see the
                // fully constructed object
            }
        }
        return tmp;
    }
}

// === BEST: use call_once (simplest and correct) ===
namespace best {
    std::once_flag flag;
    Expensive* instance = nullptr;

    Expensive* get() {
        std::call_once(flag, [] {
            instance = new Expensive();
        });
        return instance;
    }
}

// === SIMPLEST: Meyer's singleton (C++11 guarantees thread safety) ===
namespace simplest {
    Expensive& get() {
        static Expensive instance; // thread-safe in C++11+
        return instance;
    }
}

int main() {
    // Test all versions
    constexpr int THREADS = 8;
    std::vector<std::thread> threads;

    for (int t = 0; t < THREADS; ++t) {
        threads.emplace_back([] {
            auto* p = fixed_atomic::get();
            std::cout << "Got instance at " << p
                      << " data[0]=" << p->data[0] << "\n";
        });
    }
    for (auto& t : threads) t.join();
    // Output: All threads see the same address, data[0]=42

    delete fixed_atomic::instance.load();
}
```

Here is exactly what goes wrong and how the fix addresses it:

```cpp
The compiler and CPU may reorder the construction sequence:

  Standard sequence:         Possible reordering:

  1. allocate memory         1. allocate memory
  2. construct object        2. store ptr (non-null now!)
  3. store ptr               3. construct object <- NOT YET DONE

  Thread A: reaches step 2 (reordered), stores non-null ptr
  Thread B: CHECK #1 sees instance != null -> returns it!
  Thread B: uses half-constructed object -> UNDEFINED BEHAVIOR

The fix: acquire/release ordering creates a happens-before relationship.
  store(release) ensures construction completes BEFORE the pointer is published.
  load(acquire) ensures the reading thread sees the complete object.
```

---

## Notes

- **The C++ memory model is hardware-agnostic.** It defines an abstract machine. The compiler maps it to hardware-specific instructions.
- **x86 is "almost" sequentially consistent:** loads have acquire semantics, stores have release semantics by default. Only store-load reordering is possible without `MFENCE`.
- **ARM/POWER are weakly ordered:** All reorderings are possible. acquire/release compile to real instructions. This is why bugs that hide on x86 can surface on ARM.
- **`seq_cst` is the default** memory order for all atomic operations. It's safe but may be slower than needed.
- **Rule of thumb:** Start with `seq_cst`, profile, then weaken to `acquire`/`release` where needed and provably correct.
- **ThreadSanitizer** (`-fsanitize=thread`) detects data races at runtime. Always test concurrent code with TSan.
- Compile with `-std=c++11 -O2 -pthread -fsanitize=thread`.
