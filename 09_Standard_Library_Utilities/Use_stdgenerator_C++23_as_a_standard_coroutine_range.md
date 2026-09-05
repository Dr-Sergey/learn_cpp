# Use std::generator (C++23) as a standard coroutine range

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/coroutine/generator>  

---

## Topic Overview

`std::generator<T>` is C++23's standard coroutine return type for producing lazy sequences. You write a function that uses `co_yield` to produce values one at a time, and the caller gets something that behaves like a range - it plugs straight into range-based for loops and composes with `std::views` pipelines. Before C++23, getting this behavior required a pile of hand-written promise type, iterator, and sentinel boilerplate. Now the library handles all of that for you.

### How It Works

A generator function suspends at each `co_yield` and hands the value back to the caller. The caller resumes the coroutine on the next iteration:

```cpp
std::generator<int> count(int from, int to) {
    for (int i = from; i < to; ++i)
        co_yield i;   // suspend, produce value
}
// caller resumes on each iteration
```

Three things are worth keeping in mind:

1. The function is a **coroutine** because it uses `co_yield`.
2. `std::generator<int>` handles the promise type, coroutine handle, and iterator protocol.
3. Values are produced **lazily** - computed only when the caller advances the iterator.

### Key Properties

| Property | Value |
| --- | --- |
| Range category | `input_range` (single-pass) |
| Copyable? | No (move-only) |
| Rewindable? | No |
| Infinite? | Yes - can produce values forever |
| Recursive? | Yes - via `co_yield std::ranges::elements_of(sub_gen)` |

### Core Syntax

Here's the classic lazy sequence example. The Fibonacci generator runs forever - values are only computed as the caller pulls them, so `std::views::take(10)` stops the process cleanly after ten:

```cpp
#include <generator>
#include <ranges>
#include <iostream>

// Simple generator: Fibonacci sequence
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        int next = a + b;
        a = b;
        b = next;
    }
}

int main() {
    // Take first 10 Fibonacci numbers
    for (int val : fibonacci() | std::views::take(10)) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: 0 1 1 2 3 5 8 13 21 34
}
```

---

## Self-Assessment

### Q1: Rewrite a recursive tree traversal as a std::generator<Node*> using co_yield

**Answer:**

The interesting thing here is `co_yield std::ranges::elements_of(sub_generator)`. Without it you'd have to manually flatten the recursion into an explicit stack. With it, each recursive call just delegates its output into the parent generator:

```cpp
#include <generator>
#include <iostream>
#include <memory>

struct Node {
    int value;
    std::unique_ptr<Node> left;
    std::unique_ptr<Node> right;

    Node(int v, std::unique_ptr<Node> l = {}, std::unique_ptr<Node> r = {})
        : value(v), left(std::move(l)), right(std::move(r)) {}
};

// In-order traversal as a generator (using recursive co_yield)
std::generator<const Node*> inorder(const Node* node) {
    if (!node) co_return;

    // Yield all elements from left subtree
    co_yield std::ranges::elements_of(inorder(node->left.get()));

    // Yield current node
    co_yield node;

    // Yield all elements from right subtree
    co_yield std::ranges::elements_of(inorder(node->right.get()));
}

// Pre-order: root, left, right
std::generator<const Node*> preorder(const Node* node) {
    if (!node) co_return;
    co_yield node;
    co_yield std::ranges::elements_of(preorder(node->left.get()));
    co_yield std::ranges::elements_of(preorder(node->right.get()));
}

int main() {
    //       4
    //      / \
    //     2   6
    //    / \ / \
    //   1  3 5  7
    auto tree = std::make_unique<Node>(4,
        std::make_unique<Node>(2,
            std::make_unique<Node>(1),
            std::make_unique<Node>(3)),
        std::make_unique<Node>(6,
            std::make_unique<Node>(5),
            std::make_unique<Node>(7))
    );

    std::cout << "inorder:  ";
    for (const Node* n : inorder(tree.get())) {
        std::cout << n->value << " ";
    }
    std::cout << "\n";
    // Output: inorder:  1 2 3 4 5 6 7

    std::cout << "preorder: ";
    for (const Node* n : preorder(tree.get())) {
        std::cout << n->value << " ";
    }
    std::cout << "\n";
    // Output: preorder: 4 2 1 3 6 5 7
}
```

**Key insight:** `co_yield std::ranges::elements_of(sub_generator)` enables **recursive generators** without flattening the recursion manually. Each recursive call produces a nested generator, and `elements_of` yields all its values into the parent.

---

### Q2: Show that std::generator satisfies std::ranges::input_range for range-based for

**Answer:**

You can prove this at compile time with `static_assert`. The `!forward_range` assertion is just as important as the `input_range` one - it's a reminder that you cannot iterate the generator a second time or hold multiple positions in it:

```cpp
#include <generator>
#include <ranges>
#include <iostream>
#include <vector>
#include <string>
#include <type_traits>

// A generator of squares
std::generator<int> squares(int n) {
    for (int i = 0; i < n; ++i) {
        co_yield i * i;
    }
}

// A generator of lines from a string (split by newline)
std::generator<std::string> lines(const std::string& text) {
    std::string::size_type start = 0;
    while (start < text.size()) {
        auto end = text.find('\n', start);
        if (end == std::string::npos) end = text.size();
        co_yield text.substr(start, end - start);
        start = end + 1;
    }
}

int main() {
    // Proof: std::generator<int> satisfies input_range
    static_assert(std::ranges::input_range<std::generator<int>>);
    static_assert(std::ranges::range<std::generator<int>>);

    // NOT a forward_range (single-pass only)
    static_assert(!std::ranges::forward_range<std::generator<int>>);

    // Use in range-based for loop
    for (int sq : squares(5)) {
        std::cout << sq << " ";
    }
    std::cout << "\n";
    // Output: 0 1 4 9 16

    // Compose with views
    auto result = squares(10)
        | std::views::filter([](int x) { return x % 2 == 0; })
        | std::views::take(3);

    for (int val : result) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: 0 4 16

    // Use with string generator
    std::string text = "line 1\nline 2\nline 3";
    for (const auto& line : lines(text)) {
        std::cout << "[" << line << "]\n";
    }
    // Output:
    // [line 1]
    // [line 2]
    // [line 3]
}
```

---

### Q3: Explain why std::generator is single-pass and cannot be rewound

**Answer:**

This is one of those things that trips people up when they first meet coroutines. A `std::generator` is backed by a **coroutine frame** - a heap-allocated block that stores the coroutine's local variables and the current suspension point. That design makes rewinding fundamentally impossible:

1. **Coroutine state is mutable and progresses forward.** When you resume a coroutine, it continues from its last `co_yield`. There is no mechanism to "reset" to the beginning - the frame only knows where it currently is.

2. **No copy semantics.** `std::generator` is move-only. You cannot create a second independent copy that starts from the beginning.

3. **Values are ephemeral.** After advancing to the next value, the previous value's storage may be reused by the coroutine.

The code below shows the concrete consequence: iterating partway through a generator advances its internal state, and you cannot go back.

```cpp
#include <generator>
#include <iostream>
#include <ranges>

std::generator<int> counting() {
    for (int i = 0; ; ++i) co_yield i;
}

int main() {
    auto gen = counting();

    // First iteration: consumes some values
    for (int val : gen | std::views::take(3)) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: 0 1 2

    // Cannot rewind! gen's internal state is now at i=3
    // A second loop would continue from where it left off (or be UB if already done)

    // Cannot copy:
    // auto gen2 = gen;  // COMPILE ERROR: generator is not copyable

    // Solution: create a new generator
    auto gen2 = counting();
    for (int val : gen2 | std::views::take(3)) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: 0 1 2  (fresh start)
}
```

If you need a quick mental model: a generator is like reading from a pipe. You can read forward, but the data is gone once consumed. A `std::vector` is more like a book - you can flip back to any page.

**Comparison:**

| Range type | Passes | Example |
| --- | --- | --- |
| `std::vector` | Multi-pass (forward+) | Random access, rewindable |
| `std::forward_list` | Multi-pass (forward) | Can iterate multiple times |
| `std::generator` | **Single-pass (input)** | Consumed on read |
| `std::istream_iterator` | Single-pass (input) | Similar to generator |

**Practical implication:** If you need the values more than once, materialize the generator into a container: `auto vec = gen | std::ranges::to<std::vector>()` (C++23).

---

## Notes

- `std::generator` is in `<generator>` (C++23), not `<coroutine>`.
- Compiler support: MSVC 17.5+, GCC 14+, Clang 18+ (partial).
- `co_yield std::ranges::elements_of(...)` enables recursive generators - the only standard way to "yield from" a sub-generator.
- `std::generator<T>` stores a `T` value in the promise, so large objects are stored by value - prefer `std::generator<const T&>` for large types.
- Memory: each active coroutine frame is typically heap-allocated (compiler may elide in some cases).
- Generators are the standard replacement for custom iterator pairs or callback-based traversals.
