# Understand and avoid TOCTOU races in concurrent code

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange>  

---

## Topic Overview

A **TOCTOU (Time-of-Check to Time-of-Use)** race occurs when a thread checks a condition and then acts on it, but another thread modifies the state between the check and the use. The window between "check" and "act" is enough for the world to change underneath you.

The reason this trips people up is that each individual operation can look perfectly correct in isolation. The load is atomic, the subtraction is atomic - but the *sequence* of two individually-atomic operations is not itself atomic. Another thread can slip in between them.

### The TOCTOU Pattern

```cpp
Thread A                    Thread B
────────                    ────────
CHECK: balance >= 100?
  -> yes
                            CHECK: balance >= 100?
                              -> yes
USE: balance -= 100
                            USE: balance -= 100   <- OVERDRAFT!

Result: balance went negative - both threads passed the check
        but one should have been rejected.
```

### Where TOCTOU Races Happen

| Pattern | Buggy | Correct |
| --- | --- | --- |
| Balance check-then-debit | `if (bal >= x) bal -= x;` | `compare_exchange` or mutex |
| Map check-then-insert | `if (!map.count(k)) map[k]=v;` | `try_emplace` or mutex |
| Atomic decrement-if-positive | `if (a > 0) a--;` | `CAS loop` |
| File check-then-open | `if (exists(f)) open(f);` | `open()` with error handling |

### Fix Strategies

1. **Atomic CAS:** Make check+act a single atomic operation
2. **Mutex scope:** Hold the lock across both check and act
3. **Design elimination:** Use APIs that combine check+act (e.g., `try_emplace`)

---

## Self-Assessment

### Q1: Show a time-of-check to time-of-use race when checking then using a shared resource

**Answer:**

The `BuggyAccount::withdraw` below has a TOCTOU race. The `balance.load()` and `balance.fetch_sub()` are each individually atomic, but there is a gap between them. Look at the comment marking the window:

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

// BUGGY: classic TOCTOU race in a bank account
struct BuggyAccount {
    std::atomic<int> balance{1000};

    bool withdraw(int amount) {
        // CHECK: is balance sufficient?
        if (balance.load() >= amount) {
            // <- TOCTOU WINDOW: another thread can withdraw here
            // USE: debit the account
            balance.fetch_sub(amount);
            return true;
        }
        return false;
    }
};

int main() {
    constexpr int TRIALS = 100'000;
    int overdraft_count = 0;

    for (int trial = 0; trial < TRIALS; ++trial) {
        BuggyAccount account; // balance = 1000

        // Two threads each try to withdraw 600
        // Only ONE should succeed (1000 >= 600, but 1000 < 600+600)
        std::atomic<int> successes{0};

        std::thread t1([&] {
            if (account.withdraw(600)) successes++;
        });
        std::thread t2([&] {
            if (account.withdraw(600)) successes++;
        });
        t1.join();
        t2.join();

        if (successes == 2) {
            // BOTH succeeded -> balance is now -200 (overdraft!)
            ++overdraft_count;
        }
    }

    std::cout << "Overdrafts in " << TRIALS << " trials: "
              << overdraft_count << "\n";
    // Output (typical):
    // Overdrafts in 100000 trials: ~500-5000
    // (varies by timing, but NOT zero - proves the bug)

    // The race: both threads load balance (1000), both see >= 600,
    // both subtract 600 -> final balance = -200.
}
```

**Explanation:** The gap between `balance.load()` (check) and `balance.fetch_sub()` (use) is the TOCTOU window. Even though each operation is individually atomic, the *sequence* is not atomic - another thread can interleave between them.

### Q2: Fix the TOCTOU by using a single atomic compare_exchange operation

**Answer:**

The fix is to collapse the check and the act into a single indivisible operation using a CAS loop. The `compare_exchange_weak` call atomically asks: "is the balance still the value I saw? If so, set it to value-minus-amount." If another thread got there first, `current` is updated with the new value and the loop re-evaluates:

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

struct SafeAccount {
    std::atomic<int> balance{1000};

    bool withdraw(int amount) {
        int current = balance.load(std::memory_order_relaxed);
        do {
            if (current < amount)
                return false; // insufficient funds
            // Try to atomically change balance from 'current' to 'current - amount'
            // If another thread modified balance, CAS fails and 'current' is updated
        } while (!balance.compare_exchange_weak(
            current,             // expected (updated on failure)
            current - amount,    // desired
            std::memory_order_relaxed,
            std::memory_order_relaxed));
        return true;
        // CHECK + USE are now a SINGLE atomic operation (CAS)
        // No TOCTOU window exists
    }
};

int main() {
    constexpr int TRIALS = 100'000;
    int overdraft_count = 0;

    for (int trial = 0; trial < TRIALS; ++trial) {
        SafeAccount account;
        std::atomic<int> successes{0};

        std::thread t1([&] {
            if (account.withdraw(600)) successes++;
        });
        std::thread t2([&] {
            if (account.withdraw(600)) successes++;
        });
        t1.join();
        t2.join();

        if (successes == 2) ++overdraft_count;
    }

    std::cout << "Overdrafts: " << overdraft_count << "\n";
    // Output: Overdrafts: 0
    // ZERO overdrafts - the CAS loop guarantees atomicity

    // === How the CAS loop eliminates TOCTOU ===
    // Thread A loads balance=1000, tries CAS(1000->400) - succeeds
    // Thread B loads balance=1000, tries CAS(1000->400) - FAILS
    //   (because balance is now 400, not 1000)
    //   CAS updates 'current' to 400, retries loop
    //   Now current(400) < amount(600) -> return false (correct)

    // === Generalized pattern ===
    SafeAccount acc;
    acc.balance.store(500);

    // Atomic "increment if below limit"
    auto increment_if_below = [](std::atomic<int>& val, int limit) {
        int cur = val.load();
        do {
            if (cur >= limit) return false;
        } while (!val.compare_exchange_weak(cur, cur + 1));
        return true;
    };

    std::cout << "Increment below 600: "
              << increment_if_below(acc.balance, 600) << "\n"; // 1 (500->501)
    std::cout << "Balance: " << acc.balance.load() << "\n";     // 501
}
```

**Explanation:** `compare_exchange_weak` atomically checks "is the value still what I expect?" and updates it only if yes. If another thread changed the value, CAS fails and returns the new value in `current`, so the loop re-evaluates the condition. The check and use are inseparable - no TOCTOU window.

### Q3: Explain how mutex scope should cover the entire check-then-act sequence

**Answer:**

A very common mistake is acquiring a lock for the check, releasing it, then re-acquiring it for the act. This looks organized and "careful," but the gap between the two lock acquisitions is exactly the TOCTOU window. The lock must stay held across the entire check-and-act sequence:

```cpp
#include <mutex>
#include <map>
#include <string>
#include <thread>
#include <vector>
#include <iostream>

// BUGGY: narrow mutex scope creates TOCTOU
struct BuggyCache {
    std::map<std::string, int> data;
    std::mutex mtx;

    void insert_if_absent(const std::string& key, int value) {
        bool exists;
        {
            std::lock_guard lock(mtx);
            exists = data.count(key); // CHECK (under lock)
        } // <- lock released here!

        // TOCTOU WINDOW: another thread can insert 'key' here

        if (!exists) {
            std::lock_guard lock(mtx);
            data[key] = value; // USE (re-acquired lock, but condition may be stale)
        }
    }
};

// FIXED: mutex covers entire check-then-act
struct SafeCache {
    std::map<std::string, int> data;
    std::mutex mtx;

    bool insert_if_absent(const std::string& key, int value) {
        std::lock_guard lock(mtx);
        // CHECK + USE under the SAME lock acquisition
        if (data.count(key) == 0) {
            data[key] = value;
            return true;  // inserted
        }
        return false;     // already existed
    }

    // Even better: use try_emplace (combines check+insert atomically)
    bool insert_if_absent_v2(const std::string& key, int value) {
        std::lock_guard lock(mtx);
        auto [it, inserted] = data.try_emplace(key, value);
        return inserted;
    }
};

int main() {
    // === Demonstrate the bug ===
    BuggyCache buggy;
    constexpr int THREADS = 100;
    std::atomic<int> insert_count{0};

    std::vector<std::thread> threads;
    for (int t = 0; t < THREADS; ++t) {
        threads.emplace_back([&] {
            buggy.insert_if_absent("key", t);
            insert_count++;
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Buggy: data[key] = " << buggy.data["key"] << "\n";
    // Multiple threads may overwrite the value - last writer wins

    // === Fixed version ===
    SafeCache safe;
    std::atomic<int> wins{0};
    threads.clear();
    for (int t = 0; t < THREADS; ++t) {
        threads.emplace_back([&, t] {
            if (safe.insert_if_absent("key", t))
                wins++;
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Safe: data[key] = " << safe.data["key"]
              << " (exactly " << wins.load() << " winner)\n";
    // Output: Safe: data[key] = <first thread's value> (exactly 1 winner)
}

// === RULE: Mutex scope must cover the ENTIRE critical section ===
//
// WRONG:                           RIGHT:
// lock()                           lock()
//   check condition                  check condition
// unlock()                           act on condition
// <- TOCTOU window               unlock()
// lock()
//   act on condition
// unlock()
//
// The lock must be held from CHECK through USE without releasing.
// If you release and re-acquire, the invariant may have changed.
```

**Explanation:**

The critical rule: **never release a mutex between checking a condition and acting on it**. When you release and re-acquire, you only know the condition *was* true - not that it *still is* true. The fix is simple: keep the lock held across the entire check-then-act sequence.

Common TOCTOU patterns and their fixes:

| TOCTOU Pattern | Fix |
| --- | --- |
| `if (!map.count(k)) { map[k] = v; }` | Hold single lock across both, or use `try_emplace` |
| `if (file_exists(f)) { open(f); }` | Just call `open()` and handle errors |
| `if (q.size() > 0) { q.pop(); }` | Use `try_pop()` or hold lock across both |
| `if (ptr != null) { ptr->use(); }` | Use `shared_ptr` or hold lock across both |

---

## Notes

- **TOCTOU in file systems:** `access()` then `open()` is a classic security vulnerability (CWE-367). Prefer opening directly and handling ENOENT.
- **TSan catches TOCTOU:** ThreadSanitizer (`-fsanitize=thread`) detects the data race between the check and use.
- **`compare_exchange_weak` vs `strong`:** `weak` may fail spuriously (fine in a loop); `strong` never fails spuriously but may be slower on ARM.
- **Design-level fix:** Often the best fix is an API that combines check+act: `try_emplace`, `try_pop`, `compare_exchange`, `fetch_add`.
- Compile with `-std=c++17 -O2 -pthread -fsanitize=thread`.
