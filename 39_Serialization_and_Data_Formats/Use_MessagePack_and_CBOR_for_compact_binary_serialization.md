# Use MessagePack and CBOR for compact binary serialization

**Category:** Serialization and Data Formats  
**Standard:** C++17  
**Reference:** <https://msgpack.org/> · <https://cbor.io/>  

---

## Topic Overview

MessagePack and CBOR are self-describing binary serialization formats that are more compact than JSON while remaining schema-free. They're ideal for logging, caching, and inter-service communication.

The key idea is "self-describing binary." Unlike FlatBuffers or a hand-rolled binary format, you do not need a separate schema file to read the data - the format embeds enough type information to decode it. But unlike JSON, it does not waste bytes on quotes, braces, and ASCII digits. You get the flexibility of JSON at roughly half the wire size.

### Comparison

Both MessagePack and CBOR compress your data the same way: small integers pack into 1-2 bytes, string lengths are binary-encoded rather than counted in ASCII, and structure delimiters are single bytes instead of `{`, `}`, `,`. The table below puts them side-by-side with JSON:

| Feature | JSON | MessagePack | CBOR |
| --- | --- | --- | --- |
| Format | Text | Binary | Binary |
| Self-describing | Yes | Yes | Yes |
| Size vs JSON | Baseline | ~50-70% smaller | ~50-70% smaller |
| Standardized | RFC 8259 | msgpack spec | RFC 8949 |
| Streaming | Limited | Yes | Yes |
| Extensions | No | Yes (ext types) | Yes (tags) |

### Using nlohmann/json for MessagePack and CBOR

If you already use nlohmann/json, you get MessagePack and CBOR support for free - no additional dependencies. The same `json` object serializes to all three formats with a single function call. The hex dump at the end of the example gives you a concrete feel for what the binary actually looks like:

<!-- compile: needs third-party library header `nlohmann/json.hpp` -->
```cpp
#include <nlohmann/json.hpp>
#include <iostream>
#include <vector>
#include <iomanip>

using json = nlohmann::json;

int main() {
    json data = {
        {"name", "Alice"},
        {"age", 30},
        {"scores", {95, 87, 92}},
        {"active", true}
    };

    // JSON (text)
    auto json_str = data.dump();
    std::cout << "JSON size: " << json_str.size() << " bytes\n";
    // ~55 bytes

    // MessagePack (binary)
    auto msgpack = json::to_msgpack(data);
    std::cout << "MessagePack size: " << msgpack.size() << " bytes\n";
    // ~35 bytes

    // CBOR (binary)
    auto cbor = json::to_cbor(data);
    std::cout << "CBOR size: " << cbor.size() << " bytes\n";
    // ~36 bytes

    // Deserialize MessagePack
    auto restored = json::from_msgpack(msgpack);
    std::cout << "\nRestored: " << restored.dump(2) << "\n";

    // Hex dump of MessagePack
    std::cout << "\nMessagePack hex: ";
    for (auto b : msgpack) {
        std::cout << std::hex << std::setw(2) << std::setfill('0')
                  << static_cast<int>(b) << " ";
    }
    std::cout << "\n";
}
```

After this runs, the round-trip through `from_msgpack` gives you back the exact same `json` object you started with. That is the self-describing property at work.

### msgpack-c Library (High Performance)

When you need maximum throughput, the dedicated `msgpack-c` library is faster than going through nlohmann/json because it serializes directly from your C++ structs. The `MSGPACK_DEFINE` macro tells it which members to include - that is the only annotation you need to add to your type:

```cpp
#include <msgpack.hpp>
#include <iostream>
#include <string>
#include <vector>
#include <sstream>

struct Sensor {
    std::string id;
    double value;
    uint64_t timestamp;
    MSGPACK_DEFINE(id, value, timestamp);
};

void msgpack_c_example() {
    // Serialize
    Sensor s{"temp-01", 23.5, 1700000000};
    msgpack::sbuffer buf;
    msgpack::pack(buf, s);
    std::cout << "Packed size: " << buf.size() << " bytes\n";

    // Deserialize
    auto handle = msgpack::unpack(buf.data(), buf.size());
    Sensor restored;
    handle.get().convert(restored);
    std::cout << "Sensor: " << restored.id
              << " value=" << restored.value << "\n";

    // Streaming: pack multiple objects into one buffer
    msgpack::sbuffer stream;
    for (int i = 0; i < 5; ++i) {
        Sensor si{"sensor-" + std::to_string(i), i * 1.1, 0};
        msgpack::pack(stream, si);
    }
    std::cout << "Stream size (5 sensors): " << stream.size() << " bytes\n";
}
```

The streaming example at the end is important for time-series use cases: you can pack many objects sequentially into one buffer and a reader can unpack them one by one without knowing the total count in advance.

---

## Self-Assessment

### Q1: Show the size advantage of MessagePack over JSON for a typical API response

This benchmark builds a 100-user API response - the kind of payload you might see from a REST endpoint - and compares the three formats:

<!-- compile: needs third-party library header `nlohmann/json.hpp` -->
```cpp
#include <nlohmann/json.hpp>
#include <iostream>

int main() {
    // Typical API response
    nlohmann::json response;
    for (int i = 0; i < 100; ++i) {
        response.push_back({
            {"id", i},
            {"name", "User " + std::to_string(i)},
            {"email", "user" + std::to_string(i) + "@example.com"},
            {"score", 75.5 + i},
            {"active", i % 3 != 0}
        });
    }

    auto json_bytes = response.dump().size();
    auto msgpack_bytes = nlohmann::json::to_msgpack(response).size();
    auto cbor_bytes = nlohmann::json::to_cbor(response).size();

    std::cout << "JSON:        " << json_bytes << " bytes\n";
    std::cout << "MessagePack: " << msgpack_bytes << " bytes ("
              << (100 - 100 * msgpack_bytes / json_bytes) << "% smaller)\n";
    std::cout << "CBOR:        " << cbor_bytes << " bytes ("
              << (100 - 100 * cbor_bytes / json_bytes) << "% smaller)\n";
}
```

The savings come mainly from numeric fields (IDs, scores) that JSON represents as variable-length ASCII strings, while MessagePack and CBOR represent as fixed-width binary integers.

### Q2: When to choose MessagePack vs CBOR

- **MessagePack**: simpler spec, more language bindings, widely used in Redis/Fluentd/MessagePack-RPC
- **CBOR**: IETF standard (RFC 8949), supports tags for dates/bignum/etc., used in COSE/FIDO2/WebAuthn
- Both are roughly equivalent in size and performance

If you are integrating with an existing ecosystem (Redis, Fluentd), MessagePack is the practical choice. If you need standardized type extensions like proper date representation or are working in a security context (FIDO2, COSE), CBOR's tag system gives you those for free.

### Q3: Handle streaming serialization of time-series data

For very high-volume time-series writes, use the low-level `msgpack::packer` API directly. It lets you write arrays without materializing an intermediate object, which keeps memory usage flat no matter how many records you write:

```cpp
// Write 1M sensor readings into a single buffer efficiently
#include <msgpack.hpp>
#include <chrono>

void stream_timeseries(const char* filename) {
    msgpack::sbuffer buf;
    for (int i = 0; i < 1'000'000; ++i) {
        msgpack::packer<msgpack::sbuffer> pk(&buf);
        pk.pack_array(3);
        pk.pack(i);           // sequence
        pk.pack(23.5 + i * 0.001); // value
        pk.pack(1700000000ULL + i); // timestamp
    }
    // Write buf to file - readers can unpack sequentially
    // No need to load entire file into memory
}
```

---

## Notes

- For nlohmann/json, MessagePack and CBOR support is built-in - no extra dependencies.
- For maximum performance, use the dedicated msgpack-c library with `MSGPACK_DEFINE` macros.
- CBOR tags (tag 0 = date string, tag 1 = epoch) provide standardized type extensions.
- Both formats handle nested objects, arrays, null, and binary data natively.
