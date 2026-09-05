# Design data serialization architecture with protobuf, flatbuffers, or msgpack

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Serialization** is the process of converting an in-memory object into a sequence of bytes for storage or transmission, and deserialization is the reverse. Every distributed system, persistent store, and inter-process channel needs to do this. Choosing the wrong format can cost you performance, break schema evolution, or tie your codebase to a single language.

In C++ production work, protobuf, FlatBuffers, and msgpack cover most use cases. The high-level tradeoffs are: protobuf is the safe default with excellent schema evolution; FlatBuffers is for latency-critical paths where you want to read fields directly from the buffer without a parse step; msgpack is for simple cases where you want compact binary with no schema files.

### Serialization Formats Comparison

| Format | Schema | Zero-copy | Size | Speed | Schema Evolution |
| --- | --- | --- | --- | --- | --- |
| **Protobuf** | Required (.proto) | No | Small | Fast | Excellent (field numbers) |
| **FlatBuffers** | Required (.fbs) | Yes | Small-Medium | Fastest | Good |
| **msgpack** | Optional | No | Smallest | Fast | Manual |
| **JSON** | Optional | No | Large | Slow | Fragile |
| **Cap'n Proto** | Required | Yes | Small | Fastest | Good |

---

## Self-Assessment

### Q1: Design a serialization abstraction layer

**Answer:**

The key architectural principle here is that your domain types should know nothing about serialization. `SensorReading` has no protobuf dependency, no FlatBuffers import, nothing. The `ISerializer` interface is the seam between your domain and the wire format. If you decide to switch from protobuf to FlatBuffers later, you swap the implementation, not the domain code.

```cpp
#include <vector>
#include <string>
#include <cstdint>
#include <optional>
#include <span>

// === Domain types (serialization-agnostic) ===
struct SensorReading {
    uint32_t sensor_id;
    double temperature;
    double humidity;
    uint64_t timestamp_ms;
    std::string location;
    std::vector<double> raw_samples;
};

// === Serializer interface ===
class ISerializer {
public:
    virtual ~ISerializer() = default;
    virtual std::vector<uint8_t> serialize(
        const SensorReading& reading) const = 0;
    virtual std::optional<SensorReading> deserialize(
        std::span<const uint8_t> data) const = 0;
    virtual const char* format_name() const = 0;
};

// === Protobuf implementation ===
// sensor.proto:
// syntax = "proto3";
// message SensorReading {
//   uint32 sensor_id = 1;
//   double temperature = 2;
//   double humidity = 3;
//   uint64 timestamp_ms = 4;
//   string location = 5;
//   repeated double raw_samples = 6;
// }

class ProtobufSerializer : public ISerializer {
public:
    std::vector<uint8_t> serialize(
            const SensorReading& reading) const override {
        proto::SensorReading msg;
        msg.set_sensor_id(reading.sensor_id);
        msg.set_temperature(reading.temperature);
        msg.set_humidity(reading.humidity);
        msg.set_timestamp_ms(reading.timestamp_ms);
        msg.set_location(reading.location);
        for (double s : reading.raw_samples)
            msg.add_raw_samples(s);

        std::vector<uint8_t> buffer(msg.ByteSizeLong());
        msg.SerializeToArray(buffer.data(), buffer.size());
        return buffer;
    }

    std::optional<SensorReading> deserialize(
            std::span<const uint8_t> data) const override {
        proto::SensorReading msg;
        if (!msg.ParseFromArray(data.data(), data.size()))
            return std::nullopt;

        return SensorReading{
            msg.sensor_id(), msg.temperature(), msg.humidity(),
            msg.timestamp_ms(), msg.location(),
            {msg.raw_samples().begin(), msg.raw_samples().end()}
        };
    }

    const char* format_name() const override { return "protobuf"; }
};
```

Notice that `deserialize` returns `std::optional<SensorReading>`. That is the right return type for an operation that can fail - corrupted data, truncated input, wrong format. Returning a null optional is more informative than throwing and silences the need for try/catch at every call site.

### Q2: FlatBuffers zero-copy implementation

**Answer:**

FlatBuffers is designed so that the serialized bytes can be read directly without any parsing step. The generated accessor functions (`fb->temperature()`, `fb->sensor_id()`) read directly from the buffer memory. That means if you receive a message over the network, you can read fields from it without any allocation or deserialization work at all - you just keep a pointer to the received buffer.

```cpp
// === sensor.fbs ===
// namespace Sensor;
// table Reading {
//   sensor_id: uint32;
//   temperature: double;
//   humidity: double;
//   timestamp_ms: uint64;
//   location: string;
//   raw_samples: [double];
// }
// root_type Reading;

#include "sensor_generated.h"  // flatc-generated

class FlatBufferSerializer : public ISerializer {
public:
    std::vector<uint8_t> serialize(
            const SensorReading& reading) const override {
        flatbuffers::FlatBufferBuilder builder(256);

        auto location = builder.CreateString(reading.location);
        auto samples = builder.CreateVector(reading.raw_samples);

        auto fb = Sensor::CreateReading(builder,
            reading.sensor_id,
            reading.temperature,
            reading.humidity,
            reading.timestamp_ms,
            location,
            samples);

        builder.Finish(fb);
        auto* buf = builder.GetBufferPointer();
        return {buf, buf + builder.GetSize()};
    }

    std::optional<SensorReading> deserialize(
            std::span<const uint8_t> data) const override {
        // Zero-copy: directly access buffer without parsing!
        auto* fb = Sensor::GetReading(data.data());
        if (!fb) return std::nullopt;

        SensorReading result;
        result.sensor_id = fb->sensor_id();
        result.temperature = fb->temperature();
        result.humidity = fb->humidity();
        result.timestamp_ms = fb->timestamp_ms();
        if (fb->location())
            result.location = fb->location()->str();
        if (fb->raw_samples())
            result.raw_samples.assign(
                fb->raw_samples()->begin(),
                fb->raw_samples()->end());
        return result;
    }

    // Zero-copy access without full deserialization:
    static double get_temperature(std::span<const uint8_t> data) {
        // No allocation, no parsing - direct memory read
        return Sensor::GetReading(data.data())->temperature();
    }

    const char* format_name() const override { return "flatbuffers"; }
};
```

The `get_temperature` static function is the clearest demonstration of the zero-copy benefit. You pass in a raw buffer and read a single field directly, with no intermediate object constructed. For high-frequency telemetry or trading systems where you need to inspect just one or two fields of a large message, this is a significant advantage.

### Q3: Handle schema evolution and versioning

**Answer:**

Schema evolution is what happens when you need to change the structure of a message type over time, while both old and new code may be reading and writing messages. This is one of the hardest parts of any distributed system design. Protobuf handles it well because its wire format uses field numbers, not field names - adding a new field with a new number is fully backward compatible, and old readers simply ignore fields with numbers they do not know.

```cpp
// === Schema evolution rules ===

// Protobuf: SAFE changes
// - Add new fields (with new field numbers)
// - Remove fields (old readers ignore unknown fields)
// - Rename fields (wire format uses numbers, not names)

// Protobuf: BREAKING changes
// - Change field number
// - Change field type (int32 -> string)
// - Remove field number and reuse it

// === Versioned serialization architecture ===
struct MessageHeader {
    uint8_t version;       // Schema version
    uint8_t format;        // 0=protobuf, 1=flatbuf, 2=msgpack
    uint16_t message_type; // Application-defined type ID
    uint32_t payload_size; // Payload length in bytes
};
static_assert(sizeof(MessageHeader) == 8);

class VersionedSerializer {
public:
    std::vector<uint8_t> serialize(uint16_t type,
                                    const std::vector<uint8_t>& payload) {
        MessageHeader header{
            .version = CURRENT_VERSION,
            .format = static_cast<uint8_t>(format_),
            .message_type = type,
            .payload_size = static_cast<uint32_t>(payload.size())
        };

        std::vector<uint8_t> result(sizeof(header) + payload.size());
        std::memcpy(result.data(), &header, sizeof(header));
        std::memcpy(result.data() + sizeof(header),
                    payload.data(), payload.size());
        return result;
    }

    // Deserialize with version handling
    template<typename T>
    std::optional<T> deserialize(std::span<const uint8_t> data,
                                  ISerializer& serializer) {
        if (data.size() < sizeof(MessageHeader))
            return std::nullopt;

        MessageHeader header;
        std::memcpy(&header, data.data(), sizeof(header));

        if (header.version > CURRENT_VERSION) {
            // Future version: try to read compatible subset
            // Protobuf/FlatBuffers handle this gracefully
        }

        auto payload = data.subspan(sizeof(header), header.payload_size);
        return serializer.deserialize(payload);
    }

    static constexpr uint8_t CURRENT_VERSION = 3;
    int format_ = 0;
};
```

The `static_assert(sizeof(MessageHeader) == 8)` is a small but important detail. It verifies that the header struct has no unexpected padding that would make it the wrong size on certain platforms. Without this check, a struct that happens to be padded to 12 bytes on one platform would silently corrupt messages received by a system that expects an 8-byte header.

---

## Notes

- **Protobuf** is the default choice for most projects: it has excellent schema evolution, wide language support, and solid performance.
- **FlatBuffers** is for latency-critical paths where zero-copy access is important and you need to avoid deserialization entirely.
- **msgpack** works well for simple cases where you want compact binary without the overhead of schema files.
- Never use raw struct `memcpy` for serialization across machines - endianness, struct padding, and alignment are all platform-dependent.
- Always version your wire format by adding a version byte to the header - this enables graceful migration when schemas change.
- The serialization layer belongs in infrastructure code, not in the domain. Domain types should never include protobuf headers directly.
- For IPC within the same process or machine, FlatBuffers or shared memory can avoid serialization entirely by sharing memory directly.
