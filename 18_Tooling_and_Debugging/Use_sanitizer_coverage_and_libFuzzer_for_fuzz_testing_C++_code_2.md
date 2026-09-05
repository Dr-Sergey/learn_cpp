# Fuzz C++ code with structured inputs and multiple sanitizers

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/LibFuzzer.html>  

---

## Topic Overview

This is an extended guide on libFuzzer with advanced techniques: custom mutators, structured fuzzing, and CI integration.

Fuzz testing works by generating a flood of random or semi-random inputs and feeding them to your code as fast as possible, looking for crashes, assertion failures, or sanitizer violations. What makes modern fuzzers like libFuzzer particularly effective is the coverage feedback loop. The fuzzer instruments your binary to detect when a new input exercises a previously unseen code path. When that happens, it saves the input to the corpus and uses it as a basis for further mutations. Over time, the corpus grows to cover more and more of your code, and the fuzzer naturally finds its way into the deep, unexpected cases that manual testing rarely reaches.

The architecture looks like this:

```cpp
Advanced fuzzing architecture:

  Corpus        libFuzzer Engine       Target
  ┌──────┐     ┌─────────────┐     ┌───────────────┐
  │ seed1 │───>│  Mutator     │───>│ LLVMFuzzer     │
  │ seed2 │    │  Coverage   │    │ TestOneInput   │
  │ ...   │<───│  Feedback   │<───│ (your code)    │
  └──────┘     └─────────────┘     └───────────────┘
  (grows as new                     │ crash? timeout?
   paths found)                     v
                              ┌───────────────┐
                              │ ASan/UBSan/MSan│
                              │ detects bug    │
                              └───────────────┘
```

The corpus starts small (often just a few seed inputs) and grows automatically as the fuzzer discovers new paths. When a crash occurs, the sanitizer layer - AddressSanitizer, UBSan, or MemorySanitizer - diagnoses exactly what went wrong and where.

---

## Self-Assessment

### Q1: Set up a fuzz target with structured input

A fuzz target is just a function with a specific signature: `LLVMFuzzerTestOneInput`. libFuzzer calls it repeatedly with mutated byte buffers. The function should return 0 in all cases - the way you signal a bug is by crashing or triggering a sanitizer violation, not by returning an error code.

The parser below has two intentional bugs planted in it. The comments identify them. Part of the exercise is letting the fuzzer find them automatically.

```cpp
// fuzz_json.cpp - fuzz a JSON-like parser
#include <cstdint>
#include <cstddef>
#include <string>
#include <string_view>
#include <stdexcept>

// Simple parser to fuzz:
struct KeyValue {
    std::string key;
    std::string value;
};

KeyValue parse_kv(std::string_view input) {
    auto pos = input.find(':');
    if (pos == std::string_view::npos)
        throw std::invalid_argument("no colon");

    // BUG: doesn't handle empty key after trim
    auto key = input.substr(0, pos);
    auto value = input.substr(pos + 1);

    // BUG: accessing pos+1 when colon is last char
    if (value.empty())
        throw std::invalid_argument("empty value");

    return {std::string(key), std::string(value)};
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    if (size > 1024) return 0;  // limit input size for performance

    std::string_view input(reinterpret_cast<const char*>(data), size);
    try {
        auto kv = parse_kv(input);
        // Optional: validate output invariants
        if (kv.key.empty() && kv.value.empty())
            __builtin_trap();  // shouldn't happen
    } catch (const std::exception&) {
        // Expected for invalid input
    }
    return 0;
}
```

A dictionary file gives the fuzzer hints about tokens the parser cares about. Without it, the fuzzer is purely random. With it, the fuzzer quickly learns that `:` is a meaningful separator and starts generating inputs around it - which is exactly where the bugs are.

```bash
# Compile with dictionary for better exploration:
$ clang++ -std=c++20 -g -O1 -fsanitize=fuzzer,address fuzz_json.cpp -o fuzz_json

# Create a dictionary file for protocol tokens:
$ cat > json.dict <<EOF
":"
"="
"{}"
"[]"
"null"
"true"
"false"
EOF

# Run with dictionary and seed corpus:
$ mkdir -p corpus
$ echo -n 'key:value' > corpus/seed1
$ echo -n 'name:John' > corpus/seed2
$ ./fuzz_json -dict=json.dict corpus/
```

### Q2: Interpret crash artifacts and reproduce

When libFuzzer finds a crash, it writes the triggering input to a file named something like `crash-abc123`. This is valuable because you can reproduce the crash deterministically at any time - no need to rerun the fuzzer and hope for the same input.

```bash
# When fuzzer finds a crash:
# ==ERROR: AddressSanitizer:
# artifact_prefix='./'; Test unit written to ./crash-abc123

# Examine the crashing input:
$ hexdump -C ./crash-abc123
# 00000000: 3a                    :
# Input is just ":" -> empty key AND empty value

# Reproduce deterministically:
$ ./fuzz_json ./crash-abc123
# Crashes again with same error

# Minimize:
$ ./fuzz_json -minimize_crash=1 ./crash-abc123

# Find coverage gaps:
$ ./fuzz_json -print_coverage=1 corpus/ 2>&1 | head -20
# COVERED_FUNC: parse_kv
# UNCOVERED_FUNC: ...  (functions fuzzer hasn't reached)

# Merge and deduplicate corpus:
$ mkdir corpus_merged
$ ./fuzz_json -merge=1 corpus_merged/ corpus/
# Merged 500 inputs -> 42 unique coverage inputs
```

The crash input `3a` - a single colon character - is exactly the edge case the code failed to handle. The crashing input is always minimal enough to add to a regression test suite directly.

### Q3: Combine fuzzing with multiple sanitizers

libFuzzer becomes especially powerful when combined with a sanitizer. Each sanitizer detects a different class of bug, so you typically run separate fuzz binaries for each one. You cannot combine ASan and MSan in the same binary because they both intercept the same memory allocation calls.

```bash
# ASan + fuzzer (most common):
$ clang++ -fsanitize=fuzzer,address -g -O1 fuzz.cpp -o fuzz_asan
# Detects: heap overflow, use-after-free, double-free, stack overflow

# UBSan + fuzzer:
$ clang++ -fsanitize=fuzzer,undefined -g -O1 fuzz.cpp -o fuzz_ubsan
# Detects: signed overflow, null deref, type confusion, shift errors

# MSan + fuzzer (uninitialized reads):
$ clang++ -fsanitize=fuzzer,memory -g -O1 fuzz.cpp -o fuzz_msan
# Detects: use of uninitialized memory
# NOTE: requires all dependencies compiled with MSan

# CI integration (GitHub Actions):
```

Running fuzz tests in CI on a schedule - rather than only on every commit - is the recommended pattern because the fuzzer needs time to explore deeply. A five-minute nightly run consistently outperforms a thirty-second per-commit run.

```yaml
# .github/workflows/fuzz.yml
name: Fuzz Tests
on:
  schedule:
    - cron: '0 2 * * *'  # nightly
  workflow_dispatch:

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build fuzz targets
        run: |
          clang++ -std=c++20 -g -O1 \
            -fsanitize=fuzzer,address \
            fuzz_parser.cpp -o fuzz_parser
      - name: Download corpus
        uses: actions/download-artifact@v4
        with:
          name: fuzz-corpus
        continue-on-error: true
      - name: Run fuzzer (5 minutes)
        run: |
          mkdir -p corpus
          timeout 300 ./fuzz_parser corpus/ || true
      - name: Upload corpus
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-corpus
          path: corpus/
```

Uploading the corpus as an artifact and downloading it at the start of the next run is the key to making CI fuzzing effective. The fuzzer picks up where it left off each night, steadily accumulating coverage over time rather than restarting from scratch every run.

---

## Notes

- Use `FuzzedDataProvider` from `<fuzzer/FuzzedDataProvider.h>` for structured input generation.
- `-max_len=4096` limits input size; helps focus on relevant inputs.
- Run separate fuzz targets for ASan, UBSan, and MSan (can't combine them).
- OSS-Fuzz (Google) provides free continuous fuzzing for open-source projects.
- Corpus is cumulative: save between CI runs for faster coverage.
