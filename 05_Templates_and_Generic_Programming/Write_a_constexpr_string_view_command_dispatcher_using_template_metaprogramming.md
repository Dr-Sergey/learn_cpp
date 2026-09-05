# Write a constexpr string_view Command Dispatcher Using Template Metaprogramming

**Category:** Templates and Generic Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/constexpr>  

---

## Topic Overview

### What Is a Compile-Time Command Dispatcher

A **command dispatcher** maps command names (strings) to handler functions. A **compile-time** dispatcher resolves this mapping at compile time using `constexpr` and template metaprogramming, producing optimal code at runtime (often a series of `if`/`else` comparisons that the compiler can optimize into direct calls or jump tables).

The contrast with the typical runtime approach is worth spelling out. When you use a runtime map, every lookup costs a hash computation plus a pointer dereference into heap memory. The compile-time version sidesteps all of that:

```cpp
// Runtime approach (unordered_map):
std::unordered_map<std::string, Handler> commands;
commands["help"] = &handle_help;
commands[cmd]();  // hash + lookup + indirect call at runtime

// Compile-time approach:
constexpr auto dispatch = make_dispatcher(
    command<"help">(handle_help),
    command<"quit">(handle_quit)
);
dispatch("help");  // resolved at compile time -> direct call
```

The compile-time version wins on cache locality (the table lives in `.rodata`), startup cost (nothing to build at runtime), and correctness (the compiler can verify command names exist).

### Key Building Blocks

Here is a quick map of the pieces you need to put this together:

| Component | Purpose |
| --- | --- |
| `constexpr` functions | Enable compile-time computation |
| `std::string_view` | Lightweight, `constexpr`-friendly string reference |
| NTTPs (C++20) | String literals as template parameters |
| `std::array` | `constexpr`-friendly fixed-size container |
| `if constexpr` | Compile-time branching |
| Fold expressions | Iterate over parameter packs |

---

## Self-Assessment

### Q1: Build a compile-time map from `string_view` command names to handler function pointers

The key idea here is that `std::string_view` supports `constexpr` comparison since C++17, so you can store the mapping in a `constexpr std::array` and iterate over it in a `constexpr` function. The `make_dispatcher` helper deduces the array size from the number of entries you pass in.

```cpp
#include <iostream>
#include <string_view>
#include <array>
#include <utility>
#include <stdexcept>

// === Compile-time command entry ===
struct CommandEntry {
    std::string_view name;
    void (*handler)();
};

// === Compile-time command dispatcher ===
template <std::size_t N>
class CommandDispatcher {
    std::array<CommandEntry, N> commands_;
public:
    constexpr CommandDispatcher(std::array<CommandEntry, N> cmds) : commands_(cmds) {}

    constexpr void dispatch(std::string_view cmd) const {
        for (const auto& entry : commands_) {
            if (entry.name == cmd) {
                entry.handler();
                return;
            }
        }
        throw std::runtime_error("Unknown command");
    }

    // Check if a command exists (can be used at compile time)
    constexpr bool has_command(std::string_view cmd) const {
        for (const auto& entry : commands_) {
            if (entry.name == cmd) return true;
        }
        return false;
    }
};

// === Helper to create dispatcher with deduced size ===
template <typename... Entries>
constexpr auto make_dispatcher(Entries... entries) {
    return CommandDispatcher<sizeof...(Entries)>{
        std::array<CommandEntry, sizeof...(Entries)>{entries...}
    };
}

// === Handler functions ===
void handle_help()  { std::cout << "Help: Available commands: help, status, quit\n"; }
void handle_status(){ std::cout << "Status: All systems operational\n"; }
void handle_quit()  { std::cout << "Quitting...\n"; }

// === Build the dispatcher at compile time ===
constexpr auto dispatcher = make_dispatcher(
    CommandEntry{"help",   &handle_help},
    CommandEntry{"status", &handle_status},
    CommandEntry{"quit",   &handle_quit}
);

// === Compile-time verification ===
static_assert(dispatcher.has_command("help"),   "help must exist");
static_assert(dispatcher.has_command("quit"),   "quit must exist");
static_assert(!dispatcher.has_command("foo"),   "foo must not exist");

int main() {
    std::cout << "=== Compile-Time Command Dispatcher ===\n\n";

    dispatcher.dispatch("help");
    dispatcher.dispatch("status");
    dispatcher.dispatch("quit");

    // Dispatching from user input at runtime also works:
    std::string_view cmd = "status";
    dispatcher.dispatch(cmd);

    return 0;
}
```

Notice the `static_assert` calls at namespace scope. Those fire at compile time with zero runtime cost - they are your first line of defence against typos in command names.

**Expected output:**

```text
=== Compile-Time Command Dispatcher ===

Help: Available commands: help, status, quit
Status: All systems operational
Quitting...
Status: All systems operational
```

### Q2: Show that an invalid command name at compile time triggers a `static_assert`

When you promote command lookup to a template parameter (using a `constexpr std::string_view` as a non-type template argument in C++20), the compiler can check at instantiation time whether the name exists. Pass an unregistered name and you get a build error, not a runtime exception.

```cpp
#include <iostream>
#include <string_view>
#include <array>
#include <type_traits>

// === Compile-time command registry with static_assert validation ===
struct CmdEntry {
    std::string_view name;
    void (*handler)();
};

template <std::size_t N>
struct CmdRegistry {
    std::array<CmdEntry, N> entries;

    constexpr bool contains(std::string_view name) const {
        for (const auto& e : entries) {
            if (e.name == name) return true;
        }
        return false;
    }

    constexpr void (*lookup(std::string_view name) const)() {
        for (const auto& e : entries) {
            if (e.name == name) return e.handler;
        }
        return nullptr;
    }
};

void do_open()  { std::cout << "Opening...\n"; }
void do_close() { std::cout << "Closing...\n"; }
void do_save()  { std::cout << "Saving...\n"; }

constexpr CmdRegistry<3> registry{{
    CmdEntry{"open",  &do_open},
    CmdEntry{"close", &do_close},
    CmdEntry{"save",  &do_save}
}};

// === Validated dispatch: fails at compile time for unknown commands ===
template <std::string_view const& Cmd>
void validated_dispatch() {
    static_assert(registry.contains(Cmd),
                  "ERROR: Unknown command! Check spelling.");
    registry.lookup(Cmd)();
}

// Command names must be constexpr to use as NTTP (C++20):
constexpr std::string_view cmd_open  = "open";
constexpr std::string_view cmd_save  = "save";
// constexpr std::string_view cmd_bad = "delete";  // uncommenting below causes compile error

int main() {
    std::cout << "=== Compile-time validated dispatch ===\n";

    validated_dispatch<cmd_open>();   // OK: "open" is registered
    validated_dispatch<cmd_save>();   // OK: "save" is registered

    // Uncommenting the next line causes a static_assert failure:
    // validated_dispatch<cmd_bad>();
    // Error: "ERROR: Unknown command! Check spelling."

    // You can also validate at compile time without calling:
    static_assert(registry.contains("open"),  "open must exist");
    static_assert(registry.contains("close"), "close must exist");
    // static_assert(registry.contains("delete"), "delete must exist");  // FAILS!

    return 0;
}
```

The reason this is valuable is that a typo in a runtime string only blows up when someone runs the bad path. Here the compiler catches it the moment you build.

### Q3: Explain the performance advantage of compile-time dispatch vs runtime `unordered_map` lookup

If the table feels like a lot, the short version is: the compile-time approach does no heap work at all, and the loop over a small contiguous array is extremely cache-friendly. Here is the full picture:

| Aspect | Compile-Time Dispatch | Runtime `unordered_map` |
| --- | --- | --- |
| **Lookup cost** | Zero (resolved at compile time) or O(n) with small n and branch prediction | O(1) amortized but with hash computation |
| **Memory overhead** | No heap allocation, data in `.rodata` | Heap-allocated buckets, nodes, strings |
| **Startup cost** | None - everything is initialized at compile time | Must populate map at startup |
| **Cache behavior** | Excellent - contiguous data, predictable branches | Poor - pointer chasing through heap nodes |
| **Hash collision** | N/A | Can degrade to O(n) |
| **Indirect calls** | Compiler can inline known handlers | Always indirect through function pointer |
| **Extensibility** | Fixed at compile time (add = recompile) | Can add/remove commands at runtime |

The benchmark below measures both paths so you can see the difference directly.

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
#include <string_view>
#include <array>
#include <chrono>
#include <functional>

// === Runtime approach ===
void rt_help()   { /* work */ }
void rt_status() { /* work */ }
void rt_quit()   { /* work */ }

// === Compile-time approach ===
struct Entry { std::string_view name; void(*fn)(); };

constexpr std::array<Entry, 3> ct_commands{{
    {"help",   &rt_help},
    {"status", &rt_status},
    {"quit",   &rt_quit}
}};

constexpr void ct_dispatch(std::string_view cmd) {
    for (const auto& e : ct_commands) {
        if (e.name == cmd) { e.fn(); return; }
    }
}

int main() {
    // Runtime: unordered_map
    std::unordered_map<std::string, std::function<void()>> rt_map;
    rt_map["help"]   = rt_help;
    rt_map["status"] = rt_status;
    rt_map["quit"]   = rt_quit;

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 1'000'000; ++i) {
        rt_map["status"]();
    }
    auto mid = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 1'000'000; ++i) {
        ct_dispatch("status");
    }
    auto end = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "Runtime unordered_map: " << ms(mid - start).count() << " ms\n";
    std::cout << "Compile-time dispatch: " << ms(end - mid).count() << " ms\n";

    std::cout << "\n=== Why compile-time wins ===\n";
    std::cout << "1. No heap allocation for the dispatch table\n";
    std::cout << "2. No hashing overhead per lookup\n";
    std::cout << "3. Compiler can inline handlers for known commands\n";
    std::cout << "4. Better cache locality (contiguous array vs hash buckets)\n";
    std::cout << "5. static_assert catches typos at compile time\n";

    return 0;
}
```

---

## Notes

- `constexpr` + `std::string_view` enables compile-time string comparison since C++17.
- With C++20 NTTPs, you can use string literals directly as template parameters.
- For small command sets (less than about 20), a linear search through `constexpr std::array` is faster than any hash map.
- The compiler can optimize a series of `constexpr` string comparisons into a jump table.
- Use `static_assert` to catch invalid command names at compile time - zero runtime cost for validation.
- For runtime-extensible systems, use `unordered_map`. For fixed command sets, prefer compile-time dispatch.
