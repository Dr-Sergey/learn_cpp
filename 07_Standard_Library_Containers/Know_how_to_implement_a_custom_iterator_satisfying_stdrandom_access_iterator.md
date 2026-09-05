# Know how to implement a custom iterator satisfying std::random_access_iterator

**Category:** Standard Library Containers  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/iterator/random_access_iterator>  

---

## Topic Overview

C++20 introduced **iterator concepts** that replace the legacy iterator tag-based classification. `std::random_access_iterator` is the strongest iterator concept (before `std::contiguous_iterator`) and enables algorithms like `std::sort`, `std::binary_search`, and all `std::ranges` algorithms.

Why does this matter? Because algorithms in `<algorithm>` and `<ranges>` check these concepts at compile time. If your iterator satisfies `std::random_access_iterator`, you get `std::sort` for free. If it only satisfies `std::forward_iterator`, `std::sort` won't compile against it - which is actually helpful, because `std::sort` genuinely needs the ability to jump to arbitrary positions in O(1).

### Iterator Concept Hierarchy (C++20)

Each level in this hierarchy adds more capabilities. The indentation shows that each level includes everything above it.

```cpp
std::input_or_output_iterator
├── std::input_iterator
│   └── std::forward_iterator
│       └── std::bidirectional_iterator
│            └── std::random_access_iterator
│                 └── std::contiguous_iterator
└── std::output_iterator
```

### Requirements for std::random_access_iterator

A type `I` satisfies `std::random_access_iterator` if it satisfies `std::bidirectional_iterator` AND provides all of these. The reason there are so many requirements is that random access means "I can jump anywhere in O(1)," and that needs arithmetic operators, distance computation, and ordering.

| Requirement | Expression | Return Type |
| --- | --- | --- |
| Advance by n | `i += n` | `I&` |
| Retreat by n | `i -= n` | `I&` |
| Offset from iterator | `i + n`, `n + i` | `I` |
| Negative offset | `i - n` | `I` |
| Distance between iterators | `i - j` | `std::iter_difference_t<I>` |
| Subscript | `i[n]` | `std::iter_reference_t<I>` |
| Three-way or relational compare | `i <=> j` or `<,>,<=,>=` | `std::partial_ordering` or stronger |

Additionally, the iterator must provide:

- **`iterator_concept`** type alias (or `iterator_category`) = `std::random_access_iterator_tag`
- **`value_type`**, **`difference_type`** type aliases
- Default constructible, copyable, destructible

### Core Example: Stride Iterator

Here's a real-world-useful iterator that walks over a range in steps. A stride-2 iterator over `{0,1,2,3,4,5,6,7,8,9}` visits `{0,2,4,6,8}`. The interesting design point is `operator-`: since the underlying pointer gap is `stride * logical_distance`, we divide by stride to recover the logical distance that algorithms need.

```cpp
#include <iostream>
#include <iterator>
#include <algorithm>
#include <concepts>
#include <vector>
#include <cassert>

// A stride iterator that walks over a contiguous range with a custom step
template <typename T>
class StrideIterator {
public:
    // Required type aliases
    using iterator_concept  = std::random_access_iterator_tag;
    using iterator_category = std::random_access_iterator_tag;
    using value_type        = T;
    using difference_type   = std::ptrdiff_t;
    using pointer           = T*;
    using reference         = T&;

private:
    T* ptr_ = nullptr;
    difference_type stride_ = 1;

public:
    // Default constructor (required)
    StrideIterator() = default;

    StrideIterator(T* ptr, difference_type stride)
        : ptr_(ptr), stride_(stride) {}

    // Dereference
    reference operator*() const { return *ptr_; }
    pointer operator->() const { return ptr_; }

    // Increment / Decrement
    StrideIterator& operator++() { ptr_ += stride_; return *this; }
    StrideIterator operator++(int) { auto tmp = *this; ++(*this); return tmp; }
    StrideIterator& operator--() { ptr_ -= stride_; return *this; }
    StrideIterator operator--(int) { auto tmp = *this; --(*this); return tmp; }

    // Random access operations
    StrideIterator& operator+=(difference_type n) { ptr_ += n * stride_; return *this; }
    StrideIterator& operator-=(difference_type n) { ptr_ -= n * stride_; return *this; }

    friend StrideIterator operator+(StrideIterator it, difference_type n) { it += n; return it; }
    friend StrideIterator operator+(difference_type n, StrideIterator it) { it += n; return it; }
    friend StrideIterator operator-(StrideIterator it, difference_type n) { it -= n; return it; }

    friend difference_type operator-(const StrideIterator& a, const StrideIterator& b) {
        return (a.ptr_ - b.ptr_) / a.stride_;
    }

    reference operator[](difference_type n) const { return *(ptr_ + n * stride_); }

    // Comparison
    friend bool operator==(const StrideIterator& a, const StrideIterator& b) {
        return a.ptr_ == b.ptr_;
    }
    friend auto operator<=>(const StrideIterator& a, const StrideIterator& b) {
        return a.ptr_ <=> b.ptr_;
    }
};

// Verify the concept at compile time
static_assert(std::random_access_iterator<StrideIterator<int>>);

int main() {
    std::vector<int> data = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

    // View every 2nd element: 0, 2, 4, 6, 8
    StrideIterator<int> begin(data.data(), 2);
    StrideIterator<int> end(data.data() + 10, 2);

    std::cout << "Every 2nd element: ";
    for (auto it = begin; it != end; ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    // Output: Every 2nd element: 0 2 4 6 8

    // std::sort works because we satisfy random_access_iterator
    std::sort(begin, end, std::greater<>{});
    std::cout << "Sorted descending: ";
    for (auto it = begin; it != end; ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    // Output: Sorted descending: 8 6 4 2 0

    return 0;
}
```

`std::sort` needs to swap elements at arbitrary positions, so it calls `operator[]` and `operator-` heavily. Our stride iterator handles both correctly by accounting for the stride in all arithmetic.

### Important Notes

- In C++20, prefer **concepts** (`std::random_access_iterator<It>`) over legacy tag dispatch.
- `iterator_concept` takes priority over `iterator_category` when checking concepts.
- `std::contiguous_iterator` additionally requires `std::to_address(it)` to work, returning a raw pointer.
- The **`operator<=>`** provides all six comparison operators automatically (C++20).
- Always verify your iterator with `static_assert` before using it with algorithms.

---

## Self-Assessment

### Q1: Write a RandomAccessIterator for a custom stride-based array view

This takes the stride iterator idea one step further by wrapping it in a proper view class with `begin()` and `end()`. The key thing to notice is that the end iterator's pointer is computed as `data + count * stride` - it points just past the last strided element in the underlying array.

```cpp
#include <iostream>
#include <iterator>
#include <algorithm>
#include <ranges>
#include <vector>

template <typename T>
class StridedView {
    T* data_;
    std::ptrdiff_t stride_;
    std::size_t count_;  // number of strided elements

public:
    class Iterator {
    public:
        using iterator_concept  = std::random_access_iterator_tag;
        using value_type        = T;
        using difference_type   = std::ptrdiff_t;
        using pointer           = T*;
        using reference         = T&;

    private:
        T* ptr_ = nullptr;
        difference_type stride_ = 1;

    public:
        Iterator() = default;
        Iterator(T* p, difference_type s) : ptr_(p), stride_(s) {}

        reference operator*() const { return *ptr_; }
        pointer operator->() const { return ptr_; }

        Iterator& operator++() { ptr_ += stride_; return *this; }
        Iterator operator++(int) { auto t = *this; ++(*this); return t; }
        Iterator& operator--() { ptr_ -= stride_; return *this; }
        Iterator operator--(int) { auto t = *this; --(*this); return t; }

        Iterator& operator+=(difference_type n) { ptr_ += n * stride_; return *this; }
        Iterator& operator-=(difference_type n) { ptr_ -= n * stride_; return *this; }

        friend Iterator operator+(Iterator it, difference_type n) { it += n; return it; }
        friend Iterator operator+(difference_type n, Iterator it) { it += n; return it; }
        friend Iterator operator-(Iterator it, difference_type n) { it -= n; return it; }
        friend difference_type operator-(const Iterator& a, const Iterator& b) {
            return (a.ptr_ - b.ptr_) / a.stride_;
        }

        reference operator[](difference_type n) const { return *(ptr_ + n * stride_); }

        friend bool operator==(const Iterator& a, const Iterator& b) { return a.ptr_ == b.ptr_; }
        friend auto operator<=>(const Iterator& a, const Iterator& b) { return a.ptr_ <=> b.ptr_; }
    };

    StridedView(T* data, std::size_t total, std::ptrdiff_t stride)
        : data_(data), stride_(stride), count_((total + stride - 1) / stride) {}

    Iterator begin() { return Iterator(data_, stride_); }
    Iterator end()   { return Iterator(data_ + count_ * stride_, stride_); }
    std::size_t size() const { return count_; }
};

int main() {
    std::vector<int> v = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100};

    // View every 3rd element: 10, 40, 70, 100
    StridedView view(v.data(), v.size(), 3);

    static_assert(std::random_access_iterator<StridedView<int>::Iterator>);

    std::cout << "Stride-3 view: ";
    for (int x : view) {
        std::cout << x << " ";
    }
    std::cout << "\n";
    // Output: Stride-3 view: 10 40 70 100

    // Sort the strided view in descending order
    std::sort(view.begin(), view.end(), std::greater<>{});
    std::cout << "After sort: ";
    for (int x : view) {
        std::cout << x << " ";
    }
    std::cout << "\n";
    // Output: After sort: 100 70 40 10

    // Original vector is modified at stride positions:
    std::cout << "Original vector: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: Original vector: 100 20 30 70 50 60 40 80 90 10

    return 0;
}
```

**How it works:**

- `StridedView` wraps a raw pointer with a stride, exposing begin/end iterators.
- The nested `Iterator` class satisfies `std::random_access_iterator` by providing all required operations.
- `operator-` between iterators divides the pointer difference by stride to get a logical distance.
- `std::sort` works directly because the iterator satisfies the random access concept - it can jump to any position in O(1).

### Q2: Verify compliance using static_asserts on iterator concept requirements

The best workflow when writing an iterator is to put your `static_assert` checks right after the class definition. If your iterator is missing something, the compiler error message from a failed concept check is far more readable than whatever you'd get trying to use the iterator with `std::sort` and seeing a ten-line template substitution failure.

```cpp
#include <iostream>
#include <iterator>
#include <concepts>
#include <type_traits>
#include <ranges>
#include <algorithm>
#include <cstddef>
#include <vector>

// Minimal random access iterator for an int array
struct MyIterator {
    using iterator_concept  = std::random_access_iterator_tag;
    using value_type        = int;
    using difference_type   = std::ptrdiff_t;
    using pointer           = int*;
    using reference         = int&;

    int* p = nullptr;

    MyIterator() = default;
    MyIterator(int* ptr) : p(ptr) {}

    reference operator*() const { return *p; }
    pointer operator->() const { return p; }

    MyIterator& operator++() { ++p; return *this; }
    MyIterator operator++(int) { auto t = *this; ++p; return t; }
    MyIterator& operator--() { --p; return *this; }
    MyIterator operator--(int) { auto t = *this; --p; return t; }

    MyIterator& operator+=(difference_type n) { p += n; return *this; }
    MyIterator& operator-=(difference_type n) { p -= n; return *this; }

    friend MyIterator operator+(MyIterator it, difference_type n) { it += n; return it; }
    friend MyIterator operator+(difference_type n, MyIterator it) { it += n; return it; }
    friend MyIterator operator-(MyIterator it, difference_type n) { it -= n; return it; }
    friend difference_type operator-(const MyIterator& a, const MyIterator& b) { return a.p - b.p; }

    reference operator[](difference_type n) const { return *(p + n); }

    friend bool operator==(const MyIterator& a, const MyIterator& b) { return a.p == b.p; }
    friend auto operator<=>(const MyIterator& a, const MyIterator& b) { return a.p <=> b.p; }
};

// Verify every level of the hierarchy
static_assert(std::input_or_output_iterator<MyIterator>);
static_assert(std::input_iterator<MyIterator>);
static_assert(std::forward_iterator<MyIterator>);
static_assert(std::bidirectional_iterator<MyIterator>);
static_assert(std::random_access_iterator<MyIterator>);
static_assert(std::contiguous_iterator<MyIterator>);  // Also contiguous (wraps raw pointer)

// Verify associated types
static_assert(std::same_as<std::iter_value_t<MyIterator>, int>);
static_assert(std::same_as<std::iter_difference_t<MyIterator>, std::ptrdiff_t>);
static_assert(std::same_as<std::iter_reference_t<MyIterator>, int&>);

// Verify it works with ranges
static_assert(std::ranges::range<std::vector<int>>);

int main() {
    int arr[] = {5, 3, 1, 4, 2};
    MyIterator b(arr), e(arr + 5);

    std::sort(b, e);
    for (auto it = b; it != e; ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    // Output: 1 2 3 4 5

    std::cout << "All static_assert checks passed!\n";
    return 0;
}
```

**How it works:**

- `static_assert` checks are evaluated at compile time - if any concept isn't satisfied, you get a clear error.
- The hierarchy is checked incrementally: if `bidirectional_iterator` fails, you know you're missing `--` operators.
- `std::iter_value_t`, `std::iter_difference_t`, etc. extract associated types through the iterator traits machinery.
- Since `MyIterator` directly wraps a raw pointer and provides `operator->` returning `int*`, it also satisfies `std::contiguous_iterator`.

### Q3: Show that a correct random access iterator enables std::sort and ranges algorithms on your type

Here's the payoff for all that boilerplate - once the concept is satisfied, the entire standard library opens up. Notice the variety of algorithms being used here, each of which has specific iterator requirements.

```cpp
#include <iostream>
#include <iterator>
#include <algorithm>
#include <ranges>
#include <numeric>
#include <vector>

// A simple fixed-size container with a random access iterator
template <typename T, std::size_t N>
class FixedArray {
    T data_[N]{};

public:
    // Iterator is just a thin wrapper around T*
    struct Iterator {
        using iterator_concept  = std::random_access_iterator_tag;
        using value_type        = T;
        using difference_type   = std::ptrdiff_t;
        using pointer           = T*;
        using reference         = T&;

        T* p = nullptr;
        Iterator() = default;
        Iterator(T* ptr) : p(ptr) {}

        reference operator*() const { return *p; }
        pointer operator->() const { return p; }
        Iterator& operator++() { ++p; return *this; }
        Iterator operator++(int) { auto t = *this; ++p; return t; }
        Iterator& operator--() { --p; return *this; }
        Iterator operator--(int) { auto t = *this; --p; return t; }
        Iterator& operator+=(difference_type n) { p += n; return *this; }
        Iterator& operator-=(difference_type n) { p -= n; return *this; }
        friend Iterator operator+(Iterator it, difference_type n) { it += n; return it; }
        friend Iterator operator+(difference_type n, Iterator it) { it += n; return it; }
        friend Iterator operator-(Iterator it, difference_type n) { it -= n; return it; }
        friend difference_type operator-(const Iterator& a, const Iterator& b) { return a.p - b.p; }
        reference operator[](difference_type n) const { return *(p + n); }
        friend bool operator==(const Iterator& a, const Iterator& b) { return a.p == b.p; }
        friend auto operator<=>(const Iterator& a, const Iterator& b) { return a.p <=> b.p; }
    };

    Iterator begin() { return Iterator(data_); }
    Iterator end()   { return Iterator(data_ + N); }
    T& operator[](std::size_t i) { return data_[i]; }
    static constexpr std::size_t size() { return N; }
};

int main() {
    FixedArray<int, 8> arr;
    // Fill: 8, 7, 6, 5, 4, 3, 2, 1
    std::iota(arr.begin(), arr.end(), 1);
    std::ranges::reverse(arr);

    std::cout << "Initial: ";
    std::ranges::for_each(arr, [](int x) { std::cout << x << " "; });
    std::cout << "\n";
    // Output: Initial: 8 7 6 5 4 3 2 1

    // std::sort (requires random_access_iterator)
    std::sort(arr.begin(), arr.end());
    std::cout << "After sort: ";
    for (int x : arr) std::cout << x << " ";
    std::cout << "\n";
    // Output: After sort: 1 2 3 4 5 6 7 8

    // std::ranges::nth_element (requires random_access_iterator)
    std::ranges::nth_element(arr, arr.begin() + 3, std::greater<>{});
    std::cout << "4th largest: " << arr[3] << "\n";
    // Output: 4th largest: 5

    // std::ranges::partial_sort
    std::ranges::partial_sort(arr, arr.begin() + 3);
    std::cout << "Top 3 smallest: ";
    for (auto it = arr.begin(); it != arr.begin() + 3; ++it)
        std::cout << *it << " ";
    std::cout << "\n";
    // Output: Top 3 smallest: 1 2 3

    // Binary search (requires sorted + random access for O(log n))
    std::sort(arr.begin(), arr.end());
    bool found = std::binary_search(arr.begin(), arr.end(), 5);
    std::cout << "Found 5: " << std::boolalpha << found << "\n";
    // Output: Found 5: true

    // ranges::lower_bound
    auto lb = std::ranges::lower_bound(arr, 5);
    std::cout << "lower_bound(5) -> " << *lb << "\n";
    // Output: lower_bound(5) -> 5

    return 0;
}
```

**How it works:**

- `FixedArray` is a custom container with no standard library inheritance - it provides `begin()` and `end()` returning our `Iterator`.
- Because `Iterator` satisfies `std::random_access_iterator`, **all** standard algorithms that require random access work:
  - `std::sort` - O(n log n) with random swaps
  - `std::nth_element` - O(n) average, needs random access partitioning
  - `std::partial_sort` - O(n log k), needs heap + random access
  - `std::binary_search` / `std::lower_bound` - O(log n), needs `it + n` jumps
- Range-based for loops work because `begin()` and `end()` are provided.
- `std::ranges::` algorithms verify concepts at call time, giving clear errors if the iterator is deficient.

---

## Notes

- **Minimum viable random access iterator:** default constructible, copyable, `*`, `++`, `--`, `+=`, `-=`, `+`, `-` (both forms), `[]`, `==`, `<=>`, plus type aliases.
- **Common pitfalls:** Forgetting default constructor, missing `n + iter` overload (only providing `iter + n`), incorrect `operator-` return type.
- **`iterator_concept` vs `iterator_category`:** Use `iterator_concept` for C++20 concept-based dispatch. `iterator_category` is for legacy `<algorithm>` tag dispatch. Provide both for maximum compatibility.
- **Testing tip:** Use `static_assert(std::random_access_iterator<YourIt>)` early - the compiler error messages when a concept fails are much clearer than runtime failures.
- **`std::contiguous_iterator`** adds one more requirement: `std::to_address(it)` must return the underlying pointer. If your iterator wraps a raw pointer, you likely satisfy this too.
