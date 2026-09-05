# Use std::iota for filling ranges with sequentially increasing values

**Category:** Standard Library Algorithms  
**Standard:** C++11, C++20 (ranges)  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/iota>  

---

## Topic Overview

`std::iota` fills a range with sequentially increasing values starting from a given value. Each element is assigned `value++`. It is the standard library way to say "fill this range with 0, 1, 2, 3, ..." without writing an explicit loop.

```cpp
#include <numeric>
std::iota(first, last, starting_value);
// Equivalent to:
// *first = starting_value;
// *(first+1) = starting_value + 1;
// ...
```

C++23 adds `std::ranges::iota` and `std::views::iota` for lazy generation. The table below summarizes the three variants and when each one is appropriate:

| Variant | Header | Eager/Lazy |
| --- | --- | --- |
| `std::iota(first, last, val)` | `<numeric>` | Eager (fills container) |
| `std::views::iota(start)` | `<ranges>` | Lazy (infinite range) |
| `std::views::iota(start, end)` | `<ranges>` | Lazy (bounded range) |

---

## Self-Assessment

### Q1: Use iota to initialize an index array [0, 1, 2, ..., n-1] for indirect sorting

One of the most practical uses of `iota` is setting up an index array that you then sort indirectly. The idea is: instead of moving the actual data around, you sort an array of integers where each integer is an index into the original data. The original array stays untouched, and you get the sorted order "for free" through the index array.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>
#include <string>

int main() {
    // === Create index array with iota ===
    std::vector<std::string> names = {"Charlie", "Alice", "Eve", "Bob", "Dave"};

    // Create indices [0, 1, 2, 3, 4]
    std::vector<int> indices(names.size());
    std::iota(indices.begin(), indices.end(), 0);

    std::cout << "Indices before sort: ";
    for (int i : indices) std::cout << i << " ";
    std::cout << "\n";
    // 0 1 2 3 4

    // Sort indices by the names they refer to (indirect sort)
    std::sort(indices.begin(), indices.end(),
              [&names](int a, int b) { return names[a] < names[b]; });

    std::cout << "Indices after sort:  ";
    for (int i : indices) std::cout << i << " ";
    std::cout << "\n";
    // 1 3 0 4 2  (indices of Alice, Bob, Charlie, Dave, Eve)

    std::cout << "Names in sorted order: ";
    for (int i : indices) std::cout << names[i] << " ";
    std::cout << "\n";
    // Alice Bob Charlie Dave Eve

    // Original array is UNCHANGED:
    std::cout << "Original names: ";
    for (const auto& n : names) std::cout << n << " ";
    std::cout << "\n";
    // Charlie Alice Eve Bob Dave

    // === Practical: rank array ===
    // rank[i] = position of element i in sorted order
    std::vector<double> scores = {85.5, 92.0, 78.3, 95.1, 88.7};
    std::vector<int> rank_idx(scores.size());
    std::iota(rank_idx.begin(), rank_idx.end(), 0);
    std::sort(rank_idx.begin(), rank_idx.end(),
              [&scores](int a, int b) { return scores[a] > scores[b]; });

    std::cout << "\nRankings:\n";
    for (int r = 0; r < static_cast<int>(rank_idx.size()); ++r) {
        std::cout << "  Rank " << (r + 1) << ": index " << rank_idx[r]
                  << " (score " << scores[rank_idx[r]] << ")\n";
    }
    // Rank 1: index 3 (score 95.1)
    // Rank 2: index 1 (score 92.0)
    // Rank 3: index 4 (score 88.7)
    // Rank 4: index 0 (score 85.5)
    // Rank 5: index 2 (score 78.3)

    return 0;
}
```

The pattern `iota` -> `sort-with-comparator` is used constantly in real code whenever you need a sorted view of data without copying or moving the underlying elements.

### Q2: Combine iota with shuffle to generate a random permutation

`iota` followed by `std::shuffle` is the standard idiom for generating a random permutation. You fill the range with a known sequence, then scramble it uniformly. The card-shuffling example below shows a practical version of this.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>
#include <random>

int main() {
    // === Generate a random permutation of [0..n-1] ===
    constexpr int N = 10;
    std::vector<int> perm(N);
    std::iota(perm.begin(), perm.end(), 0);  // {0, 1, 2, ..., 9}

    std::mt19937 rng(42);  // seeded for reproducibility
    std::shuffle(perm.begin(), perm.end(), rng);

    std::cout << "Random permutation: ";
    for (int x : perm) std::cout << x << " ";
    std::cout << "\n";
    // e.g.: 1 5 0 7 3 9 4 6 2 8

    // === Shuffle a deck of cards ===
    std::vector<int> deck(52);
    std::iota(deck.begin(), deck.end(), 0);  // cards 0-51
    std::shuffle(deck.begin(), deck.end(), rng);

    auto card_name = [](int c) -> std::string {
        const char* suits[] = {"♠", "♥", "♦", "♣"};
        const char* ranks[] = {"A","2","3","4","5","6","7","8","9","10","J","Q","K"};
        return std::string(ranks[c % 13]) + suits[c / 13];
    };

    std::cout << "First 5 cards: ";
    for (int i = 0; i < 5; ++i) std::cout << card_name(deck[i]) << " ";
    std::cout << "\n";

    // === Random sample using iota + shuffle + take first k ===
    std::vector<int> indices(100);
    std::iota(indices.begin(), indices.end(), 0);
    std::shuffle(indices.begin(), indices.end(), rng);

    std::cout << "Random sample of 5 from [0..99]: ";
    for (int i = 0; i < 5; ++i) std::cout << indices[i] << " ";
    std::cout << "\n";

    return 0;
}
```

The random-sample trick at the end - fill with `iota`, shuffle, take the first k - is essentially a Fisher-Yates partial shuffle and gives you a uniform random sample without repetition.

### Q3: Show iota with a user-defined type that supports prefix operator++

`std::iota` is not limited to integers. Any type that supports copy assignment and prefix `operator++` will work. The `Date` type below shows this cleanly - the only requirement is that `++` advances the value in whatever way makes sense for that type.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>

// === User-defined type with prefix operator++ ===
struct Date {
    int year, month, day;

    // Required by iota: prefix increment
    Date& operator++() {
        ++day;
        if (day > 28) {  // simplified: all months have 28 days
            day = 1;
            ++month;
            if (month > 12) {
                month = 1;
                ++year;
            }
        }
        return *this;
    }

    friend std::ostream& operator<<(std::ostream& os, const Date& d) {
        return os << d.year << "-"
                  << (d.month < 10 ? "0" : "") << d.month << "-"
                  << (d.day < 10 ? "0" : "") << d.day;
    }
};

// === Another example: letter range ===
struct Letter {
    char ch;

    Letter& operator++() { ++ch; return *this; }

    friend std::ostream& operator<<(std::ostream& os, const Letter& l) {
        return os << l.ch;
    }
};

int main() {
    // === Fill with dates ===
    std::vector<Date> week(7);
    std::iota(week.begin(), week.end(), Date{2025, 1, 1});

    std::cout << "Week:\n";
    for (const auto& d : week) std::cout << "  " << d << "\n";
    // 2025-01-01
    // 2025-01-02
    // ...
    // 2025-01-07

    // === Fill with letters ===
    std::vector<Letter> alphabet(26);
    std::iota(alphabet.begin(), alphabet.end(), Letter{'A'});

    std::cout << "Alphabet: ";
    for (const auto& l : alphabet) std::cout << l;
    std::cout << "\n";
    // ABCDEFGHIJKLMNOPQRSTUVWXYZ

    // === iota with iterators on array ===
    int arr[5];
    std::iota(std::begin(arr), std::end(arr), 10);
    std::cout << "Array: ";
    for (int x : arr) std::cout << x << " ";
    std::cout << "\n";
    // 10 11 12 13 14

    // === C++20: std::views::iota (lazy) ===
    // for (int i : std::views::iota(0, 10)) { ... }
    // auto naturals = std::views::iota(1);  // infinite: 1, 2, 3, ...

    return 0;
}
```

Notice that `std::iota` calls `operator++` on the value after each assignment - it does not compute `starting_value + n` for each slot. This is what makes custom types like `Date` and `Letter` work, and it is also what makes the type requirements so minimal.

---

## Notes

- **`std::iota`** is in `<numeric>`, NOT `<algorithm>`.
- **Name origin:** From the APL programming language's iota (ι) function that generates integer sequences.
- **C++20 `std::views::iota`:** Lazy version that generates values on-demand without allocating a container. Prefer this for pipelines: `for (int i : std::views::iota(0, n))`.
- **C++23 `std::ranges::iota`:** Eager version like `std::iota` but with range interface.
- **Common pattern:** `iota` -> `shuffle` for random permutations. `iota` -> `sort` with comparator for indirect sorting.
