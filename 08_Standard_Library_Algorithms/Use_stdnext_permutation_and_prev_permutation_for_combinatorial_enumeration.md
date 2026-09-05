# Use std::next_permutation and prev_permutation for combinatorial enumeration

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/next_permutation>  

---

## Topic Overview

`std::next_permutation` rearranges elements into the next lexicographically greater permutation **in place**. `std::prev_permutation` does the reverse. Together they let you walk through every possible arrangement of a sequence without generating them all upfront.

### Key Properties

| Function | Transforms to | Returns `false` when |
| --- | --- | --- |
| `next_permutation` | Next lexicographically greater | Range was last permutation (descending) -> wraps to first (ascending) |
| `prev_permutation` | Previous lexicographically smaller | Range was first permutation (ascending) -> wraps to last (descending) |

### Permutation Count

For n distinct elements: **n!** permutations.

| n | n! |
| --- | --- |
| 3 | 6 |
| 4 | 24 |
| 5 | 120 |
| 8 | 40,320 |
| 10 | 3,628,800 |

Brute-force over all permutations is only practical for small n (<=~10-12). Beyond that, the factorial growth makes it impractical and you need dynamic programming or heuristics instead.

---

## Self-Assessment

### Q1: Enumerate all permutations of {1,2,3} using next_permutation in a do-while loop

The `do-while` loop is the standard idiom here. You must use `do-while` rather than `while` because `next_permutation` transforms the range first and returns the result after - so a plain `while` loop would skip the initial (sorted) permutation. Start sorted, process in the `do` body, then advance in the `while` condition.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === Enumerate all permutations of {1, 2, 3} ===
    std::vector<int> v = {1, 2, 3};

    // MUST start sorted for complete enumeration
    std::sort(v.begin(), v.end());

    int count = 0;
    do {
        std::cout << "  ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
        ++count;
    } while (std::next_permutation(v.begin(), v.end()));

    std::cout << "Total: " << count << " permutations\n";
    // Output:
    //   1 2 3
    //   1 3 2
    //   2 1 3
    //   2 3 1
    //   3 1 2
    //   3 2 1
    // Total: 6 permutations

    // After the loop, v is back to {1, 2, 3} (sorted ascending)
    // because next_permutation wraps around and returns false

    // === Permutations of a string ===
    std::string s = "abc";
    std::sort(s.begin(), s.end());

    std::cout << "\nString permutations:\n";
    do {
        std::cout << "  " << s << "\n";
    } while (std::next_permutation(s.begin(), s.end()));
    // abc, acb, bac, bca, cab, cba

    // === prev_permutation: enumerate in reverse order ===
    std::vector<int> desc = {3, 2, 1};  // start from last permutation
    std::cout << "\nReverse enumeration:\n";
    do {
        for (int x : desc) std::cout << x << " ";
        std::cout << "\n";
    } while (std::prev_permutation(desc.begin(), desc.end()));
    // 3 2 1, 3 1 2, 2 3 1, 2 1 3, 1 3 2, 1 2 3

    return 0;
}
```

After the loop completes, `v` is back to `{1, 2, 3}` - the sorted starting state. That is part of the contract: `next_permutation` wraps around to the first permutation when it reaches the last one.

### Q2: Explain why next_permutation returns false on the last permutation (sorted descending)

Understanding why the descending sequence is the "last" permutation requires thinking about what "lexicographically greater" means. Just as 999 is the largest 3-digit number (no digit can be incremented without carrying), a fully descending sequence has no position where you can make a "local increment." The algorithm formalizes this intuition.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    // === The algorithm explained ===
    // next_permutation works by:
    // 1. Find the rightmost element that is smaller than its right neighbor
    //    (the "pivot"). This is the rightmost position where we can increment.
    // 2. Find the rightmost element larger than the pivot.
    // 3. Swap them.
    // 4. Reverse everything after the pivot position.
    //
    // For {3, 2, 1} (descending = last permutation):
    // Step 1 fails — NO element has a larger right neighbor.
    // Therefore, there is no "next" permutation.
    // The algorithm reverses the entire sequence -> {1, 2, 3} (first permutation)
    // and returns FALSE to signal the wrap-around.

    std::vector<int> v = {3, 2, 1};  // last permutation

    std::cout << "Before: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    bool has_next = std::next_permutation(v.begin(), v.end());

    std::cout << "After:  ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    std::cout << "Returned: " << std::boolalpha << has_next << "\n";
    // Before: 3 2 1
    // After:  1 2 3  (wrapped to first permutation)
    // Returned: false

    // === Step-by-step example: {1, 3, 2} -> {2, 1, 3} ===
    std::vector<int> demo = {1, 3, 2};
    // Step 1: rightmost element < right neighbor: v[0]=1 < v[1]=3, pivot at index 0
    // Step 2: rightmost element > pivot (1): v[2]=2, swap with pivot
    // Step 3: After swap: {2, 3, 1}
    // Step 4: Reverse after pivot: reverse {3, 1} -> {1, 3}
    // Result: {2, 1, 3}

    std::next_permutation(demo.begin(), demo.end());
    std::cout << "\n{1,3,2} -> ";
    for (int x : demo) std::cout << x << " ";
    std::cout << "\n";
    // {1,3,2} -> 2 1 3

    return 0;
}
```

**Key insight:** A descending sequence has no element smaller than its right neighbor, so no "increment point" exists. This makes it the lexicographically largest permutation, analogous to 999 being the largest 3-digit number.

### Q3: Use next_permutation to solve TSP by brute force for small N

For small N, brute-forcing all permutations is both simple and correct. The Travelling Salesman Problem (TSP) asks for the shortest cycle through all cities - with 5 cities you only need to check 4! = 24 routes, which is trivial. This is exactly where `next_permutation` earns its keep as a combinatorial enumeration tool.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>
#include <climits>

int main() {
    // === Brute-force Travelling Salesman Problem ===
    // Cities: 0, 1, 2, 3, 4 (5 cities)
    // Find the shortest Hamiltonian cycle starting and ending at city 0

    // Distance matrix (symmetric)
    const int N = 5;
    int dist[N][N] = {
        { 0, 10, 15, 20, 25},
        {10,  0, 35, 25, 30},
        {15, 35,  0, 30, 20},
        {20, 25, 30,  0, 15},
        {25, 30, 20, 15,  0}
    };

    // We fix city 0 as the start/end (reduces permutations from 5! to 4!)
    // Permute cities {1, 2, 3, 4}
    std::vector<int> cities = {1, 2, 3, 4};
    std::sort(cities.begin(), cities.end());

    int best_cost = INT_MAX;
    std::vector<int> best_route;

    int permutation_count = 0;
    do {
        // Calculate tour cost: 0 -> cities[0] -> ... -> cities[n-1] -> 0
        int cost = dist[0][cities[0]];  // from start
        for (size_t i = 0; i + 1 < cities.size(); ++i) {
            cost += dist[cities[i]][cities[i + 1]];
        }
        cost += dist[cities.back()][0];  // return to start

        if (cost < best_cost) {
            best_cost = cost;
            best_route = cities;
        }
        ++permutation_count;
    } while (std::next_permutation(cities.begin(), cities.end()));

    std::cout << "TSP Brute Force (" << N << " cities)\n";
    std::cout << "Permutations checked: " << permutation_count << "\n";
    std::cout << "Best route: 0";
    for (int c : best_route) std::cout << " -> " << c;
    std::cout << " -> 0\n";
    std::cout << "Cost: " << best_cost << "\n";
    // Permutations checked: 24 (4!)
    // Best route: 0 -> 1 -> 3 -> 4 -> 2 -> 0  (or similar)
    // Cost: depends on distances

    // === Complexity warning ===
    // n=10: 9! = 362,880 permutations (fast)
    // n=15: 14! = 87 billion (impractical)
    // For larger N, use dynamic programming, branch-and-bound, or heuristics.

    return 0;
}
```

Fixing city 0 as the start and end reduces the problem from 5! to 4! permutations without missing any routes - the starting city is irrelevant to which cycle is shortest, so you only need to vary the order of the remaining cities.

---

## Notes

- **Always sort first** for complete enumeration. If you start from a non-sorted state, `next_permutation` only generates permutations that come AFTER the current one lexicographically.
- **Duplicate elements:** `next_permutation` handles duplicates correctly - it generates only distinct permutations. For `{1, 1, 2}`, it generates 3 permutations, not 6.
- **Complexity:** Each call is O(n) worst case (for the reversal step). Amortized over all n! permutations, each call is O(1).
- **`prev_permutation`** is the mirror - useful when you want to enumerate backwards or need the previous arrangement.
- **Alternative for combinations:** `next_permutation` generates permutations. For combinations (n choose k), use bit manipulation or a custom algorithm - there's no `next_combination` in the standard library.
