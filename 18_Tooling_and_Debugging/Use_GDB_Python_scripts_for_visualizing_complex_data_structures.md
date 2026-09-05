# Use GDB Python scripts for visualizing complex data structures

**Category:** Tooling and Debugging  
**Standard:** C++20  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb/Python-API.html>  

---

## Topic Overview

By default, GDB shows your data structures as raw memory: a pointer value, maybe a size, and no readable representation of the contents. For complex types like linked lists, trees, or custom containers, that is nearly useless. GDB's Python API solves this by letting you write scripts that traverse and format your data structures into something human-readable, right inside the debugger.

The API hierarchy you will work with most often is:

```cpp
GDB Python API hierarchy:
  gdb.Value          -- access C++ values from Python
  gdb.Type           -- inspect C++ types
  gdb.Command        -- define custom GDB commands
  gdb.PrettyPrinter  -- custom display for types
  gdb.Breakpoint     -- programmatic breakpoints
```

The most commonly used feature is the pretty-printer. You register a Python class that GDB calls whenever it needs to print a value of your type, and the result appears instead of the raw pointer dump.

---

## Self-Assessment

### Q1: Write a GDB pretty-printer for a custom linked list

Start with the C++ code. Compile it with debug info (`-g -O0`) so GDB has full type information to work with.

```cpp
// linked_list.cpp - compile: g++ -g -O0 -std=c++20 linked_list.cpp -o linked_list
#include <iostream>
#include <string>

struct Node {
    int data;
    Node* next;
    Node(int d, Node* n = nullptr) : data(d), next(n) {}
};

struct LinkedList {
    Node* head;
    int size;

    LinkedList() : head(nullptr), size(0) {}

    void push_front(int val) {
        head = new Node(val, head);
        ++size;
    }

    ~LinkedList() {
        while (head) {
            Node* tmp = head;
            head = head->next;
            delete tmp;
        }
    }
};

int main() {
    LinkedList list;
    list.push_front(30);
    list.push_front(20);
    list.push_front(10);
    // list: 10 -> 20 -> 30 -> null
    std::cout << "head = " << list.head->data << '\n';
    return 0;  // breakpoint here
}
```

Now write the pretty-printer in Python. The `to_string` method provides the summary line, and `children` yields the individual elements as `(name, value)` pairs. Notice the cycle detection - without it, a corrupted list would loop forever.

```python
# linked_list_printer.py
import gdb

class LinkedListPrinter:
    """Pretty-printer for LinkedList"""
    def __init__(self, val):
        self.val = val

    def to_string(self):
        size = int(self.val['size'])
        return f'LinkedList of size {size}'

    def children(self):
        """Yield (name, value) pairs for each node"""
        node = self.val['head']
        idx = 0
        seen = set()  # cycle detection
        while node != 0:  # nullptr check
            addr = int(node)
            if addr in seen:
                yield (f'[{idx}]', '<CYCLE DETECTED>')
                break
            seen.add(addr)
            yield (f'[{idx}]', node.dereference()['data'])
            node = node.dereference()['next']
            idx += 1

    def display_hint(self):
        return 'array'  # display as indexed array

class NodePrinter:
    def __init__(self, val):
        self.val = val
    def to_string(self):
        data = self.val['data']
        nxt = self.val['next']
        if nxt == 0:
            return f'Node({data}) -> null'
        return f'Node({data}) -> ...'

def lookup(val):
    t = str(val.type.strip_typedefs())
    if t == 'LinkedList':
        return LinkedListPrinter(val)
    if t == 'Node':
        return NodePrinter(val)
    return None

gdb.pretty_printers.append(lookup)
```

Load the printer and run the program in GDB to see the difference:

```bash
$ gdb ./linked_list
(gdb) source linked_list_printer.py
(gdb) break 38   # return 0
(gdb) run
(gdb) print list
# $1 = LinkedList of size 3 = {[0] = 10, [1] = 20, [2] = 30}
#
# Without the printer it would show:
# $1 = {head = 0x5555...60, size = 3}  <-- just pointers
```

The difference is dramatic. Instead of an opaque pointer address, you see the entire list contents at a glance.

### Q2: Load pretty-printer in `.gdbinit` and verify type matching

Typing `source printer.py` every GDB session quickly becomes annoying. The `.gdbinit` file is the right place to automate this. GDB requires that auto-load paths be explicitly trusted as a security measure, so you need to declare them safe first.

```bash
# ~/.gdbinit (or project-local .gdbinit):

# Auto-load project printers:
add-auto-load-safe-path /home/user/projects/myapp/.gdbinit
source /home/user/projects/myapp/gdb_printers/linked_list_printer.py

# Verify printers are loaded:
(gdb) info pretty-printer
# global pretty-printers:
#   builtin
#     mpx_bound128
#   objfile /home/user/projects/myapp/linked_list printers:
#     LinkedListPrinter
#     NodePrinter

# Test type matching:
(gdb) print list
# LinkedList of size 3 = {[0] = 10, [1] = 20, [2] = 30}  <-- printer active

# For auto-loading per-objfile printers, create:
#   <objfile>-gdb.py  (e.g., linked_list-gdb.py)
# GDB loads it automatically when the binary is loaded
```

The `info pretty-printer` command is your first debugging step when a printer is not behaving as expected. It shows you exactly which printers are registered and what their names are.

### Q3: STL pretty-printers displaying std::map in GDB

Modern GDB ships with libstdc++ pretty-printers pre-installed, so standard containers like `std::map` and `std::vector` already display nicely without any extra setup on your part. Here is what you get:

```cpp
// stl_demo.cpp - compile: g++ -g -O0 -std=c++20 stl_demo.cpp -o stl_demo
#include <map>
#include <string>
#include <vector>
#include <set>

int main() {
    std::map<std::string, std::vector<int>> courses = {
        {"math", {90, 85, 92}},
        {"physics", {88, 76}},
        {"cs", {95, 100, 98, 97}}
    };

    std::set<int> primes = {2, 3, 5, 7, 11, 13};

    return 0;  // breakpoint
}
```

```bash
$ gdb ./stl_demo
(gdb) break 14
(gdb) run

# libstdc++ pretty-printers auto-loaded:
(gdb) print courses
# $1 = std::map with 3 elements = {
#   ["cs"] = std::vector of length 4 = {95, 100, 98, 97},
#   ["math"] = std::vector of length 3 = {90, 85, 92},
#   ["physics"] = std::vector of length 2 = {88, 76}
# }

(gdb) print primes
# $2 = std::set with 6 elements = {2, 3, 5, 7, 11, 13}

# Access individual elements:
(gdb) print courses["math"]
# Not directly supported. Use:
(gdb) python
import gdb
courses = gdb.parse_and_eval('courses')
# Iterate via the pretty-printer's children() method
end
```

The output is immediately readable without knowing anything about the internal tree structure of `std::map`. When you write your own containers, you can give them the same treatment by following exactly this pattern.

---

## Notes

- Custom GDB commands: subclass `gdb.Command` for project-specific debugging.
- Use `gdb.events.stop.connect(callback)` to run Python on every breakpoint hit.
- Cycle detection is important for linked structures to avoid infinite loops.
- `display_hint()` returning `'array'`, `'map'`, or `'string'` changes display format.
- GDB Python scripts work in VS Code's debug console too.
