# Use gdb pretty-printers for STL types

**Category:** Tooling and Debugging  
**Standard:** C++23  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb/Pretty-Printing.html>  

---

## Topic Overview

If you have ever tried to inspect a `std::map` or a `std::vector` in GDB without pretty-printers, you know the experience: instead of the contents of your container, you get the raw internal representation - red-black tree node pointers, allocator state, and dozens of fields that tell you nothing about what is actually stored. GDB pretty-printers solve this by translating those internal representations into something a human can actually read.

```text
Without pretty-printers:                With pretty-printers:
(gdb) print my_map                      (gdb) print my_map
$1 = {                                  $1 = std::map with 3 elements = {
  _M_t = {                                [1] = "hello",
    _M_impl = {                            [2] = "world",
      _M_header = {                        [42] = "answer"
        _M_color = 0,                    }
        _M_parent = 0x55...,
        _M_left = 0x55...,                 ^^ READABLE!
        ...20 more lines...
```

The pretty-printer for `std::map` knows how to walk the red-black tree and display the key-value pairs. The libstdc++ pretty-printers for GCC ship with the compiler and load automatically in most configurations, so you may already have this working without realizing it.

---

## Self-Assessment

### Q1: Load libstdc++ pretty-printers and print std::map and std::variant

The libstdc++ pretty-printers cover the full range of STL types, including newer additions like `std::variant` and `std::optional`. Here is a program that exercises several of them so you can see what the output looks like in a debugger session.

```cpp
// compile: g++ -g -O0 -std=c++23 pretty_demo.cpp -o pretty_demo
#include <map>
#include <variant>
#include <string>
#include <vector>
#include <optional>

int main() {
    std::map<int, std::string> scores = {
        {1, "Alice"}, {2, "Bob"}, {3, "Charlie"}
    };

    std::variant<int, double, std::string> v1 = 42;
    std::variant<int, double, std::string> v2 = 3.14;
    std::variant<int, double, std::string> v3 = std::string("hello");

    std::optional<int> opt_val = 7;
    std::optional<int> opt_empty;

    std::vector<std::vector<int>> nested = {{1, 2}, {3, 4, 5}};

    return 0;  // Set breakpoint here
}
```

Run the program under GDB and step to the breakpoint, then inspect each variable:

```bash
# GDB session with pretty-printers:
$ gdb ./pretty_demo
(gdb) break main
(gdb) run
(gdb) next  # step past initializations
...

# Pretty-printers auto-loaded from libstdc++:
(gdb) print scores
# $1 = std::map with 3 elements = {
#   [1] = "Alice",
#   [2] = "Bob",
#   [3] = "Charlie"
# }

(gdb) print v1
# $2 = std::variant<int, double, std::string> [index 0] = {42}

(gdb) print v2
# $3 = std::variant<int, double, std::string> [index 1] = {3.14}

(gdb) print v3
# $4 = std::variant<int, double, std::string> [index 2] = {"hello"}

(gdb) print opt_val
# $5 = std::optional<int> = {7}

(gdb) print opt_empty
# $6 = std::optional<int> [no value]

(gdb) print nested
# $7 = std::vector of length 2 = {
#   std::vector of length 2 = {1, 2},
#   std::vector of length 3 = {3, 4, 5}
# }

# If pretty-printers aren't loading automatically:
(gdb) python
import sys
sys.path.insert(0, '/usr/share/gcc/python')
from libstdcxx.v6.printers import register_libstdcxx_printers
register_libstdcxx_printers(None)
end
```

Notice how `std::variant` shows the active index alongside the stored value, and `std::optional` tells you plainly when it holds no value. This is vastly more useful than digging through the raw internal storage manually.

### Q2: Write a custom GDB pretty-printer for a user-defined type

The same Python API that powers the libstdc++ printers is available for your own types. Writing a custom printer is straightforward: you provide a class with a `to_string` method that GDB calls whenever it needs to display a value of that type.

```cpp
// my_ptr.cpp
#include <iostream>

template<typename T>
class MyPtr {
    T* raw_;
    int ref_count_;
public:
    explicit MyPtr(T* p) : raw_(p), ref_count_(1) {}
    ~MyPtr() { delete raw_; }
    T& operator*() { return *raw_; }
    int refs() const { return ref_count_; }
};

int main() {
    MyPtr<int> p(new int(42));
    MyPtr<std::string> s(new std::string("hello"));
    return 0;  // breakpoint here
}
```

The Python printer script reads the raw fields from GDB's value representation and formats them into a human-friendly string:

```python
# my_pretty_printers.py - GDB Python pretty-printer
import gdb
import re

class MyPtrPrinter:
    """Pretty-printer for MyPtr<T>"""
    def __init__(self, val):
        self.val = val

    def to_string(self):
        raw = self.val['raw_']
        refs = self.val['ref_count_']
        try:
            pointed = raw.dereference()
            return f'MyPtr(refs={refs}) -> {pointed}'
        except gdb.MemoryError:
            return f'MyPtr(refs={refs}) -> <null>'

def my_lookup(val):
    type_name = str(val.type.strip_typedefs())
    if re.match(r'^MyPtr<.*>$', type_name):
        return MyPtrPrinter(val)
    return None

gdb.pretty_printers.append(my_lookup)
```

Loading the printer and using it in a session:

```bash
# Load in GDB:
(gdb) source my_pretty_printers.py
(gdb) print p
# $1 = MyPtr(refs=1) -> 42
(gdb) print s
# $2 = MyPtr(refs=1) -> "hello"

# Auto-load via .gdbinit:
# echo "source /path/to/my_pretty_printers.py" >> ~/.gdbinit
```

Adding it to `~/.gdbinit` means it loads every time GDB starts, so you never have to source it manually again.

### Q3: Show STL container without pretty-printers

It is worth seeing the raw output at least once. If you ever need to inspect the actual internal structure of a container - perhaps to understand a memory layout or debug a corrupted node - you can disable pretty-printers temporarily.

```bash
# Disable pretty-printers to see raw internals:
(gdb) disable pretty-printer
(gdb) print scores
# $1 = {
#   _M_t = {
#     _M_impl = {
#       _M_key_compare = {<std::binary_function<int, int, bool>> = {<No data fields>}},
#       _M_header = {
#         _M_color = std::_S_red,
#         _M_parent = 0x5555555596c0,
#         _M_left = 0x5555555596e0,
#         _M_right = 0x555555559720
#       },
#       _M_node_count = 3
#     }
#   }
# }
# ^^ Completely unreadable! Red-black tree implementation details

# Re-enable:
(gdb) enable pretty-printer
(gdb) print scores
# $2 = std::map with 3 elements = {[1]="Alice", [2]="Bob", [3]="Charlie"}
# ^^ Human readable again

# List all registered pretty-printers:
(gdb) info pretty-printer
# global pretty-printers:
#   libstdc++-v6
#     std::map
#     std::vector
#     std::string
#     std::variant
#
```

The raw output is a reminder of what the pretty-printers are actually doing for you every time you print a standard container. Without them, understanding the state of your program would require knowing the implementation details of every standard library type.

---

## Notes

- libstdc++ pretty-printers ship with GCC and are usually auto-loaded when you start a debugging session. If they do not load, check the Python path as shown in Q1.
- libc++ (Clang's standard library) has its own separate pretty-printers; LLDB handles libc++ types natively without any manual setup.
- Place custom printers in a `gdb` subfolder of your project with an `__init__.py` file to enable GDB's auto-loading mechanism for project-specific types.
- Use `set print pretty on` in GDB for nicely indented output even when pretty-printers are not available - it at least makes the raw struct output easier to scan.
- VS Code's C/C++ extension uses GDB and LLDB pretty-printers to populate the Variables and Watch panes, so the printers you write for the command line also improve your IDE debugging experience.
