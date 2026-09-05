# Use std::transform, std::for_each, and std::generate correctly

**Category:** Standard Library Algorithms  
**Standard:** C++98, C++11, C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/transform>  

---

## Topic Overview

These three algorithms are the workhorses for applying callables to ranges, but they have distinct contracts that are easy to confuse. The key is to match the right algorithm to what you actually need: produce output from input, apply side effects, or fill from scratch.

| Algorithm | Purpose | Reads input? | Writes output? | Returns |
| --- | --- | --- | --- | --- |
| `std::transform` | Map each element to an output range | Yes | Yes (output iterator) | Output iterator past last written |
| `std::for_each` | Apply side-effect to each element | Yes | May modify in-place | The functor (moved) |
| `std::generate` | Fill range from a generator | No | Yes (overwrites) | void |

### std::transform

Two overloads - unary for one input, binary for two:

```cpp
// Unary: out[i] = op(in[i])
transform(first, last, d_first, unary_op);

// Binary: out[i] = op(in1[i], in2[i])
transform(first1, last1, first2, d_first, binary_op);
```

- The output range **may** alias the input range (in-place transform) as long as `d_first == first`.
- `unary_op` / `binary_op` must not invalidate iterators or modify elements through the input iterators.

### std::for_each

```cpp
auto f = std::for_each(first, last, func);
// func is applied to *it for each it in [first, last)
// f is the (possibly moved) functor - useful for stateful functors
```

- Unlike range-for, `for_each` **returns the functor**, allowing you to extract accumulated state.
- Since C++17, parallel `for_each` with execution policies is available.

### std::generate / std::generate_n

```cpp
std::generate(first, last, generator);     // fill [first, last)
std::generate_n(first, n, generator);      // fill first n elements
```

- `generator()` is called with no arguments for each position.
- Useful for filling with sequential values, random numbers, etc.

### Core Example

Here's all three in a short program to see the pattern differences side by side:

```cpp
#include <algorithm>
#include <numeric>
#include <vector>
#include <iostream>
#include <random>

int main() {
    std::vector<int> src{1, 2, 3, 4, 5};

    // --- transform: square each element into a new vector ---
    std::vector<int> dst(src.size());
    std::transform(src.begin(), src.end(), dst.begin(),
                   [](int x) { return x * x; });
    // dst = {1, 4, 9, 16, 25}

    // --- for_each with stateful functor ---
    struct Sum {
        int total = 0;
        void operator()(int x) { total += x; }
    };
    Sum result = std::for_each(src.begin(), src.end(), Sum{});
    std::cout << "sum = " << result.total << "\n";
    // Output: sum = 15

    // --- generate: fill with increasing values ---
    std::vector<int> seq(5);
    int counter = 0;
    std::generate(seq.begin(), seq.end(), [&counter]{ return ++counter; });
    // seq = {1, 2, 3, 4, 5}

    // --- generate: fill with random values ---
    std::mt19937 rng{42};
    std::uniform_int_distribution<int> dist(1, 100);
    std::vector<int> rnd(5);
    std::generate(rnd.begin(), rnd.end(), [&]{ return dist(rng); });
    for (int v : rnd) std::cout << v << " ";
    std::cout << "\n";
    // Output: (5 random numbers in [1,100])
}
```

The `for_each` example makes the functor-return trick visible right away: `Sum{}` is passed in, accumulates state, and the returned `result` holds the final total. That's something a plain range-for loop can't do without an external variable.

---

## Self-Assessment

### Q1: Use std::transform with two input ranges to compute a dot product

The binary overload of `transform` takes elements from two ranges simultaneously and passes them to a binary operation. For a dot product you want element-wise multiplication, then a reduction - which is a two-step process with `transform` alone.

```cpp
#include <algorithm>
#include <numeric>
#include <vector>
#include <iostream>

int main() {
    std::vector<double> a{1.0, 2.0, 3.0};
    std::vector<double> b{4.0, 5.0, 6.0};

    // Step 1: transform (binary) - element-wise multiply into a temp
    std::vector<double> products(a.size());
    std::transform(a.begin(), a.end(), b.begin(), products.begin(),
                   [](double x, double y) { return x * y; });
    // products = {4.0, 10.0, 18.0}

    // Step 2: accumulate the products
    double dot = std::accumulate(products.begin(), products.end(), 0.0);
    std::cout << "dot product = " << dot << "\n";
    // Output: dot product = 32

    // --- Better: single-pass with transform_reduce (C++17) ---
    double dot2 = std::transform_reduce(a.begin(), a.end(), b.begin(), 0.0);
    std::cout << "dot product (transform_reduce) = " << dot2 << "\n";
    // Output: dot product (transform_reduce) = 32

    // --- Or: inner_product (classic) ---
    double dot3 = std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
    std::cout << "dot product (inner_product)     = " << dot3 << "\n";
    // Output: dot product (inner_product)     = 32
}
```

The two-step approach creates an unnecessary intermediate `products` vector. The `transform_reduce` version avoids that allocation entirely by fusing the map and reduce into a single pass - which is usually what you want in performance-sensitive code.

---

### Q2: Explain why std::for_each returns the functor (unlike range-for) and when this is useful

`std::for_each` returns `std::move(f)` - the functor after it has been applied to every element. This matters when the functor is **stateful**:

```cpp
#include <algorithm>
#include <vector>
#include <iostream>
#include <string>

struct Stats {
    int count = 0;
    double sum = 0.0;
    double min_val = std::numeric_limits<double>::max();
    double max_val = std::numeric_limits<double>::lowest();

    void operator()(double x) {
        ++count;
        sum += x;
        if (x < min_val) min_val = x;
        if (x > max_val) max_val = x;
    }

    double mean() const { return sum / count; }
};

int main() {
    std::vector<double> data{3.1, 1.4, 1.5, 9.2, 6.5};

    // for_each returns the functor with accumulated state
    Stats s = std::for_each(data.begin(), data.end(), Stats{});

    std::cout << "count = " << s.count   << "\n";
    std::cout << "sum   = " << s.sum     << "\n";
    std::cout << "mean  = " << s.mean()  << "\n";
    std::cout << "min   = " << s.min_val << "\n";
    std::cout << "max   = " << s.max_val << "\n";
    // Output:
    // count = 5
    // sum   = 21.7
    // mean  = 4.34
    // min   = 1.4
    // max   = 9.2
}
```

**Why range-for can't do this:**
A range-based for loop has no mechanism to return accumulated state. You would need to capture variables externally. With `for_each`, the state is neatly encapsulated inside the functor object, and the returned copy carries the final state - no external variables needed, and the functor is reusable.

**When it is useful:**

- Single-pass statistics (count, sum, min, max, histogram)
- Collecting results into a functor-owned container
- Any time you want a self-contained, reusable "visitor" over a range

---

### Q3: Use std::generate to fill a container with increasing values and with random values

`generate` is the right tool when you need to fill a range from a generator function rather than from an existing source range. The generator takes no arguments, so all state has to live inside it - either captured by a lambda or as functor members.

```cpp
#include <algorithm>
#include <vector>
#include <iostream>
#include <random>
#include <numeric>  // for std::iota

int main() {
    // --- Increasing values with generate ---
    std::vector<int> inc(10);
    int n = 0;
    std::generate(inc.begin(), inc.end(), [&n]{ return n++; });
    for (int v : inc) std::cout << v << " ";
    std::cout << "\n";
    // Output: 0 1 2 3 4 5 6 7 8 9

    // Simpler alternative: std::iota
    std::vector<int> inc2(10);
    std::iota(inc2.begin(), inc2.end(), 0);
    // inc2 = {0, 1, 2, ..., 9}

    // --- Even numbers with generate ---
    int even = 0;
    std::vector<int> evens(5);
    std::generate(evens.begin(), evens.end(), [&even]{ return even += 2; });
    for (int v : evens) std::cout << v << " ";
    std::cout << "\n";
    // Output: 2 4 6 8 10

    // --- Random values with generate ---
    std::mt19937 rng{12345};
    std::uniform_real_distribution<double> dist(0.0, 1.0);
    std::vector<double> rnd(8);
    std::generate(rnd.begin(), rnd.end(), [&]{ return dist(rng); });
    for (double v : rnd) std::cout << v << " ";
    std::cout << "\n";
    // Output: (8 random doubles in [0, 1))

    // --- generate_n: fill first N elements ---
    std::vector<int> partial(10, 0);
    std::generate_n(partial.begin(), 5, [i=100]() mutable { return i++; });
    for (int v : partial) std::cout << v << " ";
    std::cout << "\n";
    // Output: 100 101 102 103 104 0 0 0 0 0
}
```

**Key points:**

- `std::generate` calls the generator with **no arguments** - all state must be captured or stored in the functor.
- Use a mutable lambda with an init-capture (`[i=0]() mutable`) to avoid external counter variables.
- For simple incrementing sequences, prefer `std::iota` - it's clearer and more concise.
- `std::generate_n` is convenient when you want to fill only the first N positions without computing the end iterator.

---

## Notes

- `std::transform` must not have a `unary_op` that modifies elements through the **input** iterator - it reads inputs and writes through the **output** iterator. In-place is allowed only when output == input.
- `std::for_each` **guarantees** in-order application (left to right) in the non-parallel overload. The parallel overload (`std::for_each(policy, ...)`) does not guarantee order.
- `std::generate` does not return anything, but `std::generate_n` returns the iterator past the last generated element - useful for chaining.
- All three accept execution policies since C++17 for parallelism.
- Ranges versions (`std::ranges::transform`, `std::ranges::for_each`, `std::ranges::generate`) are available since C++20 and return richer result types.
