# Use Sanitizer coverage and libFuzzer for fuzz testing C++ code

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/LibFuzzer.html>  

---

## Topic Overview

Fuzz testing is the practice of feeding random, malformed, or edge-case inputs to a function and letting the tool find crashes, undefined behavior, and assertion failures that your hand-written tests would never think of. libFuzzer is Clang's built-in coverage-guided fuzzer - it does not just throw random bytes at your code, it watches which branches each input exercises and steers mutations toward inputs that reach new code paths. That is what "coverage-guided" means, and it is what makes libFuzzer far more effective than a naive random input generator.

The workflow is straightforward:

```cpp
libFuzzer workflow:

  1. You write LLVMFuzzerTestOneInput(data, size)
  2. libFuzzer generates random inputs
  3. Coverage instrumentation guides mutation
  4. New code paths -> keep the input in corpus
  5. Crash detected -> save crashing input as artifact

  Mutations: bit flips, byte insertions, dictionary words,
             cross-over between corpus entries
```

You write a single entry-point function. libFuzzer owns the main loop, generates inputs, watches coverage counters, and saves any input that causes a crash. Your job is to interpret the data bytes as something your code can process.

---

## Self-Assessment

### Q1: Set up a fuzz target with `LLVMFuzzerTestOneInput`

The structure of every fuzz target is the same: interpret the raw bytes, call the code under test, and return 0. The key rule is that `LLVMFuzzerTestOneInput` must never crash on its own - any crash should come from the function you are testing, not from bad pointer arithmetic in the harness itself.

This example deliberately contains a bug (an out-of-bounds read when `key_len` is zero) so you can see the fuzzer catch it:

```cpp
// fuzz_parser.cpp — fuzz target for a simple parser
#include <cstdint>
#include <cstddef>
#include <cstring>
#include <stdexcept>

// Function under test: parses a key=value string
bool parse_config(const char* input, size_t len) {
    if (len < 3) return false;

    // Find '=' separator
    const char* eq = static_cast<const char*>(std::memchr(input, '=', len));
    if (!eq) return false;

    size_t key_len = eq - input;
    size_t val_len = len - key_len - 1;

    // BUG: off-by-one when key_len == 0
    if (key_len == 0)
        return input[key_len - 1] == 'x';  // underflow!

    return key_len > 0 && val_len > 0;
}

// Fuzz target entry point:
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Call the function under test:
    parse_config(reinterpret_cast<const char*>(data), size);
    return 0;  // must return 0
}
```

Compile with both the fuzzer and AddressSanitizer enabled, then run. The corpus directory accumulates inputs that exercise new coverage:

```bash
# Compile with fuzzer and sanitizer:
$ clang++ -std=c++20 -g -O1 \
    -fsanitize=fuzzer,address \
    fuzz_parser.cpp -o fuzz_parser

# Run the fuzzer:
$ mkdir corpus
$ ./fuzz_parser corpus/
# INFO: Seed: 1234567
# INFO: Loaded 0 files from corpus/
# #2    INITED
# #128  NEW    cov: 12 ft: 13 corp: 3/15b
# #512  NEW    cov: 15 ft: 18 corp: 5/32b
# ==12345==ERROR: AddressSanitizer: stack-buffer-underflow
# READ of size 1 at 0x7ffd... (1 byte before buffer)
#   #0 parse_config(char const*, unsigned long) fuzz_parser.cpp:17
#
# artifact_prefix='./'; Test unit written to ./crash-abc123

# The fuzzer found input "=" (key_len==0) that triggers the bug
```

Notice how quickly it finds the problem. The coverage instrumentation told libFuzzer that inputs containing `=` at position 0 reach a new branch, so it kept mutating in that direction until it triggered the underflow. This is the power of coverage-guided fuzzing: it finds the exact input class that causes trouble, not just random noise.

### Q2: Interpret crash artifacts and minimize

When libFuzzer finds a crash, it saves the crashing input to a file named `crash-<hash>`. That file is the raw bytes that triggered the problem - no metadata, just the input. You can reproduce the crash deterministically by passing the file as an argument.

```bash
# View the crashing input:
$ xxd ./crash-abc123
# 00000000: 3d                    =
# The input is just "=" (a single equals sign)

# Reproduce the crash:
$ ./fuzz_parser ./crash-abc123
# ==ERROR: AddressSanitizer: stack-buffer-underflow
# Same crash, deterministic

# Minimize the crashing input:
$ ./fuzz_parser -minimize_crash=1 -exact_artifact_path=./minimized ./crash-abc123
# Tries progressively smaller inputs that still crash
# Output: minimized from 1 bytes to 1 byte (already minimal)

# For larger crashes, minimization removes irrelevant bytes:
# Original: "aaaa=bbbb\x00\xff\x01" (12 bytes)
# Minimized: "=" (1 byte) <- much easier to debug

# Merge corpus (remove redundant entries):
$ ./fuzz_parser -merge=1 corpus_merged/ corpus/
# Keeps only inputs that cover unique code paths
```

The minimization step is valuable because a 300-byte crashing input with extraneous data is hard to reason about, but a 3-byte minimal reproducer shows you the exact structural condition that matters. Always minimize before filing a bug or writing a regression test.

### Q3: Combine fuzzing with AddressSanitizer

The real value of combining libFuzzer with ASan is that many memory bugs produce no crash on their own - they just silently corrupt memory and cause problems elsewhere. ASan turns those silent corruptions into immediate, precise aborts with exact stack traces. This example shows a heap buffer overflow that would be very hard to find without the sanitizer:

```cpp
// fuzz_buffer.cpp — finds heap buffer overflow
#include <cstdint>
#include <cstddef>
#include <cstring>

void process_packet(const uint8_t* data, size_t size) {
    if (size < 4) return;

    // Read length from first 2 bytes
    uint16_t payload_len;
    std::memcpy(&payload_len, data, 2);

    // BUG: trusts user-controlled length without bounds check
    uint8_t* buffer = new uint8_t[64];
    if (payload_len > 0) {
        std::memcpy(buffer, data + 2, payload_len);  // heap overflow if payload_len > 62!
    }
    delete[] buffer;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    process_packet(data, size);
    return 0;
}
```

```bash
# Compile with ASan + fuzzer:
$ clang++ -std=c++20 -g -O1 \
    -fsanitize=fuzzer,address \
    fuzz_buffer.cpp -o fuzz_buffer

$ ./fuzz_buffer corpus_buffer/
# Within seconds:
# ==ERROR: AddressSanitizer: heap-buffer-overflow
# WRITE of size 200 at 0x60200000efb0
#   #0 process_packet fuzz_buffer.cpp:14
# artifact: ./crash-def456

# The fuzzer crafted input where first 2 bytes = large number,
# causing memcpy to overflow the 64-byte buffer

# Fix:
void process_packet_fixed(const uint8_t* data, size_t size) {
    if (size < 4) return;
    uint16_t payload_len;
    std::memcpy(&payload_len, data, 2);
    if (payload_len > 62 || payload_len + 2 > size) return;  // bounds check!
    uint8_t* buffer = new uint8_t[64];
    std::memcpy(buffer, data + 2, payload_len);
    delete[] buffer;
}
```

The fuzzer figured out within seconds that setting the first two bytes to a large value and providing enough data to satisfy the `size >= 4` check would overflow the heap allocation. This is exactly the kind of input an attacker would craft. The fix adds the two missing guard conditions.

---

## Notes

- libFuzzer requires Clang; GCC does not support it natively.
- Always use ASan with fuzzing to catch memory bugs that would otherwise be silent.
- Use `-max_total_time=60` to limit fuzzing duration in CI so it does not run forever.
- Seed the corpus directory with valid sample inputs to give libFuzzer a better starting point and reach interesting code paths faster.
- Use `-dict=protocol.dict` with protocol-specific tokens (keywords, magic bytes) for better coverage on structured input formats.
