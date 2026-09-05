# Understand micro-architectural bottlenecks: port pressure, ROB size, store buffers

**Category:** Performance and CPU Architecture  
**Standard:** C++17 (architecture-aware coding)  
**Reference:** <https://uops.info/> <https://agner.org/optimize/>  

---

## Topic Overview

### Beyond Big-O: Micro-Architecture Matters

Two algorithms with the same big-O complexity can differ by 10x in performance due to **micro-architectural** effects. Understanding the CPU pipeline is essential for low-level optimization. The roofline model tells you whether you are memory-bound or compute-bound; micro-architectural analysis tells you which specific hardware resource is the bottleneck within that.

### Execution Ports and Port Pressure

Modern CPUs have multiple **execution ports**, each capable of handling specific types of µops. Not all operations can go through all ports - multiplications can only use certain ports, loads can only use load ports, and so on. When your loop generates too many µops that compete for the same port, that port becomes a bottleneck even if every other port is sitting idle.

Here is the port layout for a typical Intel Alder Lake P-core to make this concrete:

```cpp
Intel Alder Lake P-core (Golden Cove):
Port 0: ALU, Vec ALU, Branch
Port 1: ALU, Vec ALU, Integer Mul
Port 2: Load (AGU)
Port 3: Load (AGU)
Port 4: Store Data
Port 5: ALU, Vec ALU, Vec Shuffle
Port 6: ALU, Branch
Port 7: Store AGU
Port 8: Store AGU
Port 9: Store Data
```

**Port pressure** occurs when too many µops compete for the same port, creating a bottleneck even though other ports are idle. The way to fix it is to restructure the code so that different operations - loads, multiplies, adds - are interleaved, spreading the work across more ports:

```cpp
// Example: high port 0/1 pressure (ALU-heavy, no memory ops)
for (int i = 0; i < n; ++i) {
    result[i] = a[i] * b[i] + c[i] * d[i];
    // All multiplications compete for port 1 (integer multiply)
}

// Better: interleave with memory operations to balance port utilization
for (int i = 0; i < n; ++i) {
    auto ai = a[i];  // load - port 2/3
    auto bi = b[i];  // load - port 2/3
    auto ab = ai * bi;  // multiply - port 1
    auto ci = c[i];  // load - port 2/3 (overlaps with multiply)
    auto di = d[i];  // load
    result[i] = ab + ci * di;  // add - port 0/5, multiply - port 1
}
```

The restructured version does the same work but the loads and multiplies are interleaved, so the load ports (2/3) and the multiply port (1) can all be active at the same time instead of one sitting idle while the other is saturated.

### Reorder Buffer (ROB) Size

The ROB tracks all in-flight µops from decode to retirement. If it fills up, the front end stalls and the CPU stops issuing new instructions. Current ROB sizes:

| CPU | ROB Size |
| --- | --- |
| Intel Alder Lake (P-core) | 512 entries |
| Intel Raptor Lake (P-core) | 512 entries |
| AMD Zen 4 | 320 entries |
| AMD Zen 5 | 448 entries |
| Apple M3 (Avalanche) | ~630 entries |

Here is why this matters. A DRAM access takes ~200 cycles. During those 200 cycles, the out-of-order engine is trying to find other useful work to execute, keeping results in the ROB until the stalled instruction finishes. But the ROB only holds ~320-512 entries. If there are not enough independent instructions to fill those cycles, the ROB fills up with instructions waiting behind the cache miss, and the entire pipeline stalls.

This is why reducing cache misses improves ILP - it keeps the ROB drained so new instructions can keep flowing in:

```cpp
// ROB-unfriendly: unpredictable memory access -> cache misses -> ROB fills
for (int i = 0; i < n; ++i) {
    sum += data[indices[i]];  // random access -> L3/DRAM miss -> ROB stall
}

// ROB-friendly: prefetch to hide latency, keeping ROB from filling
for (int i = 0; i < n; ++i) {
    __builtin_prefetch(&data[indices[i + 16]]);  // prefetch future access
    sum += data[indices[i]];  // hopefully in cache by now
}
```

The prefetch hint asks the hardware to start fetching `data[indices[i+16]]` early, so that by the time iteration `i+16` runs, the data is already in cache. This hides latency by overlapping memory access with computation.

### Store Buffer

The **store buffer** holds stores before they commit to the cache hierarchy. It does three important things:

- Allows stores to retire before the write actually reaches the cache (write combining).
- Supports store-to-load forwarding (a load can read from a pending store without waiting for it to reach cache).
- Has limited capacity (~56-72 entries on modern x86).

When your loop issues many stores to different cache lines faster than the store buffer can drain them, it fills up and stalls the pipeline. Non-temporal stores bypass the store buffer and cache entirely, writing directly to memory - useful for large write-only operations where you will not be reading the data back immediately:

```cpp
// Many stores can fill the store buffer
void fill_array(int* arr, int n) {
    for (int i = 0; i < n; ++i)
        arr[i] = i;  // each iteration = 1 store
    // If stores are to different cache lines, they cannot be combined
    // Store buffer may fill, causing a stall
}

// Non-temporal stores bypass the store buffer and cache
#include <immintrin.h>
void fill_array_nt(int* arr, int n) {
    for (int i = 0; i < n; i += 4) {
        __m128i val = _mm_set_epi32(i+3, i+2, i+1, i);
        _mm_stream_si128((__m128i*)&arr[i], val);  // bypasses cache
    }
}
```

### Detecting Bottlenecks with `perf`

The top-down analysis methodology divides pipeline stalls into four categories. This is the fastest way to know which micro-architectural resource to investigate:

```bash
# Top-down micro-architecture analysis
perf stat -e \
  topdown-retiring,topdown-bad-spec,topdown-fe-bound,topdown-be-bound \
  ./my_app

# Typical output:
# 35.2% topdown-retiring      <- useful work
# 5.1%  topdown-bad-spec      <- branch mispredictions
# 12.3% topdown-fe-bound      <- front-end stalls (instruction fetch/decode)
# 47.4% topdown-be-bound      <- back-end stalls (memory, execution units) <- bottleneck
```

---

## Self-Assessment

### Q1: What causes a front-end stall vs. a back-end stall

The front-end is responsible for fetching and decoding instructions. The back-end is responsible for executing them. They are two separate pipeline stages, and they can stall independently:

- **Front-end stall**: The instruction fetch/decode pipeline can't keep up with the back-end's appetite for work. Common causes: instruction cache miss, complex instructions that decode into many µops, branch mispredictions that flush the pipeline, and instruction alignment issues.
- **Back-end stall**: The execution engine cannot make progress. Common causes: D-cache misses (memory-bound), port pressure (compute-bound - a specific execution port saturated), ROB full (long-latency stall blocking retirement), or store buffer full.

When `perf` shows high `topdown-be-bound`, the next step is to drill down into whether it is memory-bound (D-cache miss counters) or execution-unit-bound (port utilization counters).

### Q2: How does store-to-load forwarding work

Store-to-load forwarding is a hardware optimization that lets a load bypass the cache entirely if there is a pending store to the same address in the store buffer. Rather than waiting for the store to travel all the way to cache and back, the CPU hands the data directly from the store buffer to the waiting load.

This works perfectly when the load completely overlaps the store's data - same address, same size. The reason this trips people up is partial overlaps. If you store 4 bytes and then load 8 bytes starting at the same address (or a different but overlapping offset), the CPU cannot forward cleanly and has to handle a forwarding failure, which causes a pipeline stall and potentially a cache lookup before the load can complete.

### Q3: How do you identify port pressure with perf

Once you know you have a back-end execution stall (high `topdown-be-bound`), the next step is to look at per-port utilization. On Intel CPUs, hardware performance counters track how many µops were dispatched through each port. A port running at saturation while others are quiet is the bottleneck:

```bash
# On Intel, use specific port utilization counters
perf stat -e uops_dispatched_port.port_0,\
uops_dispatched_port.port_1,\
uops_dispatched_port.port_2,\
uops_dispatched_port.port_3,\
uops_dispatched_port.port_5,\
uops_dispatched_port.port_6 \
./my_app
```

If one port has significantly more µops than others, that port is the bottleneck. Look up which instruction types use that port at <https://uops.info/>, then restructure the loop to balance the µop mix across ports - for example, replacing sequences of integer multiplications with a mix of multiplies and adds that can use multiple ports.

---

## Notes

- Use <https://uops.info/> to check which ports specific instructions use on your target microarchitecture.
- Agner Fog's instruction tables are the definitive reference for latency and throughput per instruction.
- Intel VTune's "Microarchitecture Exploration" analysis automates bottleneck detection and presents port utilization visually.
- Compiler flags like `-march=native` allow the compiler to use all available ports and instruction types for the specific CPU it is targeting.
