# Leverage AI to generate property-based tests and fuzzing harnesses

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

Writing fuzz harnesses and property-based tests by hand is tedious work - you need to think carefully about what invariants your code should satisfy, how to express them as assertions, and how to give the fuzzer enough structural guidance to explore interesting inputs. AI assistants can generate **property-based tests** (tests that verify invariants over random inputs) and **fuzz testing harnesses** (for libFuzzer, AFL, or Honggfuzz). These are high-value and tedious to write manually. AI can identify the right properties to test, generate corpus seeds, write coverage-guided fuzz targets, and create structure-aware fuzzers. What would normally take an afternoon to set up can be sketched out in minutes with a well-formed prompt.

### Fuzz Target Generation Workflow

Here's the overall pipeline: you describe your API and invariants, the AI generates the harness, and then libFuzzer takes over and mutates inputs looking for crashes.

```cpp
Source Code --> Prompt AI --> Fuzz Harness --> libFuzzer --> Crashes
     |               |              |              |            |
     v               v              v              v            v
  API surface    Identify       LLVMFuzzer    Coverage-    Triage &
  + invariants   attack         TestOneInput  guided       fix bugs
                 surface                      mutation
```

---

## Self-Assessment

### Q1: Generate fuzz harnesses with AI

**Answer:**

The key to a good fuzzing prompt is telling the AI specifically what kinds of bad inputs you're worried about. Vague prompts produce generic harnesses; specific prompts produce harnesses that actually exercise the risky paths.

```cpp
=== PROMPT ===

"Generate a libFuzzer harness for this JSON parser.
Target:

- Malformed input (truncated, invalid UTF-8, deeply nested)
- Integer overflow in number parsing
- Stack overflow from recursive descent
- Memory safety (buffer overruns in string parsing)

Include a seed corpus and fuzzing dictionary."
```

The AI generates a harness around the parser's public API. Notice the round-trip property check inside - if parsing succeeds, we serialize and re-parse, asserting the results match. That single assertion catches a whole class of bugs where parsing and serialization aren't inverses of each other.

```cpp
// === Target: a simple JSON parser ===
class JsonParser {
public:
    struct Value { /* variant of object, array, string, number, bool, null */ };
    static std::expected<Value, ParseError> parse(std::string_view input);
    static std::expected<Value, ParseError> parse(std::span<const uint8_t> input);
};

// === AI-generated fuzz harness ===
// Compile: clang++ -fsanitize=fuzzer,address,undefined json_fuzz.cpp -o json_fuzz

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Limit input size to prevent timeout
    if (size > 1024 * 1024) return 0;

    std::string_view input(reinterpret_cast<const char*>(data), size);

    // Parse should never crash, regardless of input
    auto result = JsonParser::parse(input);

    if (result) {
        // Round-trip property: serialize and re-parse should match
        auto serialized = result->to_string();
        auto result2 = JsonParser::parse(serialized);
        assert(result2.has_value() && "Round-trip parse failed");
        assert(*result == *result2 && "Round-trip value mismatch");
    }

    return 0;
}

// === Seed corpus (save each as a file in corpus/ directory) ===
// 1. corpus/empty:            (empty file)
// 2. corpus/null:             null
// 3. corpus/bool:             true
// 4. corpus/number:           42
// 5. corpus/float:            3.14e10
// 6. corpus/string:           "hello world"
// 7. corpus/array:            [1,"two",null]
// 8. corpus/object:           {"key":"value","n":42}
// 9. corpus/nested:           {"a":{"b":{"c":[1,2,3]}}}
// 10. corpus/unicode:         "Aé世界"

// === Fuzzing dictionary (json.dict) ===
// Save as json.dict, run with: ./json_fuzz -dict=json.dict corpus/
//
// "true"
// "false"
// "null"
// "\"\""          # empty string
// "{}"             # empty object
// "[]"             # empty array
// "0"
// "-1"
// "1e308"          # near double max
// "1e-308"         # near double min
// "9999999999999999999"  # integer overflow
// "\\u0000"        # null byte in string
// "\\n"
// "\\t"
```

The seed corpus and dictionary together give libFuzzer a head start - it doesn't have to discover from random bytes that `{` and `"` are meaningful characters. This dramatically speeds up coverage.

### Q2: Generate property-based tests

**Answer:**

Property-based testing is about expressing the mathematical invariants your data structure should always satisfy, then letting a framework like RapidCheck throw thousands of random inputs at those invariants. The hard part is identifying the right properties. This is where AI is genuinely useful - it tends to suggest the classic set (insert/find, commutativity, sorted order, idempotency) that catch real bugs.

```cpp
=== PROMPT ===

"Generate property-based tests for a C++ BTree<K,V>
implementation using the RapidCheck framework.
Properties to verify:

1. All inserted keys are findable
2. Size equals number of unique insertions
3. Iteration produces sorted order
4. Remove + find returns empty
5. Insert order doesn't affect final tree content"
```

Each `RC_GTEST_PROP` is a property that gets checked against hundreds of randomly generated inputs. If RapidCheck finds a failing case, it automatically shrinks the input to the minimal example that reproduces the failure - which is a huge help for debugging.

```cpp
#include <rapidcheck.h>

// Property 1: Insert then find always succeeds
RC_GTEST_PROP(BTreeProperties, InsertThenFind,
    (std::vector<std::pair<int, std::string>> entries)) {
    BTree<int, std::string> tree;
    for (const auto& [k, v] : entries)
        tree.insert(k, v);

    for (const auto& [k, v] : entries) {
        auto result = tree.find(k);
        RC_ASSERT(result.has_value());
        // Value should be last inserted for this key
    }
}

// Property 2: Size equals unique key count
RC_GTEST_PROP(BTreeProperties, SizeEqualsUniqueKeys,
    (std::vector<std::pair<int, std::string>> entries)) {
    BTree<int, std::string> tree;
    std::set<int> unique_keys;
    for (const auto& [k, v] : entries) {
        tree.insert(k, v);
        unique_keys.insert(k);
    }
    RC_ASSERT(tree.size() == unique_keys.size());
}

// Property 3: Iteration is sorted
RC_GTEST_PROP(BTreeProperties, IterationIsSorted,
    (std::vector<int> keys)) {
    BTree<int, int> tree;
    for (int k : keys) tree.insert(k, k);

    std::vector<int> result;
    for (auto it = tree.begin(); it != tree.end(); ++it)
        result.push_back(it->first);

    RC_ASSERT(std::is_sorted(result.begin(), result.end()));
}

// Property 4: Insert order doesn't matter (commutativity)
RC_GTEST_PROP(BTreeProperties, InsertOrderIndependent,
    (std::vector<std::pair<int, int>> entries)) {
    RC_PRE(!entries.empty());

    BTree<int, int> tree1, tree2;
    for (const auto& [k, v] : entries)
        tree1.insert(k, v);

    auto shuffled = entries;
    std::shuffle(shuffled.begin(), shuffled.end(),
                 std::mt19937{42});
    for (const auto& [k, v] : shuffled)
        tree2.insert(k, v);

    // Same keys present
    std::vector<int> keys1, keys2;
    for (auto it = tree1.begin(); it != tree1.end(); ++it)
        keys1.push_back(it->first);
    for (auto it = tree2.begin(); it != tree2.end(); ++it)
        keys2.push_back(it->first);
    RC_ASSERT(keys1 == keys2);
}

// Property 5: Remove + find returns nullopt
RC_GTEST_PROP(BTreeProperties, RemoveThenFindFails,
    (std::vector<int> keys)) {
    RC_PRE(!keys.empty());
    BTree<int, int> tree;
    for (int k : keys) tree.insert(k, k);

    int to_remove = *rc::gen::elementOf(keys);
    tree.remove(to_remove);
    RC_ASSERT(!tree.find(to_remove).has_value());
}
```

The `RC_PRE` macro is a precondition - it tells RapidCheck to skip test cases where the precondition doesn't hold, rather than fail on them.

### Q3: Structure-aware fuzzing with AI

**Answer:**

Random byte mutation is fine for simple formats, but for structured binary protocols it's mostly wasted effort - random bytes will fail the magic-number check immediately and the fuzzer never reaches the interesting parsing logic. A structure-aware fuzzer knows what the format looks like and produces mutations that are structurally plausible.

```cpp
=== PROMPT ===

"Generate a structure-aware fuzzer for a protocol buffer
parser using libprotobuf-mutator + libFuzzer.
The protocol handles:

- Header (magic bytes, version, length)
- Payload (variable-length TLV records)
- CRC32 checksum

The fuzzer should mutate the structure intelligently,
not just random bytes."
```

The custom mutator intercepts libFuzzer's mutation step and replaces it with targeted edits that stay within the structural grammar of the protocol. Each `case` targets a different potential bug class.

```cpp
// === AI-generated structure-aware fuzzer ===

#include <cstdint>
#include <cstring>
#include <vector>

struct Header {
    uint32_t magic;     // 0xDEADBEEF
    uint16_t version;
    uint32_t payload_len;
};

struct TLVRecord {
    uint8_t type;
    uint16_t length;
    std::vector<uint8_t> value;
};

// Custom mutator: understands the protocol structure
extern "C" size_t LLVMFuzzerCustomMutator(
    uint8_t* data, size_t size, size_t max_size, unsigned int seed) {

    std::mt19937 rng(seed);
    std::uniform_int_distribution<int> action(0, 4);

    // Start with valid structure if too small
    if (size < sizeof(Header)) {
        Header hdr{0xDEADBEEF, 1, 0};
        std::memcpy(data, &hdr, sizeof(hdr));
        return sizeof(hdr);
    }

    switch (action(rng)) {
    case 0:  // Corrupt magic bytes
        data[rng() % 4] ^= (1 << (rng() % 8));
        break;
    case 1:  // Set extreme version
        reinterpret_cast<Header*>(data)->version = 0xFFFF;
        break;
    case 2:  // Mismatch payload_len
        reinterpret_cast<Header*>(data)->payload_len =
            static_cast<uint32_t>(rng());
        break;
    case 3:  // Truncate mid-TLV record
        if (size > sizeof(Header) + 3)
            return sizeof(Header) + 1;  // Cut inside TLV header
        break;
    case 4:  // Insert oversized TLV record
        if (size + 5 <= max_size) {
            data[size] = static_cast<uint8_t>(rng());      // type
            uint16_t big_len = 0xFFFF;
            std::memcpy(data + size + 1, &big_len, 2);     // length
            size += 3;
        }
        break;
    }
    return size;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Parser should handle all inputs without crashing
    auto result = ProtocolParser::parse(
        std::span<const uint8_t>(data, size));

    if (result) {
        // Invariant: parsed payload length matches actual records
        size_t total = 0;
        for (const auto& rec : result->records)
            total += 3 + rec.value.size();  // type + len + value
        assert(total <= result->header.payload_len);
    }
    return 0;
}
```

Case 2 (mismatched `payload_len`) and case 3 (truncation mid-record) are particularly valuable - those are the conditions most likely to expose length-check bugs or buffer overreads in the parser.

---

## Notes

- AI can generate **seed corpora** that cover diverse input patterns - always ask for them, because a good initial corpus makes fuzzing orders of magnitude more effective.
- For property-based testing, AI excels at identifying the **right properties** to check: commutativity, idempotency, round-trip, and invariant preservation are the classic ones worth starting from.
- **Structure-aware fuzzers** are much more effective than random byte mutation for protocols - the investment in a custom mutator pays off quickly in coverage.
- Always enable sanitizers when fuzzing: `-fsanitize=fuzzer,address,undefined` - without them, many memory bugs will silently pass.
- Ask AI to generate a **CMakeLists.txt** for the fuzz target with correct sanitizer flags, since the build configuration for fuzzing has non-obvious requirements.
- RapidCheck generators for custom types can be AI-generated: `rc::gen::build<MyType>(...)` - this is the main thing that makes property tests for domain types feasible.
- Run fuzz tests in CI with a **time budget** (e.g., 60 seconds) per target to catch regressions without blocking the build indefinitely.
