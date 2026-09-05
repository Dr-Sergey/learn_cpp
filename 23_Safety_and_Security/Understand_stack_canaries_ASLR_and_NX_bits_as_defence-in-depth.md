# Understand stack canaries, ASLR, and NX bits as defence-in-depth

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Buffer_overflow_protection>  

---

## Topic Overview

Stack canaries, ASLR, and NX bits are compiler and OS-level mitigations that make buffer overflow exploits significantly harder - but they don't eliminate the root cause. Together they form a defence-in-depth strategy: each blocks a different exploitation technique, so an attacker must bypass all of them.

The important framing here is that these are not a substitute for writing correct code. They're a safety net that raises the attacker's cost. A buffer overflow that you've allowed to exist in your code is still a bug, even if the OS makes it hard to exploit today.

### Defence-in-Depth Layer Model

Think of this as a series of walls an attacker must climb over. Each wall stops a specific attack technique:

```cpp
Attack: buffer overflow -> overwrite return address -> execute shellcode

Layer 1: Stack Canary
  Detects: return address overwrite
  Bypass:  leak canary value (info disclosure)

Layer 2: NX (No-eXecute)
  Detects: execution of code on stack/heap
  Bypass:  ROP/JOP (reuse existing code gadgets)

Layer 3: ASLR
  Detects: nothing (preventive - randomizes addresses)
  Bypass:  info leak -> derive base addresses

Layer 4: CFI (Control Flow Integrity)
  Detects: indirect calls to unexpected targets
  Bypass:  data-only attacks, gadgets within valid targets

Layer 5: Shadow Stack (Intel CET)
  Detects: return address tampering (hardware-backed)
  Bypass:  data-only attacks (no ROP needed)
```

Each layer has a bypass - no single mitigation is impenetrable. But chaining them forces the attacker to chain their bypasses too, which raises the complexity and reduces reliability.

### Core Mechanism Summary

| Mitigation | What it does | Compiler/OS flag | Overhead |
| --- | --- | --- | --- |
| **Stack canary** | Random value between locals and return address; checked before `ret` | `-fstack-protector-strong` | ~1% |
| **ASLR** | Randomize stack, heap, shared library, and executable base addresses | OS + `-fPIE -pie` | Negligible |
| **NX bit** | Mark data pages (stack, heap) non-executable | `-z noexecstack` (default) | None |

All three are essentially free in performance terms - there's no reason not to enable them in every build.

### Core Example

All three mitigations are transparent to application code. You don't have to write anything special - they operate at the binary and OS level. But you can observe ASLR's effect at runtime by printing addresses:

```cpp
#include <iostream>
#include <cstdint>

// All three mitigations are transparent to application code.
// They protect at the binary/OS level.

void demonstrate_addresses() {
    int stack_var = 42;
    int* heap_var = new int(99);

    std::cout << "Stack address: " << &stack_var << "\n";
    std::cout << "Heap address:  " << heap_var << "\n";
    std::cout << "Code address:  " << reinterpret_cast<void*>(&demonstrate_addresses) << "\n";
    // Run twice - with ASLR, all addresses differ each run.

    delete heap_var;
}

int main() {
    demonstrate_addresses();
}
// Compile: g++ -std=c++20 -fstack-protector-strong -fPIE -pie -Wl,-z,noexecstack demo.cpp
```

If you run this binary twice and the addresses change, ASLR is working. If they stay the same, either ASLR is disabled or the binary wasn't compiled with PIE.

---

## Self-Assessment

### Q1: Explain how a stack canary detects stack buffer overflows before a return address is used

**Answer:**

The canary works because of the stack's memory layout. To overwrite the return address, an attacker's overflow must first pass through the canary value. Checking the canary before returning catches the corruption before control is transferred to the attacker's code:

```cpp
#include <iostream>
#include <cstdint>
#include <cstring>

// Stack canary mechanism (what the compiler generates):
//
// Original function:
//   void foo(const char* input) {
//       char buf[16];
//       strcpy(buf, input);
//   }
//
// With -fstack-protector-strong, compiler transforms to:
//
//   void foo(const char* input) {
//       uintptr_t canary = __stack_chk_guard;   // load random canary
//       char buf[16];
//       strcpy(buf, input);                      // potential overflow
//       if (canary != __stack_chk_guard)         // check BEFORE return
//           __stack_chk_fail();                  // abort! stack smashed!
//       return;                                  // safe to return
//   }
//
// Stack layout:
//   +------------------------+ High address
//   | Return address          | <- attacker wants to overwrite this
//   +------------------------+
//   | Saved frame pointer     |
//   +------------------------+
//   | CANARY VALUE            | <- random, set at function entry
//   +------------------------+
//   | buf[16]                 | <- overflow starts here
//   +------------------------+      and grows upward
//   | ...                     |
//   +------------------------+ Low address
//
// If attacker overflows buf to reach return address,
// they MUST overwrite the canary first.
// Function checks canary before returning -> mismatch -> __stack_chk_fail -> abort.

// How the canary value is generated:
// - At program startup, a random value is read from /dev/urandom
// - Stored in __stack_chk_guard (thread-local on most platforms)
// - Includes a null byte (0x00) at the low position to block strcpy
//   (strcpy stops at null -> can't write past canary cleanly)
//
// Protection levels:
//   -fstack-protector         : only functions with char arrays > 8 bytes
//   -fstack-protector-strong  : functions with any arrays, address-taken locals
//   -fstack-protector-all     : ALL functions (highest overhead)

int main() {
    // We can observe the effect indirectly:
    std::cout << "Stack canary detection is transparent to C++ code.\n";
    std::cout << "Compile with: -fstack-protector-strong\n";
    std::cout << "If overflow occurs, program aborts with:\n";
    std::cout << "  *** stack smashing detected ***: terminated\n";
    std::cout << "  Aborted (core dumped)\n";

    // To verify canary is present in your binary:
    // objdump -d a.out | grep -A5 '__stack_chk'
    // Or: checksec --file=a.out (shows Stack canary: Enabled)
}
```

The compiler inserts a random value (canary) between local variables and the saved return address at function entry. Before the function returns, it compares the canary to the expected value. A stack buffer overflow that reaches the return address must overwrite the canary first, which is detected. The canary includes a null byte to defeat `strcpy`-based attacks (since `strcpy` stops at `'\0'`).

### Q2: Show how ASLR randomises heap and stack base addresses and why it complicates exploits

**Answer:**

Without ASLR, an attacker can hardcode the address of `system()` or a known gadget because those addresses are the same on every machine running the same OS version. With ASLR, they'd need to guess from a large space - or find an information leak first:

```cpp
#include <iostream>
#include <cstdint>
#include <cstdlib>

// ASLR: Address Space Layout Randomization
//
// Without ASLR (predictable layout):
//   Executable:  0x00400000  (always)
//   Heap:        0x00602000  (always)
//   Stack:       0x7fffffffe000  (always)
//   libc:        0x7ffff7a00000  (always)
//
// With ASLR (randomized each run):
//   Executable:  0x5555557a0000  (random, with PIE)
//   Heap:        0x555555b21000  (random)
//   Stack:       0x7ffd38a21000  (random)
//   libc:        0x7f38a2100000  (random)
//
// Attacker can't hardcode addresses for:
//   - Return-to-libc (system() address unknown)
//   - ROP gadget chains (gadget addresses unknown)
//   - Shellcode on heap (heap address unknown)

void function_on_stack() {
    int stack_local = 42;
    int* heap_alloc = new int(99);

    // Print addresses - differ each run with ASLR
    std::cout << "Run this multiple times to see ASLR effect:\n";
    std::cout << "  main() code:     " << reinterpret_cast<void*>(&main) << "\n";
    std::cout << "  Stack variable:  " << &stack_local << "\n";
    std::cout << "  Heap allocation: " << heap_alloc << "\n";

    // With ASLR disabled:  same addresses every run
    // With ASLR enabled:   different addresses every run
    //
    // To test:
    //   # Disable ASLR temporarily (Linux):
    //   echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
    //   # Run: ./a.out   ->  same addresses
    //   # Run: ./a.out   ->  same addresses
    //
    //   # Re-enable ASLR:
    //   echo 2 | sudo tee /proc/sys/kernel/randomize_va_space
    //   # Run: ./a.out   ->  different addresses
    //   # Run: ./a.out   ->  different addresses

    delete heap_alloc;
}

// Why ASLR complicates exploits:
//
// Classical exploit: overflow buf -> overwrite RET -> jump to shellcode
//   Without ASLR: shellcode at known address 0x7fffffffe100
//   With ASLR:    shellcode at ???  (27+ bits of entropy)
//   Attacker must guess: 2^27 = 134 million possibilities on 64-bit
//
// Return-to-libc: overflow -> overwrite RET -> jump to system()
//   Without ASLR: system() at 0x7ffff7a42440 (known)
//   With ASLR:    system() at ??? + 0x42440 (base unknown)
//
// Bypass technique: Information leak
//   1. Find a bug that prints a pointer (format string, use-after-free)
//   2. Leaked pointer reveals library base address
//   3. Calculate all function addresses from the base
//   4. Now exploit as if ASLR were off
//
// For full protection, need ASLR + PIE (Position-Independent Executable)
// Compile: g++ -fPIE -pie prog.cpp

int main() {
    function_on_stack();
    // Run twice:
    // First:  Stack variable: 0x7ffd3a8b1234
    // Second: Stack variable: 0x7fff129c5678  (different!)
}
```

ASLR randomizes the base addresses of the stack, heap, shared libraries, and (with PIE) the executable itself. This means the addresses of functions, variables, and gadgets differ between runs - an attacker with a buffer overflow can redirect control flow but doesn't know WHERE to redirect it. Bypassing requires an information leak to determine the randomized base address. 64-bit ASLR provides 27+ bits of entropy, making brute-force infeasible.

### Q3: Verify that your binary has NX (non-executable stack) set using readelf -l or checksec

**Answer:**

Knowing how to verify your mitigations are actually active is as important as knowing how to enable them. Here's how to check on each platform, plus the complete set of recommended build flags:

```cpp
// NX (No-eXecute) / DEP (Data Execution Prevention) / W^X (Write XOR Execute)
//
// Verification commands:
//
// Method 1: readelf -l
//
//   $ readelf -l a.out | grep -i stack
//     GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
//                                                                   ^^
//   RW  = Read + Write only         -> NX ENABLED (good!)
//   RWE = Read + Write + Execute    -> NX DISABLED (bad!)
//
//   The GNU_STACK segment tells the kernel what permissions the stack should have.
//   Modern compilers default to RW (no execute).
//
// Method 2: checksec
//
//   $ checksec --file=a.out
//   RELRO           STACK CANARY      NX            PIE
//   Full RELRO      Canary found      NX enabled    PIE enabled
//                                     ^^^^^^^^^^
//   checksec reports all mitigations at once.
//   Install: pip install checksec.py  or  apt install checksec
//
// Method 3: scanelf (Gentoo hardened-sources)
//
//   $ scanelf -e a.out
//   TYPE   STK/REL/PTL  FILE
//   ET_DYN RW- R-- RW-  a.out
//          ^^^
//   RW- = NX enabled (no X)
//
// How NX works
//
// x86-64 page table entry (PTE):
//   Bit 63 (NX bit): 0 = executable, 1 = non-executable
//
//   Stack pages:  NX=1 (data, not code)
//   Heap pages:   NX=1 (data, not code)
//   .text pages:  NX=0, Write=0 (code, not writable)
//
// When CPU fetches an instruction from a page with NX=1:
//   -> Hardware exception -> OS terminates process with SIGSEGV
//
// Effect on attacks:
//   Classic shellcode on stack -> BLOCKED (stack is NX)
//   JIT spray on heap         -> BLOCKED (heap is NX)
//   ROP (return-oriented prog) -> NOT blocked (uses existing .text gadgets)

#include <iostream>
#include <cstdlib>

int main() {
    std::cout << "NX/DEP verification commands:\n\n";

    std::cout << "Linux:\n";
    std::cout << "  readelf -l a.out | grep STACK\n";
    std::cout << "  checksec --file=a.out\n\n";

    std::cout << "Windows:\n";
    std::cout << "  dumpbin /headers a.exe | findstr \"NX\"\n";
    std::cout << "  Process Explorer -> Properties -> DEP\n\n";

    std::cout << "macOS:\n";
    std::cout << "  codesign -dvvv a.out 2>&1 | grep flags\n\n";

    // Complete recommended build flags:
    std::cout << "Recommended compile flags for all mitigations:\n";
    std::cout << "  g++ -std=c++20 -O2 \\\n";
    std::cout << "    -fstack-protector-strong \\\n";   // canaries
    std::cout << "    -D_FORTIFY_SOURCE=2 \\\n";        // bounds checks
    std::cout << "    -fPIE -pie \\\n";                  // ASLR for executable
    std::cout << "    -Wl,-z,relro,-z,now \\\n";         // RELRO (GOT protection)
    std::cout << "    -Wl,-z,noexecstack \\\n";          // NX stack
    std::cout << "    -fcf-protection=full \\\n";         // Intel CET
    std::cout << "    -Wall -Wextra -Werror\n";
}
```

The `GNU_STACK` program header in ELF binaries specifies stack permissions. `RW` means read+write only (NX enabled - good). `RWE` means the stack is executable (NX disabled - vulnerable). `readelf -l` reads this header directly. `checksec` is a convenience tool that checks all mitigations (canary, NX, ASLR/PIE, RELRO) in one command. NX is enabled by default in modern compilers, but inline assembly or certain legacy libraries can disable it.

---

## Notes

- **Stack canaries** are defeated by information leaks (read canary before overwriting) or by overwriting variables without crossing the canary (e.g., adjacent variable overwrites).
- **ASLR entropy:** 64-bit Linux provides ~28 bits for stack, ~28 for mmap, ~23 for heap. 32-bit has only ~8-16 bits - brute-forceable.
- **NX + JIT:** JIT compilers (V8, LuaJIT) must create RWX pages. These are prime targets - prefer W^X with mprotect toggles.
- **`-fstack-protector-strong`** is the recommended level: protects functions with arrays, address-taken locals, and unions - balances coverage vs overhead.
- **`-Wl,-z,relro,-z,now`** (RELRO) makes the GOT (Global Offset Table) read-only after loading, preventing GOT overwrite attacks.
- All three mitigations are free or near-free in performance. Enable them always.
