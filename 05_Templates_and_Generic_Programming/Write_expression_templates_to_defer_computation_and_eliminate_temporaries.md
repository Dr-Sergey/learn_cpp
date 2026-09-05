# Write Expression Templates to Defer Computation and Eliminate Temporaries

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/template>  

---

## Topic Overview

### What Are Expression Templates

**Expression templates** are a C++ technique where arithmetic operators return lightweight **proxy objects** instead of computed results. The actual computation is deferred until assignment, eliminating temporary allocations.

The reason this matters in practice is that naive vector arithmetic creates a fresh heap allocation for every operator. If you chain four additions, you get three temporaries that exist just long enough to be added and discarded. Expression templates replace all of those allocations with a single loop at the end.

### The Problem

```cpp
// Naive operator+: creates temporaries
Vec a, b, c, d;
Vec result = a + b + c + d;
// Step 1:  temp1 = a + b       <- allocates temp1
// Step 2:  temp2 = temp1 + c   <- allocates temp2
// Step 3:  result = temp2 + d  <- allocates result
// 2 unnecessary temporary vectors allocated and deallocated!
```

### The Solution

```cpp
// Expression template: no temporaries
Vec result = a + b + c + d;
// Step 1: expr = Add(Add(Add(a, b), c), d)   <- lightweight proxy, no allocation
// Step 2: result[i] = a[i] + b[i] + c[i] + d[i]  <- single loop, no temps
```

### How It Works

The key insight is that `operator+` returns a proxy object (a `VecAdd`) that stores references to its operands. No elements are computed yet. When that proxy is finally assigned to a `Vec`, the `Vec` constructor drives a single loop that calls `operator[]` on the proxy, which recurses through the proxy tree and computes each element on the spot.

```cpp
    a + b + c                        Evaluation (at assignment)
    ---------                        -------------------------

        +                            for (i = 0; i < n; ++i)

       / \                               result[i] = a[i] + b[i] + c[i]

      +   c    <- proxy objects
     / \       (no computation)
    a   b
```

---

## Self-Assessment

### Q1: Implement a minimal expression template for vector addition that avoids intermediate vectors

The CRTP base `VecExpr<Derived>` is what glues everything together. Any expression - whether it is a concrete `Vec` or a `VecAdd` proxy - inherits from `VecExpr` so that `operator+` can accept and return any combination of them. The actual data only moves when you construct or assign a `Vec` from a `VecExpr`.

```cpp
#include <iostream>
#include <vector>
#include <cstddef>
#include <cassert>

// === Expression base: CRTP for static polymorphism ===
template <typename Derived>
struct VecExpr {
    double operator[](std::size_t i) const {
        return static_cast<const Derived&>(*this)[i];
    }
    std::size_t size() const {
        return static_cast<const Derived&>(*this).size();
    }
};

// === Concrete vector: owns data ===
class Vec : public VecExpr<Vec> {
    std::vector<double> data_;
public:
    Vec(std::size_t n, double val = 0.0) : data_(n, val) {}
    Vec(std::initializer_list<double> init) : data_(init) {}

    // Construct from any expression - THIS is where evaluation happens
    template <typename E>
    Vec(const VecExpr<E>& expr) : data_(expr.size()) {
        for (std::size_t i = 0; i < data_.size(); ++i)
            data_[i] = expr[i];  // single loop, element-by-element
    }

    // Assign from any expression
    template <typename E>
    Vec& operator=(const VecExpr<E>& expr) {
        data_.resize(expr.size());
        for (std::size_t i = 0; i < data_.size(); ++i)
            data_[i] = expr[i];
        return *this;
    }

    double  operator[](std::size_t i) const { return data_[i]; }
    double& operator[](std::size_t i)       { return data_[i]; }
    std::size_t size() const { return data_.size(); }
};

// === Addition expression: lightweight proxy, stores references ===
template <typename L, typename R>
class VecAdd : public VecExpr<VecAdd<L, R>> {
    const L& lhs_;
    const R& rhs_;
public:
    VecAdd(const L& l, const R& r) : lhs_(l), rhs_(r) {
        assert(l.size() == r.size());
    }

    double operator[](std::size_t i) const {
        return lhs_[i] + rhs_[i];  // computed on demand, no storage
    }
    std::size_t size() const { return lhs_.size(); }
};

// === Operator+: returns a proxy, not a computed vector ===
template <typename L, typename R>
VecAdd<L, R> operator+(const VecExpr<L>& lhs, const VecExpr<R>& rhs) {
    return VecAdd<L, R>(static_cast<const L&>(lhs), static_cast<const R&>(rhs));
}

// === Scalar multiplication expression ===
template <typename E>
class VecScale : public VecExpr<VecScale<E>> {
    double scalar_;
    const E& expr_;
public:
    VecScale(double s, const E& e) : scalar_(s), expr_(e) {}
    double operator[](std::size_t i) const { return scalar_ * expr_[i]; }
    std::size_t size() const { return expr_.size(); }
};

template <typename E>
VecScale<E> operator*(double s, const VecExpr<E>& expr) {
    return VecScale<E>(s, static_cast<const E&>(expr));
}

int main() {
    Vec a = {1.0, 2.0, 3.0};
    Vec b = {4.0, 5.0, 6.0};
    Vec c = {7.0, 8.0, 9.0};

    // a + b + c creates: VecAdd<VecAdd<Vec, Vec>, Vec>
    // No temporaries! Evaluation happens in Vec's constructor
    Vec result = a + b + c;

    std::cout << "a + b + c = ";
    for (std::size_t i = 0; i < result.size(); ++i)
        std::cout << result[i] << " ";
    std::cout << "\n";  // 12 15 18

    // Scalar multiplication
    Vec scaled = 2.0 * a + b;
    std::cout << "2*a + b = ";
    for (std::size_t i = 0; i < scaled.size(); ++i)
        std::cout << scaled[i] << " ";
    std::cout << "\n";  // 6 9 12

    return 0;
}
```

Watch what happens at the assignment: `Vec result = a + b + c` calls `Vec`'s constructor with a `VecAdd<VecAdd<Vec,Vec>,Vec>`. That constructor runs a single loop, and each `expr[i]` recursively computes `a[i] + b[i] + c[i]` in place - no intermediate storage anywhere.

**Expected output:**

```text
a + b + c = 12 15 18
2*a + b = 6 9 12
```

### Q2: Show that the expression template evaluates element-by-element in a single loop

This version instruments the `Vec` constructor to print every allocation. You will see that forming the proxy expression `a + b + c` produces no allocations at all, and the evaluation into `result` is a single allocation with a single loop.

```cpp
#include <iostream>
#include <vector>
#include <cstddef>
#include <cassert>

// Reusing the expression template from Q1 with instrumentation

template <typename Derived>
struct VecExpr {
    double operator[](std::size_t i) const {
        return static_cast<const Derived&>(*this)[i];
    }
    std::size_t size() const {
        return static_cast<const Derived&>(*this).size();
    }
};

// Instrumented Vec - tracks allocations
static int allocation_count = 0;

class Vec : public VecExpr<Vec> {
    std::vector<double> data_;
public:
    Vec(std::size_t n, double val = 0.0) : data_(n, val) {
        ++allocation_count;
        std::cout << "  [ALLOC] Vec(" << n << ") - allocation #" << allocation_count << "\n";
    }
    Vec(std::initializer_list<double> init) : data_(init) {
        ++allocation_count;
        std::cout << "  [ALLOC] Vec(init) - allocation #" << allocation_count << "\n";
    }

    template <typename E>
    Vec(const VecExpr<E>& expr) : data_(expr.size()) {
        ++allocation_count;
        std::cout << "  [ALLOC] Vec(expr) - allocation #" << allocation_count << "\n";
        std::cout << "  [EVAL] Single loop evaluating all elements:\n";
        for (std::size_t i = 0; i < data_.size(); ++i) {
            data_[i] = expr[i];
            std::cout << "    result[" << i << "] = " << data_[i] << "\n";
        }
    }

    double  operator[](std::size_t i) const { return data_[i]; }
    double& operator[](std::size_t i)       { return data_[i]; }
    std::size_t size() const { return data_.size(); }
};

template <typename L, typename R>
class VecAdd : public VecExpr<VecAdd<L, R>> {
    const L& lhs_;
    const R& rhs_;
public:
    VecAdd(const L& l, const R& r) : lhs_(l), rhs_(r) {}
    double operator[](std::size_t i) const { return lhs_[i] + rhs_[i]; }
    std::size_t size() const { return lhs_.size(); }
};

template <typename L, typename R>
VecAdd<L, R> operator+(const VecExpr<L>& l, const VecExpr<R>& r) {
    return VecAdd<L, R>(static_cast<const L&>(l), static_cast<const R&>(r));
}

int main() {
    std::cout << "=== Creating source vectors ===\n";
    allocation_count = 0;
    Vec a = {1.0, 2.0, 3.0};  // alloc #1
    Vec b = {4.0, 5.0, 6.0};  // alloc #2
    Vec c = {7.0, 8.0, 9.0};  // alloc #3

    std::cout << "\n=== Expression template: a + b + c ===\n";
    std::cout << "operator+ returns proxies (no allocation)...\n";
    auto expr = a + b + c;  // <- NO allocation! Just proxy objects
    std::cout << "Proxy created. Now evaluating into result vector:\n";
    Vec result(expr);  // alloc #4 - single allocation, single loop

    std::cout << "\nTotal allocations: " << allocation_count << "\n";
    std::cout << "Without expression templates: would be 5 (3 source + 2 temps + 1 result)\n";
    std::cout << "With expression templates: " << allocation_count << " (3 source + 1 result)\n";

    return 0;
}
```

### Q3: Explain the compile-time cost (template instantiation depth) and when it outweighs the benefit

Expression templates are a classic example of a technique that improves runtime performance at the cost of compile-time complexity. Each `+` creates a new unique type, so long chains produce deeply nested template types. That makes error messages painful and compilation slow. Here is the full trade-off picture:

| Factor | Expression Templates | Naive Operators |
| --- | --- | --- |
| **Compile time** | Slower - each `+` creates a new unique type | Fast - all return the same type |
| **Template depth** | `a+b+c+d` -> `VecAdd<VecAdd<VecAdd<Vec,Vec>,Vec>,Vec>` (depth 3) | No nesting |
| **Error messages** | Very verbose - deeply nested template types | Clear and short |
| **Debug builds** | Slower - proxies may not be optimized away | Temporaries are straightforward |
| **Optimized builds** | Excellent - single loop, no temps, auto-vectorizable | Extra allocations, cache misses |
| **Code complexity** | High - must define proxy types for every operator | Simple operator overloading |

```cpp
#include <iostream>

int main() {
    std::cout << "=== When Expression Templates Are Worth It ===\n\n";

    std::cout << "USE expression templates when:\n";
    std::cout << "  - Vectors/matrices are large (1000+ elements)\n";
    std::cout << "  - Operations chain frequently (a + b + c + d)\n";
    std::cout << "  - Allocation/copy cost dominates runtime\n";
    std::cout << "  - You're building a math/linear algebra library\n";
    std::cout << "  - Performance is critical (HPC, graphics, ML)\n";

    std::cout << "\nAVOID expression templates when:\n";
    std::cout << "  - Data is small (Vec3, Vec4 - compiler optimizes anyway)\n";
    std::cout << "  - Compile time matters more than runtime\n";
    std::cout << "  - Team is unfamiliar with template metaprogramming\n";
    std::cout << "  - Expressions are simple (a + b only, no chaining)\n";
    std::cout << "  - Debug build performance is important\n";

    std::cout << "\n=== Compile-Time Cost Details ===\n";
    std::cout << "Expression: a + b + c + d + e\n";
    std::cout << "Type generated:\n";
    std::cout << "  VecAdd<VecAdd<VecAdd<VecAdd<Vec,Vec>,Vec>,Vec>,Vec>\n";
    std::cout << "  -> 4 unique template instantiations\n";
    std::cout << "  -> Each has its own operator[] and size()\n";
    std::cout << "  -> Compiler must inline all of them for performance\n";
    std::cout << "  -> Error messages show the full nested type\n";

    std::cout << "\n=== Template Depth Limits ===\n";
    std::cout << "Default limits: GCC=900, Clang=1024, MSVC=500\n";
    std::cout << "Each + adds 1 level of nesting\n";
    std::cout << "Hitting limit: '-ftemplate-depth=N' to increase\n";

    return 0;
}
```

---

## Notes

- Expression templates are used in production libraries: Eigen, Blaze, Armadillo, Boost.uBLAS.
- CRTP (`VecExpr<Derived>`) provides static polymorphism - no virtual call overhead.
- **Dangling reference danger:** Expression proxies store references to operands. Don't store an expression and use it after operands are destroyed.
- In C++20, `std::ranges` views use a similar lazy-evaluation approach.
- For small fixed-size vectors (Vec3, Vec4), modern compilers often optimize naive code equally well - expression templates add unnecessary complexity.
- Always benchmark before introducing expression templates into a codebase.
