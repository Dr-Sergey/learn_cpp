# Use objdump or llvm-objdump to inspect binary structure and symbol sizes

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/CommandGuide/llvm-objdump.html>  

---

## Topic Overview

Understanding what ended up in your binary - and why - is one of the most useful debugging skills in C++ development. Compiled binaries are not just a bag of machine code; they are organized into sections with specific purposes, each with its own size and memory characteristics. Inspecting those sections lets you diagnose linking issues, catch binary size regressions, and understand exactly what the compiler chose to emit.

```cpp
ELF binary structure:
┌────────────────────────┐
│  ELF Header            │
├────────────────────────┤
│  .text (code)          │  <- executable instructions
├────────────────────────┤
│  .rodata (read-only)   │  <- string literals, const data
├────────────────────────┤
│  .data (initialized)   │  <- global/static initialized vars
├────────────────────────┤
│  .bss (uninitialized)  │  <- zero-initialized globals (no file space)
├────────────────────────┤
│  .symtab (symbols)     │  <- function/variable names + sizes
└────────────────────────┘
```

One non-obvious detail worth remembering: `.bss` contains zero-initialized global and static variables, but it takes up no space in the file on disk. The OS knows to allocate zeroed memory for it at load time. That is why you can have a large `.bss` section without a correspondingly large binary file.

---

## Self-Assessment

### Q1: Find which symbols contribute most to binary size

The `nm` tool lists all symbols in a binary along with their sizes. Sorting by size with `--size-sort` immediately shows you where your binary bytes are coming from.

```bash
# nm: list all symbols with sizes, sorted largest first
$ nm --size-sort --reverse-sort --demangle myapp | head -15
# 00000000000045a0 0000089c T void std::__introsort_loop<...>(...)
# 00000000000023b0 00000567 T process_data(std::vector<int>&)
# 0000000000001230 00000345 T main
#

# Filter only defined text (code) symbols:
$ nm --size-sort --demangle myapp | grep ' [Tt] ' | tail -10

# Total code size from all symbols:
$ nm --size-sort -t d myapp | awk '/[Tt]/{sum+=$1} END{print sum " bytes of code"}'

# objdump: section headers with sizes
$ objdump -h myapp
# Idx Name          Size      VMA
#   1 .text         0000abc0  0000000000401000
#   2 .rodata       00001234  000000000040bc00
#   3 .data         00000100  000000000060d000
#   4 .bss          00000200  000000000060d100
```

When a binary is larger than you expected, this output usually reveals the culprit quickly. Common surprises include large template instantiations from the standard library, unexpectedly large functions that defeated inlining, or symbol duplication from header-only libraries.

### Q2: Measure text/data/bss with `llvm-size`

`llvm-size` provides a compact, structured summary of the section sizes in a binary or a set of object files. The Berkeley format (the default) gives you the four key categories in a single line.

```bash
# Basic section sizes:
$ llvm-size myapp
#    text    data     bss     dec     hex filename
#   43968    1024     512   45504    b1c0 myapp

# Berkeley format (default) vs System V format:
$ llvm-size --format=sysv myapp
# myapp  :
# section          size     addr
# .text           43968   4198400
# .rodata          4660   4242432
# .data            1024   6316032
# .bss              512   6317056
#

# Compare multiple objects:
$ llvm-size *.o
#    text    data     bss     dec     hex filename
#    2345     100       0    2445     98d main.o
#    5678     200      16    5894    1706 util.o
#    8901     300      32    9233    2411 math.o

# GNU size equivalent:
$ size myapp
```

Comparing individual object files is handy during a build where you are trying to understand which translation unit is contributing the most code. If `math.o` is unexpectedly large, that is where you start investigating for template instantiation bloat or missing `inline` keywords.

### Q3: Detect template instantiation explosion

Templates are one of the most powerful features in C++, but they come with a cost: each unique combination of template arguments generates a separate instantiation of the template body. In a large codebase with many template parameters, this can silently cause the binary to grow much larger than expected. `nm` can surface this directly.

```cpp
// bloat.cpp - generates many template instantiations
#include <vector>
#include <list>
#include <deque>
#include <map>
#include <set>
#include <string>

// Each unique type creates a new instantiation:
template<typename T>
void process(T& container) {
    for (auto& x : container)
        x = {};
}

int main() {
    std::vector<int> v1(10);
    std::vector<double> v2(10);
    std::vector<std::string> v3(10);
    std::list<int> l1;
    std::list<double> l2;
    std::deque<int> d1;
    std::map<int, std::string> m1;
    std::set<double> s1;

    process(v1); process(v2); process(v3);
    process(l1); process(l2);
    process(d1); process(m1); process(s1);
}
```

Running `nm` and filtering for the template instantiations reveals the cost of each one:

```bash
# Find template bloat by sorting symbols by size:
$ nm --size-sort --demangle bloat | grep 'process<' | sort -rn
# 00000345 T void process<std::map<int, std::string>>(std::map<...>&)
# 00000234 T void process<std::vector<std::string>>(std::vector<...>&)
# 00000189 T void process<std::list<double>>(std::list<...>&)
# 00000145 T void process<std::vector<int>>(std::vector<...>&)
# ...8 instantiations

# Count unique template instantiations:
$ nm --demangle bloat | grep 'process<' | wc -l
# 8

# Total size of template instantiations:
$ nm --size-sort -t d --demangle bloat | grep 'process<' | awk '{sum+=$1} END{print sum}'
# 2345 bytes just for process<> variants

# Fix: use type erasure or explicit instantiation to reduce bloat
```

Each call to `process` with a different container type generates a completely separate copy of the function body in the binary. For a small function like this the cost is modest, but in real codebases with complex template bodies called with many type combinations, this multiplication can easily add hundreds of kilobytes to the binary. Identifying and quantifying the bloat with `nm` is the first step toward addressing it.

---

## Notes

- `c++filt` demangles symbol names at the command line: `echo _Z7processi | c++filt` produces `process(int)`. Useful when `--demangle` is not available.
- `.bss` takes no space in the binary file but occupies memory at runtime - a distinction that matters when comparing file size against runtime memory usage.
- `strip --strip-all` removes the symbol table from a binary, reducing file size by 30-50%. The trade-off is that you lose the ability to demangle names and get meaningful stack traces in crash reports.
- On macOS, use `otool -l` for section information and `otool -tV` for disassembly, since macOS uses the Mach-O binary format rather than ELF.
- Template instantiation bloat is the most common cause of unexpectedly large C++ binaries. The fix is usually type erasure (erasing the type and using a virtual interface), explicit instantiation control, or redesigning to reduce the number of unique type arguments.
