# Write const-correct code throughout

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.org/wiki/faq/const-correctness>  

---

## Topic Overview

**Const correctness** means marking everything `const` that doesn't need to change. It's not just style - it prevents a whole class of accidental mutation bugs, signals intent to the reader, and allows the compiler to make better optimization decisions.

The practical rule is simple: start with `const` everywhere, then remove it only where you actually need to mutate.

```cpp
void print(const std::string& s);     // won't modify s
int Widget::size() const;              // won't modify *this
const int MAX = 100;                   // immutable value
```

---

## Self-Assessment

### Q1: Audit a class and add `const` wherever possible

Here's a fully const-correct class. Notice that every method that doesn't mutate state is marked `const`, the getter returns a `const` reference to avoid unnecessary copies, and the range-`for` loop uses `const int`. The non-`const` method `add_grade` is the single exception - it's the only one that actually changes anything.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <numeric>

class StudentRecord {
    std::string name_;
    std::vector<int> grades_;
public:
    StudentRecord(std::string name, std::vector<int> grades)
        : name_(std::move(name)), grades_(std::move(grades)) {}

    // const: doesn't modify state
    const std::string& name() const { return name_; }

    // const: doesn't modify state
    size_t grade_count() const { return grades_.size(); }

    // const: doesn't modify state
    double average() const {
        if (grades_.empty()) return 0.0;
        return static_cast<double>(
            std::accumulate(grades_.begin(), grades_.end(), 0)
        ) / grades_.size();
    }

    // const: read-only access to grades
    const std::vector<int>& grades() const { return grades_; }

    // NOT const: modifies state
    void add_grade(int g) { grades_.push_back(g); }

    // const: only reads data
    void print_report() const {
        std::cout << "Student: " << name_ << '\n';
        std::cout << "Grades: ";
        for (const int g : grades_) std::cout << g << ' ';
        std::cout << "\nAverage: " << average() << '\n';
    }
};

int main() {
    StudentRecord alice("Alice", {90, 85, 92});
    alice.print_report();

    // const object: only const methods callable
    const StudentRecord& ref = alice;
    std::cout << "Name: " << ref.name() << '\n';
    std::cout << "Avg: " << ref.average() << '\n';
    // ref.add_grade(100);  // ERROR: can't call non-const on const ref
}
// Expected output:
// Student: Alice
// Grades: 90 85 92
// Average: 89
// Name: Alice
// Avg: 89
```

The commented-out `ref.add_grade(100)` line is key: because `ref` is a `const` reference, the compiler simply won't let you call a mutating method through it. That's the safety guarantee you get for free.

### Q2: Const correctness enables compiler optimizations

Beyond safety, `const` gives the compiler information it can exploit. A `const` global can go into read-only memory. A `const` parameter tells the compiler the data won't be aliased away from under it. These aren't micro-optimizations - they affect whole loops.

```cpp
#include <iostream>

// const global: compiler can place in .rodata (read-only memory)
const int LOOKUP_TABLE[] = {0, 1, 1, 2, 3, 5, 8, 13, 21, 34};
constexpr int TABLE_SIZE = sizeof(LOOKUP_TABLE) / sizeof(int);

// Without const: compiler must assume the data may change
int mutable_data[] = {0, 1, 1, 2, 3, 5, 8, 13, 21, 34};

// const reference parameter: compiler knows it won't be modified
int sum_values(const int* data, int n) {
    int total = 0;
    for (int i = 0; i < n; ++i)
        total += data[i];
    return total;
}

// Non-const: compiler must reload from memory after any function call
int sum_values_mut(int* data, int n) {
    int total = 0;
    for (int i = 0; i < n; ++i)
        total += data[i];  // must reload each time (aliasing!)
    return total;
}

int main() {
    // const enables:
    // 1. Read-only memory placement (crash on accidental write)
    // 2. Constant folding (compile-time evaluation)
    // 3. Better register allocation (no need to reload)
    // 4. Dead store elimination

    std::cout << "Sum: " << sum_values(LOOKUP_TABLE, TABLE_SIZE) << '\n';

    constexpr int x = 42;
    // Compiler replaces x with 42 everywhere - no memory access!
    std::cout << "x = " << x << '\n';
}
// Expected output:
// Sum: 88
// x = 42
```

The aliasing comment in `sum_values_mut` is the subtle one: without `const`, the compiler has to assume some other pointer might point to the same memory, so it can't keep the array values in registers across iterations.

### Q3: Logical const vs physical const and `mutable`

Here's where things get more nuanced. A method that modifies a cache or locks a mutex is not really changing the *observable* state of the object - it's changing internal bookkeeping. You want to declare that method `const` so callers can use it on a `const` object, but the compiler won't let you modify members inside a `const` method unless those members are marked `mutable`.

The rule of thumb: `mutable` is appropriate for members that support the const interface without affecting observable state. The three classic uses are mutexes, caches, and debug counters.

```cpp
#include <iostream>
#include <mutex>
#include <unordered_map>
#include <string>

class DataStore {
    std::unordered_map<std::string, int> data_;

    // mutable: these can change even in const methods
    mutable std::mutex mutex_;                           // logical const
    mutable std::unordered_map<std::string, int> cache_; // logical const
    mutable int access_count_ = 0;                       // logical const

public:
    DataStore() : data_{{"x", 1}, {"y", 2}, {"z", 3}} {}

    // Physically modifies mutex_, cache_, access_count_
    // But LOGICALLY const: observable state doesn't change
    int get(const std::string& key) const {
        std::lock_guard lock(mutex_);  // modifies mutex_ (mutable)
        ++access_count_;               // modifies counter (mutable)

        // Check cache first
        if (auto it = cache_.find(key); it != cache_.end())
            return it->second;

        // Compute and cache
        if (auto it = data_.find(key); it != data_.end()) {
            cache_[key] = it->second;  // modifies cache_ (mutable)
            return it->second;
        }
        return -1;
    }

    int access_count() const { return access_count_; }
};

int main() {
    const DataStore store;  // const object!

    // These all work because get() is const
    std::cout << "x = " << store.get("x") << '\n';
    std::cout << "y = " << store.get("y") << '\n';
    std::cout << "x = " << store.get("x") << '\n';  // cached
    std::cout << "Accesses: " << store.access_count() << '\n';
}
// Expected output:
// x = 1
// y = 2
// x = 1
// Accesses: 3
```

**Logical vs Physical const:**

| Concept | Physical const | Logical const |
| --- | --- | --- |
| What | No bits change | Observable state doesn't change |
| With mutable | Not possible | Members can change |
| Use cases | Simple data | Caches, mutexes, counters |
| The `mutable` keyword | Not needed | Allows physical mutation in const methods |

---

## Notes

- Start with everything `const`; remove only when mutation is needed.
- `const` on parameters prevents accidental modification.
- `const` member functions can be called on const objects and const references.
- Use `mutable` only for thread-safety primitives (mutex), caches, and debug counters.
- `const` is shallow for pointers: `const int*` vs `int* const` are different.
