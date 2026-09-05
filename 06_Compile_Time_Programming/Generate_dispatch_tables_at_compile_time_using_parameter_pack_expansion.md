# Generate Dispatch Tables at Compile Time Using Parameter Pack Expansion

**Category:** Compile Time Programming  
**Standard:** C++11, C++17 (parameter packs)  
**Reference:** <https://en.cppreference.com/w/cpp/language/constexpr>  

---

## Topic Overview

### What Is a Compile-Time Dispatch Table

A **dispatch table** is an array of function pointers indexed by an enum or integer. Instead of a `switch` statement that the compiler might or might not turn into a jump table, you explicitly build the array yourself and let indexing do the work. When you build it with `constexpr` and parameter pack expansion, you get **O(1) dispatch with zero runtime setup cost** - the table lives in the read-only data segment and is ready before `main` runs.

The practical difference between a switch and a dispatch table looks like this:

```cpp
// Instead of:
switch (op) {
    case Op::Add: return add(a, b);
    case Op::Sub: return sub(a, b);
    case Op::Mul: return mul(a, b);
}

// Use:
constexpr auto table = make_dispatch_table<add, sub, mul>();
return table[static_cast<int>(op)](a, b);  // O(1) indexed lookup
```

### Advantages

The switch statement looks simpler to write but has a hidden fragility: for large or sparse enums the compiler may generate cascading comparisons instead of a true jump table. The dispatch table makes the behavior explicit and guaranteed.

| Feature | Switch Statement | Dispatch Table |
| --- | --- | --- |
| Lookup time | O(n) or jump table | Always O(1) - array index |
| Adding cases | Edit switch | Just add to pack |
| Generated code | Depends on compiler | Guaranteed array lookup |
| Code size | Grows with cases | Fixed (one array + index) |

---

## Self-Assessment

### Q1: Generate a `constexpr` array of function pointers indexed by an enum using a parameter pack expansion

The trick here is the `make_dispatch_table` template that takes function pointers as non-type template arguments and packs them into a `std::array` using pack expansion (`{Ops...}`). The compiler then places the entire array into `.rodata` at compile time.

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// === Operations ===
enum class Op { Add, Sub, Mul, Div, Count };

int add_fn(int a, int b) { return a + b; }
int sub_fn(int a, int b) { return a - b; }
int mul_fn(int a, int b) { return a * b; }
int div_fn(int a, int b) { return b != 0 ? a / b : 0; }

// === Function pointer type ===
using BinaryOp = int(*)(int, int);

// === Generate dispatch table from parameter pack ===
template <BinaryOp... Ops>
constexpr auto make_dispatch_table() {
    return std::array<BinaryOp, sizeof...(Ops)>{Ops...};
}

// === Build at compile time ===
constexpr auto dispatch_table = make_dispatch_table<add_fn, sub_fn, mul_fn, div_fn>();

// Verify at compile time
static_assert(dispatch_table.size() == static_cast<std::size_t>(Op::Count));

// === Dispatch function ===
int dispatch(Op op, int a, int b) {
    return dispatch_table[static_cast<std::size_t>(op)](a, b);
}

// === Alternative: generate with index_sequence for automatic enum mapping ===
template <std::size_t... Is>
constexpr auto make_table_impl(std::index_sequence<Is...>) {
    constexpr BinaryOp handlers[] = {add_fn, sub_fn, mul_fn, div_fn};
    return std::array<BinaryOp, sizeof...(Is)>{handlers[Is]...};
}

constexpr auto dispatch_table_v2 =
    make_table_impl(std::make_index_sequence<static_cast<std::size_t>(Op::Count)>{});

int main() {
    std::cout << "=== Compile-Time Dispatch Table ===\n\n";

    int a = 20, b = 5;
    std::cout << "Add: " << dispatch(Op::Add, a, b) << "\n";  // 25
    std::cout << "Sub: " << dispatch(Op::Sub, a, b) << "\n";  // 15
    std::cout << "Mul: " << dispatch(Op::Mul, a, b) << "\n";  // 100
    std::cout << "Div: " << dispatch(Op::Div, a, b) << "\n";  // 4

    // Looping over all operations:
    const char* names[] = {"Add", "Sub", "Mul", "Div"};
    for (int i = 0; i < static_cast<int>(Op::Count); ++i) {
        std::cout << names[i] << "(10, 3) = "
                  << dispatch_table[i](10, 3) << "\n";
    }

    return 0;
}
```

**Expected output:**

```text
=== Compile-Time Dispatch Table ===

Add: 25
Sub: 15
Mul: 100
Div: 4
Add(10, 3) = 13
Sub(10, 3) = 7
Mul(10, 3) = 30
Div(10, 3) = 3
```

### Q2: Show that the dispatch table has zero runtime overhead compared to a `switch` statement

This benchmark exercises both a `switch` and a dispatch table for 10 million iterations so you can see that the generated code is equivalent in practice. More importantly, the code also explains *when* the difference matters most.

```cpp
#include <iostream>
#include <array>
#include <chrono>

// === Setup ===
enum class Color { Red, Green, Blue, Count };

const char* name_switch(Color c) {
    switch (c) {
        case Color::Red:   return "Red";
        case Color::Green: return "Green";
        case Color::Blue:  return "Blue";
        default:           return "?";
    }
}

using NameFn = const char*(*)();

const char* red_name()   { return "Red"; }
const char* green_name() { return "Green"; }
const char* blue_name()  { return "Blue"; }

constexpr std::array<NameFn, 3> name_table = {red_name, green_name, blue_name};

const char* name_dispatch(Color c) {
    return name_table[static_cast<int>(c)]();
}

int main() {
    constexpr int N = 10'000'000;

    // Benchmark switch
    auto t1 = std::chrono::high_resolution_clock::now();
    volatile const char* result;
    for (int i = 0; i < N; ++i) {
        result = name_switch(static_cast<Color>(i % 3));
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark dispatch table
    for (int i = 0; i < N; ++i) {
        result = name_dispatch(static_cast<Color>(i % 3));
    }
    auto t3 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "Switch:         " << ms(t2 - t1).count() << " ms\n";
    std::cout << "Dispatch table: " << ms(t3 - t2).count() << " ms\n";

    std::cout << "\n=== Assembly Comparison (with -O2) ===\n";
    std::cout << "Switch (small): compiler generates a jump table -> same as dispatch table\n";
    std::cout << "Switch (large): may generate cascading comparisons -> O(n)\n";
    std::cout << "Dispatch table: always array[index]() -> always O(1)\n";

    std::cout << "\n=== Key Insight ===\n";
    std::cout << "For small enums (<10 cases), the compiler optimizes switch to a\n";
    std::cout << "jump table anyway. For large enums (50+), explicit dispatch tables\n";
    std::cout << "guarantee O(1) behavior and avoid compiler heuristics.\n";

    (void)result;
    return 0;
}
```

### Q3: Explain how `constexpr` function pointer arrays enable O(1) dispatch for large enums

This is where the technique really pays off. With 50+ opcodes, a `switch` is at the mercy of the compiler's heuristics about whether to generate a jump table or a chain of comparisons. The `constexpr` array removes that uncertainty entirely - it's always just an array load plus an indirect call.

Notice the `constexpr` lambda initializer - this is a C++17 pattern for building a `constexpr` array with complex initialization logic (defaulting everything to `handle_nop`, then patching specific entries).

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// === Large enum (50 entries) - switch would be long, dispatch table stays O(1) ===
enum class Opcode : std::size_t {
    NOP, LOAD, STORE, ADD, SUB, MUL, DIV, MOD,
    AND, OR, XOR, NOT, SHL, SHR,
    JMP, JZ, JNZ, CALL, RET,
    PUSH, POP, DUP, SWAP,
    CMP, TEST, INC, DEC,
    // ... imagine 50+ opcodes
    COUNT
};

using Handler = void(*)(int);

// Each handler is trivial for this demo
void handle_nop(int)   { }
void handle_load(int)  { std::cout << "LOAD\n"; }
void handle_store(int) { std::cout << "STORE\n"; }
void handle_add(int)   { std::cout << "ADD\n"; }
void handle_sub(int)   { std::cout << "SUB\n"; }
// ... (abbreviated)

// === Build constexpr table ===
// The table is placed in .rodata (read-only data segment) at compile time
// No runtime initialization needed
constexpr std::array<Handler, static_cast<std::size_t>(Opcode::COUNT)> opcode_handlers = [] {
    std::array<Handler, static_cast<std::size_t>(Opcode::COUNT)> t{};
    // Default all to NOP
    for (auto& h : t) h = handle_nop;
    // Set specific handlers
    t[static_cast<std::size_t>(Opcode::LOAD)]  = handle_load;
    t[static_cast<std::size_t>(Opcode::STORE)] = handle_store;
    t[static_cast<std::size_t>(Opcode::ADD)]   = handle_add;
    t[static_cast<std::size_t>(Opcode::SUB)]   = handle_sub;
    return t;
}();

// O(1) dispatch - always just: table[opcode](arg)
void execute(Opcode op, int arg) {
    opcode_handlers[static_cast<std::size_t>(op)](arg);
}

int main() {
    std::cout << "=== O(1) Dispatch for Large Enums ===\n\n";

    execute(Opcode::LOAD, 42);   // LOAD
    execute(Opcode::ADD, 10);    // ADD
    execute(Opcode::NOP, 0);     // (silent)

    std::cout << "\n=== How It Works ===\n";
    std::cout << "1. constexpr array -> placed in .rodata at compile time\n";
    std::cout << "2. Indexing: table[enum_value] -> O(1) array access\n";
    std::cout << "3. Function pointer call: table[i](arg) -> indirect call\n";
    std::cout << "\n=== vs switch ===\n";
    std::cout << "Switch with 50 cases:\n";
    std::cout << "  - Compiler MAY generate a jump table (not guaranteed)\n";
    std::cout << "  - May fall back to if/else chain -> O(n)\n";
    std::cout << "  - Depends on compiler heuristics (value density, gaps)\n";
    std::cout << "\nDispatch table with 50 entries:\n";
    std::cout << "  - ALWAYS O(1) - it's an array index\n";
    std::cout << "  - Generated at compile time - .rodata section\n";
    std::cout << "  - Predictable: one load + one indirect call\n";
    std::cout << "  - Works regardless of enum value density\n";

    return 0;
}
```

---

## Notes

- `constexpr` dispatch tables are placed in the read-only data segment - no runtime initialization.
- For contiguous enums (0, 1, 2, ...), array indexing is the optimal dispatch strategy.
- For sparse enums, consider a compile-time hash map or `constexpr` binary search.
- Parameter pack expansion (`{Ops...}`) is the cleanest way to initialize the table.
- Use `std::index_sequence` when you need to generate the table programmatically.
- This pattern is heavily used in interpreters, bytecode VMs, protocol parsers, and command processors.
