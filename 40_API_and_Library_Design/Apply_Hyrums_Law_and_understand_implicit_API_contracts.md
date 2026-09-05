# Apply Hyrum's Law and understand implicit API contracts

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://www.hyrumslaw.com/>  

---

## Topic Overview

**Hyrum's Law** states: "With a sufficient number of users of an API, all observable behaviors of your system will be depended on by somebody." In other words, even undocumented behavior becomes part of your API the moment enough people rely on it.

This is one of those concepts that sounds obvious until it bites you. You fix a typo in an error message, or you tweak the internal iteration order of a container, and suddenly a user's code breaks - not because you changed anything documented, but because they were depending on something you never promised. The lesson is that users will find and depend on anything your API does, regardless of whether you intended it.

### Practical Implications

Here is a concrete example of how this plays out. The code below shows two versions of a function that throws on failure. Between v1 and v2, the error message text changed - that is all. But some users were parsing that string to extract the filename, so even a cosmetic rewording became a breaking change.

```cpp
#include <map>
#include <string>
#include <iostream>

// std::map iterates in sorted order - this is documented.
// But users also depend on:
// - The exact formatting of error messages (even though it's unspecified)
// - The iteration order of std::unordered_map (implementation-defined)
// - The exact sizeof (which differs between compilers/platforms)
// - Performance characteristics (O(1) amortized is not O(1) worst case)

// Example: changing message text broke real users
void before_v2() {
    // v1: throws with message "file not found: config.yaml"
    // Users parsed this string to extract the filename!
}

void after_v2() {
    // v2: changed to "cannot open file: config.yaml"
    // BROKE users who did: msg.substr(msg.find(": ") + 2)
}

// LESSON: error messages, whitespace in output, iteration order
// are all implicit API contracts.
```

The main takeaway here is not that those users were being unreasonable - it is that their dependency was predictable. If your API exposes an observable behavior, someone will eventually rely on it.

### Defensive API Design

The way to fight Hyrum's Law is to minimize what you expose in the first place. If users cannot see your internal container, they cannot depend on its properties. This example shows a bad cache that leaks its `std::map` internals versus a good cache that hides them behind a clean interface.

```cpp
#include <string>
#include <optional>

// BAD: leaking implementation details
class BadCache {
public:
    // Users can see the internal map and depend on its properties
    const std::map<std::string, std::string>& entries() const { return map_; }
private:
    std::map<std::string, std::string> map_;
};

// GOOD: minimal interface - internal representation is hidden
class GoodCache {
public:
    std::optional<std::string> get(const std::string& key) const;
    void put(const std::string& key, std::string value);
    size_t size() const;
    // No access to internal container - free to change implementation
private:
    std::unordered_map<std::string, std::string> map_; // Could be anything
};
```

Once `GoodCache` is in production, you can swap its internal `unordered_map` for a custom hash table, a sorted vector, or a database-backed store - and no caller will notice. That freedom is exactly what you want. With `BadCache`, every user who called `entries()` is now a hostage to the fact that you used a `std::map`.

---

## Self-Assessment

### Q1: Give examples of implicit API contracts in the C++ standard library

- `std::vector` elements are contiguous in memory (documented, but users also depend on `&v[0]` being a valid C array pointer).
- `std::string` is null-terminated (observable, and relied upon by every piece of C interop code that calls `.c_str()`).
- `std::sort` is not stable, but some users rely on the specific behavior of common introsort implementations.
- Move operations leave objects in a "valid but unspecified state" - but users frequently depend on a moved-from `string` being empty, even though that is not guaranteed.

### Q2: How to minimize implicit contract surface

- Export the minimal interface you can; hide implementation details with Pimpl or opaque types.
- Do not let internal containers leak through the API.
- Use opaque error types instead of string messages - string content is a notorious implicit contract magnet.
- Document explicitly what IS guaranteed, and mark everything else as "may change" so users know they cannot depend on it.

### Q3: What are "Chesterton's fence" changes in API evolution

Before removing or changing any behavior, understand WHY it exists - even if it was never documented. Hyrum's Law means someone almost certainly depends on it. The safe approach is: introduce the new behavior alongside the old, deprecate the old, and remove it only after a migration period long enough for users to adapt.

---

## Notes

- Hyrum's Law applies to all APIs: C++, REST, CLI tools, config file formats - anything with observable behavior.
- Fewer observable behaviors mean fewer implicit contracts, which means easier evolution over time.
- Semantic versioning helps communicate breakage: breaking changes increment the major version so users know to expect disruption.
- "Don't expose what you don't want to maintain forever" is the practical design rule that follows directly from Hyrum's Law.
