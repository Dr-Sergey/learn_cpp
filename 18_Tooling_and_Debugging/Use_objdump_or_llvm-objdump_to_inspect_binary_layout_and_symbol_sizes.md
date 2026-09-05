# Use objdump or llvm-objdump to inspect binary layout and symbol sizes

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/CommandGuide/llvm-objdump.html>  

---

## Topic Overview

After your code compiles and links, the binary is a black box unless you know how to inspect it. Binary inspection tools let you ask questions like "which functions are taking up the most space?", "is the vectorization I expected actually in the binary?", and "is my binary growing over time?". These are questions that no amount of reading source code can answer - you have to look at what the compiler actually produced.

| Tool | Purpose |
| --- | --- |
| `nm` | List symbols and their sizes |
| `objdump -d` | Disassemble functions |
| `llvm-objdump` | LLVM version of objdump |
| `size` / `llvm-size` | Section sizes (text/data/bss) |
| `bloaty` | Hierarchical binary size breakdown |
| `readelf` | ELF header and section details |

---

## Self-Assessment

### Q1: Find the largest functions with `nm --size-sort`

When a binary is larger than you expect, the first thing to do is find out which symbols are taking up the most space. `nm` with size sorting answers that question directly.

```cpp
// example.cpp - compile: g++ -O2 -std=c++20 example.cpp -o example
#include <vector>
#include <algorithm>
#include <string>
#include <map>
#include <iostream>

std::string process_strings(const std::vector<std::string>& v) {
    std::string result;
    for (const auto& s : v)
        result += s;
    return result;
}

void sort_map(std::map<int, std::string>& m) {
    // complex template instantiations
    for (auto& [k, v] : m)
        std::sort(v.begin(), v.end());
}

int main() {
    std::vector<std::string> v = {"hello", "world"};
    std::cout << process_strings(v) << '\n';
}
```

Running `nm` with size sorting and demangling:

```bash
# List symbols sorted by size (largest first):
$ nm --size-sort --reverse-sort --demangle example | head -20
# 0000000000001234 00000456 T std::__cxx11::basic_string<char>::_M_mutate(...)
# 0000000000002345 00000234 T process_strings(std::vector<std::string>...)
# 0000000000003456 00000198 T main
# 0000000000004567 00000167 T sort_map(std::map<int, std::string>&)
#

# Show only text (code) symbols:
$ nm --size-sort -t d --demangle example | grep ' T ' | tail -10

# Count total code size per object file (in archive):
$ nm --size-sort libfoo.a | awk '{sum[$NF]+=$1} END {for(s in sum) print sum[s], s}' | sort -rn

# LLVM equivalent:
$ llvm-nm --size-sort --demangle example
```

The three columns in the output are: address, size in hex, type (`T` means a text/code symbol), and demangled name. Scanning the largest entries often reveals template instantiation bloat, unexpectedly large standard library functions that were pulled in, or functions that grew because of inlining.

### Q2: Disassemble a function and verify SIMD usage

When you use `-O3` or `-march=native` and expect the compiler to auto-vectorize a loop, `objdump -d` lets you confirm whether it actually did. Look for instruction mnemonics like `vpaddd`, `vmulps`, or `addps` - those are SIMD instructions that process multiple values simultaneously.

```cpp
// simd_sum.cpp - compile: g++ -O3 -march=native -std=c++20 simd_sum.cpp -o simd_sum
#include <numeric>
#include <vector>

int sum_vec(const std::vector<int>& v) {
    return std::accumulate(v.begin(), v.end(), 0);
}

int main() {
    std::vector<int> v(1000, 1);
    return sum_vec(v);
}
```

Disassembling the `sum_vec` function and checking for SIMD instructions:

```bash
# Disassemble specific function:
$ objdump -d --demangle simd_sum | grep -A 30 '<sum_vec'
# sum_vec(std::vector<int> const&):
#
#   vpaddd %ymm1, %ymm0, %ymm0    <-- AVX2 SIMD! Adding 8 ints at once
#   add    $0x20, %rax
#   cmp    %rax, %rcx
#   jne
#
#   vpsrldq $0x8, %xmm0, %xmm1   <-- horizontal reduction
#   vpaddd  %xmm1, %xmm0, %xmm0

# Look for SIMD instruction patterns:
# vpaddd  = AVX integer add (8x32-bit)
# paddd   = SSE integer add (4x32-bit)
# addps   = SSE float add (4x32-bit)
# vmulpd  = AVX double multiply (4x64-bit)

# LLVM objdump (more readable output):
$ llvm-objdump -d --demangle --x86-asm-syntax=intel simd_sum

# Disassemble to file for analysis:
$ objdump -d --demangle simd_sum > disasm.txt
$ grep -c 'vpaddd\|paddd\|vmulps\|addps' disasm.txt
# Count of SIMD instructions
```

The presence of `vpaddd %ymm` instructions confirms AVX2 vectorization - each of those instructions adds 8 integers at once. If you only see scalar `add` instructions, the auto-vectorizer did not trigger, and you can investigate why (perhaps alignment, aliasing assumptions, or a loop structure that the compiler could not analyze).

### Q3: Track binary size regressions in CI with `bloaty` / `llvm-size`

Binary size tends to grow gradually unless you actively track it. `llvm-size` and `bloaty` give you the measurements you need to catch regressions before they accumulate.

```bash
# llvm-size shows section breakdown:
$ llvm-size example
#    text    data     bss     dec     hex filename
#   12456    1024     256   13736    35a8 example

# Track in CI:
$ llvm-size example | tail -1 | awk '{print $1}' > size_text.txt
# Compare with baseline:
# if [ $(cat size_text.txt) -gt 15000 ]; then echo "Binary too large!"; exit 1; fi

# Bloaty - hierarchical breakdown:
$ bloaty example
#     FILE SIZE        VM SIZE
#  -------------- --------------
#   42.3%  12.4Ki  52.1%  12.4Ki .text
#   18.4%   5.4Ki  22.6%   5.4Ki .rodata
#    8.2%   2.4Ki   0.0%       0 .symtab
#    7.1%   2.1Ki  10.1%   2.4Ki .data
#

# Bloaty diff between versions:
$ bloaty new_binary -- old_binary
#     FILE SIZE        VM SIZE
#  -------------- --------------
#  [ = ]       0   +256  +2.1% .text
#  [ = ]       0   -128  -5.0% .rodata

# CI script:
$ bloaty build/app -- baseline/app --csv > size_report.csv
```

The `bloaty` diff mode is particularly useful in code review: you can compare the binary before and after a pull request and see exactly which sections grew or shrank. A CI check that enforces a size budget on the `.text` section catches the kind of gradual growth that is easy to miss in any single change.

```yaml
# GitHub Actions CI for binary size tracking:

- name: Check binary size

  run: |
    llvm-size build/app | tee size.txt
    TEXT=$(awk 'NR==2{print $1}' size.txt)
    echo "Text section: $TEXT bytes"
    if [ "$TEXT" -gt 50000 ]; then
      echo "::error::Binary text section exceeds 50KB limit"
      exit 1
    fi
```

---

## Notes

- Use `c++filt` to demangle symbol names if `--demangle` is not available: `echo _Z7processi | c++filt` produces `process(int)`.
- `readelf -S` shows all ELF sections with sizes and flags, which is useful when you want more detail than `llvm-size` provides.
- Template-heavy code often has large `.text` sections because every unique type argument produces a new instantiation of the template body.
- `strip` removes the symbol table from a binary, reducing file size but losing the ability to demangle function names - useful for shipping but not for debugging.
- On macOS, use `otool -tV` instead of `objdump -d` for disassembly.
