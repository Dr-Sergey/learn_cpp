# Implement fuzz testing with libFuzzer or AFL for C++ code

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Fuzz testing** feeds random and mutated inputs to your code to find crashes, hangs, and memory errors. The key insight is that humans are terrible at imagining edge cases, especially for parsers, protocol handlers, and serialization code. A fuzzer does not think like a human - it mutates inputs in ways that trigger off-by-one errors, unexpected empty inputs, malformed headers, and integer boundaries that your hand-written tests would never cover.

Combined with sanitizers, fuzz testing is extraordinarily effective. The fuzzer finds the inputs; the sanitizers detect the bugs those inputs trigger. Together they catch buffer overflows, use-after-free, and undefined behavior that might otherwise lurk in production for years.

### Fuzzer Comparison

| Feature | libFuzzer | AFL++ | Honggfuzz |
| --- | --- | --- | --- |
| **Type** | In-process, coverage-guided | Fork-based, coverage-guided | Fork/in-process |
| **Speed** | Very fast (no fork) | Fast with persistent mode | Fast |
| **Integration** | Built into Clang | Separate tool | Separate tool |
| **Corpus management** | Built-in | Built-in | Built-in |
| **Sanitizer support** | Native (ASan, UBSan, MSan) | Via compiler flags | Via compiler flags |
| **Dictionaries** | Yes | Yes | Yes |
| **Structure-aware** | Custom mutators | Custom mutators | Custom mutators |

libFuzzer is the easiest to get started with for C++ since it ships with Clang and requires no separate installation.

---

## Self-Assessment

### Q1: Write a libFuzzer fuzz target for a parser

**Answer:**

A libFuzzer fuzz target is just a function named `LLVMFuzzerTestOneInput` that takes a byte array and a size. libFuzzer calls this function thousands of times per second with different mutations, tracking which inputs cause new code paths to execute. Every time it finds a new path, it keeps that input in the corpus and uses it as a seed for further mutations.

```cpp
// === fuzz_json_parser.cpp ===
// Fuzz target: entry point called by libFuzzer with random data

#include <cstdint>
#include <cstddef>
#include <string>
#include <string_view>
#include <stdexcept>

// Our JSON-like parser under test
struct JsonValue {
    enum Type { Null, Bool, Number, String, Array, Object };
    Type type;
    std::string string_val;
    double number_val;
};

JsonValue parse_json(std::string_view input);  // Function under test

// === libFuzzer entry point ===
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Convert raw bytes to string_view
    std::string_view input(reinterpret_cast<const char*>(data), size);

    try {
        auto result = parse_json(input);
        // Optional: verify invariants on successful parse
        // e.g., serialized result parses back to same value
    } catch (const std::exception&) {
        // Expected: invalid inputs should throw, not crash
    }
    // Any crash, ASAN error, or UBSan error is a bug
    return 0;
}
```

The `try/catch` is intentional: throwing an exception for invalid input is correct behavior. What you are hunting for is anything that *bypasses* the exception - crashes, infinite loops, or sanitizer violations.

To build and run the fuzzer, you need Clang with `-fsanitize=fuzzer`:

```bash
# === Build with libFuzzer + sanitizers ===
clang++ -g -O1 \
    -fsanitize=fuzzer,address,undefined \
    -fno-omit-frame-pointer \
    fuzz_json_parser.cpp json_parser.cpp \
    -o fuzz_json

# === Run ===
mkdir -p corpus
./fuzz_json corpus/ \
    -max_len=4096 \
    -timeout=10 \
    -dict=json.dict \
    -jobs=4 \
    -workers=4

# === Optional seed corpus ===
echo '{"key": "value"}' > corpus/seed1
echo '[1, 2, 3]' > corpus/seed2
echo 'null' > corpus/seed3
```

A seed corpus of known-valid inputs dramatically accelerates the fuzzer. It starts from working inputs and mutates them, which explores real parsing paths rather than spending most of its time on inputs that are rejected at the very first byte.

Providing a dictionary of protocol-specific tokens helps even more. Here is an example for JSON:

```cpp
# === json.dict (fuzzer dictionary) ===
"{"
"}"
"["
"]"
":"
","
"true"
"false"
"null"
"\""
```

### Q2: Build a CMake project with fuzzing targets

**Answer:**

Fuzz targets need to be guarded behind an option because they require Clang and produce binaries that are not useful for normal testing. The CMake setup below also wires the saved crash inputs into the regular test suite as regression tests, which is good practice - once you find a bug, you want to make sure it never comes back.

```cmake
# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.16)
project(my_project)

option(ENABLE_FUZZING "Build fuzz targets" OFF)

add_library(my_parser src/parser.cpp)
target_include_directories(my_parser PUBLIC include/)

if(ENABLE_FUZZING)
    # Require Clang for libFuzzer
    if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        message(FATAL_ERROR "Fuzzing requires Clang")
    endif()

    # Common fuzzing flags
    set(FUZZ_FLAGS "-fsanitize=fuzzer,address,undefined -fno-omit-frame-pointer")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${FUZZ_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${FUZZ_FLAGS}")

    # Add fuzz targets
    add_executable(fuzz_parser fuzz/fuzz_parser.cpp)
    target_link_libraries(fuzz_parser my_parser)

    add_executable(fuzz_serializer fuzz/fuzz_serializer.cpp)
    target_link_libraries(fuzz_serializer my_parser)

    # Regression test: run corpus as unit tests
    file(GLOB PARSER_CORPUS fuzz/corpus/parser/*)
    foreach(testcase ${PARSER_CORPUS})
        get_filename_component(name ${testcase} NAME)
        add_test(
            NAME fuzz_regression_${name}
            COMMAND fuzz_parser ${testcase}
        )
    endforeach()
endif()
```

When you pass a single file (instead of a corpus directory) to a libFuzzer binary, it runs just that one input and exits. That is how the regression tests work - each `add_test` entry runs the fuzzer binary against a single saved crash input.

```bash
# Build and run
cmake -B build-fuzz -DENABLE_FUZZING=ON \
    -DCMAKE_CXX_COMPILER=clang++
cmake --build build-fuzz

# Run fuzzer
./build-fuzz/fuzz_parser fuzz/corpus/parser/ -max_total_time=300

# When a crash is found, it's saved as crash-<hash>
# Add it to corpus for regression testing
cp crash-* fuzz/corpus/parser/
```

### Q3: Write a structure-aware fuzzer with custom mutators

**Answer:**

Pure random mutation is inefficient for binary protocols because most randomly-mutated inputs fail at header validation before reaching any interesting parsing logic. A custom mutator solves this by generating structurally valid mutations - tweaking header fields intelligently while leaving the overall message format intact.

```cpp
// === fuzz_protocol.cpp ===
// Structure-aware fuzzer for a binary protocol

#include <cstdint>
#include <cstddef>
#include <cstring>
#include <algorithm>

// Protocol header
struct __attribute__((packed)) Header {
    uint8_t  version;
    uint8_t  msg_type;
    uint16_t payload_length;
    uint32_t checksum;
};

void process_message(const uint8_t* data, size_t size);  // Under test

// Basic fuzz target
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    if (size < sizeof(Header)) return 0;  // Need at least a header

    process_message(data, size);
    return 0;
}

// === Custom mutator: generates valid protocol structures ===
extern "C" size_t LLVMFuzzerCustomMutator(
    uint8_t* data, size_t size, size_t max_size, unsigned int seed) {

    // Ensure we have at least a header
    if (max_size < sizeof(Header)) return 0;

    // Sometimes mutate header fields intelligently
    std::minstd_rand rng(seed);

    if (size >= sizeof(Header)) {
        Header* hdr = reinterpret_cast<Header*>(data);

        switch (rng() % 5) {
            case 0: hdr->version = rng() % 4; break;        // Valid versions: 0-3
            case 1: hdr->msg_type = rng() % 10; break;      // Valid types: 0-9
            case 2: {
                // Fix payload_length to match actual payload
                uint16_t payload = size - sizeof(Header);
                hdr->payload_length = payload;
                break;
            }
            case 3: {
                // Corrupt checksum (should be caught by validation)
                hdr->checksum ^= rng();
                break;
            }
            default:
                // Let default mutation handle it
                break;
        }
    }

    // Use libFuzzer's default mutation on the payload portion
    return size;
}

// === Custom cross-over for combining two inputs ===
extern "C" size_t LLVMFuzzerCustomCrossOver(
    const uint8_t* data1, size_t size1,
    const uint8_t* data2, size_t size2,
    uint8_t* out, size_t max_out_size,
    unsigned int seed) {

    // Take header from one, payload from another
    if (size1 < sizeof(Header) || size2 < sizeof(Header))
        return 0;

    size_t payload2_len = size2 - sizeof(Header);
    size_t total = sizeof(Header) + payload2_len;
    if (total > max_out_size) return 0;

    // Header from input 1
    std::memcpy(out, data1, sizeof(Header));
    // Payload from input 2
    std::memcpy(out + sizeof(Header), data2 + sizeof(Header), payload2_len);

    // Fix length field
    auto* hdr = reinterpret_cast<Header*>(out);
    hdr->payload_length = static_cast<uint16_t>(payload2_len);

    return total;
}
```

The crossover mutator is a nice touch - it combines the header from one corpus entry with the payload from another. This can create combinations that neither input alone would produce, covering code paths that depend on specific header/payload interactions.

For CI integration, run the fuzzer on a time budget rather than to completion (it never "completes"). Save the growing corpus as an artifact so future runs start from a richer set of seeds.

```yaml
# === CI integration: OSS-Fuzz style ===
name: Fuzz Testing
on:
  schedule:
    - cron: '0 2 * * *'  # Nightly

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build fuzz targets
        run: |
          cmake -B build-fuzz -DENABLE_FUZZING=ON \
              -DCMAKE_CXX_COMPILER=clang++
          cmake --build build-fuzz
      - name: Run fuzzer (time-limited)
        run: |
          ./build-fuzz/fuzz_parser fuzz/corpus/parser/ \
              -max_total_time=600 \
              -print_final_stats=1
      - name: Upload new corpus
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-corpus
          path: fuzz/corpus/
      - name: Upload crashes
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-crashes
          path: crash-*
```

---

## Notes

- Always fuzz with sanitizers (ASan + UBSan at minimum) - the fuzzer finds the inputs; the sanitizers detect the bugs those inputs trigger.
- Save crash-reproducing inputs as regression tests in your corpus directory so fixed bugs can never come back silently.
- Use dictionaries (`.dict` files) to seed the fuzzer with protocol-specific tokens - this can multiply coverage speed by an order of magnitude for structured input formats.
- Set `max_len` to match realistic input sizes. Too large wastes fuzzer effort on inputs your code would never realistically see; too small misses overflow bugs in large inputs.
- Structure-aware fuzzing with custom mutators is dramatically more effective for binary protocols and complex file formats, where purely random bytes almost never pass initial validation.
- Coverage-guided fuzzing finds bugs in minutes that random testing would miss in hours, because it actively steers toward unexplored code paths.
- Consider OSS-Fuzz for open-source projects - Google runs it continuously for free and has found tens of thousands of bugs in popular libraries.
- libFuzzer is built into Clang and requires no separate installation.
