# Use `std::array` as a Compile-Time Lookup Table

**Category:** Compile Time Programming  
**Standard:** C++11, C++14, C++17 (`std::array`)  
**Reference:** <https://en.cppreference.com/w/cpp/container/array>  

---

## Topic Overview

### Why `std::array` for Lookup Tables

A `constexpr std::array` is placed in the read-only data segment (`.rodata`) at compile time. Lookups are simple O(1) array indexing with **zero runtime initialization cost**. The table is computed once during compilation and baked directly into the binary.

Here's the canonical pattern - an immediately-invoked lambda builds the table, and the compiler evaluates the whole thing at build time:

```cpp
constexpr std::array<int, 256> hex_table = [] {
    std::array<int, 256> t{};
    // Fill with -1 for invalid
    for (auto& v : t) v = -1;
    // Map '0'-'9', 'a'-'f', 'A'-'F'
    for (int i = 0; i <= 9; ++i) t['0' + i] = i;
    for (int i = 0; i < 6; ++i) { t['a' + i] = 10 + i; t['A' + i] = 10 + i; }
    return t;
}();
```

### Benefits

It's worth comparing `constexpr std::array` against the alternatives so you know when to reach for it:

| Feature | `constexpr std::array` | `switch` statement | `std::unordered_map` |
| --- | --- | --- | --- |
| Initialization | Compile time | N/A | Runtime |
| Lookup speed | O(1) - array index | O(1) if jump table | O(1) amortized hash |
| Memory | Stack/rodata | Code section | Heap |
| Cache-friendly | Yes - contiguous | Depends | No - heap-allocated |

The `std::unordered_map` comparison is the most practical one: it initializes at runtime, allocates on the heap, and involves hashing overhead. A `constexpr std::array` avoids all three of those costs when the key is a dense integer (like an ASCII character value).

---

## Self-Assessment

### Q1: Create a `constexpr std::array<int,256>` mapping ASCII characters to their hex digit values

This is a classic use case for character-keyed lookup tables: given a character, return its hex digit value (0-15), or -1 if the character isn't a valid hex digit. The table is built as a named function for clarity, then the `static_assert`s confirm it's correct before any runtime code runs.

```cpp
#include <iostream>
#include <array>

// === Compile-time hex digit lookup table ===
constexpr std::array<int, 256> make_hex_table() {
    std::array<int, 256> table{};
    for (auto& v : table) v = -1;  // -1 = invalid

    for (int i = 0; i <= 9; ++i)
        table['0' + i] = i;

    for (int i = 0; i < 6; ++i) {
        table['a' + i] = 10 + i;
        table['A' + i] = 10 + i;
    }

    return table;
}

constexpr auto hex_table = make_hex_table();

// === Compile-time verification ===
static_assert(hex_table['0'] == 0);
static_assert(hex_table['9'] == 9);
static_assert(hex_table['a'] == 10);
static_assert(hex_table['F'] == 15);
static_assert(hex_table['z'] == -1);  // invalid
static_assert(hex_table[' '] == -1);  // invalid

// === Use in a hex parser ===
constexpr int parse_hex_byte(char hi, char lo) {
    int h = hex_table[static_cast<unsigned char>(hi)];
    int l = hex_table[static_cast<unsigned char>(lo)];
    if (h < 0 || l < 0) return -1;
    return (h << 4) | l;
}

static_assert(parse_hex_byte('F', 'F') == 255);
static_assert(parse_hex_byte('0', 'A') == 10);
static_assert(parse_hex_byte('1', '0') == 16);
static_assert(parse_hex_byte('z', '0') == -1);  // invalid

int main() {
    std::cout << "=== Hex Lookup Table ===\n";
    std::cout << "hex('0') = " << hex_table['0'] << "\n";
    std::cout << "hex('9') = " << hex_table['9'] << "\n";
    std::cout << "hex('a') = " << hex_table['a'] << "\n";
    std::cout << "hex('F') = " << hex_table['F'] << "\n";
    std::cout << "hex('z') = " << hex_table['z'] << " (invalid)\n";

    std::cout << "\n=== Hex Byte Parsing ===\n";
    std::cout << "parse_hex_byte('F','F') = " << parse_hex_byte('F', 'F') << "\n";
    std::cout << "parse_hex_byte('0','A') = " << parse_hex_byte('0', 'A') << "\n";
    std::cout << "parse_hex_byte('1','0') = " << parse_hex_byte('1', '0') << "\n";

    // Runtime usage with string
    const char* hex_str = "48656C6C6F";
    std::cout << "\nDecoding \"" << hex_str << "\": ";
    for (int i = 0; hex_str[i] && hex_str[i + 1]; i += 2) {
        int byte = parse_hex_byte(hex_str[i], hex_str[i + 1]);
        if (byte >= 0) std::cout << static_cast<char>(byte);
    }
    std::cout << "\n";

    return 0;
}
```

**Expected output:**

```text
=== Hex Lookup Table ===
hex('0') = 0
hex('9') = 9
hex('a') = 10
hex('F') = 15
hex('z') = -1 (invalid)

=== Hex Byte Parsing ===
parse_hex_byte('F','F') = 255
parse_hex_byte('0','A') = 10
parse_hex_byte('1','0') = 16

Decoding "48656C6C6F": Hello
```

### Q2: Use the lookup table at both compile time (`static_assert`) and runtime

One of the nice properties of `constexpr` tables is that the same table and the same lookup functions work in both contexts. You write the code once, and it's usable in `static_assert`, `if constexpr`, template arguments, and also in ordinary runtime loops.

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// === Compile-time character classification table ===
enum CharClass : int {
    OTHER = 0,
    DIGIT = 1,
    UPPER = 2,
    LOWER = 3,
    SPACE = 4,
    PUNCT = 5
};

constexpr auto make_char_class_table() {
    std::array<int, 128> t{};
    for (int c = '0'; c <= '9'; ++c) t[c] = DIGIT;
    for (int c = 'A'; c <= 'Z'; ++c) t[c] = UPPER;
    for (int c = 'a'; c <= 'z'; ++c) t[c] = LOWER;
    for (int c : {' ', '\t', '\n', '\r'}) t[c] = SPACE;
    for (int c : {'.', ',', '!', '?', ';', ':'}) t[c] = PUNCT;
    return t;
}

constexpr auto char_class = make_char_class_table();

// === Compile-time usage: static_assert ===
static_assert(char_class['A'] == UPPER);
static_assert(char_class['z'] == LOWER);
static_assert(char_class['5'] == DIGIT);
static_assert(char_class[' '] == SPACE);
static_assert(char_class['.'] == PUNCT);
static_assert(char_class['@'] == OTHER);

// === Compile-time usage: in constexpr functions ===
constexpr bool is_identifier_start(char c) {
    return char_class[static_cast<unsigned char>(c)] == UPPER ||
           char_class[static_cast<unsigned char>(c)] == LOWER ||
           c == '_';
}

constexpr bool is_identifier_char(char c) {
    return is_identifier_start(c) ||
           char_class[static_cast<unsigned char>(c)] == DIGIT;
}

static_assert(is_identifier_start('a'));
static_assert(is_identifier_start('_'));
static_assert(!is_identifier_start('5'));
static_assert(is_identifier_char('5'));

// === Runtime usage ===
const char* class_name(int c) {
    switch (c) {
        case DIGIT: return "digit";
        case UPPER: return "upper";
        case LOWER: return "lower";
        case SPACE: return "space";
        case PUNCT: return "punct";
        default:    return "other";
    }
}

int main() {
    // Runtime: classify each character in a string
    const char* text = "Hello, World! 123";
    std::cout << "Classifying: \"" << text << "\"\n";
    for (const char* p = text; *p; ++p) {
        int cls = (*p >= 0 && *p < 128) ? char_class[*p] : OTHER;
        std::cout << "  '" << *p << "' -> " << class_name(cls) << "\n";
    }

    // Runtime: count by class
    int counts[6] = {};
    for (const char* p = text; *p; ++p) {
        if (*p >= 0 && *p < 128) counts[char_class[*p]]++;
    }
    std::cout << "\nCounts: digits=" << counts[DIGIT]
              << " upper=" << counts[UPPER]
              << " lower=" << counts[LOWER]
              << " space=" << counts[SPACE]
              << " punct=" << counts[PUNCT]
              << " other=" << counts[OTHER] << "\n";

    return 0;
}
```

**Expected output:**

```text
Classifying: "Hello, World! 123"
  'H' -> upper
  'e' -> lower
  'l' -> lower
  'l' -> lower
  'o' -> lower
  ',' -> punct
  ' ' -> space
  'W' -> upper
  'o' -> lower
  'r' -> lower
  'l' -> lower
  'd' -> lower
  '!' -> punct
  ' ' -> space
  '1' -> digit
  '2' -> digit
  '3' -> digit

Counts: digits=3 upper=2 lower=7 space=2 punct=2 other=0
```

### Q3: Compare the binary output of a `constexpr` table vs a `switch` statement for the same lookup

This is a question about what the compiler actually generates. Both approaches can produce similar machine code in some cases, but the `constexpr` table approach is more predictable: it always compiles to a bounds check plus an indexed load, regardless of the number of cases.

```cpp
#include <iostream>
#include <array>

// === Method 1: constexpr table ===
constexpr std::array<const char*, 7> day_names_table = {
    "Sunday", "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday"
};

const char* day_name_table(int day) {
    if (day >= 0 && day < 7) return day_names_table[day];
    return "Invalid";
}

// === Method 2: switch statement ===
const char* day_name_switch(int day) {
    switch (day) {
        case 0: return "Sunday";
        case 1: return "Monday";
        case 2: return "Tuesday";
        case 3: return "Wednesday";
        case 4: return "Thursday";
        case 5: return "Friday";
        case 6: return "Saturday";
        default: return "Invalid";
    }
}

int main() {
    std::cout << "=== Table vs Switch ===\n";
    for (int i = 0; i < 7; ++i) {
        std::cout << "Table:  day " << i << " = " << day_name_table(i) << "\n";
        std::cout << "Switch: day " << i << " = " << day_name_switch(i) << "\n";
    }

    std::cout << "\n=== Binary Comparison (with -O2 on x86-64) ===\n\n";

    std::cout << "constexpr table:\n";
    std::cout << "  .rodata: 7 pointers (56 bytes on 64-bit) + string literals\n";
    std::cout << "  Code: bounds check + load from array\n";
    std::cout << "    cmp edi, 6\n";
    std::cout << "    ja .Linvalid\n";
    std::cout << "    mov rax, [day_names_table + rdi*8]   ; single indexed load\n";
    std::cout << "    ret\n";
    std::cout << "  Total: ~4 instructions\n\n";

    std::cout << "switch statement:\n";
    std::cout << "  Option A (compiler generates jump table - same as above):\n";
    std::cout << "    cmp edi, 6\n";
    std::cout << "    ja .Ldefault\n";
    std::cout << "    jmp [.Ljumptable + rdi*8]   ; jump table\n";
    std::cout << "  Option B (compiler generates if-else chain):\n";
    std::cout << "    cmp edi, 3\n";
    std::cout << "    je .Lwednesday\n";
    std::cout << "    cmp edi, 4\n";
    std::cout << "    ... (cascading comparisons)\n";
    std::cout << "  Total: varies (4-14 instructions)\n\n";

    std::cout << "=== Key Differences ===\n";
    std::cout << "1. constexpr table: ALWAYS one indexed load - predictable\n";
    std::cout << "2. switch: compiler DECIDES - may be jump table or if-else chain\n";
    std::cout << "3. Table code size is constant; switch code size grows with cases\n";
    std::cout << "4. Table is cache-friendly - contiguous memory\n";
    std::cout << "5. For <4 cases, switch may be faster (branch prediction)\n";
    std::cout << "6. For >8 cases, table is consistently faster and smaller\n";

    return 0;
}
```

The important takeaway from the assembly comparison: a `switch` statement may or may not generate a jump table depending on how dense the cases are and what the optimizer decides. A `constexpr std::array` always generates a single indexed load. You get consistent, predictable performance, and the behavior doesn't change as you add more cases.

---

## Notes

- `constexpr std::array` tables are placed in `.rodata` - zero runtime init cost.
- O(1) indexed lookup: `table[key]` compiles to a single memory load.
- For dense integer keys (0..N-1), `std::array` is the optimal lookup structure.
- For sparse keys, consider compile-time hash maps or sorted + binary search.
- The compiler may optimize a `switch` to a jump table, but this is not guaranteed.
- `std::array` lookup tables are universally used in parsers, character classifiers, and protocol decoders.
