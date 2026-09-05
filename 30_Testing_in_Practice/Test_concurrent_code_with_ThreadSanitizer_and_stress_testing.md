# Test concurrent code with ThreadSanitizer and stress testing

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Concurrency bugs are among the hardest category of bugs to deal with because they're **non-deterministic**. A data race might only manifest when two threads happen to be scheduled in a particular order, which can depend on load, CPU speed, OS scheduling, and a dozen other factors you can't fully control. The same buggy code might run fine in development, pass all your tests, and then fail in production under higher load.

The reason this trips people up is the false sense of security that comes from "my multithreaded code worked 1000 times in a row." That's not evidence of correctness - it's evidence that the particular race you have wasn't triggered by those 1000 runs. The bug is still there.

Two strategies complement each other here. **ThreadSanitizer (TSan)** instruments every memory access at compile time and reports data races deterministically whenever they occur during a run - if a race is reachable by your test, TSan will find it. **Stress testing** uses many threads and tight loops to maximize the probability of triggering timing-dependent bugs that might not show up with normal thread counts and workloads. You want both: TSan for the races it can find precisely, stress testing for the emergent timing issues it exposes.

### Detection Strategies

| Strategy | What It Finds | Deterministic? | Overhead |
| --- | --- | --- | --- |
| **TSan** | Data races, lock ordering | Yes (instruments all accesses) | 5-15x slower |
| **Stress testing** | Timing-dependent bugs, starvation | No (probabilistic) | Varies |
| **Helgrind** (Valgrind) | Races, lock misuse | Yes | 20-100x slower |
| **Code review** | Design-level issues | N/A | Human time |
| **Model checking** | Exhaustive state exploration | Yes | Exponential |

---

## Self-Assessment

### Q1: Write tests that TSan can detect races in, and show how to fix them

**Answer:**

The key to using TSan effectively is to write your tests in a way that exercises the concurrent paths. If your test never actually has two threads writing to the same variable at the same time, TSan has nothing to report. The test below deliberately creates the race condition so TSan can catch it:

```cpp
#include <gtest/gtest.h>
#include <thread>
#include <mutex>
#include <atomic>
#include <vector>
#include <numeric>

// BUG: Data race on shared counter
class BuggyCounter {
public:
    void increment() { count_++; }  // RACE: no synchronization
    int get() const { return count_; }
private:
    int count_ = 0;  // Shared, mutable, not synchronized
};

TEST(DataRaceTest, BuggyCounterHasRace) {
    BuggyCounter counter;
    constexpr int THREADS = 4;
    constexpr int INCREMENTS = 10000;

    std::vector<std::thread> threads;
    for (int i = 0; i < THREADS; ++i) {
        threads.emplace_back([&] {
            for (int j = 0; j < INCREMENTS; ++j)
                counter.increment();
        });
    }
    for (auto& t : threads) t.join();

    // Without TSan, this may "pass" with wrong count.
    // With TSan: ERROR: data race detected!
    // EXPECT_EQ(counter.get(), THREADS * INCREMENTS);  // Likely fails
}

// FIX 1: Mutex-protected counter
class MutexCounter {
public:
    void increment() {
        std::lock_guard lock(mtx_);
        count_++;
    }
    int get() const {
        std::lock_guard lock(mtx_);
        return count_;
    }
private:
    mutable std::mutex mtx_;
    int count_ = 0;
};

// FIX 2: Atomic counter (better for simple ops)
class AtomicCounter {
public:
    void increment() { count_.fetch_add(1, std::memory_order_relaxed); }
    int get() const { return count_.load(std::memory_order_relaxed); }
private:
    std::atomic<int> count_{0};
};

TEST(DataRaceTest, AtomicCounterIsCorrect) {
    AtomicCounter counter;
    constexpr int THREADS = 4;
    constexpr int INCREMENTS = 10000;

    std::vector<std::thread> threads;
    for (int i = 0; i < THREADS; ++i) {
        threads.emplace_back([&] {
            for (int j = 0; j < INCREMENTS; ++j)
                counter.increment();
        });
    }
    for (auto& t : threads) t.join();

    EXPECT_EQ(counter.get(), THREADS * INCREMENTS);  // Always correct
}
```

Notice that the atomic fix uses `memory_order_relaxed` for a simple increment counter. This is correct here because we only need the atomicity of the operation itself - we don't need any ordering guarantees relative to other memory accesses. Using `relaxed` when it's sufficient is worth doing because stronger orderings carry synchronization overhead that can be significant in hot paths.

### Q2: Design stress tests for concurrent data structures

**Answer:**

A good stress test makes the scheduler work hard. You want many threads, many operations, and minimal artificial delays - the goal is to hit every possible interleaving. The "no lost items" test below verifies that a thread-safe queue never drops elements under heavy producer/consumer contention:

```cpp
#include <gtest/gtest.h>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <chrono>
#include <set>

// Thread-safe queue under test
template<typename T>
class ConcurrentQueue {
public:
    void push(T value) {
        {
            std::lock_guard lock(mtx_);
            queue_.push(std::move(value));
        }
        cv_.notify_one();
    }

    std::optional<T> try_pop() {
        std::lock_guard lock(mtx_);
        if (queue_.empty()) return std::nullopt;
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    T wait_and_pop() {
        std::unique_lock lock(mtx_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    size_t size() const {
        std::lock_guard lock(mtx_);
        return queue_.size();
    }

private:
    mutable std::mutex mtx_;
    std::condition_variable cv_;
    std::queue<T> queue_;
};

// === Stress test: no lost items ===
TEST(ConcurrentQueueStress, NoLostItems) {
    ConcurrentQueue<int> queue;
    constexpr int PRODUCERS = 4;
    constexpr int ITEMS_PER_PRODUCER = 10000;
    constexpr int TOTAL = PRODUCERS * ITEMS_PER_PRODUCER;

    // Producers push unique values
    std::vector<std::thread> producers;
    for (int p = 0; p < PRODUCERS; ++p) {
        producers.emplace_back([&, p] {
            for (int i = 0; i < ITEMS_PER_PRODUCER; ++i)
                queue.push(p * ITEMS_PER_PRODUCER + i);
        });
    }

    // Consumers collect values
    std::atomic<int> consumed{0};
    std::mutex result_mtx;
    std::set<int> results;

    constexpr int CONSUMERS = 4;
    std::vector<std::thread> consumers;
    for (int c = 0; c < CONSUMERS; ++c) {
        consumers.emplace_back([&] {
            while (consumed.load() < TOTAL) {
                auto val = queue.try_pop();
                if (val) {
                    std::lock_guard lock(result_mtx);
                    results.insert(*val);
                    consumed.fetch_add(1);
                }
                // No sleep - tight loop stresses scheduling
            }
        });
    }

    for (auto& t : producers) t.join();
    for (auto& t : consumers) t.join();

    // Verify: every item consumed exactly once
    EXPECT_EQ(results.size(), TOTAL);
}

// === Stress test: producer-consumer with wait ===
TEST(ConcurrentQueueStress, WaitAndPopUnderContention) {
    ConcurrentQueue<int> queue;
    std::atomic<bool> done{false};
    std::vector<int> received;
    std::mutex recv_mtx;

    // Consumer starts BEFORE producer (must not deadlock)
    auto consumer = std::thread([&] {
        while (!done.load() || queue.size() > 0) {
            auto val = queue.try_pop();
            if (val) {
                std::lock_guard lock(recv_mtx);
                received.push_back(*val);
            }
        }
    });

    // Producer sends items with slight delays to vary timing
    for (int i = 0; i < 1000; ++i) {
        queue.push(i);
        if (i % 100 == 0)
            std::this_thread::yield();  // Vary timing
    }
    done.store(true);

    consumer.join();
    EXPECT_EQ(received.size(), 1000);
}
```

Using a `std::set` to collect results (rather than a vector) means the test automatically catches both lost items (size too small) and duplicate items (set deduplicates). The comment "No sleep - tight loop stresses scheduling" is important design intent: adding sleeps would actually reduce the effectiveness of the stress test by giving the scheduler more breathing room.

### Q3: Show TSan integration patterns and suppression of known issues

**Answer:**

Sometimes you have intentionally racy code - approximate performance counters are the classic example. You deliberately skip synchronization because you don't care about the exact count, only the approximate magnitude. TSan will flag these as races unless you annotate them explicitly. The annotation approach below is preferable to simply suppressing the whole function in a suppressions file, because it documents the intentional decision at the point in the code where it matters:

```cpp
// === Annotate intentionally racy code for TSan ===
// Sometimes code has benign races (e.g., performance counters).
// Use TSan annotations to suppress false positives.

#if defined(__SANITIZE_THREAD__) || defined(__has_feature)
#if __has_feature(thread_sanitizer)
#define TSAN_ENABLED 1
#endif
#endif

#ifdef TSAN_ENABLED
#include <sanitizer/tsan_interface.h>
#define TSAN_ANNOTATE_BENIGN_RACE(addr, desc) \
    AnnotateBenignRace(__FILE__, __LINE__, addr, desc)
#else
#define TSAN_ANNOTATE_BENIGN_RACE(addr, desc)
#endif

class PerformanceCounters {
public:
    void record_request() {
        // Benign race: counter is approximate, never used for correctness
        TSAN_ANNOTATE_BENIGN_RACE(&request_count_, "Approximate counter");
        ++request_count_;
    }

    int64_t request_count() const { return request_count_; }

private:
    int64_t request_count_ = 0;  // Intentionally not atomic for speed
};
```

For third-party code and known system library issues, a suppressions file is the right tool. Keep it in your repository so the suppressions are version-controlled:

```bash
# === tsan_suppressions.txt ===
# Known benign race in third-party logging library
race:spdlog::details::registry::instance

# Benign race in our approximate counters
race:PerformanceCounters::record_request

# Known issue in glibc (fixed in newer versions)
race:__cxa_guard_acquire

# Usage:
# TSAN_OPTIONS="suppressions=tsan_suppressions.txt" ./test_runner
```

The suppressions format matches function names by substring, so `race:PerformanceCounters::record_request` suppresses any race where that function appears in either of the conflicting stack traces. Be conservative with suppressions - add them only when you've verified the race is genuinely benign, because a suppression on the wrong function can hide real bugs.

---

## Notes

- TSan must be in a separate build - it's incompatible with ASan and MSan.
- TSan detects races on C++ objects, not on raw hardware - it may miss races in inline assembly.
- Stress tests should run for at least several seconds to exercise scheduling variations.
- Use `std::this_thread::yield()` in stress tests to vary timing patterns.
- `ctest --repeat until-fail:100` is a simple way to stress test non-deterministic code.
- TSan reports are very precise: they show both conflicting accesses with full stack traces.
- False positives in TSan are rare - investigate before suppressing.
- Prefer `std::atomic` for shared counters over manual locking - less error-prone.
