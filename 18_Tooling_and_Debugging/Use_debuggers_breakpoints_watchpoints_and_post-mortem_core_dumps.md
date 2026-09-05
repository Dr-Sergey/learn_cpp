# Use debuggers: breakpoints, watchpoints, and post-mortem core dumps

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://lldb.llvm.org/use/tutorial.html>  

---

## Topic Overview

A debugger is the most direct way to understand what your program is actually doing at runtime. GDB, LLDB, and WinDbg all let you pause a running program, inspect its state, and step through its execution one instruction at a time. The three features you will reach for most often are:

| Feature | What It Does |
| --- | --- |
| **Breakpoint** | Pauses execution at a line or function |
| **Conditional breakpoint** | Pauses only when a condition is true |
| **Watchpoint** | Pauses when a memory location is read or written |
| **Core dump** | Snapshot of crashed process for post-mortem analysis |

The mental model for breakpoints and watchpoints looks like this:

```cpp
Program flow with breakpoint:
  main() -> init() -> process() ->  [BREAKPOINT]  -> process() -> finish()
                                    |                ^
                                    v                |
                                    (inspect vars,   |
                                     step, continue)--
```

You freeze time at a point of interest, look around, and then let the program continue.

---

## Self-Assessment

### Q1: Set a conditional breakpoint that triggers only when a variable exceeds a threshold

A plain breakpoint on a loop body will pause on every iteration. That is fine for small loops, but for a loop that runs millions of times you want the debugger to stop only when something interesting happens. That is what conditional breakpoints are for.

```cpp
// compile: g++ -g -O0 -std=c++20 breakpoint_demo.cpp -o breakpoint_demo
#include <iostream>
#include <vector>

void process(const std::vector<int>& data) {
    for (size_t i = 0; i < data.size(); ++i) {
        int value = data[i] * 2;  // Line where we set breakpoint
        std::cout << "[" << i << "] = " << value << '\n';
    }
}

int main() {
    std::vector<int> data = {10, 20, 50, 80, 120, 5, 200};
    process(data);
}
```

Setting up and running the conditional breakpoint looks like this:

```bash
# GDB session:
$ gdb ./breakpoint_demo
(gdb) break breakpoint_demo.cpp:7 if value > 100
# Breakpoint 1 at 0x...: file breakpoint_demo.cpp, line 7
(gdb) run
# Starting program: ./breakpoint_demo
# [0] = 20
# [1] = 40
# [2] = 100
# Breakpoint 1, process() at breakpoint_demo.cpp:7
# 7         int value = data[i] * 2;
(gdb) print value
# $1 = 160        <-- data[3]=80, 80*2=160 > 100, triggered
(gdb) print i
# $2 = 3
(gdb) continue
# [3] = 160
# Breakpoint 1, process() at breakpoint_demo.cpp:7
(gdb) print value
# $3 = 240        <-- data[4]=120, 120*2=240 > 100
(gdb) continue
#

# LLDB equivalent:
$ lldb ./breakpoint_demo
(lldb) breakpoint set -f breakpoint_demo.cpp -l 7 -c 'value > 100'
(lldb) run
```

Notice that the debugger ran through the first three iterations without pausing - it only stopped when `value` actually exceeded 100. When you are hunting a bug that only manifests under specific conditions, this is far more useful than stepping through thousands of normal iterations.

### Q2: Use a watchpoint to catch a write to a specific variable

Sometimes you know a variable is being corrupted but you do not know where. That is the scenario where a watchpoint earns its keep. Instead of pausing at a location in code, it pauses whenever a particular memory address is written (or read).

```cpp
// compile: g++ -g -O0 -std=c++20 watchpoint_demo.cpp -o watchpoint_demo
#include <iostream>

int global_counter = 0;

void sneaky_modifier() {
    global_counter = 999;  // Bug: unexpected modification!
}

void normal_work() {
    for (int i = 0; i < 3; ++i) {
        ++global_counter;
        if (i == 1) sneaky_modifier();  // Hidden bug
    }
}

int main() {
    global_counter = 0;
    normal_work();
    std::cout << "counter = " << global_counter << '\n';
    // Expected: 3, Actual: 1000 -- who modified it?
}
```

Setting a watchpoint and following the writes shows you exactly where the corruption happens:

```bash
# GDB: find who writes to global_counter
$ gdb ./watchpoint_demo
(gdb) break main
(gdb) run
(gdb) watch global_counter        # hardware watchpoint
# Hardware watchpoint 2: global_counter
(gdb) continue
# Hardware watchpoint 2: global_counter
# Old value = 0
# New value = 1
# normal_work() at watchpoint_demo.cpp:11   <-- ++global_counter
(gdb) continue
# Old value = 1
# New value = 2                              <-- ++global_counter
(gdb) continue
# Old value = 2
# New value = 999
# sneaky_modifier() at watchpoint_demo.cpp:6 <-- FOUND THE BUG
(gdb) backtrace
# #0 sneaky_modifier() at watchpoint_demo.cpp:6
# #1 normal_work() at watchpoint_demo.cpp:12
# #2 main() at watchpoint_demo.cpp:17

# LLDB equivalent:
(lldb) watchpoint set variable global_counter
# Watch types: write (default), read, read_write
(lldb) watchpoint set variable -w read_write global_counter
```

The watchpoint catches every single write to `global_counter` in order. When the value jumps from 2 to 999, you get the exact source location and a full backtrace. That combination of "what changed" and "who did it" is usually everything you need to fix the bug.

### Q3: Load a core dump and identify the crash location

When a program crashes in production - or in a test environment without a debugger attached - the OS can write a core dump: a snapshot of the process's memory, registers, and call stack at the moment of the crash. You can load that snapshot into GDB after the fact and debug it as if you had been attached all along.

```cpp
// compile: g++ -g -O0 -std=c++20 crash_demo.cpp -o crash_demo
#include <iostream>
#include <vector>

void deep_function(int* ptr) {
    *ptr = 42;  // Crash: null pointer dereference
}

void middle_function() {
    int* bad_ptr = nullptr;
    deep_function(bad_ptr);
}

int main() {
    std::cout << "Starting...\n";
    middle_function();
    std::cout << "Done\n";  // Never reached
}
```

The post-mortem workflow has three steps - enable core dumps, let the program crash, and then load the dump:

```bash
# Step 1: Enable core dumps
$ ulimit -c unlimited

# Step 2: Run and crash
$ ./crash_demo
# Starting
# Segmentation fault (core dumped)

# Step 3: Load core dump in GDB
$ gdb ./crash_demo core
# Core was generated by './crash_demo'
# Program terminated with signal SIGSEGV, Segmentation fault
# #0  0x000055... in deep_function(int*) at crash_demo.cpp:5
# 5         *ptr = 42;

(gdb) backtrace
# #0 deep_function(ptr=0x0) at crash_demo.cpp:5     <-- NULL ptr
# #1 middle_function() at crash_demo.cpp:10
# #2 main() at crash_demo.cpp:15

(gdb) frame 0
(gdb) print ptr
# $1 = (int *) 0x0     <-- confirmed nullptr

(gdb) info locals       # local variables in current frame
(gdb) info registers    # CPU register state at crash

# LLDB equivalent:
$ lldb --core core ./crash_demo
(lldb) bt               # backtrace
(lldb) frame select 0
(lldb) p ptr

# On Linux, configure core pattern:
# echo "/tmp/cores/core.%e.%p" | sudo tee /proc/sys/kernel/core_pattern
```

The backtrace tells you the exact call chain at the moment of the crash. `frame 0` puts you at the innermost frame, where `ptr` is shown to be `0x0` - a null pointer, confirming the crash cause without any guesswork.

---

## Notes

- Always compile with `-g -O0` for debugging. Without `-g` you lose line number information; without `-O0` the optimizer may reorder or eliminate variables in ways that make the debugger output confusing.
- Hardware watchpoints are limited in number - typically 4 on x86. If you need more, GDB will fall back to software watchpoints, which check the condition at every instruction and are much slower.
- Use `rwatch` for read watchpoints and `awatch` for access (read + write) watchpoints in GDB.
- On macOS, use `lldb` rather than GDB. Apple's system integrity protection makes it difficult to attach GDB to processes on modern macOS.
- VS Code integrates with both GDB and LLDB via the C/C++ extension, so you can do all of this from a GUI if you prefer.
- Core dumps can be large on memory-intensive programs. Use `gcore <pid>` to capture a core dump from a running process without killing it - useful for investigating a live but misbehaving process.
