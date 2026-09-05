# Use the Strategy pattern with templates for zero-overhead policy injection

**Category:** Design Patterns Modern Takes  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Strategy_pattern>  

---

## Topic Overview

**Policy-based design** uses template parameters to inject behavior at compile time. Unlike `std::function` (which erases types and adds indirection), template policies let the compiler **inline** the strategy entirely, producing code as efficient as hand-written specializations - a "pay for what you use" approach.

The intuition here is that the compiler can only inline something if it knows exactly what it is at compile time. When you pass a `std::function`, the compiler sees an opaque callable and has to call through an indirection layer. When you pass a type parameter, the compiler sees the exact function body and can fold it directly into the generated machine code.

### Multi-Policy Design

A class can accept multiple independent policy parameters, each controlling a different axis of behavior. Every combination of policies is a distinct type with fully inlined behavior.

```cpp
Sorter<ComparePolicy, PartitionPolicy>
    |           |              |
    |    AscendingCmp     LomutoPartition     -> Sorter<AscendingCmp, LomutoPartition>
    |    DescendingCmp    HoarePartition      -> Sorter<DescendingCmp, HoarePartition>
    |    CustomCmp        MedianOf3Partition  -> Sorter<CustomCmp, MedianOf3Partition>
    |
    +-- Each combination is a unique type with fully inlined behavior
```

---

## Self-Assessment

### Q1: Implement a Sorter<ComparePolicy, PartitionPolicy> that composes sorting behavior at compile time

This example shows two independent policies working together. `ComparePolicy` decides the ordering; `PartitionPolicy` decides the partitioning algorithm used during quicksort. Because both are template parameters, the compiler sees through both abstractions and inlines the entire sort.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <utility>

// Compare Policies
struct AscendingCmp {
    template<typename T>
    bool operator()(const T& a, const T& b) const { return a < b; }
};

struct DescendingCmp {
    template<typename T>
    bool operator()(const T& a, const T& b) const { return a > b; }
};

// Partition Policies
struct LomutoPartition {
    template<typename Iter, typename Cmp>
    Iter operator()(Iter lo, Iter hi, Cmp cmp) const {
        auto pivot = *std::prev(hi);
        auto i = lo;
        for (auto j = lo; j != std::prev(hi); ++j) {
            if (cmp(*j, pivot)) {
                std::iter_swap(i, j);
                ++i;
            }
        }
        std::iter_swap(i, std::prev(hi));
        return i;
    }
};

struct HoarePartition {
    template<typename Iter, typename Cmp>
    Iter operator()(Iter lo, Iter hi, Cmp cmp) const {
        auto pivot = *lo;
        auto i = lo;
        auto j = std::prev(hi);
        while (true) {
            while (cmp(*i, pivot)) ++i;
            while (cmp(pivot, *j)) --j;
            if (i >= j) return j;
            std::iter_swap(i, j);
            ++i; --j;
        }
    }
};

// Sorter: composed from two policies
template<typename ComparePolicy = AscendingCmp,
         typename PartitionPolicy = LomutoPartition>
class Sorter {
    ComparePolicy cmp_;
    PartitionPolicy partition_;

    template<typename Iter>
    void quicksort(Iter lo, Iter hi) {
        if (lo >= hi || std::next(lo) == hi) return;
        auto pivot = partition_(lo, hi, cmp_);
        quicksort(lo, pivot);
        quicksort(std::next(pivot), hi);
    }

public:
    template<typename Container>
    void sort(Container& c) {
        quicksort(c.begin(), c.end());
    }
};

int main() {
    std::vector<int> data1 = {5, 2, 8, 1, 9, 3};
    Sorter<AscendingCmp, LomutoPartition> asc_sorter;
    asc_sorter.sort(data1);
    for (int x : data1) std::cout << x << ' ';  // 1 2 3 5 8 9
    std::cout << '\n';

    std::vector<int> data2 = {5, 2, 8, 1, 9, 3};
    Sorter<DescendingCmp, LomutoPartition> desc_sorter;
    desc_sorter.sort(data2);
    for (int x : data2) std::cout << x << ' ';  // 9 8 5 3 2 1
    std::cout << '\n';
}
```

Notice the default template arguments on `Sorter<>`. You can write `Sorter<> s;` and get ascending order with Lomuto partitioning without spelling out the types. That's a nice touch for user-facing APIs where a sensible default exists.

### Q2: Show that the template strategy generates inlined code with no virtual dispatch overhead

The benchmark here shows the concrete performance difference between the two approaches. The reason template policy is faster comes down to what the compiler can see: with a concrete type, it inlines the comparison directly into the sort's inner loop. With virtual dispatch, there's a vtable pointer dereference on every comparison, which also prevents the auto-vectorizer from analyzing the loop.

**Answer:**

```cpp
#include <iostream>
#include <chrono>
#include <vector>
#include <algorithm>
#include <functional>
#include <numeric>

// Template policy: compiles to inlined code
struct InlinedCompare {
    bool operator()(int a, int b) const { return a < b; }
};

template<typename Cmp>
void template_sort(std::vector<int>& v, Cmp cmp) {
    std::sort(v.begin(), v.end(), cmp);
    // Compiler sees the exact type of cmp -> inlines operator()
    // No function pointer, no vtable lookup
}

// Virtual dispatch: runtime indirection
struct ICompare {
    virtual ~ICompare() = default;
    virtual bool compare(int a, int b) const = 0;
};

struct VirtualAscending : ICompare {
    bool compare(int a, int b) const override { return a < b; }
};

void virtual_sort(std::vector<int>& v, const ICompare& cmp) {
    std::sort(v.begin(), v.end(), [&](int a, int b) {
        return cmp.compare(a, b);  // Virtual call per comparison!
    });
}

int main() {
    constexpr int N = 5'000'000;

    // Benchmark template policy
    std::vector<int> v1(N);
    std::iota(v1.begin(), v1.end(), 0);
    std::reverse(v1.begin(), v1.end());
    auto t1 = std::chrono::high_resolution_clock::now();
    template_sort(v1, InlinedCompare{});
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark virtual dispatch
    std::vector<int> v2(N);
    std::iota(v2.begin(), v2.end(), 0);
    std::reverse(v2.begin(), v2.end());
    VirtualAscending vcmp;
    auto t3 = std::chrono::high_resolution_clock::now();
    virtual_sort(v2, vcmp);
    auto t4 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "Template policy: " << ms(t2 - t1).count() << " ms\n";
    std::cout << "Virtual dispatch: " << ms(t4 - t3).count() << " ms\n";
    // Template policy ~= same as raw std::sort (inlined)
    // Virtual dispatch ~20-50% slower (indirect call prevents inlining)

    /*
    Why template is faster:

    - Compiler knows exact type -> inlines compare function
    - No branch prediction penalty from indirect calls
    - Enables SIMD auto-vectorization in tight loops
    - No vtable pointer dereference

    */
}
```

The 20-50% slowdown from virtual dispatch doesn't come from the vtable lookup itself being slow - it comes from what that lookup prevents. Once the compiler can't see the comparison body, it can't fuse it with the surrounding loop, can't vectorize, and can't hoist common subexpressions.

### Q3: Compare template strategy with std::function strategy for stored vs inline callables

This example puts the decision guide in concrete code. Notice that `TemplateValidator` can't change its policy after construction - the type is baked in. `RuntimeValidator` can call `set_policy` and swap to a completely different rule based on a config flag, user input, or any runtime condition.

**Answer:**

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <string>
#include <algorithm>
#include <utility>

// Template policy: inline, compile-time fixed
template<typename ValidationPolicy>
class TemplateValidator {
    ValidationPolicy policy_;
public:
    bool validate(const std::string& input) {
        return policy_(input);  // Inlined - compiler sees exact function body
    }
};

struct NotEmpty {
    bool operator()(const std::string& s) const { return !s.empty(); }
};
struct MaxLength {
    size_t max;
    bool operator()(const std::string& s) const { return s.size() <= max; }
};

// std::function: type-erased, runtime swappable
class RuntimeValidator {
    std::function<bool(const std::string&)> policy_;
public:
    explicit RuntimeValidator(std::function<bool(const std::string&)> p)
        : policy_(std::move(p)) {}

    void set_policy(std::function<bool(const std::string&)> p) {
        policy_ = std::move(p);  // Can change at runtime!
    }

    bool validate(const std::string& input) {
        return policy_(input);  // Indirect call - not inlined
    }
};

int main() {
    // Template: strategy fixed at compile time
    TemplateValidator<NotEmpty> v1;
    TemplateValidator<MaxLength> v2{MaxLength{10}};

    std::cout << v1.validate("hello") << '\n';       // 1
    std::cout << v2.validate("hello world!") << '\n'; // 0 (12 > 10)

    // std::function: strategy changeable at runtime
    RuntimeValidator rv([](const std::string& s) { return !s.empty(); });
    std::cout << rv.validate("hello") << '\n';  // 1

    // Swap strategy based on user config (not possible with templates!)
    bool strict_mode = true;
    if (strict_mode) {
        rv.set_policy([](const std::string& s) {
            return !s.empty() && s.size() <= 10 && s.find(' ') == std::string::npos;
        });
    }
    std::cout << rv.validate("hello world") << '\n';  // 0 (has space)
    std::cout << rv.validate("hello") << '\n';         // 1

    /*
    Decision guide:
    Need                         | Use Template | Use std::function
    -----------------------------|--------------|------------------
    Maximum performance          | Yes          |
    Strategy known at compile    | Yes          |
    Swap strategy at runtime     |              | Yes
    Plugin / user-configured     |              | Yes
    Minimize binary size         |              | Yes
    Strategy captures state      | Yes (functor)| Yes (closure)
    */
}
```

The decision guide in the comment is worth keeping in mind. The two approaches are complementary, not competing. A common pattern is to use template policies for the hot inner loop and `std::function` for the configuration callbacks that fire occasionally.

---

## Notes

- Template policies are the backbone of STL design: `std::sort` takes a comparator policy, `std::allocator` is a policy, `std::basic_string` has a traits policy.
- Multi-policy classes (two or more template parameters) should provide sensible defaults so `Sorter<> s;` works without spelling out every type.
- `std::function` costs ~40 bytes of storage plus type erasure overhead; template policies add 0 bytes per instance (the policy is usually a stateless struct).
- Combine both when it makes sense: use a template policy for the hot path, and `std::function` for configuration callbacks that run rarely.
