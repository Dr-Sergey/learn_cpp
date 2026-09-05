# Understand and prevent buffer overflow vulnerabilities in C++ code

**Category:** Safety and Security  
**Standard:** C++23  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

Buffer overflow is the most classic vulnerability in C/C++ - writing beyond the bounds of an allocated buffer. It enables attackers to overwrite return addresses, function pointers, and adjacent variables to gain control of program execution. If you remember one thing from this section: every time you use a C string function, you're handling memory manually, and manual memory handling goes wrong.

### Buffer Overflow Anatomy

Here's what the stack actually looks like when a vulnerable function runs. Notice that a buffer growing upward sits below the return address - which is exactly what an attacker wants to overwrite:

```cpp
Stack layout (x86-64, simplified):
+------------------+ High addresses
| Return address    | <- Overwrite target!
+------------------+
| Saved RBP         |
+------------------+
| Stack canary      | <- Defense: random value checked before return
+------------------+
| local_buffer[16]  | <- strcpy writes here...
+------------------+    ...and overflows upward
| other locals      |
+------------------+ Low addresses
```

When the overflow reaches the return address, the attacker controls where the function "returns" to. Everything above that line in the diagram is theirs.

### Unsafe C Functions vs Safe C++ Alternatives

Every one of these unsafe functions has been the root cause of real CVEs. The C++ alternatives are not just stylistic preferences - they're the fixes:

| Unsafe | Problem | Safe alternative |
| --- | --- | --- |
| `strcpy(dst, src)` | No bounds check | `std::string`, `strncpy`, `strlcpy` |
| `sprintf(buf, fmt, ...)` | No size limit | `snprintf`, `std::format` |
| `gets(buf)` | No limit at all | `std::getline`, `fgets` |
| `scanf("%s", buf)` | No field width | `scanf("%15s", buf)`, `std::cin` |
| `memcpy(d, s, n)` | Wrong `n` | `std::copy_n`, `std::span` |

### Core Example

Here are three versions of the same function - the dangerous one, the safe one, and the fixed-size variant using modern C++ tools:

```cpp
#include <cstring>
#include <string>
#include <iostream>
#include <span>
#include <array>

void vulnerable(const char* input) {
    char buffer[16];
    std::strcpy(buffer, input); // NO bounds check - overflow if input > 15 chars!
    std::cout << buffer << "\n";
}

void safe_version(const char* input) {
    std::string buffer = input; // std::string manages its own memory
    std::cout << buffer << "\n";
}

void safe_fixed_buffer(std::span<const char> input) {
    std::array<char, 16> buffer{};
    auto count = std::min(input.size(), buffer.size() - 1);
    std::copy_n(input.begin(), count, buffer.begin());
    buffer[count] = '\0';
    std::cout << buffer.data() << "\n";
}

int main() {
    safe_version("This is safe regardless of length");
    safe_fixed_buffer(std::span("Short"));
}
```

The `std::string` version is the simplest: there's no buffer to overflow because the string manages its own memory. The `std::span` version shows how to work with a fixed-size buffer safely by tracking both the buffer size and the input size.

---

## Self-Assessment

### Q1: Show a classic buffer overflow via std::strcpy into a fixed-size buffer and how ASAN catches it

**Answer:**

This example shows what happens at the machine level when you overflow a buffer, and what ASan's output looks like when it catches it. The key insight is that without ASan, the overflow can be completely silent - the program just quietly corrupts the stack:

```cpp
#include <cstring>
#include <iostream>
#include <cstdlib>

// Compile: clang++ -fsanitize=address -fno-omit-frame-pointer -O1 -std=c++20 overflow.cpp

void process_input(const char* user_input) {
    char buffer[8]; // only 8 bytes!

    // VULNERABLE: strcpy doesn't check buffer size
    std::strcpy(buffer, user_input);
    // If user_input is longer than 7 chars -> BUFFER OVERFLOW

    std::cout << "Received: " << buffer << "\n";
}

int main() {
    // Trigger the overflow
    const char* long_input = "AAAAAAAABBBBCCCC"; // 16 bytes into 8-byte buffer!

    // Without ASan: may silently corrupt stack, crash later, or "work"
    // With ASan:
    //   ==12345==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x...
    //   WRITE of size 17 at 0x... thread T0
    //     #0 0x... in strcpy
    //     #1 0x... in process_input(char const*)
    //   Address 0x... is located in stack of thread T0 at offset 40 in frame
    //     #0 0x... in process_input(char const*)
    //   This frame has 1 object(s):
    //     [32, 40) 'buffer' (line 7) <== Memory access OVERFLOWS this variable

    // process_input(long_input); // Uncomment to demonstrate

    // ASan detection mechanisms
    // 1. Red zones: ASan inserts "poisoned" memory around stack/heap allocations
    // 2. Shadow memory: tracks which bytes are valid to access
    // 3. When code writes beyond buffer, it hits poisoned red zone
    // 4. ASan reports: exact address, stack trace, variable name, size of overflow

    // How to use ASan
    // Build: g++ -fsanitize=address -fno-omit-frame-pointer -O1 -g code.cpp
    // Run:   ./a.out  (crashes with detailed report at overflow point)
    // CI:    Run full test suite with ASan-enabled build

    std::cout << "Demo: buffer is 8 bytes, input is "
              << std::strlen(long_input) << " bytes\n";
    // Output: Demo: buffer is 8 bytes, input is 16 bytes
}
```

`strcpy` copies bytes until `'\0'` with no size limit - if the source is larger than the destination, it overflows. ASan detects this by surrounding every stack and heap allocation with "red zones" of poisoned memory. Any access to a red zone triggers an immediate, detailed error report with the exact location, variable name, and overflow size.

### Q2: Replace all unsafe C string functions (strcpy, sprintf, gets) with bounds-checked C++ alternatives

**Answer:**

Let's go through each dangerous function and show exactly what to use instead. The general principle is: if the function's signature doesn't take a maximum size, it's dangerous:

```cpp
#include <string>
#include <cstring>
#include <cstdio>
#include <iostream>
#include <array>
#include <format>
#include <algorithm>
#include <span>

void demonstrate_replacements() {
    // strcpy -> std::string
    // BAD:
    char dst1[16];
    // std::strcpy(dst1, very_long_string);  // OVERFLOW!

    // GOOD: std::string allocates as needed
    std::string safe_str = "any length string is fine here";
    std::cout << safe_str << "\n";

    // GOOD: if you NEED a fixed buffer, use strncpy + ensure null terminator
    const char* src = "hello world, this is a long string";
    std::strncpy(dst1, src, sizeof(dst1) - 1);
    dst1[sizeof(dst1) - 1] = '\0'; // strncpy doesn't guarantee null terminator!

    // BEST: std::copy with span
    std::array<char, 16> dst2{};
    auto to_copy = std::min(std::strlen(src), dst2.size() - 1);
    std::copy_n(src, to_copy, dst2.begin());

    // sprintf -> std::format / snprintf
    // BAD:
    char buf[32];
    // std::sprintf(buf, "User: %s, ID: %d", long_name, id);  // OVERFLOW!

    // GOOD: snprintf is bounded
    std::snprintf(buf, sizeof(buf), "User: %s, ID: %d", "Alice", 42);

    // BEST: std::format returns std::string (no overflow possible)
    auto formatted = std::format("User: {}, ID: {}", "Alice", 42);
    std::cout << formatted << "\n";

    // gets -> std::getline
    // BAD: gets() was REMOVED from C11 - unbounded read
    // char input[100];
    // gets(input);  // REMOVED - reads unlimited bytes

    // GOOD: std::getline with std::string
    // std::string line;
    // std::getline(std::cin, line); // allocates as needed

    // GOOD: fgets with fixed buffer (C-style)
    // char input[100];
    // fgets(input, sizeof(input), stdin); // bounded

    // scanf("%s") -> std::cin with limits
    // BAD:
    // char word[16];
    // std::scanf("%s", word);  // no width limit!

    // GOOD: width specifier
    // std::scanf("%15s", word);  // reads at most 15 chars

    // BEST: std::string input
    // std::string word;
    // std::cin >> word; // std::string manages memory

    // memcpy -> std::copy_n with span
    // BAD:
    // std::memcpy(dst, src, user_provided_size); // if size is wrong -> overflow!

    // GOOD: std::span enforces bounds
    std::array<int, 4> source{1, 2, 3, 4};
    std::array<int, 4> dest{};
    auto src_span = std::span(source);
    auto dst_span = std::span(dest);
    std::copy(src_span.begin(), src_span.end(), dst_span.begin());
}

int main() {
    demonstrate_replacements();
    std::cout << "All safe!\n";
}
```

Every unsafe C function has a safe C++ alternative. The general principle: use `std::string` for dynamic strings, `std::format` for formatting, `std::span` for fixed-size buffer access, and always prefer C++ containers over raw arrays. When C-style APIs are unavoidable, always use the bounded variants (`strncpy`, `snprintf`, `fgets`) and manually ensure null termination.

### Q3: Explain stack canaries, ASLR, and NX bits as defense-in-depth for buffer overflow exploits

**Answer:**

These are OS and compiler mitigations, not C++ features. They raise the difficulty bar for attackers considerably, but they don't fix the root cause. Each mitigation blocks a different attack technique, so an attacker needs to bypass all of them:

```cpp
// These are OS/compiler mitigations, not C++ features.
// They make exploitation HARDER but don't fix the root cause.

#include <iostream>
#include <cstdint>

int main() {
    // STACK CANARIES
    //
    // What: Random value placed between local variables and return address
    // How:  Compiler inserts canary check before every function return
    //
    // Stack layout WITH canary:
    //   [locals] [CANARY] [saved RBP] [return address]
    //       ^ overflow corrupts canary
    //   Before return: if (canary != expected) -> abort()
    //
    // Enable: -fstack-protector-strong (GCC/Clang, default in most distros)
    // Bypass: Attacker must leak canary value first (partial overwrite, info leak)
    //
    // Cost: ~1% performance overhead

    // ASLR (Address Space Layout Randomization)
    //
    // What: Randomize base addresses of stack, heap, libraries, executable
    // How:  OS kernel randomizes at process startup
    //
    //   Without ASLR:  libc always at 0x7ffff7a00000
    //   With ASLR:     libc at    0x7f38a2100000 (random each run)
    //
    // Enable: OS-level (Linux: /proc/sys/kernel/randomize_va_space = 2)
    //         Compile with -fPIE -pie for the executable itself
    // Bypass: Information leak (print a pointer -> derive base address)
    //
    // Cost: negligible

    // NX / W^X (No-eXecute / Write XOR Execute)
    //
    // What: Memory pages are either writable OR executable, never both
    // How:  Hardware (NX bit on x86-64) + OS enforces page permissions
    //
    //   Stack:  readable + writable, NOT executable
    //   Heap:   readable + writable, NOT executable
    //   Code:   readable + executable, NOT writable
    //
    // Effect: Attacker can inject shellcode onto stack/heap but CAN'T execute it
    // Bypass: ROP (Return-Oriented Programming) - chain existing code snippets
    //
    // Enable: Default on modern OS. Compile with -z noexecstack

    // Defense-in-depth: each mitigation blocks different attack techniques
    //
    // Attack                    Blocked by
    // -------                   ----------
    // Stack smash -> overwrite   Stack canary (detected at return)
    // return address
    //
    // Jump to known address      ASLR (address is randomized)
    // in libc (ret2libc)
    //
    // Execute injected           NX bit (stack/heap not executable)
    // shellcode on stack
    //
    // ROP (chain gadgets)        CFI + Shadow Stack (Control Flow Integrity)
    //
    // The REAL fix: Don't overflow in the first place.
    // Use std::string, std::vector, std::span, bounds checking.

    std::cout << "Stack pointer region: " << reinterpret_cast<uintptr_t>(&main) << "\n";
    // Will differ each run if ASLR is enabled
}

// Compile flags for maximum protection:
// g++ -std=c++20 -Wall -Wextra \
//     -fstack-protector-strong \    # stack canaries
//     -D_FORTIFY_SOURCE=2 \         # runtime bounds checking (glibc)
//     -fPIE -pie \                   # position-independent -> ASLR for executable
//     -Wl,-z,relro,-z,now \          # read-only relocations (GOT hardening)
//     -Wl,-z,noexecstack \           # NX for stack
//     -fcf-protection=full \         # Intel CET (shadow stack + IBT)
//     -fsanitize=address \           # ASan for development
//     code.cpp
```

Stack canaries detect corruption (the canary value changes if a buffer overflow occurs). ASLR randomizes memory layout so attackers can't hardcode addresses for their exploits. NX bits prevent execution of injected code in data regions. Together they form defense-in-depth - each blocks a different attack technique. But they're all mitigations, not fixes. The real solution is eliminating buffer overflows with bounds-checked C++ types.

---

## Notes

- **CWE-120 (Buffer Copy without Checking Size of Input)** is perennially in the Top 25.
- **`_FORTIFY_SOURCE=2`** (glibc) adds runtime bounds checking to `memcpy`, `strcpy`, etc. when the buffer size is known at compile time. Zero cost when no overflow occurs.
- **`std::span`** (C++20) is the modern way to pass fixed-size buffer references with compile-time size information.
- **`gets()`** was removed from C11 and C++14 - it's the only standard library function ever removed for being unconditionally unsafe.
- **Hardened STL:** libstdc++ debug mode and libc++ hardening mode add bounds checking to `operator[]`, catching overflows even in containers.
- Compile with `-std=c++20 -Wall -Wextra -fstack-protector-strong -D_FORTIFY_SOURCE=2`.
