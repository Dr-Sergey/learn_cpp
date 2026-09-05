# Understand instruction-level parallelism (ILP) and out-of-order execution

**Category:** Performance and CPU Architecture  
**Standard:** C++17 (architecture-aware coding)  
**Reference:** <https://en.wikipedia.org/wiki/Instruction-level_parallelism>  

---

## Topic Overview

### What Is ILP

**Instruction-Level Parallelism (ILP)** is the ability of a CPU to execute multiple instructions simultaneously by overlapping their execution stages. Modern CPUs are superscalar - they can dispatch 4-8+ instructions per cycle across multiple execution units. The key insight is that the CPU does not just process instructions one at a time; it looks at many in-flight instructions at once and executes whichever ones have their inputs ready.

### Out-of-Order Execution

Modern CPUs don't execute instructions in strict program order. The **out-of-order execution** engine works in five steps:

1. **Fetches** instructions in order from the instruction stream.
2. **Decodes** them into micro-operations (µops).
3. **Renames** registers to eliminate false dependencies (WAR, WAW hazards).
4. **Dispatches** µops to execution ports as soon as operands are ready - regardless of original order.
5. **Retires** instructions in original program order (to maintain correctness).

The important step is number 4. The CPU watches which µops have their inputs ready and sends them to available execution ports, even if earlier instructions in program order are still waiting for a cache miss or a slow operation. Step 5 ensures the observable result is the same as if everything ran in order.

Here is a simple illustration. Instructions 1 and 3 are independent loads - the CPU can issue both in the same cycle:

```cpp
Program order:        Execution order (possible):

1. load  r1, [addr1]  -> port 2 (cycle 1)
2. add   r2, r1, 5    -> port 0 (cycle 5, waits for load)
3. load  r3, [addr2]  -> port 3 (cycle 1, parallel with line 1!)
4. mul   r4, r3, 10   -> port 1 (cycle 5, waits for load)
```

Instructions 1 and 3 are independent - the CPU executes them **in parallel**.

### Data Dependencies Limit ILP

The thing that kills ILP is when each instruction depends on the result of the previous one. The CPU cannot overlap them no matter how many execution ports it has, because it must wait for each result before starting the next computation:

```cpp
// Low ILP - each computation depends on the previous one (serial chain)
int serial(int a) {
    int x = a * 3;      // must wait for a
    int y = x + 7;      // must wait for x
    int z = y * 2;      // must wait for y
    return z + 1;       // must wait for z - total: 4 serial steps
}

// High ILP - independent computations can execute in parallel
int parallel(int a, int b, int c, int d) {
    int x = a * 3;   // independent
    int y = b + 7;   // independent - runs in parallel with x
    int z = c * 2;   // independent - runs in parallel
    int w = d + 1;   // independent - runs in parallel
    return x + y + z + w;  // depends on all, but computed in a tree
}
```

The `parallel` version is faster not because it does less work, but because the four multiplications/additions can all start in the same cycle.

### Writing ILP-Friendly C++

**1. Break dependency chains:**

The single-accumulator sum is a classic ILP killer. Each addition must wait for the previous addition to complete before it can start, creating a serial chain as long as the array. With four accumulators, the four addition chains are independent and the CPU runs them in parallel, roughly quadrupling the throughput:

```cpp
// BAD: single accumulator - serial dependency chain
double sum_serial(const double* data, size_t n) {
    double sum = 0.0;
    for (size_t i = 0; i < n; ++i)
        sum += data[i];  // each add depends on previous sum
    return sum;
}

// GOOD: multiple accumulators - independent chains
double sum_parallel(const double* data, size_t n) {
    double sum0 = 0.0, sum1 = 0.0, sum2 = 0.0, sum3 = 0.0;
    size_t i = 0;
    for (; i + 3 < n; i += 4) {
        sum0 += data[i];      // independent chain
        sum1 += data[i + 1];  // independent chain
        sum2 += data[i + 2];  // independent chain
        sum3 += data[i + 3];  // independent chain
    }
    for (; i < n; ++i) sum0 += data[i];
    return sum0 + sum1 + sum2 + sum3;
}
```

**2. Avoid long latency chains:**

Division has a latency of 20-40 cycles on most CPUs. Chaining two divisions in sequence means the second cannot start until the first finishes. If you can rearrange the math to multiply two cheap operands first and divide once, you win:

```cpp
// Division has high latency (~20-40 cycles) - avoid chaining divisions
double bad(double a, double b, double c) {
    return a / b / c;  // serial: div -> div
}

double better(double a, double b, double c) {
    return a / (b * c);  // multiply is faster, then one division
}
```

### The Reorder Buffer (ROB) and Its Limits

The CPU's **Reorder Buffer** holds all in-flight instructions from decode until retirement. On modern x86, these buffers are large but finite:

| CPU | ROB Size |
| --- | --- |
| Intel Alder Lake (P-core) | 512 entries |
| Intel Raptor Lake (P-core) | 512 entries |
| AMD Zen 4 | 320 entries |
| AMD Zen 5 | 448 entries |
| Apple M3 (Avalanche) | ~630 entries |

Here is why this matters for ILP: a long-latency operation like a DRAM access takes ~200 cycles. During those 200 cycles, the CPU is trying to keep doing useful work by executing later instructions. But it can only hold ~320-512 instructions in the ROB at once. If the ROB fills up before the DRAM access completes, the pipeline stalls. Cache misses are not just a memory problem - they block the entire ILP machinery.

---

## Self-Assessment

### Q1: Why do multiple accumulators improve performance

With a single accumulator, each `sum += data[i]` depends on the previous value of `sum` - creating a serial dependency chain with latency equal to N times the addition latency (3-4 cycles for floating point).

With 4 accumulators, each chain is independent. The CPU's out-of-order engine dispatches additions from all 4 chains in parallel, achieving up to 4x throughput. The final reduction `sum0 + sum1 + sum2 + sum3` is a tree with only 3 additions, which is negligible compared to the N additions in the main loop.

### Q2: What is register renaming and why does it help

Register renaming is one of the most important tricks in modern CPUs. The hardware has far more physical registers (hundreds) than the architectural register file (16 on x86-64). When the CPU detects that two instructions write to the same architectural register but are logically independent, it assigns them different physical registers. This eliminates the false dependency so both can execute without waiting:

```assembly
; Without renaming - false dependency (WAW hazard):
mov eax, [addr1]    ; write eax
add eax, 5          ; read eax, write eax
mov eax, [addr2]    ; write eax - must wait even though values are unrelated!

; With renaming - CPU uses different physical registers:
mov r100, [addr1]   ; physical r100
add r101, r100, 5   ; physical r101
mov r102, [addr2]   ; physical r102 - NO dependency on r100/r101!
```

Register renaming eliminates false dependencies (Write-After-Read, Write-After-Write), allowing independent instruction sequences to execute in parallel even when they use the same architectural registers. This is one reason why straightforward-looking scalar code can still achieve high ILP on modern hardware.

### Q3: Show a code pattern that kills ILP and how to fix it

The canonical ILP killer is pointer chasing. Each iteration of a linked-list traversal cannot start until the previous iteration has finished loading the `next` pointer, because the next address is unknown until that load completes. No amount of out-of-order execution can help - there is a true data dependency on every step. A contiguous array breaks this chain because the hardware prefetcher can predict the next address without waiting for the current load:

```cpp
// ILP killer: linked-list traversal - each step depends on the previous pointer
Node* ptr = head;
int count = 0;
while (ptr) {
    count += ptr->value;
    ptr = ptr->next;    // pointer-chasing: next address unknown until load completes
}

// Fix: use contiguous data (std::vector) for predictable access patterns
std::vector<int> values = /* ... */;
int count = 0;
for (int v : values) {
    count += v;  // hardware prefetcher predicts sequential accesses
}
```

---

## Notes

- Modern CPUs can typically dispatch 4-8 µops per cycle - but only if there are enough independent instructions to keep the execution units busy.
- Compilers help with ILP through instruction scheduling, but algorithmic data dependencies are fundamental and the compiler cannot remove them.
- Profile with `perf stat` and check IPC (instructions per cycle). An IPC below 1.0 usually means the pipeline is stalling somewhere.
- Multiple accumulators, loop unrolling, and SoA (struct-of-arrays) layouts all improve ILP by reducing dependency chains.
