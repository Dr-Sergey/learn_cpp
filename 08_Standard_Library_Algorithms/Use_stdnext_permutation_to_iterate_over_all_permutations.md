# Use std::next_permutation to iterate over all permutations

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/next_permutation>  

---

## Topic Overview

This file focuses on the practical patterns of iterating over permutations, including larger element counts and optimization problems.

### The do-while Pattern

```cpp
std::sort(v.begin(), v.end());  // MUST start sorted
do {
    // process current permutation
} while (std::next_permutation(v.begin(), v.end()));
```

Why `do-while`? Because the first permutation (sorted ascending) should also be processed. A regular `while` loop would skip it since `next_permutation` transforms the range *then* returns. The reason this trips people up is that the return value signals "there is a next permutation after this one" - so by the time it returns `false`, it has already wrapped the sequence back to sorted and you must have processed the last permutation in the `do` body before that final call.

---

## Self-Assessment

### Q1: Iterate all permutations of a 4-element vector using next_permutation in a do-while loop

With 4 elements there are 4! = 24 permutations. This example prints the first five and the last five so you can see both ends of the lexicographic ordering, then confirms the total count. Notice also how `next_permutation` handles duplicate elements: `{1, 1, 2, 2}` produces 6 distinct permutations (4! / (2! * 2!)) rather than 24.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

int main() {
    std::vector<int> v = {1, 2, 3, 4};
    std::sort(v.begin(), v.end());  // ensure starting from first permutation

    int count = 0;
    do {
        if (count < 5 || count >= 19) {  // print first 5 and last 5
            for (int x : v) std::cout << x;
            std::cout << "\n";
        } else if (count == 5) {
            std::cout << "... (14 more) ...\n";
        }
        ++count;
    } while (std::next_permutation(v.begin(), v.end()));

    std::cout << "Total permutations: " << count << "\n";
    // First 5:  1234, 1243, 1324, 1342, 1423
    // Last 5:   4132, 4213, 4231, 4312, 4321
    // Total: 24 (= 4!)

    // After loop: v is back to {1, 2, 3, 4}

    // === Permutations of a subset (k of n) ===
    // To get all 3-element permutations from {1,2,3,4}:
    // Use next_permutation on the full array but only consider first 3 elements
    // Or: for each combination of 3, permute those 3

    // === Count distinct permutations with duplicates ===
    std::vector<int> with_dups = {1, 1, 2, 2};
    std::sort(with_dups.begin(), with_dups.end());
    int dup_count = 0;
    do {
        for (int x : with_dups) std::cout << x;
        std::cout << "  ";
        ++dup_count;
    } while (std::next_permutation(with_dups.begin(), with_dups.end()));
    std::cout << "\nWith duplicates: " << dup_count << " distinct permutations\n";
    // 1122  1212  1221  2112  2121  2211
    // With duplicates: 6 (= 4!/(2!*2!) = 6)

    return 0;
}
```

The duplicate-handling behavior is built into the algorithm. You do not need to deduplicate the output yourself - `next_permutation` automatically skips arrangements that would be identical due to repeated elements.

### Q2: Explain that next_permutation modifies in place and returns false when the sequence wraps

This example steps through a 3-element vector seven times to make the in-place modification and wrap-around behavior visible. Watch how the sequence evolves on each call, and notice that the seventh iteration is identical to the first - the cycle has completed.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    // === next_permutation modifies IN PLACE ===
    std::vector<int> v = {1, 2, 3};

    // Each call CHANGES v directly — no copy is made
    std::cout << "Step-by-step:\n";
    for (int i = 0; i < 7; ++i) {
        std::cout << "  v = {";
        for (size_t j = 0; j < v.size(); ++j)
            std::cout << (j ? "," : "") << v[j];
        std::cout << "}";

        bool result = std::next_permutation(v.begin(), v.end());
        std::cout << "  -> next_permutation returns " << std::boolalpha << result << "\n";
    }
    // Step-by-step:
    //   v = {1,2,3}  -> returns true  (now {1,3,2})
    //   v = {1,3,2}  -> returns true  (now {2,1,3})
    //   v = {2,1,3}  -> returns true  (now {2,3,1})
    //   v = {2,3,1}  -> returns true  (now {3,1,2})
    //   v = {3,1,2}  -> returns true  (now {3,2,1})
    //   v = {3,2,1}  -> returns FALSE (now {1,2,3} — wrapped!)
    //   v = {1,2,3}  -> returns true  (cycle repeats)

    // === Key implication for the do-while pattern ===
    // The do-while loop works because:
    // 1. Process the current permutation first (do body)
    // 2. Then advance to next (while condition)
    // 3. When it wraps (returns false), we've processed ALL permutations
    //    and the range is back to sorted = initial state

    // === WARNING: if you start unsorted, you miss permutations ===
    std::vector<int> mid = {2, 1, 3};  // NOT the first permutation
    int n = 0;
    do { ++n; } while (std::next_permutation(mid.begin(), mid.end()));
    std::cout << "\nStarting from {2,1,3}: only " << n << " permutations seen\n";
    // Only 4 permutations: {2,1,3} {2,3,1} {3,1,2} {3,2,1}
    // Missed: {1,2,3} {1,3,2}

    return 0;
}
```

The warning at the end is important: if you start from a mid-sequence arrangement, the `do-while` loop will only see the permutations from that point to the lexicographic end. It will not wrap around to cover the ones before your starting point. This is why you always `std::sort` before starting the loop.

### Q3: Use permutation iteration to solve a small combinatorial optimization problem

Brute-forcing all permutations with `next_permutation` is the simplest possible approach to assignment problems. For 5 workers and 5 jobs there are only 5! = 120 assignments to check, so this is perfectly fast. The second example shows maximizing a diagonal sum after row reordering, another classic application of the same pattern.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <string>
#include <climits>

int main() {
    // === Job Assignment Problem ===
    // N workers, N jobs. cost[i][j] = cost of assigning worker i to job j.
    // Find the assignment that minimizes total cost.
    // This is equivalent to the Hungarian Algorithm problem, solved here by brute force.

    const int N = 5;
    // Cost matrix: cost[worker][job]
    int cost[N][N] = {
        { 9,  2,  7,  8,  6},
        { 6,  4,  3,  7,  5},
        { 5,  8,  1,  8,  9},
        { 7,  6,  9,  4,  3},
        { 2,  3,  8,  5,  7}
    };

    // Permutation: perm[i] = job assigned to worker i
    std::vector<int> perm(N);
    std::iota(perm.begin(), perm.end(), 0);

    int best_cost = INT_MAX;
    std::vector<int> best_assignment;
    int total_perms = 0;

    do {
        int total = 0;
        for (int i = 0; i < N; ++i) {
            total += cost[i][perm[i]];
        }
        if (total < best_cost) {
            best_cost = total;
            best_assignment = perm;
        }
        ++total_perms;
    } while (std::next_permutation(perm.begin(), perm.end()));

    std::cout << "Job Assignment Problem (brute force)\n";
    std::cout << "Workers: " << N << ", Permutations checked: " << total_perms << "\n";
    std::cout << "Optimal assignment:\n";
    for (int i = 0; i < N; ++i) {
        std::cout << "  Worker " << i << " -> Job " << best_assignment[i]
                  << " (cost " << cost[i][best_assignment[i]] << ")\n";
    }
    std::cout << "Total cost: " << best_cost << "\n";

    // === Another example: maximize sum of diagonal after row permutation ===
    int matrix[4][4] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
        {13, 14, 15, 16}
    };

    std::vector<int> rows = {0, 1, 2, 3};
    int best_diag = 0;
    std::vector<int> best_rows;

    do {
        int diag_sum = 0;
        for (int j = 0; j < 4; ++j) {
            diag_sum += matrix[rows[j]][j];
        }
        if (diag_sum > best_diag) {
            best_diag = diag_sum;
            best_rows = rows;
        }
    } while (std::next_permutation(rows.begin(), rows.end()));

    std::cout << "\nMax diagonal sum: " << best_diag << "\n";
    std::cout << "Row order: ";
    for (int r : best_rows) std::cout << r << " ";
    std::cout << "\n";

    return 0;
}
```

For the job assignment problem, `perm[i]` encodes which job is assigned to worker `i`. Each permutation of `{0, 1, 2, 3, 4}` represents a unique one-to-one assignment where every worker gets exactly one job and every job is covered.

---

## Notes

- **Start sorted** for complete enumeration. Starting mid-sequence only iterates from that point to the end.
- **Factorial growth:** Only feasible for n <=~12 in practice. For larger n, use dynamic programming or heuristics.
- **Partial permutations:** To permute only k elements of n, separate the problem into choosing k elements (combinations) and permuting them.
- **`std::is_permutation`:** Checks if one range is a permutation of another in O(n^2) or O(n) with hashing.
- **Custom comparator:** `next_permutation(first, last, comp)` generates permutations according to the ordering defined by `comp`.
