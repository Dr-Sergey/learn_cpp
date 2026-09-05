# Understand Partial and Full Template Specialization

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/partial_specialization>  

---

## Topic Overview

### What Is Template Specialization

Template specialization lets you provide **custom implementations** for specific type combinations. Instead of the general template, the compiler picks a more specific version when the types match exactly.

| Type | Syntax | Usage |
| --- | --- | --- |
| **Primary template** | `template<typename T, typename U> class X {}` | General case |
| **Partial specialization** | `template<typename T> class X<T, T> {}` | When some params are fixed/constrained |
| **Full (explicit) specialization** | `template<> class X<int, int> {}` | Exact type combo |

### Selection Priority

The compiler always picks the most specific match. Full specialization is more specific than partial, which is more specific than the primary:

```cpp
Full specialization  >  Partial specialization  >  Primary template
(most specific)                                    (most general)
```

### Key Rules

- **Class templates:** support both partial and full specialization
- **Function templates:** support **only full** specialization (use overloading instead of partial)
- Specializations must be declared **after** the primary template
- Specializations in namespace `std` are allowed for user-defined types (e.g., `std::hash`)

---

## Self-Assessment

### Q1: Write a class template `MyPair<T,U>` and specialize it for `MyPair<T,T>` (partial) and `MyPair<int,int>` (full)

Here you can see all three levels in action. Pay attention to which version is selected for each variable in `main` - the priority rule is "most specific wins":

```cpp
#include <iostream>
#include <string>

// Primary template: general case
template <typename T, typename U>
class MyPair {
    T first_;
    U second_;
public:
    MyPair(T a, U b) : first_(std::move(a)), second_(std::move(b)) {}

    void describe() const {
        std::cout << "Primary<T,U>: (" << first_ << ", " << second_ << ")\n";
    }
};

// Partial specialization: both types are the same
template <typename T>
class MyPair<T, T> {
    T first_;
    T second_;
public:
    MyPair(T a, T b) : first_(std::move(a)), second_(std::move(b)) {}

    void describe() const {
        std::cout << "Partial<T,T>: (" << first_ << ", " << second_ << ")\n";
    }

    // Bonus: same-type pairs can compute distance
    T diff() const { return first_ - second_; }
};

// Full specialization: exactly int, int
template <>
class MyPair<int, int> {
    int first_;
    int second_;
public:
    MyPair(int a, int b) : first_(a), second_(b) {}

    void describe() const {
        std::cout << "Full<int,int>: (" << first_ << ", " << second_
                  << ") sum=" << (first_ + second_) << "\n";
    }

    int sum() const { return first_ + second_; }
};

int main() {
    MyPair<std::string, int> p1("hello", 5);
    p1.describe();  // Primary<T,U>: (hello, 5)

    MyPair<double, double> p2(3.14, 2.71);
    p2.describe();  // Partial<T,T>: (3.14, 2.71)
    std::cout << "  diff = " << p2.diff() << "\n";

    MyPair<int, int> p3(10, 20);
    p3.describe();  // Full<int,int>: (10, 20) sum=30
    std::cout << "  sum = " << p3.sum() << "\n";

    // Selection priority:
    // MyPair<string, int> -> Primary (different types)
    // MyPair<double, double> -> Partial (same types, not int,int)
    // MyPair<int, int> -> Full (exact match beats partial)

    return 0;
}
```

Notice that `MyPair<int, int>` matches both the partial specialization `<T,T>` and the full specialization `<int,int>`. The full specialization wins because it is more specific - that is why `p3` reports "Full<int,int>" and not "Partial<T,T>".

**Expected output:**

```text
Primary<T,U>: (hello, 5)
Partial<T,T>: (3.14, 2.71)
  diff = 0.43
Full<int,int>: (10, 20) sum=30
  sum = 30
```

### Q2: Explain why function templates cannot be partially specialized and what to use instead

This is one of those C++ rules that trips people up. The standard simply does not define partial specialization for function templates - only class templates (and variable templates) support it. The good news is that function overloading and `if constexpr` cover all the same ground, often more cleanly:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <type_traits>

// Function templates CANNOT be partially specialized!
// This is ILLEGAL:
//   template <typename T>
//   void process<std::vector<T>>(std::vector<T> v) { ... }  // ERROR

// Reason: The C++ standard simply does not define partial specialization
// for function templates. Only class/variable templates support it.

// FULL specialization of function templates IS allowed (but often unwise):
template <typename T>
void greet(T val) {
    std::cout << "General: " << val << "\n";
}

// Full specialization for int:
template <>
void greet<int>(int val) {
    std::cout << "Int specialization: " << val << "\n";
}

// Problem: function overloads are preferred over specializations!
// An overload for int would be chosen over the specialization above.

// === Better alternatives ===

// Alternative 1: Function OVERLOADING (preferred)
void process(int val) {
    std::cout << "overload(int): " << val << "\n";
}
void process(const std::string& val) {
    std::cout << "overload(string): " << val << "\n";
}
template <typename T>
void process(const std::vector<T>& val) {
    std::cout << "overload(vector<T>): size=" << val.size() << "\n";
}

// Alternative 2: if constexpr (C++17)
template <typename T>
void dispatch(const T& val) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "if-constexpr(integral): " << val << "\n";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "if-constexpr(float): " << val << "\n";
    } else {
        std::cout << "if-constexpr(other): " << val << "\n";
    }
}

// Alternative 3: Tag dispatch with a helper class (can be partially specialized)
template <typename T>
struct ProcessHelper {
    static void exec(const T& v) { std::cout << "helper general: " << v << "\n"; }
};

template <typename T>
struct ProcessHelper<std::vector<T>> {  // Partial specialization - class, so OK
    static void exec(const std::vector<T>& v) {
        std::cout << "helper vector<T>: size=" << v.size() << "\n";
    }
};

template <typename T>
void process_via_helper(const T& v) {
    ProcessHelper<T>::exec(v);
}

int main() {
    std::cout << "=== Function overloading (preferred) ===\n";
    process(42);
    process(std::string("hello"));
    process(std::vector<int>{1,2,3});

    std::cout << "\n=== if constexpr ===\n";
    dispatch(42);
    dispatch(3.14);
    dispatch("hello");

    std::cout << "\n=== Tag dispatch via helper class ===\n";
    process_via_helper(42);
    process_via_helper(std::vector<double>{1.1, 2.2});

    return 0;
}
```

The helper class trick (Alternative 3) is the most flexible - it lets you partially specialize for patterns like `std::vector<T>` by delegating to a class template, which does support partial specialization.

### Q3: Show how specialization of `std::hash` for a user type enables use in unordered containers

Specializing `std::hash` is the canonical example of adding a full specialization in namespace `std`. The standard explicitly permits this for user-defined types, and it is the standard way to make your type usable as an unordered container key:

```cpp
#include <iostream>
#include <functional>
#include <unordered_set>
#include <unordered_map>
#include <string>

struct Employee {
    int id;
    std::string name;

    bool operator==(const Employee& other) const {
        return id == other.id;  // Equality based on ID
    }
};

// Specialize std::hash in namespace std (this is explicitly allowed for user types)
template <>
struct std::hash<Employee> {
    std::size_t operator()(const Employee& e) const noexcept {
        // Combine hash of id and name
        std::size_t h1 = std::hash<int>{}(e.id);
        std::size_t h2 = std::hash<std::string>{}(e.name);
        return h1 ^ (h2 << 1);  // Simple combining (production: use boost::hash_combine)
    }
};

// Now Employee can be used in unordered containers!

struct Point {
    int x, y;

    bool operator==(const Point&) const = default;
};

// Alternative: provide hash as a functor (no std namespace specialization)
struct PointHash {
    std::size_t operator()(const Point& p) const noexcept {
        auto h1 = std::hash<int>{}(p.x);
        auto h2 = std::hash<int>{}(p.y);
        return h1 ^ (h2 << 1);
    }
};

int main() {
    // std::hash specialization -> works directly with unordered containers
    std::unordered_set<Employee> team;
    team.insert({1, "Alice"});
    team.insert({2, "Bob"});
    team.insert({1, "Alice"});  // Duplicate - not inserted

    std::cout << "Team size: " << team.size() << "\n";  // 2
    for (const auto& e : team) {
        std::cout << "  " << e.id << ": " << e.name << "\n";
    }

    // unordered_map with Employee keys
    std::unordered_map<Employee, double> salaries;
    salaries[{1, "Alice"}] = 95000;
    salaries[{2, "Bob"}]   = 88000;
    std::cout << "\nAlice salary: " << salaries[{1, "Alice"}] << "\n";  // 95000

    // Custom hash functor (alternative to std specialization)
    std::unordered_set<Point, PointHash> points;
    points.insert({1, 2});
    points.insert({3, 4});
    points.insert({1, 2});  // Duplicate
    std::cout << "\nPoints: " << points.size() << "\n";  // 2

    return 0;
}
```

If you prefer not to touch namespace `std`, the functor approach (`PointHash` above) is equally valid - you just pass it as the second template argument to the container. Both approaches work well; the choice is mostly a matter of style.

---

## Notes

- **Full specialization** (`template<>`) provides exact implementation for specific types.
- **Partial specialization** (`template<typename T> class X<T,T>`) matches a pattern of types.
- Selection: full > partial > primary (most specific wins).
- Function templates **cannot** be partially specialized - use overloading, `if constexpr`, or helper class patterns.
- `std::hash<UserType>` specialization is the standard way to enable unordered container support.
- Full specializations do not participate in overload resolution - overloads are preferred.
