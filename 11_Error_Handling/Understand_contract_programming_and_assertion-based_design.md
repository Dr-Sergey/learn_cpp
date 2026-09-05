# Understand contract programming and assertion-based design

**Category:** Error Handling  
**Standard:** C++26  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#i-in-and-out-parameters>  

---

## Topic Overview

**Contract programming** (also called "Design by Contract") is a methodology where functions explicitly declare their **preconditions** (what the caller must guarantee), **postconditions** (what the function guarantees), and **invariants** (what must always be true). The key insight is that violations of these contracts are **programming bugs**, not runtime errors. They should crash the program immediately rather than be "handled" - because there is nothing to handle, the code is simply wrong.

### Defensive Programming vs Contract Programming

Here is the core tension. Defensive programming tries to keep things running no matter what. Contract programming says: if a caller violates the API, crash immediately so the bug is obvious. Look at the same function written both ways:

```cpp
// Defensive Programming:                Contract Programming:
// double sqrt(double x) {              // // Precondition: x >= 0
//   if (x < 0)                         // double sqrt(double x) {
//     return -1; // silent             //   assert(x >= 0);
//   return std::sqrt(x);               //   return std::sqrt(x);
// }                                    // }
//
// Problem: caller doesn't             // Benefit: bug is caught
// know -1 means "error"               // immediately at the source
// vs a valid negative result          // with a clear message
```

The defensive version looks safer but is actually more dangerous. The bug propagates silently: the caller gets back `-1`, uses it as a valid number, and the corruption spreads. With the contract version, the program crashes the instant the contract is violated - loud, immediate, debuggable.

### The Three Contract Types

The three contract types map to three different questions about a function's relationship with its callers:

| Contract | Who Must Guarantee? | Example |
| --- | --- | --- |
| **Precondition** | Caller | `x > 0` before calling `sqrt(x)` |
| **Postcondition** | Function | "returned pointer is non-null" |
| **Class invariant** | All member functions | "size() <= capacity()" always |

### Assertion Mechanisms in C++

There are several ways to express contracts in C++ today, and they differ in when they run and how much you can configure them:

| Mechanism | Removed in Release? | Message? | Standard |
| --- | --- | --- | --- |
| `assert(expr)` | Yes (with `NDEBUG`) | File/line only | C / C++ |
| `static_assert(expr, msg)` | Compile-time only | Custom message | C++11 |
| Custom `ASSERT` macro | Configurable | Custom message + expression | User-defined |
| Contracts (`pre`/`post`) | Configurable | Annotation-based | C++26 (proposed) |
| `[[assume(expr)]]` | Hint to optimizer only | No runtime check | C++23 |

The big gap in plain `assert()` is that it disappears entirely in release builds (when `NDEBUG` is defined). If you need contract checking in production, you need a custom macro.

### Best Practices

When you face an error condition, the first question is: is this a programmer mistake, or a legitimate runtime situation? The answer tells you exactly which tool to reach for:

```cpp
//                 Is this a PROGRAMMER BUG?
//                 (caller violates API contract)
//                        |
//             +-----------+-----------+
//             YES                     NO
//             |                       |
//     Use assertion           Is it a COMMON expected failure?
//     (crash fast)            (user input, file not found)
//             |                       |
//             v                  +----+----+
//       assert() or             YES       NO (truly rare)
//       custom ASSERT           |         |
//                           std::expected  throw exception
//                           or error code
```

### Important Notes

- **Assertions are for bug detection**, not for handling user input or external errors.
- **Never put side effects in assertions** - they're removed with `NDEBUG`.
- **Crash fast** on invariant violation - continuing with corrupt state causes worse bugs later.
- Mark non-throwing functions `noexcept` for better codegen and container optimization.

---

## Self-Assessment

### Q1: Replace a defensive if-check inside a function with a precondition assertion and document the contract

The "before" versions below are real code patterns you will encounter in production codebases. The "after" versions show what they should look like with proper contract annotations.

**Solution - Before and After:**

```cpp
#include <iostream>
#include <cassert>
#include <vector>
#include <string>
#include <cmath>

// ========== BEFORE: Defensive Programming ==========
// Problems: silent error, caller doesn't know -1 means error,
// bug propagates silently through the system

namespace defensive {

double divide(double a, double b) {
    if (b == 0.0)
        return 0.0;  // silent corruption! caller thinks result is valid
    return a / b;
}

int element_at(const std::vector<int>& v, size_t idx) {
    if (idx >= v.size())
        return -1;  // is -1 an error or a valid element?
    return v[idx];
}

std::string substring(const std::string& s, size_t pos, size_t len) {
    if (pos > s.size())
        return "";  // hides a bug in the caller
    return s.substr(pos, len);
}

}  // namespace defensive

// ========== AFTER: Contract Programming ==========
// Benefits: bugs caught immediately at the source,
// clear documentation of what the caller must guarantee

namespace contract {

// @pre b != 0.0
double divide(double a, double b) {
    assert(b != 0.0 && "Precondition violated: divisor must not be zero");
    return a / b;
}

// @pre idx < v.size()
int element_at(const std::vector<int>& v, size_t idx) {
    assert(idx < v.size() && "Precondition violated: index out of bounds");
    return v[idx];
}

// @pre pos <= s.size()
std::string substring(const std::string& s, size_t pos, size_t len) {
    assert(pos <= s.size() && "Precondition violated: pos beyond string end");
    return s.substr(pos, len);
}

// Class with invariant
class BankAccount {
    double balance_;  // invariant: balance_ >= 0

    void check_invariant() const {
        assert(balance_ >= 0.0 && "Invariant violated: negative balance");
    }

public:
    // @pre initial_balance >= 0
    explicit BankAccount(double initial_balance) : balance_(initial_balance) {
        assert(initial_balance >= 0.0 && "Precondition: initial balance >= 0");
        check_invariant();  // postcondition + invariant
    }

    // @pre amount > 0
    // @post balance() == old balance() + amount
    void deposit(double amount) {
        assert(amount > 0.0 && "Precondition: deposit amount must be positive");
        balance_ += amount;
        check_invariant();
    }

    // @pre amount > 0 && amount <= balance()
    // @post balance() == old balance() - amount
    void withdraw(double amount) {
        assert(amount > 0.0 && "Precondition: withdrawal amount must be positive");
        assert(amount <= balance_ && "Precondition: insufficient funds");
        balance_ -= amount;
        check_invariant();
    }

    double balance() const { return balance_; }
};

}  // namespace contract

int main() {
    // These work correctly:
    contract::BankAccount acct(100.0);
    acct.deposit(50.0);
    acct.withdraw(30.0);
    std::cout << "Balance: " << acct.balance() << "\n";  // 120.0

    // This would CRASH with a clear assertion message:
    // acct.withdraw(200.0);  // "Precondition: insufficient funds"
    // contract::divide(10.0, 0.0);  // "divisor must not be zero"
}
// Expected output:
//   Balance: 120
```

Notice how `check_invariant()` is called at the end of `deposit` and `withdraw`. This verifies that the function left the object in a valid state before returning. If something ever goes wrong with the balance logic, you get an immediate crash inside the function that caused it - not a mysterious negative balance discovered three operations later.

---

### Q2: Show the difference between defensive programming (handle silently) and contract programming (assert)

Here the binary search example makes the contrast extremely clear. Passing unsorted data to a binary search is a bug - there is no sensible "result" to return. A defensive function returns `-1` (wrong answer). A contract function crashes (correct behavior).

**Solution - Side-by-Side Comparison:**

```cpp
#include <iostream>
#include <cassert>
#include <vector>
#include <algorithm>
#include <numeric>

// ========== Scenario: Binary search on sorted data ==========

namespace defensive {
// "Handle" unsorted input silently - hides the bug
int find(const std::vector<int>& data, int target) {
    if (data.empty()) return -1;
    // Doesn't check if sorted - just runs and returns wrong answer silently
    auto it = std::lower_bound(data.begin(), data.end(), target);
    if (it != data.end() && *it == target)
        return static_cast<int>(it - data.begin());
    return -1;  // "not found" - but maybe it WAS there, data was just unsorted
}
}

namespace contract {
// Assert the precondition - crash if caller violates contract
int find(const std::vector<int>& data, int target) {
    assert(!data.empty() && "Precondition: data must not be empty");
    assert(std::is_sorted(data.begin(), data.end()) &&
           "Precondition: data must be sorted for binary search");

    auto it = std::lower_bound(data.begin(), data.end(), target);
    if (it != data.end() && *it == target) {
        int idx = static_cast<int>(it - data.begin());
        assert(data[idx] == target && "Postcondition: found element matches target");
        return idx;
    }
    return -1;
}
}

// ========== Custom ASSERT macro for production ==========
// Unlike assert(), this is NOT removed with NDEBUG

#define CONTRACT_ASSERT(expr, msg) \
    do { \
        if (!(expr)) { \
            std::cerr << "CONTRACT VIOLATION at " << __FILE__ << ":" << __LINE__ \
                      << "\n  " << #expr << "\n  " << msg << "\n"; \
            std::abort(); \
        } \
    } while(0)

// Production-grade version with the custom macro:
namespace production {
int find(const std::vector<int>& data, int target) {
    CONTRACT_ASSERT(!data.empty(), "data must not be empty");
    CONTRACT_ASSERT(std::is_sorted(data.begin(), data.end()),
                    "data must be sorted for binary search");

    auto it = std::lower_bound(data.begin(), data.end(), target);
    if (it != data.end() && *it == target)
        return static_cast<int>(it - data.begin());
    return -1;
}
}

int main() {
    std::vector<int> sorted = {1, 3, 5, 7, 9};
    std::vector<int> unsorted = {5, 1, 9, 3, 7};

    // Works correctly on sorted data:
    std::cout << "Found 5 at: " << contract::find(sorted, 5) << "\n";  // 2

    // Defensive: silently returns wrong answer (bug hidden):
    std::cout << "Defensive on unsorted: " << defensive::find(unsorted, 3) << "\n";
    // May return -1 (wrong!) because binary search on unsorted data is undefined

    // Contract: CRASHes immediately, bug caught:
    // contract::find(unsorted, 3);  // assertion failure!
}
// Expected output:
//   Found 5 at: 2
//   Defensive on unsorted: -1  (wrong - 3 IS in the data!)
```

**Key Differences:**

| Aspect | Defensive | Contract |
| --- | --- | --- |
| Invalid input | Return error value silently | Crash immediately |
| Bug detection | Bug may propagate for weeks | Bug caught at the source |
| Code clarity | Unclear what's an error vs feature | Explicit preconditions |
| Performance | Extra branches on hot paths | Assertions removed in release (with `NDEBUG`) |
| User input | Appropriate (validate and reject) | Not for user input |
| API contracts | Hides violations | Enforces contracts |

---

### Q3: Explain why crashing fast on contract violation is often safer than continuing with invalid state

This is the concept that feels counterintuitive at first. "How is crashing safer than continuing?" The answer is that continuing with corrupt state is an **unbounded** failure - you do not know how far the corruption spreads or what it will overwrite before anyone notices. A crash is a **bounded** failure - it stops the damage immediately and gives you a precise location.

**The Case for Fail-Fast:**

```cpp
// Scenario: Image processing application
// Contract: buffer pointer must not be null, width > 0, height > 0

// Defensive (continue with invalid state):
//   process_image(nullptr, -1, -1)
//     -> if (!buf) return;        // silently skip
//     -> save_output("result.png") // saves empty/garbage file
//     -> upload_to_server()        // uploads corrupted data
//     -> customer sees garbage     // 3 hours later someone notices
//     -> Root cause: hard to trace back to the original null pointer

// Contract (crash immediately):
//   process_image(nullptr, -1, -1)
//     -> assert(buf != nullptr)   // BOOM - crash at line 42
//     -> Core dump + stack trace  // developer sees exactly what happened
//     -> Fix deployed in 30 minutes
```

**Real-World Consequences of NOT Crashing Fast:**

| Continuing with invalid state... | Crashing fast... |
| --- | --- |
| Corrupt data written to database | No corrupt data - stopped before write |
| Security vulnerability (buffer overrun) | No exploitation possible - process dead |
| Wrong calculation cascades through system | Cascade prevented at the source |
| Bug surfaces far from root cause | Bug pinpointed exactly |
| Intermittent "weird behavior" reports | Clear crash report with stack trace |

The security case is particularly important and worth looking at directly:

```cpp
// Example: The security case
void vulnerable(const char* user_data, size_t len) {
    // DEFENSIVE: silently truncate
    char buf[256];
    if (len > 256) len = 256;  // Hides buffer overflow bug in caller
    memcpy(buf, user_data, len);
    // Caller never learns they passed too much data

    // CONTRACT: crash on violation
    assert(len <= 256 && "Buffer overflow: caller must validate length");
    // In production, this could be a custom assert that logs + aborts
}
```

The defensive version silently "fixes" an overflow that might represent an attacker probing boundaries. The contract version crashes, which is exactly what you want - no data gets read, no output gets produced.

**The Principle:**

- **Bugs should be loud and immediate**, not quiet and delayed.
- **Invalid state is poison** - every operation on invalid state produces more invalid state.
- **A crash is a bounded failure** - continuing with corruption is an unbounded failure.
- **Core dumps are debuggable** - silent corruption is not.

**Important Caveat:** Assertions are for **programmer bugs**, not user errors. User input should always be validated and rejected gracefully (with error messages or `std::expected`), never asserted.

---

## Notes

- **`NDEBUG` removes `assert()`** - if you need runtime checks in release builds, use a custom assertion macro.
- **`static_assert()`** (C++11) checks at compile time - use for template constraints, type sizes, alignment.
- **`[[assume(expr)]]`** (C++23) tells the optimizer the expression is always true - no runtime check, UB if wrong. Use only for proven facts.
- **C++26 contracts** (`pre`, `post`, `contract_assert`) will provide standardized contract annotations with configurable enforcement levels (ignore, observe, enforce).
- **GSL `Expects()` / `Ensures()`** from Microsoft's Guidelines Support Library are widely used custom assertion macros for pre/postconditions.
- **Never write:** `assert(important_function())` - the call is removed in release mode! Separate side effects from assertions.
