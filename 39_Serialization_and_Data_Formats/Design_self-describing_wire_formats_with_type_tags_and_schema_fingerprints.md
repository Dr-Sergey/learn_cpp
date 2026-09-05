# Design self-describing wire formats with type tags and schema fingerprints

**Category:** Serialization and Data Formats  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value>  

---

## Topic Overview

A **self-describing** wire format carries enough metadata for a reader to interpret the data without prior schema agreement. The key idea is that every field on the wire announces its own type - so a reader can traverse and decode the message even without a schema document in hand. Type tags identify field types, and schema fingerprints allow fast compatibility checks between sender and receiver.

This is different from a schema-based format like Protocol Buffers, where both sides share a compiled `.proto` file and the wire bytes contain only field numbers and raw values, with no embedded type information. Self-describing formats trade some extra wire overhead for a lot of flexibility.

### Type Tag System

The example below builds a minimal self-describing wire format from scratch. Each field is prefixed with a `FieldHeader` that carries the wire type, field ID, and data length. A reader can process any message from this format, printing values, by walking the headers - even if it has never seen this particular schema before.

```cpp
#include <cstdint>
#include <string>
#include <variant>
#include <vector>
#include <iostream>
#include <cstring>

enum class WireType : uint8_t {
    Uint32   = 0,
    Int32    = 1,
    Float64  = 2,
    String   = 3,
    Bytes    = 4,
    Bool     = 5,
    Array    = 6,  // Followed by element WireType + count
    Null     = 7,
};

// Each field on the wire:
//   [1 byte] WireType
//   [2 bytes] field_id
//   [4 bytes] length (for variable-length types)
//   [N bytes] data

struct FieldHeader {
    WireType  type;
    uint16_t  field_id;
    uint32_t  data_length;  // 0 for fixed-size types
};

class SelfDescribingWriter {
    std::vector<uint8_t> buf_;
public:
    void write_uint32(uint16_t field_id, uint32_t val) {
        FieldHeader h{WireType::Uint32, field_id, sizeof(val)};
        append(h);
        append(val);
    }

    void write_string(uint16_t field_id, const std::string& s) {
        FieldHeader h{WireType::String, field_id, static_cast<uint32_t>(s.size())};
        append(h);
        buf_.insert(buf_.end(), s.begin(), s.end());
    }

    void write_float64(uint16_t field_id, double val) {
        FieldHeader h{WireType::Float64, field_id, sizeof(val)};
        append(h);
        append(val);
    }

    const std::vector<uint8_t>& buffer() const { return buf_; }

private:
    template<typename T>
    void append(const T& v) {
        auto p = reinterpret_cast<const uint8_t*>(&v);
        buf_.insert(buf_.end(), p, p + sizeof(T));
    }
};

class SelfDescribingReader {
    const uint8_t* data_;
    size_t size_, pos_ = 0;
public:
    SelfDescribingReader(const uint8_t* data, size_t size)
        : data_(data), size_(size) {}

    bool has_next() const { return pos_ + sizeof(FieldHeader) <= size_; }

    void read_next() {
        FieldHeader h;
        std::memcpy(&h, data_ + pos_, sizeof(h));
        pos_ += sizeof(h);

        std::cout << "Field " << h.field_id << " [";
        switch (h.type) {
            case WireType::Uint32: {
                uint32_t val;
                std::memcpy(&val, data_ + pos_, sizeof(val));
                std::cout << "uint32]: " << val;
                break;
            }
            case WireType::String: {
                std::string s(reinterpret_cast<const char*>(data_ + pos_), h.data_length);
                std::cout << "string]: \"" << s << "\"";
                break;
            }
            case WireType::Float64: {
                double val;
                std::memcpy(&val, data_ + pos_, sizeof(val));
                std::cout << "float64]: " << val;
                break;
            }
            default:
                std::cout << "unknown]: skipping " << h.data_length << " bytes";
        }
        std::cout << "\n";
        pos_ += h.data_length;
    }
};

int main() {
    SelfDescribingWriter w;
    w.write_string(1, "Alice");
    w.write_uint32(2, 30);
    w.write_float64(3, 98.6);

    SelfDescribingReader r(w.buffer().data(), w.buffer().size());
    while (r.has_next()) r.read_next();
    // Output:
    //   Field 1 [string]: "Alice"
    //   Field 2 [uint32]: 30
    //   Field 3 [float64]: 98.6
}
```

Watch the `default:` branch in `read_next()` - that is forward compatibility in action. A reader that encounters an unknown `WireType` can still skip over the field correctly using `data_length`, and then continue reading the next field. This is the same principle that makes Protocol Buffers forward-compatible, just expressed in hand-rolled code.

---

## Self-Assessment

### Q1: Compare self-describing formats vs schema-based formats

The trade-off comes down to payload size versus decoupling. Self-describing formats are larger but more autonomous; schema-based formats are leaner but require both sides to share the schema definition out-of-band.

| Aspect | Self-Describing | Schema-Based |
| --- | --- | --- |
| Wire overhead | Larger (type info per field) | Smaller (types known a priori) |
| Reader needs schema? | No | Yes |
| Debugging | Easy (can dump any message) | Need schema to decode |
| Performance | Slightly slower (type checks) | Faster (no type dispatch) |
| Use case | Logging, debugging, ad-hoc | RPC, databases, high-throughput |

### Q2: Implement a schema fingerprint from field definitions

A schema fingerprint is a hash of the set of `(field_id, type)` pairs. The idea is that if two sides have the same fingerprint, they are speaking the same schema version. If fingerprints differ, the reader can decide whether to attempt a compatible read or reject the message outright. Here is a simple implementation using a combine-style hash:

```cpp
#include <cstdint>
#include <iostream>
#include <vector>

uint64_t hash_combine(uint64_t seed, uint64_t value) {
    seed ^= value + 0x9e3779b97f4a7c15 + (seed << 6) + (seed >> 2);
    return seed;
}

struct FieldSchema {
    uint16_t id;
    WireType type;
    bool required;
};

uint64_t compute_fingerprint(const std::vector<FieldSchema>& schema) {
    uint64_t fp = 0;
    for (auto& f : schema) {
        fp = hash_combine(fp, f.id);
        fp = hash_combine(fp, static_cast<uint64_t>(f.type));
    }
    return fp;
}

int main() {
    std::vector<FieldSchema> v1 = {{1, WireType::String, true}, {2, WireType::Uint32, true}};
    std::vector<FieldSchema> v2 = {{1, WireType::String, true}, {2, WireType::Uint32, true},
                                    {3, WireType::Float64, false}};

    std::cout << "v1: 0x" << std::hex << compute_fingerprint(v1) << "\n";
    std::cout << "v2: 0x" << std::hex << compute_fingerprint(v2) << "\n";
}
```

The two fingerprints will differ, which tells a reader that the schemas are not identical - even though v2 is a backward-compatible superset of v1.

### Q3: When should you prefer self-describing formats in production

Self-describing formats are preferred when:

- **Log aggregation**: heterogeneous log types from many services where you cannot guarantee every consumer has the latest schema.
- **Schema registry unavailable**: ad-hoc communication between independently deployed components.
- **Debugging tools**: message inspectors that must decode without preloaded schemas.
- **Migration periods**: transitioning between schema versions where both old and new messages coexist in the same stream.
- **Event sourcing**: stored events must be readable decades later without the original compiled code.

---

## Notes

- CBOR and MessagePack are standardized self-describing binary formats - prefer them over a custom TLV implementation.
- For high-throughput paths, schema-based formats (FlatBuffers, Protobuf) are better due to lower overhead.
- A hybrid approach is also common: embed a schema fingerprint in the message header, and carry type tags only for unknown extensions.
