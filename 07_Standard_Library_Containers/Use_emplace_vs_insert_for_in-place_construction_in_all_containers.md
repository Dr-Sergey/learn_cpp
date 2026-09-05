# Use emplace vs insert for in-place construction in all containers

**Category:** Standard Library Containers  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/container/map/emplace>  

---

## Topic Overview

`emplace` constructs elements **in-place** directly inside the container, while `insert` requires an already-constructed object that gets **copied or moved** into the container. This distinction matters for types that are expensive to construct, non-copyable, or non-movable.

The reason this trips people up is that the performance difference is real for some types and completely nonexistent for others. Understanding when emplace actually helps - and when it can surprise you - is more useful than applying it blindly everywhere.

### How They Differ

Here is the fundamental difference in what happens at the assembly level:

```cpp
insert(value):

  1. Construct element outside container
  2. Copy/move it into container storage
  3. Destroy the temporary (if copy)

emplace(args...):

  1. Forward args to container's internal storage
  2. Construct element directly in-place
  3. No copy, no move, no temporary
```

With `emplace`, the constructor arguments travel through a forwarding chain directly to the memory location inside the container. The object is born there. With `insert`, the object is born somewhere else and then relocated.

### Comparison Table

| Operation | Constructs temporary? | Copies/moves? | Works with non-movable types? |
| --- | --- | --- | --- |
| `v.push_back(T(a,b))` | Yes | Move | No (if non-movable) |
| `v.emplace_back(a,b)` | No | No | Yes |
| `m.insert({k,v})` | Yes (pair) | Move pair | No |
| `m.emplace(k,v)` | Maybe* | Maybe* | Yes (with piecewise) |
| `m.try_emplace(k,v_args...)` | No (value only if new) | No | Yes |

*`map::emplace` may construct a temporary pair even if key exists - `try_emplace` (C++17) avoids this.

### Emplace Variants Across Containers

| Container | emplace method | hint variant | Special |
| --- | --- | --- | --- |
| `vector` | `emplace_back`, `emplace` | - | `emplace` at position |
| `deque` | `emplace_back`, `emplace_front`, `emplace` | - | Both ends |
| `list` | `emplace_back`, `emplace_front`, `emplace` | - | Both ends + position |
| `set` | `emplace` | `emplace_hint` | - |
| `map` | `emplace` | `emplace_hint` | `try_emplace` (C++17) |
| `unordered_map` | `emplace` | `emplace_hint` | `try_emplace` (C++17) |

### Core Example

The `Heavy` type here prints every construction and move so you can see exactly what happens with each approach. The difference between `push_back` and `emplace_back` is one extra move:

```cpp
#include <iostream>
#include <vector>
#include <string>

struct Heavy {
    std::string name;
    int value;

    Heavy(std::string n, int v) : name(std::move(n)), value(v) {
        std::cout << "  Constructed: " << name << "\n";
    }
    Heavy(const Heavy& o) : name(o.name), value(o.value) {
        std::cout << "  Copied: " << name << "\n";
    }
    Heavy(Heavy&& o) noexcept : name(std::move(o.name)), value(o.value) {
        std::cout << "  Moved: " << name << "\n";
    }
};

int main() {
    std::vector<Heavy> v;
    v.reserve(4);  // Avoid reallocation noise

    std::cout << "push_back(Heavy{...}):\n";
    v.push_back(Heavy{"Alpha", 1});
    // Output: Constructed: Alpha
    //         Moved: Alpha (moved into vector)

    std::cout << "\nemplace_back(args...):\n";
    v.emplace_back("Beta", 2);
    // Output: Constructed: Beta (directly in vector, no move!)

    return 0;
}
```

`push_back` creates `Heavy{"Alpha",1}` first, then moves it into the vector. `emplace_back` skips the temporary entirely and constructs `Heavy` directly in the vector's storage. For an expensive-to-move type that savings is significant; for an `int` it is zero.

### Important Notes

- `emplace_back` is **not always faster** - for simple types like `int`, `double`, or small structs, the compiler optimizes both paths identically.
- `emplace` can cause subtle bugs: `v.emplace_back(10, 20)` might call an unexpected constructor.
- Prefer `push_back`/`insert` for braced-init-lists: `v.push_back({1, 2})` works, `v.emplace_back({1, 2})` does not (brace elision issue).
- `try_emplace` (C++17) is strictly better than `emplace` for maps when you don't want to construct the value if the key already exists.

---

## Self-Assessment

### Q1: Show that map.emplace(key, value) avoids constructing a pair explicitly

This example traces every construction, copy, and move so you can count exactly how many operations each insertion style incurs. Pay attention to the progression: `insert` -> `emplace` -> `piecewise_construct` -> `try_emplace`:

```cpp
#include <iostream>
#include <map>
#include <string>

struct Tracked {
    std::string data;

    Tracked() : data("default") {
        std::cout << "  Default constructed\n";
    }
    explicit Tracked(const std::string& s) : data(s) {
        std::cout << "  Constructed: " << data << "\n";
    }
    Tracked(const Tracked& o) : data(o.data) {
        std::cout << "  Copy constructed: " << data << "\n";
    }
    Tracked(Tracked&& o) noexcept : data(std::move(o.data)) {
        std::cout << "  Move constructed\n";
    }
    ~Tracked() {
        std::cout << "  Destroyed: " << data << "\n";
    }
};

int main() {
    std::map<int, Tracked> m;

    // === insert: requires constructing a pair first ===
    std::cout << "--- insert(make_pair) ---\n";
    m.insert(std::make_pair(1, Tracked("one")));
    // Output:
    //   Constructed: one
    //   Move constructed     <- moved into pair inside map
    //   Destroyed:           <- temporary destroyed

    std::cout << "\n--- insert({key, value}) ---\n";
    m.insert({2, Tracked("two")});
    // Output:
    //   Constructed: two
    //   Move constructed     <- moved into map's internal pair
    //   Destroyed:           <- temporary destroyed

    // === emplace: constructs pair in-place ===
    std::cout << "\n--- emplace(key, value) ---\n";
    m.emplace(3, Tracked("three"));
    // Output:
    //   Constructed: three
    //   Move constructed     <- still one move (pair construction)
    //   Destroyed:

    // === emplace with piecewise_construct: truly zero copies ===
    std::cout << "\n--- emplace(piecewise_construct) ---\n";
    m.emplace(std::piecewise_construct,
              std::forward_as_tuple(4),
              std::forward_as_tuple("four"));
    // Output:
    //   Constructed: four    <- ONE construction, no copies, no moves!

    // === try_emplace (C++17): best option ===
    std::cout << "\n--- try_emplace(key, args...) ---\n";
    m.try_emplace(5, "five");
    // Output:
    //   Constructed: five    <- ONE construction, no copies, no moves!

    // try_emplace won't construct if key exists:
    std::cout << "\n--- try_emplace(5, ...) again ---\n";
    m.try_emplace(5, "FIVE_DUPLICATE");
    // Output: (nothing! value not constructed because key 5 exists)

    std::cout << "\n--- Final map ---\n";
    for (const auto& [k, v] : m)
        std::cout << "  " << k << " -> " << v.data << "\n";

    std::cout << "\n--- Cleanup ---\n";
    return 0;
}
```

The progression from `insert` to `try_emplace` reduces the construction count from 2 operations (construct + move) down to exactly 1, and `try_emplace` adds the bonus of skipping construction entirely if the key already exists. For the map case, `try_emplace` is almost always the right choice.

**How it works:**

- `insert(make_pair(...))` creates a temporary pair, then moves it into the map - at minimum one construction + one move.
- `emplace(key, value)` forwards arguments to a pair constructor, but the pair still constructs the Tracked object, then potentially moves it.
- `emplace(piecewise_construct, ...)` uses `std::forward_as_tuple` to pass constructor arguments directly to key and value - the Tracked is constructed in-place with zero copies/moves.
- `try_emplace(key, args...)` (C++17) achieves the same in-place construction but additionally skips value construction entirely if the key already exists. This is the preferred pattern.

### Q2: Use set.emplace_hint(it, val) for O(1) amortized insertion when position is known

If you're inserting pre-sorted data into a `set` or `map`, you already know where each element goes. The hint tells the container to start its search there instead of from the root, turning O(log n) into O(1) amortized:

```cpp
#include <iostream>
#include <set>
#include <chrono>

int main() {
    constexpr int N = 1'000'000;

    // === Without hint: O(log n) per insert ===
    {
        std::set<int> s;
        auto start = std::chrono::steady_clock::now();

        for (int i = 0; i < N; ++i)
            s.emplace(i);  // O(log n) - searches from root each time

        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "emplace (no hint): " << ms << " ms\n";
    }

    // === With hint: O(1) amortized when hint is correct ===
    {
        std::set<int> s;
        auto start = std::chrono::steady_clock::now();

        auto hint = s.end();
        for (int i = 0; i < N; ++i)
            hint = s.emplace_hint(hint, i);  // O(1) - hint points just before insert pos

        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "emplace_hint:      " << ms << " ms\n";
    }

    // === Hint rules ===
    std::set<int> s = {10, 20, 30, 40, 50};

    // Good hint: iterator to element just AFTER where new element goes
    // Insert 25 -> should go between 20 and 30
    auto it_30 = s.find(30);
    s.emplace_hint(it_30, 25);  // O(1) - hint is correct

    // Bad hint: still works, but falls back to O(log n)
    auto it_10 = s.find(10);
    s.emplace_hint(it_10, 45);  // O(log n) - hint is wrong, 45 doesn't go near 10

    // end() as hint: optimal when inserting the largest element
    s.emplace_hint(s.end(), 100);  // O(1) - 100 > all existing elements

    // begin() as hint: optimal when inserting the smallest element
    s.emplace_hint(s.begin(), 1);  // O(1) - 1 < all existing elements

    std::cout << "\nSet contents: ";
    for (int x : s) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 10 20 25 30 40 45 50 100

    return 0;
}
// Typical timing:
// emplace (no hint): ~450 ms
// emplace_hint:      ~200 ms  (2x+ faster for sequential data)
```

The key detail here is that a wrong hint doesn't cause incorrect behavior - it just falls back to the normal O(log n) search. So the strategy "use `s.end()` as a running hint when inserting ascending data" is both correct and fast.

**How it works:**

- `emplace_hint(hint, args...)` inserts an element with a "hint" iterator suggesting where the element should go.
- If the hint is **correct** (points to the element just after the insert position), insertion is O(1) amortized instead of O(log n).
- If the hint is **wrong**, the container falls back to a normal O(log n) search - no harm, just no benefit.
- **Best use case:** Inserting pre-sorted data. Keep a running hint (the return value of each `emplace_hint` is the iterator to the inserted element).
- Works for `set`, `multiset`, `map`, `multimap`, and their `unordered_` variants (though hints in unordered containers are less useful).

### Q3: Explain when emplace and insert are equivalent (trivially constructible types)

The honest answer is: for most everyday types, `emplace_back` and `push_back` compile to the same code. Here is the full picture of where the difference matters and where it doesn't - plus the gotchas to watch for:

```cpp
#include <iostream>
#include <vector>
#include <set>
#include <map>
#include <initializer_list>
#include <string>
#include <utility>

int main() {
    // === For scalar/trivial types: identical codegen ===
    {
        std::vector<int> v;
        v.reserve(4);

        // These produce identical machine code:
        v.push_back(42);     // Copy int into vector
        v.emplace_back(42);  // Construct int in-place -> same thing!
        // For int, "construction" IS just a copy of 4 bytes.
        // No temporary to avoid, no constructor to call.
    }

    // === For std::string with string literal: also equivalent ===
    {
        std::vector<std::string> v;
        v.reserve(4);

        // Both call string(const char*) in the vector's storage:
        v.push_back("hello");     // Implicitly converts to string, then moves
        v.emplace_back("hello");  // Constructs string in-place from const char*
        // In practice, compilers optimize both to the same code.
    }

    // === When emplace IS different / better ===
    {
        // 1. Multi-argument construction:
        std::vector<std::pair<int, std::string>> v;
        // v.push_back(1, "one");   // ERROR: push_back takes one arg
        v.emplace_back(1, "one");   // OK: forwards (1, "one") to pair constructor

        // 2. Non-copyable, non-movable types:
        struct Immovable {
            int x;
            Immovable(int x) : x(x) {}
            Immovable(const Immovable&) = delete;
            Immovable(Immovable&&) = delete;
        };
        std::vector<Immovable> vi;
        vi.reserve(2);
        // vi.push_back(Immovable{1});  // ERROR: deleted move ctor
        vi.emplace_back(1);             // OK: constructs in-place, no move

        // 3. Expensive-to-move types:
        // Types with large internal buffers benefit from emplace
        // because zero bytes need to be moved.
    }

    // === When emplace is WORSE (surprises) ===
    {
        std::vector<std::vector<int>> vv;
        vv.reserve(4);

        // push_back is clear about what's being inserted:
        vv.push_back({1, 2, 3});     // initializer_list -> vector<int>

        // emplace_back with initializer_list doesn't compile:
        // vv.emplace_back({1, 2, 3});  // ERROR: can't deduce initializer_list

        // emplace_back with explicit type:
        vv.emplace_back(std::initializer_list<int>{1, 2, 3});  // Verbose but works

        // emplace_back with single size_t: SURPRISE!
        vv.emplace_back(3);  // Creates vector<int>(3) = {0, 0, 0}!
        // Did you mean {3}? push_back({3}) would give a single-element vector.
    }

    // === Summary ===
    std::cout << "When emplace == insert (no benefit):\n";
    std::cout << "  - Scalar types (int, double, pointers)\n";
    std::cout << "  - Types with trivial copy/move\n";
    std::cout << "  - Single-argument construction from same type\n";
    std::cout << "  - Already-constructed objects being moved\n\n";

    std::cout << "When emplace > insert (real benefit):\n";
    std::cout << "  - Multi-argument construction (pair, tuple, custom)\n";
    std::cout << "  - Non-copyable/non-movable types\n";
    std::cout << "  - Expensive-to-construct types\n";
    std::cout << "  - map: try_emplace avoids value creation if key exists\n\n";

    std::cout << "When insert > emplace (prefer insert):\n";
    std::cout << "  - Initializer lists: push_back({...})\n";
    std::cout << "  - Explicit type is clearer for readability\n";
    std::cout << "  - Avoiding accidental implicit conversions\n";

    return 0;
}
```

The `vv.emplace_back(3)` line is the gotcha worth memorizing: you meant to insert the integer 3, but `vector<int>` has a constructor that takes a `size_t` and creates a vector of that many zero-initialized elements. `push_back({3})` does what you intended; `emplace_back(3)` does something else entirely.

**How it works:**

- For **trivially copyable/movable types** (int, double, pointers, small PODs), the compiler generates identical code for `push_back(x)` and `emplace_back(x)`. There's no temporary to eliminate - construction IS copying a few bytes.
- For **`std::string` from `const char*`**, both paths call the same `string(const char*)` constructor in the storage, so they're equivalent.
- `emplace` provides real savings when:
  - Construction takes **multiple arguments** (avoids creating a temporary pair/tuple)
  - The type is **non-copyable/non-movable** (emplace is the only option)
  - The type is **expensive to move** (large buffer, deep copy)
- `emplace` has pitfalls: it can call **unexpected constructors** (like `vector(size_t)` when you meant to insert a value), and it doesn't work with brace-init-lists.

---

## Notes

- **Rule of thumb:** Use `emplace_back` for multi-argument construction and non-movable types. Use `push_back` for clarity with simple values and initializer lists.
- **`try_emplace` (C++17)** is the gold standard for maps: it never constructs the value unless needed, and it clearly separates key from value arguments.
- **`insert_or_assign` (C++17)** is the counterpart: it always overwrites, like `operator[]` but returns whether insertion happened.
- **Performance measurement:** Always benchmark. In many cases, compilers optimize both paths to identical code with `-O2`.
- **Braced-init-list issue:** `emplace_back({1,2,3})` cannot deduce the type. Use `push_back({1,2,3})` or `emplace_back(std::initializer_list<int>{1,2,3})`.
