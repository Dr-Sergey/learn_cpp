# Write Expression Templates to Eliminate Temporaries in Arithmetic Chains

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/templates>  

---

## Topic Overview

### Expression Templates for Fixed-Size Vectors

While the previous topic covered dynamic vector expression templates, this topic focuses on **fixed-size types** like `Vec3` where the goal is eliminating temporaries in chains like `a + b + c`:

```cpp
// Without expression templates:
Vec3 a + b + c;
// temp1 = a + b     <- copy 3 doubles
// result = temp1 + c <- copy 3 more doubles
// Total: 2 temporary Vec3 objects

// With expression templates:
Vec3 result = a + b + c;
// result.x = a.x + b.x + c.x   <- single pass, no temps
// result.y = a.y + b.y + c.y
// result.z = a.z + b.z + c.z
```

### Key Insight: Assembly Comparison

With `-O2`, modern compilers often optimize the naive version to match expression templates for small types. But for **medium-sized types** (10-100 elements) or when the compiler can't see through the abstraction, expression templates guarantee optimal code.

---

## Self-Assessment

### Q1: Implement a simple `Vec3` expression template that avoids temporaries in `a + b + c`

The structure here is the same CRTP approach as with dynamic vectors, but simpler because there is no dynamic allocation to worry about. Every expression proxy just returns scalars for `x()`, `y()`, and `z()`. The `Vec3` constructor that takes a `const Expr<E>&` is the single evaluation point where all the chained additions actually compute.

```cpp
#include <iostream>
#include <cmath>

// === CRTP Expression base ===
template <typename E>
struct Expr {
    double x() const { return static_cast<const E&>(*this).x(); }
    double y() const { return static_cast<const E&>(*this).y(); }
    double z() const { return static_cast<const E&>(*this).z(); }
};

// === Concrete Vec3 ===
struct Vec3 : Expr<Vec3> {
    double x_, y_, z_;

    Vec3() : x_(0), y_(0), z_(0) {}
    Vec3(double x, double y, double z) : x_(x), y_(y), z_(z) {}

    // Construct from any expression - single evaluation point
    template <typename E>
    Vec3(const Expr<E>& expr)
        : x_(expr.x()), y_(expr.y()), z_(expr.z()) {}

    template <typename E>
    Vec3& operator=(const Expr<E>& expr) {
        x_ = expr.x(); y_ = expr.y(); z_ = expr.z();
        return *this;
    }

    double x() const { return x_; }
    double y() const { return y_; }
    double z() const { return z_; }

    double length() const { return std::sqrt(x_*x_ + y_*y_ + z_*z_); }

    friend std::ostream& operator<<(std::ostream& os, const Vec3& v) {
        return os << "(" << v.x_ << ", " << v.y_ << ", " << v.z_ << ")";
    }
};

// === Addition expression proxy ===
template <typename L, typename R>
struct AddExpr : Expr<AddExpr<L, R>> {
    const L& lhs;
    const R& rhs;
    AddExpr(const L& l, const R& r) : lhs(l), rhs(r) {}
    double x() const { return lhs.x() + rhs.x(); }
    double y() const { return lhs.y() + rhs.y(); }
    double z() const { return lhs.z() + rhs.z(); }
};

template <typename L, typename R>
AddExpr<L, R> operator+(const Expr<L>& l, const Expr<R>& r) {
    return {static_cast<const L&>(l), static_cast<const R&>(r)};
}

// === Subtraction expression proxy ===
template <typename L, typename R>
struct SubExpr : Expr<SubExpr<L, R>> {
    const L& lhs;
    const R& rhs;
    SubExpr(const L& l, const R& r) : lhs(l), rhs(r) {}
    double x() const { return lhs.x() - rhs.x(); }
    double y() const { return lhs.y() - rhs.y(); }
    double z() const { return lhs.z() - rhs.z(); }
};

template <typename L, typename R>
SubExpr<L, R> operator-(const Expr<L>& l, const Expr<R>& r) {
    return {static_cast<const L&>(l), static_cast<const R&>(r)};
}

// === Scalar multiply expression proxy ===
template <typename E>
struct ScaleExpr : Expr<ScaleExpr<E>> {
    double s;
    const E& expr;
    ScaleExpr(double s, const E& e) : s(s), expr(e) {}
    double x() const { return s * expr.x(); }
    double y() const { return s * expr.y(); }
    double z() const { return s * expr.z(); }
};

template <typename E>
ScaleExpr<E> operator*(double s, const Expr<E>& e) {
    return {s, static_cast<const E&>(e)};
}

int main() {
    Vec3 a(1.0, 2.0, 3.0);
    Vec3 b(4.0, 5.0, 6.0);
    Vec3 c(7.0, 8.0, 9.0);

    // a + b + c -> AddExpr<AddExpr<Vec3,Vec3>, Vec3>
    // No temporaries until assignment!
    Vec3 sum = a + b + c;
    std::cout << "a + b + c = " << sum << "\n";  // (12, 15, 18)

    // Complex expression: 2*a + b - c
    Vec3 complex = 2.0 * a + b - c;
    std::cout << "2*a + b - c = " << complex << "\n";  // (-1, 1, 3)

    // Length of expression (evaluates inline)
    Vec3 diff = a - b;
    std::cout << "|a - b| = " << diff.length() << "\n";  // ~5.196

    return 0;
}
```

Notice that `2.0 * a + b - c` builds up a proxy tree of type `SubExpr<AddExpr<ScaleExpr<Vec3>,Vec3>,Vec3>` without computing a single number. The three coordinates are computed all at once when `complex` is initialized.

**Expected output:**

```text
a + b + c = (12, 15, 18)
2*a + b - c = (-1, 1, 3)
|a - b| = 5.19615
```

### Q2: Show the assembly difference between expression templates and operator-overloaded temporaries

The key point here is that with `-O2`, modern compilers often produce identical assembly for naive `Vec3` and expression-template `Vec3`. The difference shows up more clearly for larger types or when you force the naive approach to store its temporaries (as `volatile` does in the snippet below). Understanding where the improvement is guaranteed matters as much as understanding the technique itself.

```cpp
#include <iostream>

// === NAIVE: Returns temporary Vec3 ===
struct NaiveVec3 {
    double x, y, z;
    NaiveVec3(double x, double y, double z) : x(x), y(y), z(z) {}
};

// Each operator creates a temporary
NaiveVec3 operator+(const NaiveVec3& a, const NaiveVec3& b) {
    return {a.x + b.x, a.y + b.y, a.z + b.z};
}

// === EXPRESSION TEMPLATE: (reusing the Expr/AddExpr from Q1 conceptually) ===
// In the expression template version:
//   Vec3 result = a + b + c;
// compiles to exactly:
//   result.x = a.x + b.x + c.x;
//   result.y = a.y + b.y + c.y;
//   result.z = a.z + b.z + c.z;

int main() {
    std::cout << "=== Assembly Comparison (use godbolt.org with -O2) ===\n\n";

    std::cout << "NAIVE (a + b + c):\n";
    std::cout << "  ; temp1 = a + b\n";
    std::cout << "  movsd   xmm0, [a.x]\n";
    std::cout << "  addsd   xmm0, [b.x]    ; temp1.x\n";
    std::cout << "  movsd   [rsp+temp1.x], xmm0\n";
    std::cout << "  ; (repeat for y, z)\n";
    std::cout << "  ; result = temp1 + c\n";
    std::cout << "  movsd   xmm0, [rsp+temp1.x]\n";
    std::cout << "  addsd   xmm0, [c.x]    ; result.x\n";
    std::cout << "  ; Extra loads/stores for temporaries!\n\n";

    std::cout << "EXPRESSION TEMPLATE (a + b + c):\n";
    std::cout << "  movsd   xmm0, [a.x]\n";
    std::cout << "  addsd   xmm0, [b.x]\n";
    std::cout << "  addsd   xmm0, [c.x]    ; result.x directly!\n";
    std::cout << "  ; No temporaries, no extra loads/stores\n\n";

    std::cout << "NOTE: With -O2, modern compilers (GCC 12+, Clang 15+)\n";
    std::cout << "often optimize the naive version to match for Vec3.\n";
    std::cout << "The difference is more pronounced for:\n";
    std::cout << "  - Larger types (Matrix4x4, VecN)\n";
    std::cout << "  - Complex expressions (a*b + c*d - e)\n";
    std::cout << "  - Heap-allocated data (std::vector<double>)\n";

    // Actual benchmark:
    NaiveVec3 a(1, 2, 3), b(4, 5, 6), c(7, 8, 9);
    volatile NaiveVec3 result = a + b + c;  // volatile prevents optimization
    std::cout << "\nNaive result: (" << result.x << ", " << result.y << ", " << result.z << ")\n";

    return 0;
}
```

### Q3: Explain why expression templates increase compile time and template error verbosity

The reason this trips people up is that the compile-time cost of expression templates scales with the length of your expressions. Each `+` introduces a new unique template instantiation. With a chain of five additions you get four new types, and the compiler must instantiate each one, generate debug symbols for each one, and inline through all of them during optimization.

```cpp
#include <iostream>

int main() {
    std::cout << "=== Compile-Time and Error Verbosity Costs ===\n\n";

    // 1. Template Instantiation Growth
    std::cout << "1. TEMPLATE INSTANTIATION GROWTH\n";
    std::cout << "   a + b           -> AddExpr<Vec3, Vec3>\n";
    std::cout << "   a + b + c       -> AddExpr<AddExpr<Vec3,Vec3>, Vec3>\n";
    std::cout << "   a + b + c + d   -> AddExpr<AddExpr<AddExpr<Vec3,Vec3>,Vec3>, Vec3>\n";
    std::cout << "   Each operator creates a UNIQUE type -> more instantiations\n\n";

    // 2. Error Messages
    std::cout << "2. ERROR MESSAGE VERBOSITY\n";
    std::cout << "   If you accidentally write: Vec3 v = a + 42;\n";
    std::cout << "   Error: no matching function for 'operator+'\n";
    std::cout << "   Candidate: AddExpr<Vec3,Vec3> operator+(\n";
    std::cout << "       const Expr<Vec3>&, const Expr<Vec3>&)\n";
    std::cout << "   Note: the types are deeply nested when errors occur\n";
    std::cout << "   mid-chain, e.g.:\n";
    std::cout << "     AddExpr<SubExpr<ScaleExpr<Vec3>,Vec3>,\n";
    std::cout << "            AddExpr<Vec3,ScaleExpr<Vec3>>>\n\n";

    // 3. Compile Time Factors
    std::cout << "3. COMPILE TIME FACTORS\n";
    std::cout << "   Factor                     Impact\n";
    std::cout << "   -------------------------  ------\n";
    std::cout << "   Unique type per expression  Each expression is a new type\n";
    std::cout << "   N operators = N types        Compiler must instantiate each\n";
    std::cout << "   Inlining depth              Optimizer must inline N levels\n";
    std::cout << "   CRTP base class             Each derived type triggers base\n";
    std::cout << "   Debug info                  Symbols for all proxy types\n\n";

    // 4. Mitigation Strategies
    std::cout << "4. MITIGATION STRATEGIES\n";
    std::cout << "   - Limit expression depth (eval() at intermediate points)\n";
    std::cout << "   - Use concepts (C++20) for clearer error messages\n";
    std::cout << "   - Use extern template to pre-instantiate common expressions\n";
    std::cout << "   - Profile compile times with -ftime-trace (Clang)\n";
    std::cout << "   - For small types: benchmark first, expression templates may not help\n";

    return 0;
}
```

---

## Notes

- For `Vec3`/`Vec4`: modern compilers with `-O2` often eliminate temporaries anyway via copy elision and inlining. **Benchmark before adding complexity.**
- For large vectors (`std::vector<double>`): expression templates provide significant speedup by avoiding heap allocations.
- Expression templates are the core technique behind Eigen, Blaze, xtensor.
- **Dangling reference pitfall:** `auto expr = a + b;` - if `a` or `b` go out of scope, `expr` holds dangling references.
- C++20 concepts can constrain expression template operators for better error messages.
- Consider `std::valarray` (has expression-template-like optimizations in some implementations) before rolling your own.
