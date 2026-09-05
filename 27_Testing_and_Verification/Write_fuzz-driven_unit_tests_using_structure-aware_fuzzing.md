# Write fuzz-driven unit tests using structure-aware fuzzing

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/LibFuzzer.html>  

---

## Topic Overview

**Structure-aware fuzzing** generates inputs that conform to a grammar or schema, reaching deep code paths that random byte mutation cannot. While basic fuzz testing throws random bytes at a function, structure-aware fuzzing understands the **format** of valid inputs. The practical payoff is enormous: a random byte fuzzer might spend 24 hours never getting past the parser's first validation check, while a structure-aware fuzzer reaches your actual business logic within minutes.

### Random vs Structure-Aware Fuzzing

The reason random byte mutation gets stuck is that most parsers and protocols reject invalid input immediately at the first byte. The odds of generating a syntactically valid JSON object by random mutation are astronomically low. Structure-aware fuzzing sidesteps this entirely by generating inputs that are valid by construction:

| Aspect | Random Byte Mutation | Structure-Aware Fuzzing |
| --- | :---: | :---: |
| Input validity | ~0.001% valid | ~90%+ valid |
| Code coverage | Stuck at parser rejection | Reaches business logic |
| Time to find deep bugs | Hours/days | Minutes |
| Setup effort | Minimal | Requires protobuf/custom mutator |
| Best for | Binary formats, simple APIs | Parsers, protocols, DSLs |

### Architecture

The most common combination is libFuzzer plus libprotobuf-mutator. The protobuf defines the grammar, the mutator generates structurally valid messages, a converter translates them to your actual format (JSON, SQL, etc.), and libFuzzer uses coverage feedback to guide mutation toward unexplored code paths:

```cpp
................................................
|             libFuzzer Engine               |
|  ..........    .......................      |
|  | Coverage|   | libprotobuf-mutator |     |
|  | Feedback|<--| (grammar-aware mut) |     |
|  ..........    .......................      |
|       |                  |                 |
|       |        ..........V............     |
|       |        |  Protobuf Message    |    |
|       |        |  (structured input)  |    |
|       |        .......................|    |
|       |                  | convert()       |
|       |        ..........V............     |
|       |        |  Domain Object       |    |
|       |        |  (JSON, SQL, Config) |    |
|       V        .......................|    |
|  Maximize               |                  |
|  coverage  -----> Function Under Test      |
................................................
```

---

## Self-Assessment

### Q1: Use libprotobuf-mutator to fuzz a parser with structured, grammar-aware inputs

**Answer:**

The workflow has three steps: define the grammar as a protobuf, write a converter from the protobuf to your target format, then write the fuzz target that drives everything. Let's walk through each step for a JSON parser:

**Step 1: Define the grammar as a protobuf:**

```protobuf
// json_grammar.proto
syntax = "proto3";

message JsonValue {
    oneof value {
        double number  = 1;
        string str     = 2;
        bool boolean   = 3;
        JsonArray arr  = 4;
        JsonObject obj = 5;
    }
}

message JsonArray {
    repeated JsonValue elements = 1;
}

message JsonObject {
    repeated JsonKeyValue pairs = 1;
}

message JsonKeyValue {
    string key   = 1;
    JsonValue val = 2;
}
```

**Step 2: Convert protobuf to the target format:**

```cpp
// json_converter.h
#include "json_grammar.pb.h"
#include <string>
#include <sstream>

std::string proto_to_json(const JsonValue& val) {
    std::ostringstream out;
    switch (val.value_case()) {
        case JsonValue::kNumber:
            out << val.number();
            break;
        case JsonValue::kStr:
            out << "\"" << val.str() << "\"";  // Note: not escaped, by design
            break;
        case JsonValue::kBoolean:
            out << (val.boolean() ? "true" : "false");
            break;
        case JsonValue::kArr: {
            out << "[";
            for (int i = 0; i < val.arr().elements_size(); ++i) {
                if (i > 0) out << ",";
                out << proto_to_json(val.arr().elements(i));
            }
            out << "]";
            break;
        }
        case JsonValue::kObj: {
            out << "{";
            for (int i = 0; i < val.obj().pairs_size(); ++i) {
                if (i > 0) out << ",";
                out << "\"" << val.obj().pairs(i).key() << "\":"
                    << proto_to_json(val.obj().pairs(i).val());
            }
            out << "}";
            break;
        }
        default:
            out << "null";
    }
    return out.str();
}
```

**Step 3: Write the fuzz target:**

```cpp
// json_fuzz.cpp
#include "src/libfuzzer/libfuzzer_macro.h"  // libprotobuf-mutator
#include "json_grammar.pb.h"
#include "json_converter.h"
#include "json_parser.h"    // YOUR parser under test

DEFINE_PROTO_FUZZER(const JsonValue& input) {
    std::string json_str = proto_to_json(input);

    // Limit input size to avoid timeouts
    if (json_str.size() > 10000) return;

    // The fuzzer will generate millions of STRUCTURALLY valid JSON inputs
    // that stress the parser with deep nesting, unicode, special floats, etc.
    try {
        auto result = parse_json(json_str);
        // Optional: verify invariants on the parse result
        // e.g., round-trip: serialize(result) should be equivalent
    } catch (const ParseError&) {
        // Expected - some proto combos produce invalid JSON (e.g., NaN)
    }
    // Any crash, ASAN error, or hang = BUG FOUND
}
```

**CMakeLists.txt:**

```cmake
find_package(Protobuf REQUIRED)
find_package(libprotobuf-mutator REQUIRED)

protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS json_grammar.proto)

add_executable(json_fuzzer json_fuzz.cpp ${PROTO_SRCS})
target_link_libraries(json_fuzzer
    PRIVATE protobuf::libprotobuf
    PRIVATE libprotobuf-mutator::libprotobuf-mutator
    PRIVATE json_parser_lib
)
target_compile_options(json_fuzzer PRIVATE -fsanitize=fuzzer,address -g)
target_link_options(json_fuzzer PRIVATE -fsanitize=fuzzer,address)
```

### Q2: Show how structure-aware fuzzing reaches deeper code paths than random byte mutation

**Answer:**

The coverage comparison below is based on a SQL-like parser. Random mutation spends almost all of its time failing the very first token check and never reaches the `JOIN` or `WHERE` logic. Structure-aware fuzzing, generating valid SQL by construction, covers all the deep paths within an hour:

```cpp
// SQL-like query parser
// To reach the 'execute_join' path, input must be:
//   SELECT <cols> FROM <table> JOIN <table2> ON <cond> WHERE <pred>

// Random fuzzing: generates "aX\x00\xff3k..." -> rejected at first token
// Structure-aware: generates valid SQL -> reaches JOIN/WHERE logic

// Coverage comparison
/*
Random byte mutation (24 hours):
  parse_token()          xxxxxxxxxxxx 100%
  parse_select()         xxx           25%  (rarely gets past SELECT)
  parse_from()           x              8%
  parse_join()                          0%  <- NEVER reached!
  parse_where()                         0%
  execute_join()                        0%

Structure-aware fuzzing (1 hour):
  parse_token()          xxxxxxxxxxxx 100%
  parse_select()         xxxxxxxxxxxx 100%
  parse_from()           xxxxxxxxxxxx 100%
  parse_join()           xxxxxxxxxx    85%  <- deep logic tested!
  parse_where()          xxxxxxxxx     75%
  execute_join()         xxxxxx        50%  <- BUG FOUND here!
*/

// Custom mutator (no protobuf)
// For simpler cases, write a custom mutator directly:

#include <cstddef>
#include <cstdint>
#include <cstring>

// Structure: 4-byte magic + 2-byte version + payload
struct Header {
    uint32_t magic;     // Must be 0xDEADBEEF
    uint16_t version;   // Must be 1 or 2
    uint16_t length;    // Payload length
};

// Random fuzzing: probability of hitting magic = 1/2^32 - never happens in practice
// Custom mutator: ALWAYS generates valid magic

extern "C" size_t LLVMFuzzerCustomMutator(
    uint8_t* data, size_t size, size_t max_size, unsigned int seed) {

    // Ensure header is always valid
    if (max_size < sizeof(Header)) return 0;

    Header hdr;
    hdr.magic = 0xDEADBEEF;     // Always valid magic
    hdr.version = (seed % 2) + 1; // Version 1 or 2
    hdr.length = (max_size - sizeof(Header)) & 0xFFFF;

    std::memcpy(data, &hdr, sizeof(Header));

    // Mutate only the payload (bytes after header)
    // libFuzzer's default mutator handles the rest
    return size;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    if (size < sizeof(Header)) return 0;

    Header hdr;
    std::memcpy(&hdr, data, sizeof(Header));

    if (hdr.magic != 0xDEADBEEF) return 0;  // Custom mutator ensures this passes
    if (hdr.version != 1 && hdr.version != 2) return 0;

    // NOW we're in the deep code path that random fuzzing never reaches
    process_payload(data + sizeof(Header), hdr.length, hdr.version);
    return 0;
}
```

The custom mutator approach is lighter weight than the full protobuf-mutator setup and works well for binary protocols with fixed headers. You guarantee the magic number and version are valid, then let libFuzzer mutate the payload freely.

### Q3: Integrate fuzzer findings into a permanent regression test suite

**Answer:**

When a fuzzer finds a crash, the failing input is saved to a file named `crash-<hash>`. The right response is to minimize the crash, fix the bug, and convert the minimized input into a permanent regression test so it can never regress:

**Step 1: Save crash corpus**

```bash
# Run fuzzer - crashes saved automatically
./json_fuzzer corpus/ -max_total_time=3600

# Crashes appear as:
#   crash-abc123def456
#   crash-789xyz000111

# Minimize the crashing input:
./json_fuzzer -minimize_crash=1 -exact_artifact_path=minimized.bin crash-abc123def456
```

**Step 2: Convert to regression tests**

```cpp
// regression_tests.cpp
#include <gtest/gtest.h>
#include <fstream>
#include <vector>

// Re-declare the fuzz target function
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size);

// Load crash file and replay as a unit test
std::vector<uint8_t> load_file(const std::string& path) {
    std::ifstream f(path, std::ios::binary);
    return {std::istreambuf_iterator<char>(f),
            std::istreambuf_iterator<char>()};
}

TEST(FuzzRegression, Crash_DeepNesting_Issue42) {
    // This crash was found by fuzzer on 2024-03-15
    // Root cause: stack overflow on deeply-nested JSON (>1000 levels)
    auto input = load_file("fuzz_corpus/crash-deep-nesting.bin");
    // Should NOT crash after fix
    LLVMFuzzerTestOneInput(input.data(), input.size());
}

TEST(FuzzRegression, Crash_InvalidUtf8_Issue58) {
    // Found by fuzzer: invalid UTF-8 sequence caused buffer over-read
    auto input = load_file("fuzz_corpus/crash-invalid-utf8.bin");
    LLVMFuzzerTestOneInput(input.data(), input.size());
}

// Alternative: inline small inputs
TEST(FuzzRegression, Crash_EmptyObject_Issue63) {
    // Minimal reproducer: "{\"\":}"
    const uint8_t input[] = {'{', '"', '"', ':', '}'};
    LLVMFuzzerTestOneInput(input, sizeof(input));
}
```

**Step 3: CI integration**

```yaml
# .github/workflows/fuzz.yml
name: Fuzz Testing

on:
  schedule:

    - cron: '0 2 * * *'  # Nightly at 2am

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build fuzzers

        run: |
          cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=fuzzer,address -g"
          cmake --build build --target json_fuzzer

      - name: Run fuzzer (10 min)

        run: |
          mkdir -p corpus crashes
          timeout 600 ./build/json_fuzzer corpus/ -artifact_prefix=crashes/ \
            -max_len=4096 -jobs=4 -workers=4 || true

      - name: Check for crashes

        run: |
          if ls crashes/crash-* 1>/dev/null 2>&1; then
            echo "::error::Fuzzer found crashes!"
            exit 1
          fi

      - name: Upload corpus

        uses: actions/upload-artifact@v4
        with:
          name: fuzz-corpus
          path: corpus/

      - name: Run regression tests

        run: |
          cmake --build build --target fuzz_regression_tests
          cd build && ctest -R FuzzRegression --output-on-failure
```

The full workflow from fuzzer finding to permanent protection looks like this:

```cpp
Fuzzer finds crash --> Minimize input --> File bug
                                            |
Fix the bug <------ Add regression test <---+
     |
     V
Commit fix + test --> CI runs regression --> Never regresses
                      CI runs fuzzer      --> Finds NEW bugs
```

The corpus upload step is important: persisting the corpus between CI runs means the fuzzer can build on previous coverage discoveries rather than starting from scratch every night.

---

## Notes

- Structure-aware fuzzing means libFuzzer + libprotobuf-mutator - this is the most common production combination.
- Always fuzz with ASAN enabled (`-fsanitize=fuzzer,address`) - the fuzzer alone cannot catch memory bugs, but ASAN will.
- `-max_len=4096` limits input size; increase this for complex formats that need larger inputs.
- `-jobs=N -workers=N` enables parallel fuzzing on multi-core CI machines.
- The corpus is **cumulative** - persist it between CI runs with artifact caching so you build on previous work.
- Google OSS-Fuzz uses structure-aware fuzzing for Chrome, OpenSSL, curl, and many other critical projects.
