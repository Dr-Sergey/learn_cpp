# Use compiler explorer (Godbolt) to inspect generated assembly

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://godbolt.org>  

---

## Topic Overview

Compiler Explorer at godbolt.org is a browser-based tool that compiles your C++ code and shows you the assembly it produces, in real time, for any compiler and optimization level you choose. If you have ever wondered whether a particular abstraction truly has zero overhead, or whether the optimizer is actually vectorizing your loop, Godbolt is the fastest way to get an honest answer.

The interface is split into two panes - C++ source on the left, assembly output on the right - and hovering a source line highlights the corresponding assembly instructions so you can see exactly what each piece of code produces.

```cpp
https://godbolt.org
┌────────────────────┬───────────────────────┐
│  C++ Source          │  Assembly Output         │
│                      │                           │
│  int square(int n) { │  square(int):              │
│    return n * n;      │    imul edi, edi           │
│  }                    │    mov  eax, edi           │
│                      │    ret                     │
└────────────────────┴───────────────────────┘
```

Three instructions. That is what the optimizer turns `n * n` into. No function call, no stack frame, just a multiply and a return. That kind of confirmation is what makes Godbolt so useful.

---

## Self-Assessment

### Q1: Verify zero-overhead abstraction - C++ class vs plain C

One of C++'s core promises is that abstractions like classes, constructors, and accessors compile away when you do not need the runtime machinery. Here is a direct comparison you can paste into Godbolt to verify that promise with your own eyes.

```cpp
// Paste both in Godbolt with -O2:

// C version:
struct Point_C { int x, y; };
int distance_sq_c(struct Point_C p) {
    return p.x * p.x + p.y * p.y;
}

// C++ version with class:
class Point {
    int x_, y_;
public:
    Point(int x, int y) : x_(x), y_(y) {}
    int x() const { return x_; }
    int y() const { return y_; }
    int distance_sq() const { return x_ * x_ + y_ * y_; }
};

int cpp_distance(int x, int y) {
    Point p(x, y);
    return p.distance_sq();
}
```

Godbolt assembly at `-O2` (x86-64 GCC):

```asm
; distance_sq_c:              cpp_distance:
  imul edi, edi              ; imul edi, edi
  imul esi, esi              ; imul esi, esi
  add  eax, edi              ; add  eax, edi
  add  eax, esi              ; add  eax, esi
  ret                        ; ret
; IDENTICAL!  Zero overhead confirmed.
```

The constructor, the accessor methods, the encapsulation - all of it disappears at `-O2`. The resulting machine code is identical to the plain C version. This is the zero-overhead principle in action, and Godbolt lets you check it for your own types whenever you are unsure.

### Q2: Compare optimization levels for a simple loop

This example shows how dramatically the optimizer transforms a trivial loop as you raise the optimization level. Try each flag in Godbolt and watch the assembly change.

```cpp
// Paste in Godbolt, compare -O0, -O2, -O3:
int sum_array(const int* arr, int n) {
    int total = 0;
    for (int i = 0; i < n; ++i)
        total += arr[i];
    return total;
}
```

| Level | Assembly Characteristics |
| --- | --- |
| `-O0` | Stores `total` and `i` on stack; loads/stores every iteration; ~15 instructions per iteration |
| `-O2` | Keeps `total` in register; loop uses `add` + `cmp` + `jl`; ~4 instructions per iteration |
| `-O3` | Vectorized: uses `paddd` (SSE) or `vpaddd` (AVX) to add 4/8 ints at once; scalar cleanup loop for remainder |

```asm
; -O0 (no optimization): slow, readable
sum_array:
    mov DWORD PTR [rbp-4], 0    ; total = 0
    mov DWORD PTR [rbp-8], 0    ; i = 0
.L2:
    cmp DWORD PTR [rbp-8], esi  ; if (i >= n)
    jge .L3                      ; goto end
    mov eax, DWORD PTR [rbp-8]
    cdqe
    mov eax, DWORD PTR [rdi+rax*4]  ; arr[i]
    add DWORD PTR [rbp-4], eax      ; total += arr[i]
    add DWORD PTR [rbp-8], 1        ; i++
    jmp .L2

; -O2 (standard): registers, no memory traffic
sum_array:
    xor eax, eax                ; total = 0
.L2:
    add eax, DWORD PTR [rdi]    ; total += *arr
    add rdi, 4                  ; arr++
    dec esi                     ; n--
    jne .L2                     ; loop
    ret

; -O3 (aggressive): SIMD vectorized
sum_array:
    ; ... setup ...
    pxor xmm0, xmm0            ; zero vector accumulator
.L3:
    paddd xmm0, XMMWORD PTR [rdi+rax]  ; add 4 ints at once!
    add rax, 16
    cmp rax, rcx
    jne .L3
    ; ... horizontal sum + scalar cleanup ...
```

Notice how `-O0` spills both `total` and `i` to the stack on every iteration. At `-O2`, the compiler figures out that it can keep `total` in a register the whole time - no memory traffic in the hot path. At `-O3`, it goes further and rewrites the loop to process multiple elements per iteration using SIMD instructions. Same source code, three very different performance profiles.

### Q3: Use diff view to compare two implementations

Godbolt has a built-in diff mode: click "Add new..." and select "Diff" to view two compiler outputs side by side. This is useful for settling questions like "does `std::unique_ptr` really cost nothing compared to a raw pointer?"

```cpp
// Implementation A: std::unique_ptr
#include <memory>
int use_unique() {
    auto p = std::make_unique<int>(42);
    return *p;
}

// Implementation B: raw pointer
int use_raw() {
    int* p = new int(42);
    int val = *p;
    delete p;
    return val;
}
```

At `-O2`, both generate **identical** assembly:

```asm
; Both become:
    mov edi, 4
    call operator new(unsigned long)
    mov DWORD PTR [rax], 42
    mov eax, 42           ; return value known at compile time!
    ; ... delete call ...
    ret
```

The optimizer even figures out that the return value is 42 at compile time and bakes that into the code. No vtable, no reference counting, no overhead from `unique_ptr` at all. The diff view makes it trivially easy to confirm results like this.

Godbolt diff tips:

- Green lines = present in left only
- Red lines = present in right only
- White lines = identical
- Use short links (Share button) to save and share comparisons

---

## Notes

- Use `__attribute__((noinline))` or `[[gnu::noinline]]` to prevent a function from being inlined when you want to inspect it in isolation - otherwise the optimizer may absorb it into its caller and you will not see it in the output at all.
- Add `-march=native` to see SIMD instructions for your specific CPU architecture.
- Use the "Opt Pipeline" view to see which optimization passes ran and in what order - useful when you want to understand why something was or was not optimized.
- Godbolt supports 100+ compilers including GCC, Clang, MSVC, and Intel ICC, so you can compare how the same code is compiled across toolchains.
- Use the "Libraries" dropdown to add Boost, Abseil, fmt, and other popular libraries without any local setup.
