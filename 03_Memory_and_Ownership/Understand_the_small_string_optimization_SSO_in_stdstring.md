# Understand the Small String Optimization (SSO) in std::string

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string>  

---

## Topic Overview

### What Is SSO

The **Small String Optimization** stores short strings directly inside the `std::string` object itself - in-situ in the object's footprint - rather than allocating heap memory. This avoids a heap allocation for the most common case: short strings. Since `std::string` typically occupies 32 bytes on a 64-bit machine anyway (to hold the pointer, size, and capacity), those bytes might as well hold small string data directly.

### How SSO Works Internally

Here is a simplified picture of what the two modes look like in memory. The union lets the same 32 bytes serve double duty: either holding a heap pointer (long mode) or holding the characters directly (short mode).

```cpp
// Without SSO (all strings heap-allocated):
struct string {            // 32 bytes on 64-bit
    char* data;            // -> heap buffer
    size_t size;
    size_t capacity;
};

// With SSO:
struct string {            // 32 bytes on 64-bit
    union {
        struct { char* data; size_t size; size_t capacity; };  // long mode
        struct { char buf[sizeof(above) - 1]; char size; };    // short mode
    };
};
// Short strings (up to ~15-22 chars) stored in buf — no heap!
```

### SSO Buffer Sizes by Implementation

The exact threshold varies by standard library. If you write code that relies on SSO to avoid allocations, you should test against your specific compiler.

| Implementation | SSO Buffer | Max SSO Length |
| --- | --- | --- |
| libstdc++ (GCC) | 15 bytes | 15 chars |
| libc++ (Clang) | 22 bytes | 22 chars |
| MSVC STL | 15 bytes | 15 chars |

### Key Implications

| Short string (SSO) | Long string (heap) |
| --- | --- |
| No `malloc`/`free` | Heap allocation |
| Data inside object | Data on heap |
| Move = copy (memcpy) | Move = pointer swap (O(1)) |
| Great cache locality | Pointer indirection |

The move row is the one that surprises people most, so let's come back to it in Q2.

---

## Self-Assessment

### Q1: Write a test that verifies short strings (< ~15 chars) don't heap-allocate on major implementations

The trick here is to compare the address of `s.data()` against the address range occupied by the `std::string` object itself. If the data pointer falls inside the object, the string is stored inline and no heap allocation occurred.

```cpp
#include <iostream>
#include <string>
#include <cstring>

void check_sso(const char* label, const std::string& s) {
    const void* obj_start = &s;
    const void* obj_end   = reinterpret_cast<const char*>(&s) + sizeof(s);
    const void* data_ptr  = s.data();

    bool is_sso = (data_ptr >= obj_start && data_ptr < obj_end);

    std::cout << label << ":\n"
              << "  length=" << s.size()
              << "  sizeof(string)=" << sizeof(s)
              << "  data_ptr=" << (const void*)s.data()
              << "  obj_addr=" << obj_start
              << (is_sso ? "  [SSO - INLINE]" : "  [HEAP]") << "\n";
}

int main() {
    std::cout << "=== SSO Detection ===\n";
    std::cout << "sizeof(std::string) = " << sizeof(std::string) << "\n\n";

    // These should all use SSO (no heap)
    check_sso("empty", std::string(""));
    check_sso("1 char", std::string("A"));
    check_sso("7 chars", std::string("1234567"));
    check_sso("15 chars", std::string("123456789012345"));

    // Boundary testing
    check_sso("16 chars", std::string("1234567890123456"));
    check_sso("22 chars", std::string("1234567890123456789012"));
    check_sso("23 chars", std::string("12345678901234567890123"));

    // Definitely heap
    check_sso("100 chars", std::string(100, 'x'));

    std::cout << "\n=== Finding the SSO threshold ===\n";
    for (size_t n = 0; n <= 30; ++n) {
        std::string s(n, 'A');
        const void* obj_start = &s;
        const void* obj_end   = reinterpret_cast<const char*>(&s) + sizeof(s);
        const void* data_ptr  = s.data();
        bool is_sso = (data_ptr >= obj_start && data_ptr < obj_end);
        if (!is_sso) {
            std::cout << "SSO threshold: " << n << " chars (first heap alloc)\n";
            std::cout << "Max SSO: " << (n - 1) << " chars\n";
            break;
        }
    }

    return 0;
}
// Example output (GCC/MSVC):
// sizeof(std::string) = 32
// empty:    length=0  [SSO - INLINE]
// 1 char:   length=1  [SSO - INLINE]
// 15 chars: length=15 [SSO - INLINE]
// 16 chars: length=16 [HEAP]
// SSO threshold: 16 chars (first heap alloc)
// Max SSO: 15 chars
```

### Q2: Explain why moving a small string that uses SSO is not faster than copying

This is one of the trickier implications of SSO, and the reason this trips people up is that "move always wins" is the common intuition. But it only holds when there is a heap pointer to steal. An SSO string has no heap pointer - its data lives inside the object. Moving it requires copying those inline bytes to the destination object and then clearing the source. That is the same work as a copy.

```cpp
#include <iostream>
#include <string>
#include <chrono>
#include <vector>

// SSO strings: data is INSIDE the object (no heap pointer to steal)
//
// Moving a heap string:   swap pointer + size -> O(1), no copy of content
// Moving an SSO string:   must copy all bytes from src buffer -> memcpy -> same cost as copy!
//
// After move, source must still be valid ("") -> must zero out its buffer
// Total work: memcpy + zero source ~ same as copy

template<typename Func>
long long bench(Func f, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) f();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
}

int main() {
    constexpr int N = 1'000'000;

    std::cout << "=== SSO: move vs copy performance ===\n\n";

    // Short string (SSO) — move ~ copy
    {
        std::string src = "Hello";  // 5 chars — SSO
        auto t_copy = bench([&] {
            std::string dst = src;  // copy
            (void)dst;
        }, N);

        auto t_move = bench([&] {
            std::string tmp = src;
            std::string dst = std::move(tmp);  // move (but SSO!)
            (void)dst;
        }, N);

        std::cout << "Short string (5 chars, SSO):\n";
        std::cout << "  Copy: " << t_copy / N << " ns/op\n";
        std::cout << "  Move: " << t_move / N << " ns/op\n";
        std::cout << "  Ratio: " << (double)t_move / t_copy << "x\n\n";
    }

    // Long string (heap) — move >>> copy
    {
        std::string src(200, 'X');  // 200 chars — heap
        auto t_copy = bench([&] {
            std::string dst = src;  // copy: allocate + memcpy 200 bytes
            (void)dst;
        }, N);

        auto t_move = bench([&] {
            std::string tmp = src;
            std::string dst = std::move(tmp);  // move: swap pointer
            (void)dst;
        }, N);

        std::cout << "Long string (200 chars, heap):\n";
        std::cout << "  Copy: " << t_copy / N << " ns/op\n";
        std::cout << "  Move: " << t_move / N << " ns/op\n";
        std::cout << "  Ratio: " << (double)t_move / t_copy << "x\n";
    }

    std::cout << "\n=== Key insight ===\n";
    std::cout << "SSO strings: move ~ copy (both memcpy the inline buffer)\n";
    std::cout << "Heap strings: move >> copy (pointer swap vs full copy)\n";
    std::cout << "Don't assume std::move always helps with strings!\n";

    return 0;
}
```

### Q3: Show how SSO affects the design of string-heavy hot paths in terms of move semantics

Once you know about SSO, it shapes how you approach string-heavy code. The three patterns below - `string_view` for read-only, `reserve` for building, and a fixed-size char array for identifiers - are all responses to SSO's constraints and strengths.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <chrono>

// Design implications of SSO for hot paths:
//
// 1. Don't move strings if they're short — no benefit
// 2. reserve() matters less for SSO-sized strings
// 3. string_view avoids copies entirely (best for read-only)
// 4. For many short strings, cache locality is already good (SSO = contiguous)
// 5. For known-short strings, consider std::array<char, N> for zero overhead

// Pattern 1: Token parsing — string_view avoids SSO/heap entirely
#include <string_view>

std::vector<std::string_view> tokenize_view(std::string_view input) {
    std::vector<std::string_view> tokens;
    size_t start = 0;
    while (start < input.size()) {
        size_t end = input.find(' ', start);
        if (end == std::string_view::npos) end = input.size();
        if (end > start)
            tokens.push_back(input.substr(start, end - start));
        start = end + 1;
    }
    return tokens;  // No string allocations at all!
}

// Pattern 2: Building result strings — reserve for known sizes
std::string build_csv(const std::vector<std::string>& fields) {
    // Estimate total size to avoid reallocations
    size_t total = 0;
    for (const auto& f : fields) total += f.size() + 1;

    std::string result;
    result.reserve(total);  // Single allocation

    for (size_t i = 0; i < fields.size(); ++i) {
        if (i > 0) result += ',';
        result += fields[i];
    }
    return result;  // NRVO — no copy
}

// Pattern 3: Fixed-size identifiers — avoid std::string entirely
struct FixedTag {
    char data[16] = {};  // Same size as SSO, but zero overhead

    FixedTag() = default;
    FixedTag(const char* s) {
        size_t len = std::min(std::strlen(s), size_t(15));
        std::memcpy(data, s, len);
    }
    std::string_view view() const { return {data, std::strlen(data)}; }
};

int main() {
    std::cout << "=== SSO-aware hot path design ===\n\n";

    // Pattern 1: string_view tokenization
    std::string input = "the quick brown fox jumps over the lazy dog";
    auto tokens = tokenize_view(input);
    std::cout << "Tokenized " << tokens.size() << " words (zero allocations):\n";
    for (auto t : tokens) std::cout << "  [" << t << "]\n";

    // Pattern 2: Pre-reserved string building
    std::vector<std::string> fields = {"name", "age", "city", "score"};
    std::string csv = build_csv(fields);
    std::cout << "\nCSV: " << csv << "\n";

    // Pattern 3: Fixed tag (no string overhead)
    FixedTag tag("PLAYER_01");
    std::cout << "\nFixed tag: " << tag.view()
              << " (sizeof=" << sizeof(tag) << ")\n";

    std::cout << "\n=== Design guidelines ===\n";
    std::cout << "1. Use string_view for read-only; avoids all allocation\n";
    std::cout << "2. Short keys/tags: may not benefit from move; consider fixed arrays\n";
    std::cout << "3. Long strings: move is valuable; avoid unnecessary copies\n";
    std::cout << "4. Batch operations: reserve() total size, then append\n";
    std::cout << "5. Profile: SSO threshold differs by compiler (15-22 chars)\n";

    return 0;
}
```

---

## Notes

- SSO eliminates heap allocation for short strings - the most common case in practice.
- SSO buffer size varies: 15 bytes (GCC/MSVC) or 22 bytes (libc++/Clang).
- `std::move()` on SSO strings provides **no speedup** - the data must still be copied from the inline buffer.
- Use `std::string_view` for read-only access to avoid all allocation regardless of length.
- `sizeof(std::string)` is typically 32 bytes on 64-bit systems, regardless of content length.
