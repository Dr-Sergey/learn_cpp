# Use Static Allocation Strategies Without Heap

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** <https://www.etlcpp.com/>  

---

## Topic Overview

In safety-critical and hard real-time systems, dynamic memory allocation (`new`, `malloc`) is forbidden or severely restricted. The heap introduces non-deterministic allocation times, fragmentation over long uptimes (months/years), and the possibility of allocation failure that is difficult to handle gracefully. MISRA C++ 2023 Rule 21.6.1 bans `malloc`/`free`; AUTOSAR C++14 Rule A18-5-1 bans `new`/`delete` in safety-critical code.

Modern C++ is **not** synonymous with heap allocation. With placement `new`, `std::array`, `std::optional`, `std::variant`, `std::span`, static memory pools, and the Embedded Template Library (ETL), you can write expressive C++ that never touches the heap. The key principle is: **all memory is allocated at compile time or system startup, with sizes known statically.**

| Strategy | Mechanism | Deterministic? | Fragmentation? | Size Known at Compile Time? |
| --- | --- | --- | --- | --- |
| Stack allocation | Local variables, `alloca` | Yes | No | Yes (frame size) |
| Static globals | `static` / global objects | Yes | No | Yes |
| `std::array<T,N>` | Fixed-size array | Yes | No | Yes |
| Placement `new` | Construct in pre-allocated buffer | Yes | No | Buffer: yes |
| Static pool allocator | Fixed-size blocks from static array | Yes | No (same-size blocks) | Yes |
| `etl::vector<T,N>` | Stack-based vector with max capacity | Yes | No | Yes |
| `std::pmr` with monotonic | Bump allocator over static buffer | Yes (until exhaustion) | No (no dealloc) | Buffer: yes |
| `std::vector<T>` | Heap | No | Yes | No |

The diagram below makes the contrast concrete. Notice that the no-heap program has its worst case known at compile/link time: either everything fits or the linker refuses to build.

```cpp
Memory Model Comparison:

Heap-based program:                No-heap program:
┌─────────────────────┐           ┌─────────────────────┐
│ .text          Code │           │ .text          Code │
│ .rodata    Constants│           │ .rodata    Constants│
│ .data    Init'd vars│           │ .data    Init'd vars│
│ .bss   Zero'd vars  │           │ .bss   Zero'd vars  │
│                     │           │ .bss   Static pools  │
│ ┌─── Heap ────────┐ │           │ .bss   Ring buffers  │
│ │ fragmented...   │ │           │ .bss   Message queues│
│ │ ??? │free│used│  │ │           │                     │
│ │ │alloc│???│    │  │ │           │ (no heap section)   │
│ └─────────────────┘ │           │                     │
│                     │           │                     │
│ ┌─── Stack ───────┐ │           │ ┌─── Stack ───────┐ │
│ │ deterministic   │ │           │ │ deterministic   │ │
│ └─────────────────┘ │           │ └─────────────────┘ │
└─────────────────────┘           └─────────────────────┘
  Risk: OOM at runtime              Guarantee: all fits or
  after weeks of uptime             won't compile/link
```

To enforce no-heap at the toolchain level, provide stubs for `_sbrk`, `operator new`, and `malloc` that trap, and use linker flags like `-Wl,--wrap=malloc` to intercept any accidental heap usage.

---

## Self-Assessment

### Q1: Implement a compile-time-sized, type-safe static memory pool allocator that supports allocation and deallocation in O(1) with no heap usage

A fixed-block pool works by maintaining a free list embedded directly in the unused slots. When a slot is free, its storage holds the index of the next free slot. When it is occupied, that same storage holds the live object. This gives O(1) alloc and free with no additional memory overhead.

```cpp
// static_pool.h - Fixed-block static memory pool
#pragma once

#include <cstdint>
#include <cstddef>
#include <new>           // placement new (freestanding)
#include <type_traits>
#include <optional>
#include <array>

namespace mem {

// Pool of N objects of type T, entirely in static memory (BSS)
// O(1) allocate and deallocate via embedded free list
template <typename T, std::size_t N>
class StaticPool {
    static_assert(N > 0, "Pool must have at least one element");
    static_assert(sizeof(T) >= sizeof(std::size_t),
                  "Element must be large enough to hold free-list index");

    // Each slot is either:
    //   - In use: contains a constructed T
    //   - Free: contains a std::size_t index to next free slot
    union Slot {
        alignas(T) std::uint8_t storage[sizeof(T)];
        std::size_t next_free;
    };

    std::array<Slot, N> pool_{};
    std::size_t free_head_ = 0;
    std::size_t used_count_ = 0;

public:
    // Initialize free list - must be called before first use
    // (or use constexpr constructor for trivial T)
    void init() {
        for (std::size_t i = 0; i < N - 1; ++i) {
            pool_[i].next_free = i + 1;
        }
        pool_[N - 1].next_free = N;  // sentinel: end of list
        free_head_ = 0;
        used_count_ = 0;
    }

    // Allocate one T, construct in-place with given args
    template <typename... Args>
    [[nodiscard]]
    T* allocate(Args&&... args) {
        if (free_head_ >= N) {
            return nullptr;  // Pool exhausted
        }
        auto idx = free_head_;
        free_head_ = pool_[idx].next_free;
        ++used_count_;

        // Construct T in the slot's storage
        return ::new (pool_[idx].storage) T(static_cast<Args&&>(args)...);
    }

    // Deallocate - destroy T and return slot to free list
    void deallocate(T* ptr) {
        if (!ptr) return;

        // Find slot index from pointer
        auto* slot_ptr = reinterpret_cast<Slot*>(ptr);
        auto idx = static_cast<std::size_t>(slot_ptr - pool_.data());

        if (idx >= N) return;  // Invalid pointer - defensive

        // Destroy the object
        ptr->~T();

        // Return slot to free list
        pool_[idx].next_free = free_head_;
        free_head_ = idx;
        --used_count_;
    }

    [[nodiscard]] std::size_t used()      const { return used_count_; }
    [[nodiscard]] std::size_t available()  const { return N - used_count_; }
    [[nodiscard]] static constexpr std::size_t capacity() { return N; }
};

// ---------- Usage example ----------

struct TcpConnection {
    std::uint32_t remote_ip;
    std::uint16_t remote_port;
    std::uint16_t local_port;
    std::uint8_t  state;
    std::uint8_t  retries;

    // Non-trivial constructor - pool handles it
    TcpConnection(std::uint32_t ip, std::uint16_t rport, std::uint16_t lport)
        : remote_ip(ip), remote_port(rport), local_port(lport),
          state(0), retries(0) {}
};

// Static pool for max 32 simultaneous TCP connections - zero heap
inline StaticPool<TcpConnection, 32> g_conn_pool;

inline TcpConnection* accept_connection(std::uint32_t ip, std::uint16_t port) {
    return g_conn_pool.allocate(ip, port, std::uint16_t{8080});
}

inline void close_connection(TcpConnection* conn) {
    g_conn_pool.deallocate(conn);
}

} // namespace mem
```

### Q2: Show how to use `std::pmr::monotonic_buffer_resource` over a static buffer to get STL-compatible containers without heap allocation

The `std::pmr` (polymorphic memory resource) machinery lets standard containers like `std::pmr::vector` use any backing allocator you provide. The key is passing `std::pmr::null_memory_resource()` as the upstream - this means the allocator will fail (throw or abort) rather than silently falling back to the heap when the static buffer runs out.

```cpp
// pmr_static.h - PMR containers on static memory
#pragma once

#include <cstdint>
#include <cstddef>
#include <array>
#include <memory_resource>  // std::pmr (C++17)
#include <string>
#include <vector>

namespace mem {

// ---------- Static buffer for PMR ----------
// All PMR allocations come from this fixed buffer - no heap.

class StaticBufferResource {
    // 4 KB static buffer - adjust per application
    static constexpr std::size_t BUFFER_SIZE = 4096;
    alignas(std::max_align_t) std::uint8_t buffer_[BUFFER_SIZE]{};

    // null_memory_resource as upstream -> allocation fails instead of heap fallback
    std::pmr::monotonic_buffer_resource resource_{
        buffer_, BUFFER_SIZE,
        std::pmr::null_memory_resource()  // CRITICAL: no heap fallback!
    };

public:
    std::pmr::memory_resource* get() { return &resource_; }

    void reset() {
        resource_.release();  // Reclaim all memory (fast: just resets pointer)
    }

    static constexpr std::size_t buffer_size() { return BUFFER_SIZE; }
};

// ---------- Usage: PMR containers with zero heap ----------

inline void process_sensor_data() {
    // Static buffer resource - lives for scope duration or longer
    StaticBufferResource mem_resource;

    // PMR vector - allocates from static buffer, NOT heap
    std::pmr::vector<std::uint16_t> readings(mem_resource.get());
    readings.reserve(128);  // One allocation from static buffer

    // Simulate reading 100 ADC samples
    for (int i = 0; i < 100; ++i) {
        readings.push_back(static_cast<std::uint16_t>(i * 33));
    }

    // PMR string - also from static buffer
    std::pmr::string status(mem_resource.get());
    status = "Sensor OK: ";
    // append does NOT go to heap

    // Process...
    std::uint32_t sum = 0;
    for (auto v : readings) sum += v;

    // When mem_resource goes out of scope (or reset() called),
    // all memory is reclaimed instantly. No fragmentation.
}

// ---------- Long-lived PMR containers ----------

// For global containers that persist, use a global resource
inline StaticBufferResource g_global_mem;

struct LogEntry {
    std::uint32_t timestamp;
    std::uint8_t  severity;
    char          message[59];  // Fixed-size to avoid string allocation
};
static_assert(sizeof(LogEntry) == 64, "LogEntry must be cache-line sized");

// PMR vector with global static backing
inline std::pmr::vector<LogEntry>& get_log() {
    static std::pmr::vector<LogEntry> log(g_global_mem.get());
    return log;
}

// ---------- Scoped arena pattern ----------
// Allocate for a function's duration, then bulk-free

class ScopedArena {
    static constexpr std::size_t SIZE = 2048;
    alignas(std::max_align_t) std::uint8_t buf_[SIZE];
    std::pmr::monotonic_buffer_resource resource_{
        buf_, SIZE, std::pmr::null_memory_resource()
    };

public:
    std::pmr::memory_resource* get() { return &resource_; }
    // Destructor releases all - O(1), no per-object dealloc
};

inline void handle_command(const std::uint8_t* data, std::size_t len) {
    ScopedArena arena;

    // Temporary containers - all memory freed when arena dies
    std::pmr::vector<std::uint8_t> decoded(arena.get());
    decoded.assign(data, data + len);

    std::pmr::string response(arena.get());
    response = "ACK";

    // ... process and respond ...
    // No memory leak possible, no fragmentation, deterministic cleanup
}

} // namespace mem
```

The scoped arena pattern is particularly powerful for message-parsing or frame-processing loops: allocate freely during the function, then free everything in one O(1) operation when the arena goes out of scope.

### Q3: Implement a no-heap firmware using ETL (Embedded Template Library) containers and demonstrate how to trap any accidental heap usage at link time

ETL gives you drop-in replacements for STL containers - same API, but with a compile-time maximum capacity baked into the type. The `etl::delegate` replacement for `std::function` is particularly important because `std::function` can heap-allocate for larger callables.

```cpp
// etl_firmware.h - Complete no-heap firmware patterns using ETL
#pragma once

// ETL: drop-in replacements for STL containers with static max capacity
// Install: header-only, just add to include path
#include <etl/vector.h>
#include <etl/string.h>
#include <etl/map.h>
#include <etl/queue.h>
#include <etl/optional.h>
#include <etl/expected.h>
#include <etl/delegate.h>
#include <etl/pool.h>

#include <cstdint>
#include <cstddef>

namespace fw {

// ---------- ETL containers: STL API, static allocation ----------

// etl::vector<T, N> - max N elements, stored inline (no heap)
struct SensorReading {
    std::uint16_t sensor_id;
    float         value;
    std::uint32_t timestamp_ms;
};

// Max 64 readings - all in BSS, zero heap
etl::vector<SensorReading, 64> g_readings;

// etl::string<N> - fixed-capacity string
etl::string<128> g_device_name{"Sensor-Node-042"};

// etl::map<K, V, N> - static red-black tree
etl::map<std::uint16_t, float, 16> g_calibration_table;

// etl::queue<T, N> - static FIFO
struct Command {
    std::uint8_t  opcode;
    std::uint32_t param;
};
etl::queue<Command, 32> g_command_queue;

// ---------- etl::delegate - replaces std::function (no heap!) ----------
// std::function can heap-allocate for large callables. etl::delegate never does.

class MotorController {
    std::uint8_t speed_ = 0;
public:
    void set_speed(std::uint8_t s) { speed_ = s; }
    std::uint8_t speed() const { return speed_; }
};

using SpeedCallback = etl::delegate<void(std::uint8_t)>;

inline void register_callback_example() {
    MotorController motor;

    // Bind member function - no heap, no std::function
    SpeedCallback cb = SpeedCallback::create<
        MotorController, &MotorController::set_speed>(motor);

    cb(128);  // Calls motor.set_speed(128)
}

// ---------- etl::pool - object pool (like our StaticPool) ----------

struct Packet {
    std::uint8_t  data[256];
    std::uint16_t length;
    std::uint8_t  priority;
};

// Pool of 16 packets - no heap
etl::pool<Packet, 16> g_packet_pool;

inline Packet* alloc_packet() {
    return g_packet_pool.allocate<Packet>();
}

inline void free_packet(Packet* p) {
    g_packet_pool.release(p);
}

// ---------- Application using all of the above ----------

inline void firmware_main_loop() {
    // Initialize calibration
    g_calibration_table[0x01] = 1.023f;
    g_calibration_table[0x02] = 0.998f;

    while (true) {
        // Process command queue
        if (!g_command_queue.empty()) {
            Command cmd = g_command_queue.front();
            g_command_queue.pop();

            if (cmd.opcode == 0x10) {
                // Store calibrated reading
                auto it = g_calibration_table.find(
                    static_cast<std::uint16_t>(cmd.param >> 16));
                float cal = (it != g_calibration_table.end())
                            ? it->second : 1.0f;

                if (!g_readings.full()) {
                    g_readings.push_back(SensorReading{
                        .sensor_id    = static_cast<std::uint16_t>(cmd.param >> 16),
                        .value        = static_cast<float>(cmd.param & 0xFFFF) * cal,
                        .timestamp_ms = 0  // filled by timer
                    });
                }
            }
        }
        asm volatile("wfi");
    }
}

} // namespace fw
```

Link this file with every firmware build - any accidental `new`, `delete`, or `malloc` call will hit a breakpoint during testing rather than failing silently in production:

```cpp
// heap_trap.cpp - Link-time enforcement of no-heap policy
// Compile and link this file with every firmware build.

#include <cstddef>
#include <cstdint>

// Override operator new/delete to trap
// If any code path reaches these, it's a bug.
void* operator new(std::size_t) {
    // In debug: breakpoint. In release: reset.
    asm volatile("bkpt #1");
    while (true) {}  // unreachable
}

void* operator new[](std::size_t) {
    asm volatile("bkpt #2");
    while (true) {}
}

void operator delete(void*) noexcept {
    asm volatile("bkpt #3");
    while (true) {}
}

void operator delete[](void*) noexcept {
    asm volatile("bkpt #4");
    while (true) {}
}

void operator delete(void*, std::size_t) noexcept {
    asm volatile("bkpt #5");
    while (true) {}
}

void operator delete[](void*, std::size_t) noexcept {
    asm volatile("bkpt #6");
    while (true) {}
}

// Also trap C malloc/free for mixed C/C++ codebases
extern "C" {

void* malloc(std::size_t) {
    asm volatile("bkpt #7");
    while (true) {}
}

void free(void*) {
    asm volatile("bkpt #8");
    while (true) {}
}

void* calloc(std::size_t, std::size_t) {
    asm volatile("bkpt #9");
    while (true) {}
}

void* realloc(void*, std::size_t) {
    asm volatile("bkpt #10");
    while (true) {}
}

// Newlib's heap growth function
void* _sbrk(int) {
    asm volatile("bkpt #11");
    while (true) {}
}

} // extern "C"
```

The breakpoint numbers make it easy to see in a debugger exactly which allocation function was called, which helps you track down the offending code path quickly.

---

## Notes

- ETL (`etlcpp.com`) is the de facto standard for no-heap C++ on embedded. It mirrors the STL API but all containers have a compile-time maximum capacity. It requires no RTTI and no exceptions.
- `std::pmr::monotonic_buffer_resource` with `null_memory_resource()` upstream is the **correct** way to use PMR without heap - omitting the upstream parameter defaults to `get_default_resource()` which **is the heap**.
- Placement `new` (`::new (buffer) T(args...)`) is freestanding and never allocates - it constructs an object at the provided address. Always pair with explicit destructor call (`obj->~T()`).
- `std::optional<T>` and `std::variant<Ts...>` store their payloads inline - they are inherently no-heap and safe for embedded.
- The heap trap file (`heap_trap.cpp`) should be included in every firmware build. Any accidental heap usage will trigger a breakpoint during testing, not a silent failure in production.
- For `std::pmr`, the `monotonic_buffer_resource` is ideal for "allocate many, free all at once" patterns (message parsing, frame processing). For general-purpose alloc/free, use `std::pmr::unsynchronized_pool_resource` over a static buffer.
- Stack usage analysis is critical when avoiding the heap - all "dynamic" storage moves to the stack. Use `-fstack-usage` (GCC) and static analysis tools to verify stack depth.
- `etl::delegate` is a non-allocating replacement for `std::function`. It stores the callable inline (up to ~2 pointers) and never heap-allocates.
