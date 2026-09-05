# Use std::transform with output iterator adaptors

**Category:** Standard Library Algorithms  
**Standard:** C++98, C++11, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/iterator/back_insert_iterator>  

---

## Topic Overview

`std::transform` writes results through an **output iterator**. The iterator you choose controls where the results land - and the standard library gives you several adaptors for common destinations so you don't have to pre-size containers or write the output loop by hand.

### The Three Insert Iterator Adaptors

| Adaptor | Helper function | Calls on container | Works with |
| --- | --- | --- | --- |
| `std::back_insert_iterator` | `std::back_inserter(c)` | `c.push_back(v)` | `vector`, `deque`, `list`, `string` |
| `std::front_insert_iterator` | `std::front_inserter(c)` | `c.push_front(v)` | `deque`, `list`, `forward_list` |
| `std::insert_iterator` | `std::inserter(c, it)` | `c.insert(it, v)` | All containers with `insert` |

### std::ostream_iterator

```cpp
std::ostream_iterator<T>(os, delimiter)
```

Writes each assigned value to `os` followed by `delimiter`. Useful for printing transformed results directly without an intermediate container - the output goes straight to a stream.

### Core Syntax

Here's all four adaptors in one place, so you can see the behavioral differences:

```cpp
#include <algorithm>
#include <vector>
#include <deque>
#include <set>
#include <iterator>
#include <iostream>
#include <string>

int main() {
    std::vector<int> src{1, 2, 3, 4, 5};

    // --- back_inserter: append to an empty vector ---
    std::vector<int> dst;  // empty!
    std::transform(src.begin(), src.end(),
                   std::back_inserter(dst),
                   [](int x) { return x * x; });
    // dst = {1, 4, 9, 16, 25}

    // --- front_inserter: prepend to a deque (reverse order) ---
    std::deque<int> dq;
    std::transform(src.begin(), src.end(),
                   std::front_inserter(dq),
                   [](int x) { return x * 10; });
    // dq = {50, 40, 30, 20, 10}  (reversed!)

    // --- inserter: insert into a set ---
    std::set<int> s;
    std::transform(src.begin(), src.end(),
                   std::inserter(s, s.end()),
                   [](int x) { return x % 3; });
    // s = {0, 1, 2}  (duplicates removed by set)

    // --- ostream_iterator: print directly ---
    std::transform(src.begin(), src.end(),
                   std::ostream_iterator<int>(std::cout, " "),
                   [](int x) { return x * x; });
    std::cout << "\n";
    // Output: 1 4 9 16 25
}
```

The `front_inserter` reversal is the surprising behavior worth noting: each `push_front` places the new element before all previously inserted ones, so the net result is reversed relative to the source.

---

## Self-Assessment

### Q1: Use std::transform with std::back_inserter to build a new container from a transformation

`back_inserter` is by far the most common adaptor. It lets you write into an initially empty container without knowing the output size upfront - the container grows automatically via `push_back`.

```cpp
#include <algorithm>
#include <vector>
#include <string>
#include <iterator>
#include <iostream>
#include <cctype>

int main() {
    std::vector<std::string> words{"hello", "world", "cpp"};

    // Transform strings to uppercase - build a NEW vector
    std::vector<std::string> upper;
    std::transform(words.begin(), words.end(),
                   std::back_inserter(upper),
                   [](std::string s) {
                       for (char& c : s) c = static_cast<char>(std::toupper(c));
                       return s;
                   });

    for (const auto& w : upper) std::cout << w << " ";
    std::cout << "\n";
    // Output: HELLO WORLD CPP

    // Numeric example - squares into a new vector
    std::vector<int> nums{2, 4, 6, 8};
    std::vector<int> squares;
    std::transform(nums.begin(), nums.end(),
                   std::back_inserter(squares),
                   [](int x) { return x * x; });

    for (int v : squares) std::cout << v << " ";
    std::cout << "\n";
    // Output: 4 16 36 64
}
```

`std::back_inserter(upper)` returns a `back_insert_iterator` that calls `upper.push_back()` for each value assigned to it. The destination container starts **empty** - no need to pre-size. Each call to `push_back` may trigger reallocation; if you know the final size, call `upper.reserve(words.size())` before the transform to avoid repeated reallocations.

---

### Q2: Show std::ostream_iterator used as output of std::transform to print transformed results

`ostream_iterator` eliminates the intermediate container entirely when you only need to print results. The transform output flows straight to the stream character by character.

```cpp
#include <algorithm>
#include <vector>
#include <iterator>
#include <iostream>
#include <sstream>
#include <cmath>

int main() {
    std::vector<double> data{1.0, 4.0, 9.0, 16.0, 25.0};

    // Print square roots directly to cout
    std::cout << "Square roots: ";
    std::transform(data.begin(), data.end(),
                   std::ostream_iterator<double>(std::cout, " "),
                   [](double x) { return std::sqrt(x); });
    std::cout << "\n";
    // Output: Square roots: 1 2 3 4 5

    // Print to a string stream instead
    std::ostringstream oss;
    std::transform(data.begin(), data.end(),
                   std::ostream_iterator<double>(oss, ", "),
                   [](double x) { return x * 2; });
    std::cout << "Doubled: " << oss.str() << "\n";
    // Output: Doubled: 2, 8, 18, 32, 50,

    // Integer to hex string output
    std::vector<int> vals{10, 255, 16, 42};
    std::cout << "Hex: ";
    std::cout << std::hex;
    std::transform(vals.begin(), vals.end(),
                   std::ostream_iterator<int>(std::cout, " "),
                   [](int x) { return x; });
    std::cout << std::dec << "\n";
    // Output: Hex: a ff 10 2a
}
```

`std::ostream_iterator<T>(os, delim)` works with any `std::ostream` - `std::cout`, `std::ofstream`, `std::ostringstream`. One thing to watch for: the delimiter appears **after** every element including the last, so you may get a trailing delimiter (notice the trailing comma in the "Doubled" output). If that bothers you, you need a separate pass or a custom separator.

---

### Q3: Explain the difference between back_insert_iterator, insert_iterator, and front_insert_iterator

| Property | `back_insert_iterator` | `front_insert_iterator` | `insert_iterator` |
| --- | --- | --- | --- |
| Helper | `std::back_inserter(c)` | `std::front_inserter(c)` | `std::inserter(c, pos)` |
| Container call | `c.push_back(v)` | `c.push_front(v)` | `c.insert(pos, v)` |
| Requires | `push_back` member | `push_front` member | `insert` member |
| Works with `vector` | Yes | No | Yes |
| Works with `deque` | Yes | Yes | Yes |
| Works with `list` | Yes | Yes | Yes |
| Works with `set/map` | No | No | Yes |
| Element order | Same as input | **Reversed** | Depends on position |

```cpp
#include <algorithm>
#include <vector>
#include <deque>
#include <list>
#include <set>
#include <iterator>
#include <iostream>

int main() {
    std::vector<int> src{1, 2, 3, 4, 5};

    // back_inserter - preserves order
    std::vector<int> v;
    std::copy(src.begin(), src.end(), std::back_inserter(v));
    std::cout << "back:  ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: back:  1 2 3 4 5

    // front_inserter - reverses order
    std::deque<int> dq;
    std::copy(src.begin(), src.end(), std::front_inserter(dq));
    std::cout << "front: ";
    for (int x : dq) std::cout << x << " ";
    std::cout << "\n";
    // Output: front: 5 4 3 2 1

    // inserter at begin - same as front but works with list
    std::list<int> lst;
    std::copy(src.begin(), src.end(), std::inserter(lst, lst.begin()));
    std::cout << "insert(begin): ";
    for (int x : lst) std::cout << x << " ";
    std::cout << "\n";
    // Output: insert(begin): 1 2 3 4 5
    // Note: inserter updates the position after each insert,
    // so order is preserved (unlike front_inserter).

    // inserter into a set - ordered by value, duplicates removed
    std::set<int> s;
    std::vector<int> dups{3, 1, 4, 1, 5, 9, 2, 6, 5};
    std::copy(dups.begin(), dups.end(), std::inserter(s, s.end()));
    std::cout << "set:   ";
    for (int x : s) std::cout << x << " ";
    std::cout << "\n";
    // Output: set:   1 2 3 4 5 6 9
}
```

**Key insight on `front_inserter` reversal:** Each call to `push_front` places the new element before all previously inserted elements, so the final order is reversed relative to the source. If you need front-insertion with original order, reverse the source first or use `std::inserter(c, c.begin())` on a container like `std::list` - `inserter` updates the stored position after each insert, which is why it preserves order even when starting at `begin()`.

---

## Notes

- Always prefer `std::back_inserter` for `vector` and `string` - it's the most common and efficient adaptor.
- `std::inserter` with `set`/`map` ignores the position hint if the value doesn't belong there - the container maintains its invariant.
- In C++20, `std::ranges::transform` returns a struct with `{in, out}` iterators, making it easier to chain operations.
- For large ranges, `reserve()` + `back_inserter` avoids repeated reallocations.
- `std::ostream_iterator` is an output iterator with category `output_iterator_tag` - it's single-pass and write-only.
