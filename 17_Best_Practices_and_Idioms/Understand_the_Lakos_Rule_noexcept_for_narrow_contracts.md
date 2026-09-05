# Understand the Lakos Rule: noexcept for narrow contracts

**Category:** Best Practices and Idioms  
**Standard:** C++11  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Re-noexcept>  

---

## Topic Overview

The **Lakos Rule** (from John Lakos): functions with **wide contracts** (valid for all inputs) can be `noexcept`. Functions with **narrow contracts** (preconditions) should **not** be `noexcept` because precondition-violation handling may need to throw.

The intuition here is simple. If a function has no preconditions, there is nothing that can go wrong before you even start - so promising not to throw is safe. If a function has preconditions, someone will eventually call it wrong, and you want to preserve the ability to signal that violation through an exception rather than calling `std::terminate`.

### Wide vs Narrow Contracts

Here is a quick reference for which side of the line common operations fall on:

| Contract | Preconditions | Example | `noexcept`? |
| --- | --- | --- | --- |
| Wide | None - works for all inputs | `vector::size()` | Yes |
| Wide | None | `vector::empty()` | Yes |
| Narrow | `index < size()` | `vector::operator[]` | Debatable |
| Narrow | `!empty()` | `stack::top()` | Debatable |
| Wide | None | `swap(a, b)` | Yes |
| Narrow | `ptr != nullptr` | `*ptr` | No |

---

## Self-Assessment

### Q1: Explain the Lakos Rule with examples of wide and narrow contracts

Let's make a stack that distinguishes its wide-contract operations (`empty`, `size`, `push`) from its narrow-contract ones (`top`, `pop`). The narrow ones need to be able to throw - so they don't get `noexcept`.

```cpp
#include <iostream>
#include <stdexcept>
#include <vector>

// WIDE contract: valid for ANY input - safe to be noexcept
class SafeStack {
    std::vector<int> data_;
public:
    // Wide: always valid, no preconditions
    [[nodiscard]] bool empty() const noexcept { return data_.empty(); }
    [[nodiscard]] size_t size() const noexcept { return data_.size(); }

    // Wide: push always works (unless OOM, which is not recoverable)
    void push(int val) { data_.push_back(val); }

    // NARROW contract: requires !empty()
    // Should NOT be noexcept per Lakos Rule!
    int top() const {
        if (data_.empty())
            throw std::logic_error("top() on empty stack");
        return data_.back();
    }

    int pop() {
        if (data_.empty())
            throw std::logic_error("pop() on empty stack");
        int val = data_.back();
        data_.pop_back();
        return val;
    }
};

int main() {
    SafeStack s;
    s.push(10);
    s.push(20);

    std::cout << "Size: " << s.size() << '\n';
    std::cout << "Top: " << s.top() << '\n';
    std::cout << "Pop: " << s.pop() << '\n';

    try {
        s.pop();
        s.pop();  // empty! throws, not std::terminate
    } catch (const std::logic_error& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }
}
// Expected output:
// Size: 2
// Top: 20
// Pop: 20
// Caught: pop() on empty stack
```

Notice that `top` and `pop` throw instead of crashing - that is exactly what the Lakos Rule is designed to preserve.

### Q2: Show why `noexcept` on a narrow-contract function prevents catching precondition violations

This is where the rule really bites. If you mark a narrow-contract function `noexcept` and it then tries to throw, the runtime calls `std::terminate` immediately. Your test framework never gets to catch the failure - the whole process just dies.

```cpp
#include <iostream>
#include <stdexcept>
#include <exception>
#include <string>

class ArrayView {
    int* data_;
    size_t size_;
public:
    ArrayView(int* d, size_t s) : data_(d), size_(s) {}

    // BAD: narrow contract (index < size_) marked noexcept
    int at_bad(size_t index) noexcept {
        if (index >= size_) {
            // We want to report the error, but we're noexcept!
            // Option 1: throw -> std::terminate (program dies)
            // Option 2: return garbage -> silent corruption
            // Option 3: abort() -> no recovery possible
            std::cerr << "Index out of bounds!\n";
            std::abort();  // only safe option, but no recovery
        }
        return data_[index];
    }

    // GOOD: narrow contract WITHOUT noexcept
    int at_good(size_t index) {
        if (index >= size_)
            throw std::out_of_range("index " + std::to_string(index)
                + " >= size " + std::to_string(size_));
        return data_[index];
    }
};

int main() {
    int arr[] = {10, 20, 30};
    ArrayView view(arr, 3);

    // Test harness CAN catch the error from at_good:
    try {
        view.at_good(10);  // out of bounds
    } catch (const std::out_of_range& e) {
        std::cout << "Test passed: caught " << e.what() << '\n';
    }

    // at_bad(10) would call std::abort() -- test framework can't catch it!
    std::cout << "Test suite continues after at_good failure\n";
}
// Expected output:
// Test passed: caught index 10 >= size 3
// Test suite continues after at_good failure
```

The `at_good` version lets the test framework catch and report the failure gracefully. The `at_bad` version would kill the process. That is the whole motivation for the Lakos Rule in testing and library contexts.

### Q3: Apply the Lakos Rule to swap, move, and comparison operators

Wide-contract operations that you write all the time - `swap`, move constructors, comparisons - are safe to mark `noexcept` because they genuinely work for any valid combination of inputs.

```cpp
#include <iostream>
#include <string>
#include <utility>

class Widget {
    std::string name_;
    int value_;
public:
    Widget(std::string n, int v) : name_(std::move(n)), value_(v) {}

    // SWAP: wide contract (always valid for any two Widgets)
    // -> noexcept: YES
    friend void swap(Widget& a, Widget& b) noexcept {
        using std::swap;
        swap(a.name_, b.name_);    // string swap is noexcept
        swap(a.value_, b.value_);  // int swap is noexcept
    }

    // MOVE constructor: wide contract (source is valid object)
    // -> noexcept: YES (critical for vector reallocation)
    Widget(Widget&& other) noexcept
        : name_(std::move(other.name_)), value_(other.value_) {}

    // MOVE assignment: wide contract
    // -> noexcept: YES
    Widget& operator=(Widget&& other) noexcept {
        name_ = std::move(other.name_);
        value_ = other.value_;
        return *this;
    }

    // COMPARISON: wide contract (any two Widgets can be compared)
    // -> noexcept: YES
    friend bool operator==(const Widget& a, const Widget& b) noexcept {
        return a.value_ == b.value_ && a.name_ == b.name_;
    }

    friend auto operator<=>(const Widget& a, const Widget& b) noexcept {
        if (auto cmp = a.value_ <=> b.value_; cmp != 0) return cmp;
        return a.name_ <=> b.name_;
    }

    friend std::ostream& operator<<(std::ostream& os, const Widget& w) {
        return os << w.name_ << '(' << w.value_ << ')';
    }
};

int main() {
    Widget a("Alpha", 1), b("Beta", 2);
    std::cout << "Before swap: " << a << ", " << b << '\n';
    swap(a, b);
    std::cout << "After swap:  " << a << ", " << b << '\n';
    std::cout << "Equal: " << std::boolalpha << (a == b) << '\n';
    std::cout << "a < b: " << (a < b) << '\n';
}
// Expected output:
// Before swap: Alpha(1), Beta(2)
// After swap:  Beta(2), Alpha(1)
// Equal: false
// a < b: false
```

Pay special attention to the move constructor: marking it `noexcept` is not just stylistic. `std::vector` checks this at reallocation time via `std::move_if_noexcept` - if your move isn't `noexcept`, the vector falls back to copying every element.

**Summary:**

| Operation | Contract | `noexcept`? | Why |
| --- | --- | --- | --- |
| `swap` | Wide | Yes | Always valid |
| Move ctor/assign | Wide | Yes | Critical for `vector` |
| `operator==` | Wide | Yes | Any two objects comparable |
| `operator<=>` | Wide | Yes | Any two objects comparable |
| `operator[]` | Narrow | No (Lakos) | Precondition: valid index |
| `at()` | Narrow | No | May throw on violation |

---

## Notes

- The Lakos Rule is debated - some argue even narrow-contract functions should be `noexcept` in release builds (with assertions disabled).
- `noexcept` affects codegen: compilers may omit exception-handling tables.
- `std::vector` uses `std::move_if_noexcept` - if your move isn't `noexcept`, vector falls back to copying on reallocation.
- C++ Contracts (C++26) will provide a standard way to express preconditions without the noexcept dilemma.
