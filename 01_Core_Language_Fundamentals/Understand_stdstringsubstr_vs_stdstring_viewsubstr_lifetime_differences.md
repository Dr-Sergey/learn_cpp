# Understand std::string::substr vs std::string_view::substr lifetime differences

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string_view/substr>  

---

## Topic Overview

`std::string::substr()` and `std::string_view::substr()` have the same interface but fundamentally different ownership semantics - and this difference is a major source of dangling-reference bugs.

### Key Difference

| Method | Returns | Owns memory? | Safe to store? |
| --- | --- | --- | --- |
| `std::string::substr(pos, count)` | `std::string` (copy) | Yes - allocates new string | Always safe |
| `std::string_view::substr(pos, count)` | `std::string_view` (view) | No - points to original | Only if source outlives view |

### string::substr() - Always Safe

`string::substr()` allocates a fresh buffer and copies the characters in. Once the call returns, the result is completely independent of the original.

```cpp
std::string original = "Hello, World!";
std::string sub = original.substr(7, 5);  // "World" — independent copy
original.clear();
std::cout << sub << "\n";  // "World" — still valid, it's a copy
```

Even after clearing the original, `sub` is unaffected because it owns its own memory.

### string_view::substr() - Requires Source Lifetime

`string_view::substr()` returns a new view - just a pointer and a length - into the same underlying buffer. No allocation, no copy. That efficiency comes with a responsibility: the source must stay alive as long as you use the view.

```cpp
std::string original = "Hello, World!";
std::string_view sv = original;
std::string_view sub = sv.substr(7, 5);  // "World" — non-owning view!
original.clear();  // Data invalidated!
std::cout << sub << "\n";  // UNDEFINED BEHAVIOR — dangling view
```

Clearing the original string invalidates the memory `sub` points into. Using `sub` after that is undefined behavior.

### The Dangerous Pattern

The most common trap is returning a `string_view` from a function that takes a `const std::string&` parameter - because the caller might pass a temporary.

```cpp
// DANGER: string_view from a temporary
std::string_view get_extension(const std::string& filename) {
    std::string_view sv = filename;
    return sv.substr(sv.rfind('.'));  // View into filename — but what if filename is temporary?
}

// This is OK:
std::string file = "test.cpp";
auto ext = get_extension(file);  // View into 'file' — safe while 'file' lives

// This is DANGEROUS:
auto ext2 = get_extension(std::string("temp.txt"));
// The temporary std::string is destroyed after this line.
// ext2 is now a DANGLING string_view!
```

The temporary `std::string("temp.txt")` is destroyed at the end of that statement. `ext2` now points into freed memory.

---

## Self-Assessment

### Q1: Show that std::string::substr() returns an owning string while string_view::substr() returns a non-owning view

The key insight here is that `string::substr()` gives you independence while `string_view::substr()` keeps you tied to the original. Watch what happens when the original changes.

```cpp
#include <iostream>
#include <string>
#include <string_view>

int main() {
    std::string original = "Hello, World!";

    // string::substr() returns std::string (owning copy)
    std::string str_sub = original.substr(0, 5);  // "Hello"

    // string_view::substr() returns std::string_view (non-owning)
    std::string_view sv = original;
    std::string_view sv_sub = sv.substr(7, 5);    // "World"

    // Prove string::substr is independent
    original[0] = 'X';  // Modify original

    std::cout << "str_sub: " << str_sub << "\n";   // "Hello" — unchanged (owns data)
    std::cout << "sv_sub:  " << sv_sub << "\n";    // "Xorld" — changed (views original)!

    // Types prove it:
    static_assert(std::is_same_v<decltype(str_sub), std::string>);
    static_assert(std::is_same_v<decltype(sv_sub), std::string_view>);

    // Performance comparison:
    // string::substr()      -> heap allocation + copy (O(n))
    // string_view::substr() -> pointer arithmetic only (O(1))
    std::cout << "sizeof(str_sub): " << sizeof(str_sub) << "\n";   // ~32 (SSO/heap)
    std::cout << "sizeof(sv_sub):  " << sizeof(sv_sub) << "\n";    // 16 (ptr + size)
}
```

Notice that `str_sub` stays "Hello" while `sv_sub` reflects the change to 'X'. The `string_view` is a live window into the original; the `string` is a snapshot.

**How this works:**

- `string::substr()` allocates a new buffer and copies characters - the result is fully independent.
- `string_view::substr()` returns a new view (pointer + length) into the same underlying data.
- Modifying the original string changes what `string_view::substr()` sees but not `string::substr()`.

### Q2: Demonstrate a dangling string_view bug from calling .substr() on a temporary std::string

Here we have two bugs in one example: a temporary argument and a scope-limited local. Both destroy the source before the view is used.

```cpp
#include <iostream>
#include <string>
#include <string_view>

// DANGEROUS: Returns a string_view that may outlive its source
std::string_view extract_name(const std::string& full) {
    std::string_view sv(full);
    auto pos = sv.find(' ');
    return sv.substr(0, pos);  // View into 'full'
}

int main() {
    // SAFE: 'person' outlives 'name'
    std::string person = "John Doe";
    std::string_view name = extract_name(person);
    std::cout << "Safe: " << name << "\n";  // "John" — OK

    // DANGEROUS: Temporary string dies at semicolon!
    std::string_view bad = extract_name(std::string("Jane Smith"));
    // The temporary std::string("Jane Smith") is DESTROYED here.
    // 'bad' now points to freed memory!
    std::cout << "Dangling: " << bad << "\n";  // UNDEFINED BEHAVIOR!

    // ANOTHER COMMON BUG:
    std::string_view sv2;
    {
        std::string temp = "scoped string";
        sv2 = std::string_view(temp).substr(0, 6);
    }  // 'temp' destroyed here!
    std::cout << "Also dangling: " << sv2 << "\n";  // UNDEFINED BEHAVIOR!

    // FIX: Return std::string instead of string_view for unknown lifetimes
    // std::string extract_name_safe(...) { return std::string(sv.substr(0, pos)); }
}
```

Both bugs may appear to work in practice - the memory hasn't been reused yet - which makes them especially treacherous. Run under AddressSanitizer to catch them reliably.

**How this works:**

- `string_view::substr()` returns a view into the same memory as the original string.
- When the source string is a temporary or goes out of scope, the view becomes dangling.
- The program may appear to work (data may still be in memory) but this is undefined behavior.
- Fix: Return `std::string` (owning) when the lifetime of the source is uncertain.

### Q3: Write a function using string_view that calls substr() safely by ensuring the source outlives the result

The patterns here show you the full toolkit: return a view only when safe, convert to `std::string` when in doubt, and process transiently when you don't need to store the result at all.

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <vector>

// SAFE pattern 1: Return string_view only when caller owns the source
// The parameter type (string_view) makes it clear the caller must manage lifetime
std::string_view first_word(std::string_view text) {
    auto end = text.find(' ');
    return text.substr(0, end);  // View into caller's data
}

// SAFE pattern 2: Convert to string when lifetime is uncertain
std::string first_word_owning(std::string_view text) {
    auto end = text.find(' ');
    return std::string(text.substr(0, end));  // Returns owning copy
}

// SAFE pattern 3: Process in place without storing
void print_words(std::string_view text) {
    while (!text.empty()) {
        auto space = text.find(' ');
        std::cout << "  [" << text.substr(0, space) << "]\n";
        if (space == std::string_view::npos) break;
        text.remove_prefix(space + 1);  // Advance the view
    }
}

// SAFE pattern 4: Store results only as std::string
std::vector<std::string> split(std::string_view text, char delim) {
    std::vector<std::string> result;
    while (!text.empty()) {
        auto pos = text.find(delim);
        result.emplace_back(text.substr(0, pos));  // Converts to string!
        if (pos == std::string_view::npos) break;
        text.remove_prefix(pos + 1);
    }
    return result;
}

int main() {
    std::string sentence = "The quick brown fox";

    // SAFE: 'sentence' outlives 'word'
    std::string_view word = first_word(sentence);
    std::cout << "First word: " << word << "\n";  // "The"

    // SAFE: Returns owning string — no lifetime concern
    std::string word2 = first_word_owning("temporary string");
    std::cout << "First word: " << word2 << "\n";  // "temporary"

    // SAFE: Process immediately, don't store the views
    std::cout << "Words:\n";
    print_words(sentence);

    // SAFE: Split into owning strings
    auto parts = split("a,b,c,d", ',');
    for (const auto& p : parts) std::cout << p << " ";
    std::cout << "\n";  // a b c d
}
```

Pattern 3 is particularly elegant: `remove_prefix` slides the view forward in O(1) without any allocation. When you don't need to store results, transient `string_view` operations are both safe and fast.

**How this works:**

- Pattern 1: Return `string_view` only when the caller guarantees the source lives long enough.
- Pattern 2: Return `std::string` when lifetime cannot be guaranteed.
- Pattern 3: Use `string_view::substr()` transiently (don't store the result).
- Pattern 4: When storing substrings, always convert to `std::string`.

---

## Notes

- Rule of thumb: `string_view::substr()` is for reading/processing; `string::substr()` is for storing.
- Never return `string_view` from a function that creates the underlying `std::string` internally.
- `string_view::remove_prefix()` and `remove_suffix()` are O(1) alternatives to `substr()` that modify the view in place.
- AddressSanitizer (`-fsanitize=address`) can often detect dangling `string_view` access at runtime.
- C++23 adds `std::string::operator string_view()` as constexpr, but the lifetime rules remain the same.
