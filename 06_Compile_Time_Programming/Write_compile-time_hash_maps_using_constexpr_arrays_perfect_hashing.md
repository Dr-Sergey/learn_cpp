# Write Compile-Time Hash Maps Using `constexpr` Arrays (Perfect Hashing)

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/consteval>  

---

## Topic Overview

### Compile-Time Hash Maps

A compile-time hash map stores its data in a `constexpr` array that is computed entirely during compilation. The table lives in `.rodata` alongside other constants, so there is no heap allocation, no startup initialization, and no synchronization overhead. Lookups hit a cache-friendly contiguous array rather than the pointer-chasing structure of `std::unordered_map`.

The catch is that `std::unordered_map` cannot be `constexpr` - it uses heap allocation. So if you want a hash map for a fixed set of keys known at compile time, you have to build the table yourself. That sounds intimidating, but the code is actually quite short once you understand the pattern.

### Approaches

| Approach | Lookup | Build Time | Memory |
| --- | --- | --- | --- |
| Sorted `constexpr` array + binary search | O(log n) | Compile-time sort | Compact |
| Perfect hash table | O(1) | Compile-time hash + probe | May have gaps |
| Linear scan `constexpr` array | O(n) | None | Compact |
| `std::unordered_map` (runtime) | O(1) amortized | Runtime | Heap |

### Perfect Hashing

A **perfect hash function** maps each key to a unique index with no collisions. For a small, known set of keys, you can find such a function at compile time. In practice, rather than hunting for a true perfect hash, the simplest approach is to use a good hash function and resolve the rare collisions with linear probing - that is what the examples below do. The idea is simple: if the slot you hash into is already occupied, just try the next slot, and keep going until you find an empty one. Use the same probing sequence during lookup.

```cpp
consteval size_t perfect_hash(std::string_view key) {
    // Custom function designed for the specific key set
    return (key[0] * 31 + key.size()) % TABLE_SIZE;
}
```

---

## Self-Assessment

### Q1: Implement a `consteval` perfect hash table that maps string literals to enum values

The build function iterates over all known keys, hashes each one, and places it in the array. If a slot is already occupied (a collision), it probes forward using linear probing until it finds an empty slot. The same probing sequence is then used during lookup. If the table size is too small to hold all keys without infinite looping, the `consteval` function throws - which becomes a compile error.

```cpp
#include <iostream>
#include <string_view>
#include <array>
#include <optional>
#include <cstddef>

// === Color enum ===
enum class Color : int {
    Red = 0, Green, Blue, Yellow, White, Black, Cyan, Magenta,
    COUNT
};

constexpr const char* color_names[] = {
    "red", "green", "blue", "yellow", "white", "black", "cyan", "magenta"
};

// === Simple hash function ===
constexpr std::size_t hash_str(std::string_view sv) {
    std::size_t h = 0;
    for (char c : sv)
        h = h * 31 + static_cast<std::size_t>(c);
    return h;
}

// === Find perfect hash table size and build it ===
constexpr std::size_t TABLE_SIZE = 16;  // power of 2, larger than key count

struct Entry {
    std::string_view key{};
    Color value{};
    bool occupied = false;
};

consteval auto build_color_table() {
    std::array<Entry, TABLE_SIZE> table{};

    for (int i = 0; i < static_cast<int>(Color::COUNT); ++i) {
        std::string_view name = color_names[i];
        std::size_t idx = hash_str(name) % TABLE_SIZE;

        // Linear probing for collision resolution
        std::size_t attempts = 0;
        while (table[idx].occupied) {
            idx = (idx + 1) % TABLE_SIZE;
            if (++attempts >= TABLE_SIZE)
                throw "Table too small - increase TABLE_SIZE";
        }

        table[idx] = {name, static_cast<Color>(i), true};
    }

    return table;
}

constexpr auto color_table = build_color_table();

// === Lookup function ===
constexpr std::optional<Color> lookup_color(std::string_view name) {
    std::size_t idx = hash_str(name) % TABLE_SIZE;
    std::size_t attempts = 0;

    while (color_table[idx].occupied) {
        if (color_table[idx].key == name)
            return color_table[idx].value;
        idx = (idx + 1) % TABLE_SIZE;
        if (++attempts >= TABLE_SIZE) break;
    }
    return std::nullopt;
}

// Compile-time verification
static_assert(lookup_color("red") == Color::Red);
static_assert(lookup_color("blue") == Color::Blue);
static_assert(lookup_color("magenta") == Color::Magenta);
static_assert(!lookup_color("orange").has_value());

int main() {
    std::cout << "=== Compile-Time Hash Map: Color Lookup ===\n";

    for (const char* name : {"red", "green", "blue", "yellow", "white",
                              "black", "cyan", "magenta", "orange", "purple"}) {
        auto result = lookup_color(name);
        if (result) {
            std::cout << name << " -> Color::" << static_cast<int>(*result) << "\n";
        } else {
            std::cout << name << " -> NOT FOUND\n";
        }
    }

    std::cout << "\nTable occupancy: " << static_cast<int>(Color::COUNT)
              << "/" << TABLE_SIZE << " ("
              << (100.0 * static_cast<int>(Color::COUNT) / TABLE_SIZE) << "%)\n";

    return 0;
}
```

The `static_assert` lines prove that `lookup_color` is genuinely `constexpr` - the compiler can resolve these lookups at compile time. At runtime you are just reading from a pre-built table in read-only memory.

**Expected output:**

```text
=== Compile-Time Hash Map: Color Lookup ===
red -> Color::0
green -> Color::1
blue -> Color::2
yellow -> Color::3
white -> Color::4
black -> Color::5
cyan -> Color::6
magenta -> Color::7
orange -> NOT FOUND
purple -> NOT FOUND

Table occupancy: 8/16 (50%)
```

### Q2: Show that runtime dispatch via a compile-time hash table is faster than `std::unordered_map`

The performance advantage of the compile-time table comes from several sources at once: no heap allocation, no `std::string` construction during lookup (we compare against `std::string_view` directly), and a contiguous array that is small enough to stay in L1 cache. The `std::string(q)` construction inside the `unordered_map` benchmark is not a cheat - that is the real cost you pay every time you look up a string in an `unordered_map<std::string, ...>`.

```cpp
#include <iostream>
#include <string_view>
#include <array>
#include <optional>
#include <unordered_map>
#include <chrono>
#include <string>

// === Compile-time hash table (from Q1) ===
enum class Color : int {
    Red = 0, Green, Blue, Yellow, White, Black, Cyan, Magenta, COUNT
};
constexpr const char* color_names_ct[] = {
    "red", "green", "blue", "yellow", "white", "black", "cyan", "magenta"
};

constexpr std::size_t hash_str(std::string_view sv) {
    std::size_t h = 0;
    for (char c : sv) h = h * 31 + static_cast<std::size_t>(c);
    return h;
}

constexpr std::size_t CT_SIZE = 16;
struct Entry { std::string_view key; Color value; bool occupied = false; };

consteval auto build_table() {
    std::array<Entry, CT_SIZE> t{};
    for (int i = 0; i < static_cast<int>(Color::COUNT); ++i) {
        std::string_view name = color_names_ct[i];
        std::size_t idx = hash_str(name) % CT_SIZE;
        while (t[idx].occupied) idx = (idx + 1) % CT_SIZE;
        t[idx] = {name, static_cast<Color>(i), true};
    }
    return t;
}
constexpr auto ct_table = build_table();

std::optional<Color> ct_lookup(std::string_view name) {
    auto idx = hash_str(name) % CT_SIZE;
    for (std::size_t i = 0; i < CT_SIZE; ++i) {
        if (!ct_table[idx].occupied) return std::nullopt;
        if (ct_table[idx].key == name) return ct_table[idx].value;
        idx = (idx + 1) % CT_SIZE;
    }
    return std::nullopt;
}

int main() {
    // === Build runtime unordered_map ===
    std::unordered_map<std::string, Color> rt_map;
    for (int i = 0; i < static_cast<int>(Color::COUNT); ++i)
        rt_map[color_names_ct[i]] = static_cast<Color>(i);

    const std::string_view queries[] = {
        "red", "blue", "green", "yellow", "white", "black", "cyan", "magenta"
    };
    constexpr int ITERATIONS = 1000000;

    // === Benchmark compile-time table ===
    auto start1 = std::chrono::high_resolution_clock::now();
    volatile int sink1 = 0;
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        for (auto q : queries) {
            auto r = ct_lookup(q);
            if (r) sink1 = static_cast<int>(*r);
        }
    }
    auto end1 = std::chrono::high_resolution_clock::now();
    auto us1 = std::chrono::duration_cast<std::chrono::microseconds>(end1 - start1).count();

    // === Benchmark unordered_map ===
    auto start2 = std::chrono::high_resolution_clock::now();
    volatile int sink2 = 0;
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        for (auto q : queries) {
            auto it = rt_map.find(std::string(q));
            if (it != rt_map.end()) sink2 = static_cast<int>(it->second);
        }
    }
    auto end2 = std::chrono::high_resolution_clock::now();
    auto us2 = std::chrono::duration_cast<std::chrono::microseconds>(end2 - start2).count();

    std::cout << "=== Benchmark: 8 queries x " << ITERATIONS << " iterations ===\n";
    std::cout << "Compile-time hash table: " << us1 << " us\n";
    std::cout << "std::unordered_map:      " << us2 << " us\n";
    std::cout << "Speedup: " << static_cast<double>(us2) / us1 << "x\n";

    std::cout << "\n=== Why compile-time table is faster ===\n";
    std::cout << "- No heap allocation (data in .rodata)\n";
    std::cout << "- Cache-friendly contiguous array\n";
    std::cout << "- No std::string construction for lookup\n";
    std::cout << "- Constexpr hash function may be inlined\n";

    return 0;
}
```

The compile-time table avoids `std::string` construction because its keys are `std::string_view` and the comparison is a simple memory compare. The `unordered_map` has to hash, allocate, and compare a full `std::string` on every lookup - you can see the cost in the timing.

**Expected output (timing varies):**

```text
=== Benchmark: 8 queries x 1000000 iterations ===
Compile-time hash table: 42000 us
std::unordered_map:      185000 us
Speedup: 4.4x

=== Why compile-time table is faster ===
- No heap allocation (data in .rodata)
- Cache-friendly contiguous array
- No std::string construction for lookup
- Constexpr hash function may be inlined
```

### Q3: Compare with sorted `constexpr` array + `std::lower_bound`

For small key sets (fewer than about 20 keys), a sorted array with `std::lower_bound` is often the simplest approach. It requires no hash function design, the binary is slightly smaller because there are no empty slots, and `std::lower_bound` on a 7-element sorted array is extremely fast. For larger key sets the asymptotic advantage of the hash table matters more.

Notice that `build_sorted_methods` uses `std::sort` inside a `consteval` function. This is valid in C++20 because `std::sort` is now `constexpr`. The compiler sorts your HTTP method table at compile time so that runtime lookups can use binary search.

```cpp
#include <iostream>
#include <array>
#include <algorithm>
#include <string_view>
#include <optional>
#include <cstddef>
#include <string>

// === Sorted constexpr array approach ===

enum class HttpMethod : int { DELETE_, GET, HEAD, OPTIONS, PATCH, POST, PUT, COUNT };

struct MethodEntry {
    std::string_view name;
    HttpMethod method;

    constexpr bool operator<(const MethodEntry& other) const {
        return name < other.name;
    }
};

consteval auto build_sorted_methods() {
    std::array<MethodEntry, static_cast<int>(HttpMethod::COUNT)> arr = {{
        {"DELETE",  HttpMethod::DELETE_},
        {"GET",     HttpMethod::GET},
        {"HEAD",    HttpMethod::HEAD},
        {"OPTIONS", HttpMethod::OPTIONS},
        {"PATCH",   HttpMethod::PATCH},
        {"POST",    HttpMethod::POST},
        {"PUT",     HttpMethod::PUT},
    }};
    std::sort(arr.begin(), arr.end());
    return arr;
}

constexpr auto sorted_methods = build_sorted_methods();

// O(log n) lookup via binary search
constexpr std::optional<HttpMethod> find_method(std::string_view name) {
    auto it = std::lower_bound(sorted_methods.begin(), sorted_methods.end(),
                                MethodEntry{name, {}});
    if (it != sorted_methods.end() && it->name == name)
        return it->method;
    return std::nullopt;
}

// Compile-time verification
static_assert(find_method("GET") == HttpMethod::GET);
static_assert(find_method("POST") == HttpMethod::POST);
static_assert(find_method("DELETE") == HttpMethod::DELETE_);
static_assert(!find_method("CONNECT").has_value());

// === Hash table approach for comparison ===
constexpr std::size_t hash_sv(std::string_view sv) {
    std::size_t h = 0;
    for (char c : sv) h = h * 31 + static_cast<std::size_t>(c);
    return h;
}

constexpr std::size_t HT_SIZE = 16;
struct HTEntry { std::string_view key; HttpMethod value; bool occupied = false; };

consteval auto build_hash_methods() {
    std::array<HTEntry, HT_SIZE> t{};
    std::array<MethodEntry, 7> entries = {{
        {"DELETE", HttpMethod::DELETE_}, {"GET", HttpMethod::GET},
        {"HEAD", HttpMethod::HEAD}, {"OPTIONS", HttpMethod::OPTIONS},
        {"PATCH", HttpMethod::PATCH}, {"POST", HttpMethod::POST},
        {"PUT", HttpMethod::PUT},
    }};
    for (auto& e : entries) {
        auto idx = hash_sv(e.name) % HT_SIZE;
        while (t[idx].occupied) idx = (idx + 1) % HT_SIZE;
        t[idx] = {e.name, e.method, true};
    }
    return t;
}

constexpr auto ht_methods = build_hash_methods();

int main() {
    std::cout << "=== Approach Comparison ===\n\n";

    std::cout << "+---------------------+-----------------+-------------------+\n";
    std::cout << "| Aspect              | Sorted + BSearch| Hash Table        |\n";
    std::cout << "+---------------------+-----------------+-------------------+\n";
    std::cout << "| Lookup complexity   | O(log n)        | O(1) average      |\n";
    std::cout << "| Memory overhead     | None (compact)  | Gaps in table     |\n";
    std::cout << "| Build complexity    | Sort (simple)   | Hash fn design    |\n";
    std::cout << "| Cache behavior      | Excellent       | Good              |\n";
    std::cout << "| Best for N <  20    | Simple, fast    | Overkill          |\n";
    std::cout << "| Best for N >= 100   | Still fine      | Better asymptotic |\n";
    std::cout << "+---------------------+-----------------+-------------------+\n";

    std::cout << "\n=== Binary Search Lookups ===\n";
    for (auto q : {"GET", "POST", "DELETE", "HEAD", "CONNECT", "TRACE"}) {
        auto r = find_method(q);
        std::cout << q << " -> "
                  << (r ? std::to_string(static_cast<int>(*r)) : "NOT FOUND") << "\n";
    }

    std::cout << "\n=== Recommendation ===\n";
    std::cout << "Small key sets (< 20): sorted array + lower_bound is simplest.\n";
    std::cout << "Large key sets (100+): compute a perfect hash at compile time.\n";
    std::cout << "Both are constexpr - zero runtime initialization cost.\n";

    return 0;
}
```

Both approaches are `constexpr` and carry zero runtime initialization cost. The choice between them comes down to the size of your key set and how much you care about worst-case lookup time.

**Expected output:**

```text
=== Approach Comparison ===

+---------------------+-----------------+-------------------+
| Aspect              | Sorted + BSearch| Hash Table        |
+---------------------+-----------------+-------------------+
| Lookup complexity   | O(log n)        | O(1) average      |
| Memory overhead     | None (compact)  | Gaps in table     |
| Build complexity    | Sort (simple)   | Hash fn design    |
| Cache behavior      | Excellent       | Good              |
| Best for N <  20    | Simple, fast    | Overkill          |
| Best for N >= 100   | Still fine      | Better asymptotic |
+---------------------+-----------------+-------------------+

=== Binary Search Lookups ===
GET -> 1
POST -> 5
DELETE -> 0
HEAD -> 2
CONNECT -> NOT FOUND
TRACE -> NOT FOUND

=== Recommendation ===
Small key sets (< 20): sorted array + lower_bound is simplest.
Large key sets (100+): compute a perfect hash at compile time.
Both are constexpr - zero runtime initialization cost.
```

---

## Notes

- **Perfect hashing** means no collisions for the known key set - O(1) guaranteed lookup.
- For small key sets (< 20), a sorted array with `std::lower_bound` is simpler and equally fast.
- `consteval` ensures the table is built at compile time - any error is a compile error.
- Linear probing is simplest for compile-time hash tables; FNV-1a and DJB2 are good hash functions.
- `std::unordered_map` cannot be `constexpr` - compile-time hash tables are the standard alternative.
- The hash table lives in `.rodata` - no heap allocation, no startup cost, cache-friendly.
