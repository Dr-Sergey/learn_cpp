# Use std::spanstream (C++23) for in-memory I/O on raw buffers

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/io/basic_spanstream>  

---

## Topic Overview

C++23 introduces `std::ispanstream`, `std::ospanstream`, and `std::spanstream` (header `<spanstream>`) - stream classes that perform I/O on a fixed-size, externally-owned buffer via `std::span<char>`. Unlike `std::stringstream` which owns and dynamically allocates its buffer, spanstream operates on pre-allocated memory with **zero allocations**. This makes it the right tool any time you need stream-style formatting but can't afford - or don't want - heap allocation.

### Comparison with stringstream

| Property | `std::stringstream` | `std::spanstream` |
| --- | --- | --- |
| Buffer ownership | Owns (internal `std::string`) | Non-owning (`std::span<char>`) |
| Allocation | Dynamic (heap) | None - user provides buffer |
| Growth | Automatic resize | Fixed size - can't grow |
| Best for | General-purpose parsing | Embedded, real-time, performance-critical |
| Copy on `str()` | Yes (returns a copy) | Returns `span` (no copy) |
| Standard | C++98 | C++23 |

### Types

| Type | Direction | Description |
| --- | --- | --- |
| `std::ispanstream` | Input | Read from a `span<const char>` |
| `std::ospanstream` | Output | Write into a `span<char>` |
| `std::spanstream` | Both | Read/write on a `span<char>` |

### Core Examples

Here's the basic read/write pattern. You hand the stream a `span` over your buffer, use the familiar `>>` and `<<` operators, then call `.span()` to get back a view of exactly what was written - no copy involved.

```cpp
#include <spanstream>
#include <span>
#include <iostream>
#include <string>
#include <string_view>

int main() {
    // === Reading from a buffer (ispanstream) ===
    char input[] = "42 3.14 hello";
    std::ispanstream iss(std::span<const char>(input, sizeof(input) - 1));

    int i;
    double d;
    std::string s;
    iss >> i >> d >> s;
    std::cout << "i=" << i << " d=" << d << " s=" << s << "\n";
    // i=42 d=3.14 s=hello

    // === Writing to a buffer (ospanstream) ===
    char output[64] = {};
    std::ospanstream oss{std::span<char>(output)};

    oss << "Value: " << 42 << " Pi: " << 3.14;

    // Get the written portion
    auto written = oss.span();
    std::cout << "Written: " << std::string_view(written.data(), written.size()) << "\n";
    // Written: Value: 42 Pi: 3.14

    // No dynamic allocation occurred!
}
```

### Serialization into a Fixed Buffer

A practical pattern: embed serialize/deserialize methods directly on your data type, taking a `span` as the buffer contract. The caller owns the memory; the stream just provides the formatting machinery.

```cpp
#include <spanstream>
#include <span>
#include <iostream>
#include <array>

struct SensorReading {
    int id;
    double temperature;
    double humidity;

    // Serialize into a pre-allocated buffer
    std::span<char> serialize(std::span<char> buf) const {
        std::ospanstream oss(buf);
        oss << id << " " << temperature << " " << humidity;
        return oss.span(); // Returns only the written portion
    }

    // Deserialize from a buffer
    static SensorReading deserialize(std::span<const char> buf) {
        std::ispanstream iss(buf);
        SensorReading r;
        iss >> r.id >> r.temperature >> r.humidity;
        return r;
    }
};

int main() {
    SensorReading reading{42, 23.5, 65.2};

    // Serialize into stack-allocated buffer
    std::array<char, 128> buffer{};
    auto written = reading.serialize(buffer);

    std::cout << "Serialized (" << written.size() << " bytes): "
              << std::string_view(written.data(), written.size()) << "\n";
    // Serialized (14 bytes): 42 23.5 65.2

    // Deserialize from the same buffer
    auto parsed = SensorReading::deserialize(written);
    std::cout << "Parsed: id=" << parsed.id
              << " temp=" << parsed.temperature
              << " hum=" << parsed.humidity << "\n";
    // Parsed: id=42 temp=23.5 hum=65.2
}
```

---

## Self-Assessment

### Q1: Parse binary data from a `std::span<const char>` using ispanstream without copying

**Answer:**

```cpp
#include <spanstream>
#include <span>
#include <iostream>
#include <vector>

int main() {
    // Raw data buffer (could come from a network packet, file mmap, etc.)
    const char data[] = "100 200 300 400 500";

    // Create ispanstream over the raw buffer — NO COPY of data
    std::ispanstream iss(std::span<const char>(data, sizeof(data) - 1));

    // Parse integers directly from the buffer
    std::vector<int> values;
    int val;
    while (iss >> val) {
        values.push_back(val);
    }

    std::cout << "Parsed " << values.size() << " values: ";
    for (int v : values) std::cout << v << " ";
    std::cout << "\n";
    // Parsed 5 values: 100 200 300 400 500

    // Compare with stringstream — would COPY the entire buffer into a string:
    // std::istringstream iss2(std::string(data, sizeof(data)-1)); // COPIES!

    // With ispanstream: zero allocations, zero copies
    // The stream reads directly from the original 'data' array

    // Parsing structured data:
    const char record[] = "Alice 30 75000.50";
    std::ispanstream record_stream(std::span<const char>(record, sizeof(record) - 1));

    std::string name;
    int age;
    double salary;
    record_stream >> name >> age >> salary;
    std::cout << name << " age=" << age << " salary=" << salary << "\n";
    // Alice age=30 salary=75000.5
}
```

`std::ispanstream` wraps a `std::span<const char>` as a `std::istream`. It reads directly from the provided buffer with zero allocation and zero copy. This is critical in embedded or real-time systems where dynamic allocation is prohibited, or in high-throughput parsers where copying large buffers would waste bandwidth.

### Q2: Write a serializer using ospanstream that formats directly into a user-provided buffer

**Answer:**

```cpp
#include <spanstream>
#include <span>
#include <iostream>
#include <array>
#include <iomanip>
#include <string_view>

struct LogEntry {
    int level;
    std::string_view message;
    double timestamp;
};

// Format a log entry into a caller-provided buffer
// Returns the number of characters written, or -1 if buffer too small
int format_log(const LogEntry& entry, std::span<char> buffer) {
    std::ospanstream oss(buffer);
    oss << "[" << entry.level << "] "
        << std::fixed << std::setprecision(3) << entry.timestamp
        << "s: " << entry.message;

    if (oss.fail()) return -1; // buffer too small

    return static_cast<int>(oss.span().size());
}

int main() {
    LogEntry entry{2, "Disk space low", 1234.567};

    // Stack-allocated buffer — no heap allocation
    std::array<char, 256> buf{};
    int written = format_log(entry, buf);

    if (written > 0) {
        std::cout << "Formatted (" << written << " chars): "
                  << std::string_view(buf.data(), written) << "\n";
    }
    // Formatted (30 chars): [2] 1234.567s: Disk space low

    // Demonstrate buffer-too-small detection:
    std::array<char, 10> tiny_buf{};
    int result = format_log(entry, tiny_buf);
    if (result == -1) {
        std::cout << "Buffer too small!\n";
    }
    // Buffer too small!

    // Can also write multiple entries into one buffer
    std::array<char, 512> big_buf{};
    std::ospanstream oss{std::span<char>(big_buf)};
    for (int i = 0; i < 3; ++i) {
        oss << "Entry " << i << ": value=" << (i * 10) << "\n";
    }
    auto result_span = oss.span();
    std::cout << std::string_view(result_span.data(), result_span.size());
    // Entry 0: value=0
    // Entry 1: value=10
    // Entry 2: value=20
}
```

`std::ospanstream` writes formatted output directly into the caller's buffer via `std::span<char>`. The stream tracks the write position; `span()` returns a view of only the written portion. If the buffer overflows, the stream enters a fail state (`fail()` returns true). This is ideal for formatting into pre-allocated ring buffers, stack buffers, or shared memory.

### Q3: Compare spanstream with stringstream for performance in embedded or real-time contexts

| Criterion | `std::stringstream` | `std::spanstream` |
| --- | --- | --- |
| **Heap allocations** | Yes - internal string allocates/reallocates | None - operates on external buffer |
| **Real-time safe** | No - allocator may lock, fragmentation risk | Yes - fully deterministic timing |
| **Buffer growth** | Automatic (may trigger allocation) | Fixed - write beyond end -> failbit |
| **Memory fragmentation** | Yes (repeated alloc/free) | No |
| **Latency** | Unpredictable (allocator jitter) | Constant - no system calls |
| **Embedded suitability** | Poor (requires heap) | Excellent (stack/static buffers) |
| **API** | `str()` returns a copy | `span()` returns a view (no copy) |
| **Extra copies** | `str()` copies internal buffer | Zero copies |

```cpp
#include <spanstream>
#include <sstream>
#include <iostream>
#include <chrono>
#include <array>
#include <span>
#include <string>

int main() {
    constexpr int ITERATIONS = 100'000;

    // Benchmark: stringstream (allocates each iteration)
    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < ITERATIONS; ++i) {
        std::ostringstream oss;
        oss << "Sensor " << i << " temp=" << 23.5 << " hum=" << 65.0;
        std::string result = oss.str(); // COPY
        (void)result;
    }
    auto t2 = std::chrono::steady_clock::now();

    // Benchmark: spanstream (zero allocation)
    std::array<char, 128> buf{};
    auto t3 = std::chrono::steady_clock::now();
    for (int i = 0; i < ITERATIONS; ++i) {
        std::ospanstream oss{std::span<char>(buf)};
        oss << "Sensor " << i << " temp=" << 23.5 << " hum=" << 65.0;
        auto result = oss.span(); // VIEW — no copy
        (void)result;
    }
    auto t4 = std::chrono::steady_clock::now();

    auto ss_time = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1);
    auto sp_time = std::chrono::duration_cast<std::chrono::microseconds>(t4 - t3);

    std::cout << "stringstream: " << ss_time.count() << " us\n";
    std::cout << "spanstream:   " << sp_time.count() << " us\n";
    std::cout << "Speedup: " << static_cast<double>(ss_time.count()) / sp_time.count()
              << "x\n";
    // Typical result: spanstream is 2-5x faster due to zero allocation
}
```

The real-time argument is worth spelling out. Real-time systems - audio processing, motor control, trading systems - require hard latency bounds. A single heap allocation can cause unbounded latency if the allocator needs to request pages from the OS. `spanstream` guarantees zero allocations, which means every formatting operation is bounded in time.

---

## Notes

- **Buffer reuse:** Reuse the same buffer across multiple `ospanstream` operations by constructing a new stream over the same `span`. The buffer doesn't need zeroing.
- **No dynamic growth:** If you write beyond the buffer capacity, the stream enters a fail state. Check `oss.fail()` or `oss.good()` after writing.
- **Wide streams:** `std::wispanstream`, `std::wospanstream`, `std::wspanstream` are available for `wchar_t`.
- **Not a string replacement:** `spanstream` is for fixed-size, allocation-free I/O. If you need dynamically growing output, use `std::ostringstream` or `std::format_to` with a `back_insert_iterator`.
- **Compiler support (2024):** GCC 14+, MSVC 17.6+. Clang support pending.
- Compile with `-std=c++23 -Wall -Wextra`.
