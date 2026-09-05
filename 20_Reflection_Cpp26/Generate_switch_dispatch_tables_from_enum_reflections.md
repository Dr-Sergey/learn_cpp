# Generate switch dispatch tables from enum reflections

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

One of the most tedious maintenance burdens in C++ is keeping a `switch` statement in sync with its enum. Every time you add a new enumerator, you have to remember to update every `switch` that handles it. Miss one, and you get a silent runtime bug.

C++26 reflection fixes this by letting you generate the dispatch logic automatically from the enum definition. Add a new enumerator, and all your reflection-based dispatch code updates itself at the next compile - no manual edits required.

Here is how the approaches compare:

| Approach | Maintenance | Performance |
| --- | --- | --- |
| Hand-written `switch` | Must update for every new enumerator | Compiler optimizes to jump table |
| Reflection dispatch table | Automatic - zero maintenance | Identical - `std::array` lookup is O(1) |
| `std::map<Enum,func>` | Manual registration | O(log N) lookup |

For a dense enum (values 0, 1, 2, ...), the reflection approach compiles down to a simple array lookup - which is exactly what a hand-written switch becomes after optimization:

```cpp
Dense enum:  enum Op { Add=0, Sub=1, Mul=2, Div=3 };

dispatch_table[Op::Mul]  ->  handlers[2]  ->  multiply(a, b)
                             ^-- array index = enum value
```

---

## Self-Assessment

### Q1: Generate a `std::array` of function pointers indexed by enum value

This builds a compile-time array of function pointers where each slot corresponds to one enumerator. The `template for` loop pairs each enumerator's integer value (via `[:e:]` cast) with a handler at the matching index, filling the array at compile time.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <array>
#include <iostream>

enum class Op { Add = 0, Sub = 1, Mul = 2, Div = 3 };

using handler_t = int(*)(int, int);

// Generate dispatch table at compile time:
template <typename E, handler_t... Handlers>
constexpr auto make_dispatch_table() {
    constexpr auto enums = std::meta::enumerators_of(^E);
    constexpr auto N = enums.size();

    // Array indexed by the underlying enum value:
    std::array<handler_t, N> table{};

    // Pair each enumerator with a handler at its index:
    std::array<handler_t, N> handlers_arr = {Handlers...};
    std::size_t idx = 0;
    template for (constexpr auto e : enums) {
        table[static_cast<std::size_t>([:e:])] = handlers_arr[idx++];
    }
    return table;
}

int do_add(int a, int b) { return a + b; }
int do_sub(int a, int b) { return a - b; }
int do_mul(int a, int b) { return a * b; }
int do_div(int a, int b) { return b != 0 ? a / b : 0; }

constexpr auto dispatch = make_dispatch_table<Op, do_add, do_sub, do_mul, do_div>();

int main() {
    int a = 10, b = 3;
    std::cout << "Add: " << dispatch[static_cast<int>(Op::Add)](a, b) << '\n'; // 13
    std::cout << "Mul: " << dispatch[static_cast<int>(Op::Mul)](a, b) << '\n'; // 30
    std::cout << "Div: " << dispatch[static_cast<int>(Op::Div)](a, b) << '\n'; // 3
}
```

The `constexpr auto dispatch = ...` line means the entire table - including populating all four slots - is computed at compile time. At runtime, `dispatch[static_cast<int>(Op::Mul)]` is just an array read followed by a call through the stored function pointer.

### Q2: Eliminate a hand-maintained switch by generating it from reflection

This approach is slightly different: instead of a pre-built array, it generates an inline chain of equality tests via `template for`, which the compiler then optimizes into a jump table. The bonus is that if you add a new `Shape` enumerator and forget to handle it in `ShapeHandler`, the compiler forces you to fix it.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string_view>

enum class Shape { Circle, Rectangle, Triangle, Hexagon };

// Instead of:
//   switch(s) { case Shape::Circle: ... case Shape::Rectangle: ... }
// We generate the dispatch automatically:

template <typename E, typename F>
auto reflected_switch(E value, F&& handler) {
    // template for expands to a chain of if/else at compile time,
    // which the compiler optimizes to a jump table:
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) {
            return handler.template operator()<[:e:]>();
        }
    }
    __builtin_unreachable();
}

struct ShapeHandler {
    template <Shape S>
    double operator()() const {
        if constexpr (S == Shape::Circle)    return 3.14159 * 5 * 5;
        if constexpr (S == Shape::Rectangle) return 4.0 * 6.0;
        if constexpr (S == Shape::Triangle)  return 0.5 * 3.0 * 4.0;
        if constexpr (S == Shape::Hexagon)   return 2.598 * 5 * 5;
    }
};

int main() {
    Shape s = Shape::Triangle;
    double area = reflected_switch(s, ShapeHandler{});
    std::cout << "Area: " << area << '\n';  // Area: 6

    // Adding a new Shape enumerator automatically includes it -
    // the compiler forces you to handle it in ShapeHandler.
}
```

The `handler.template operator()<[:e:]>()` syntax looks intimidating but it just calls the handler's `operator()` with the current enumerator value as a template argument. Each `template for` iteration instantiates a new specialization, so the handler gets the exact compile-time enum value it needs for the `if constexpr` branches.

### Q3: Demonstrate that generated dispatch matches hand-written switch performance

It is worth confirming that all this reflection machinery does not cost you anything at runtime. This example builds a name table using reflection and compares it side-by-side with a classic hand-written switch. At `-O2` they compile to essentially the same assembly.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <array>
#include <iostream>
#include <chrono>

enum class Color { Red = 0, Green = 1, Blue = 2, Yellow = 3 };

// Hand-written switch:
const char* color_name_switch(Color c) {
    switch (c) {
        case Color::Red:    return "Red";
        case Color::Green:  return "Green";
        case Color::Blue:   return "Blue";
        case Color::Yellow: return "Yellow";
    }
    return "?";
}

// Reflection-generated dispatch table:
constexpr auto color_names = []{
    constexpr auto enums = std::meta::enumerators_of(^Color);
    std::array<const char*, enums.size()> names{};
    template for (constexpr auto e : enums) {
        // identifier_of returns string_view; .data() for const char*
        names[static_cast<int>([:e:])] =
            std::meta::identifier_of(e).data();
    }
    return names;
}();

const char* color_name_table(Color c) {
    return color_names[static_cast<int>(c)];
}

int main() {
    // Both produce identical results:
    for (int i = 0; i < 4; ++i) {
        Color c = static_cast<Color>(i);
        std::cout << color_name_switch(c) << " == "
                  << color_name_table(c) << '\n';
    }

    // Performance: both compile to a single indexed memory load.
    // The switch compiles to a jump table; the array IS the table.
    // Assembly (x86-64, -O2):
    //   color_name_switch: jmp [rax*8 + .Ljt]
    //   color_name_table:  mov rax, [rdi*8 + color_names]
    // Both are O(1) with identical throughput.
}
```

The key insight from the assembly comments is that the switch and the array do the same thing at the hardware level - both are a single indexed load. The reflection version's advantage is not performance, it is that you cannot forget to update it when you add a new color.

---

## Notes

- For dense enums (contiguous values starting at 0), array indexing is optimal and produces code identical to a hand-written switch.
- For sparse enums (values like 100, 404, 500), consider a `constexpr` sorted array of pairs with binary search instead of direct indexing.
- The `template for` expansion generates the same control flow as a hand-written switch, and the compiler applies the same jump table optimization.
- Adding a new enumerator automatically updates all reflection-based dispatch - this is the main maintenance win over hand-written code.
- This pattern eliminates entire classes of bugs that come from forgotten switch cases in large enums.
