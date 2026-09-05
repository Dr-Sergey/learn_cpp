# Design versioned binary protocols with backward and forward compatibility

**Category:** Serialization and Data Formats  
**Standard:** C++17, C++20  
**Reference:** <https://en.wikipedia.org/wiki/Wire_protocol>  

---

## Topic Overview

Binary protocols must evolve without breaking existing readers or writers. **Backward compatibility** means new readers can handle old data. **Forward compatibility** means old readers can handle new data gracefully, typically by skipping unknown fields. Getting both right requires deliberate design from day one - retrofitting compatibility into an existing protocol is painful.

The reason this trips people up is that it is easy to think "I control both sides, so I can update them together." In practice there are always old binaries still running, old log files waiting to be replayed, or a client you forgot to update.

### Key Design Principles

The table below captures the techniques that make both directions of compatibility possible. The single most important one is tagged fields with length-prefixing - it is what allows old readers to skip fields they do not recognise.

| Principle | Technique |
| --- | --- |
| Version tagging | Magic bytes + version number at the start |
| Tagged fields | Each field carries its ID and type |
| Length-prefixing | Unknown fields can be skipped by length |
| Never reuse field IDs | Deprecated fields leave their ID permanently reserved |
| Default values | Missing fields get sensible defaults |

### Wire Format Header

Here is a minimal TLV (Type-Length-Value) protocol implementation. The magic number at the start catches accidental misreads (e.g. reading a data file as a protocol stream). The version field lets the reader decide how to handle newer messages. The `skip()` method is the key forward-compatibility primitive - it advances over any field the reader does not understand.

```cpp
#include <cstdint>
#include <vector>
#include <cstring>
#include <stdexcept>
#include <iostream>

// Simple TLV (Type-Length-Value) protocol
struct WireHeader {
    uint32_t magic = 0x4350504D;  // "CPPM"
    uint16_t version = 1;
    uint16_t reserved = 0;
};

struct TLVField {
    uint16_t tag;
    uint32_t length;
    // followed by `length` bytes of data
};

class MessageWriter {
    std::vector<uint8_t> buffer_;
public:
    MessageWriter() {
        WireHeader hdr{};
        append_raw(hdr);
    }

    void add_uint32(uint16_t tag, uint32_t value) {
        TLVField f{tag, sizeof(value)};
        append_raw(f);
        append_raw(value);
    }

    void add_string(uint16_t tag, const std::string& value) {
        TLVField f{tag, static_cast<uint32_t>(value.size())};
        append_raw(f);
        buffer_.insert(buffer_.end(), value.begin(), value.end());
    }

    const std::vector<uint8_t>& data() const { return buffer_; }

private:
    template<typename T>
    void append_raw(const T& val) {
        auto p = reinterpret_cast<const uint8_t*>(&val);
        buffer_.insert(buffer_.end(), p, p + sizeof(T));
    }
};

class MessageReader {
    const uint8_t* data_;
    size_t size_;
    size_t pos_ = 0;
public:
    MessageReader(const uint8_t* data, size_t size)
        : data_(data), size_(size) {
        WireHeader hdr{};
        read_raw(hdr);
        if (hdr.magic != 0x4350504D)
            throw std::runtime_error("Invalid magic");
        // Version check - forward compatible: accept >= our version
        std::cout << "Protocol version: " << hdr.version << "\n";
    }

    // Returns false when no more fields
    bool next_field(TLVField& field) {
        if (pos_ + sizeof(TLVField) > size_) return false;
        read_raw(field);
        return true;
    }

    uint32_t read_uint32() {
        uint32_t val;
        read_raw(val);
        return val;
    }

    std::string read_string(uint32_t len) {
        std::string s(reinterpret_cast<const char*>(data_ + pos_), len);
        pos_ += len;
        return s;
    }

    void skip(uint32_t len) { pos_ += len; }  // Skip unknown fields!

private:
    template<typename T>
    void read_raw(T& val) {
        if (pos_ + sizeof(T) > size_)
            throw std::runtime_error("Buffer underrun");
        std::memcpy(&val, data_ + pos_, sizeof(T));
        pos_ += sizeof(T);
    }
};
```

Notice the comment on `skip()` - that single method is what makes forward compatibility work. An old reader encountering a field with an unrecognised tag can call `skip(field.length)` and land cleanly at the start of the next field.

### Version Evolution Example

This shows the progression across three versions of a message. The critical discipline here is that tag 2 (`TAG_AGE`) is never reassigned even though the field was dropped in v3. A v1 reader receiving a v3 message will simply not see tag 2 in the stream and produce a sensible default, while a v3 reader receiving a v1 message will encounter the old tag and print it normally.

```cpp
// Version 1 message: Name (tag=1) + Age (tag=2)
// Version 2 adds:    Email (tag=3) - old readers skip tag=3
// Version 3 removes: Age is deprecated - but tag=2 is NEVER reused

enum FieldTag : uint16_t {
    TAG_NAME  = 1,
    TAG_AGE   = 2,  // Deprecated in v3, but tag 2 is reserved forever
    TAG_EMAIL = 3,  // Added in v2
    TAG_PHONE = 4,  // Added in v3
};

void write_v3_message() {
    MessageWriter w;
    w.add_string(TAG_NAME, "Alice");
    // TAG_AGE omitted in v3
    w.add_string(TAG_EMAIL, "alice@example.com");
    w.add_string(TAG_PHONE, "+1-555-0100");

    auto& data = w.data();
    std::cout << "Message size: " << data.size() << " bytes\n";
}

void read_message_any_version(const uint8_t* data, size_t len) {
    MessageReader r(data, len);
    TLVField field;
    while (r.next_field(field)) {
        switch (field.tag) {
            case TAG_NAME:
                std::cout << "Name: " << r.read_string(field.length) << "\n";
                break;
            case TAG_AGE:
                std::cout << "Age: " << r.read_uint32() << "\n";
                break;
            case TAG_EMAIL:
                std::cout << "Email: " << r.read_string(field.length) << "\n";
                break;
            default:
                // Forward compatibility: skip unknown fields
                std::cout << "Skipping unknown tag " << field.tag
                          << " (" << field.length << " bytes)\n";
                r.skip(field.length);
                break;
        }
    }
}
```

The `default:` case is the forward-compatibility mechanism. A v1 reader processing a v3 message will hit this branch for `TAG_PHONE` (tag 4, which it does not know about) and skip it cleanly, printing a diagnostic. This is far better than crashing or corrupting the read position.

---

## Self-Assessment

### Q1: Design a header that allows readers to skip entire messages they don't understand

Sometimes you want message-level skipping, not just field-level. The pattern is to length-prefix every message at the outermost envelope level, so a reader can jump over an entire unknown message type in one move.

```cpp
#include <cstdint>
#include <vector>
#include <cstring>
#include <iostream>

// Each message starts with:
//   [4 bytes] message_type
//   [4 bytes] total_message_length (excluding this 8-byte prefix)
//   [payload...]
//
// An old reader encountering an unknown message_type can skip
// total_message_length bytes to reach the NEXT message.

struct MessageEnvelope {
    uint32_t message_type;
    uint32_t payload_length;  // Length of everything after this header
};

void write_messages(std::vector<uint8_t>& buf) {
    // Message type 1: a known type
    MessageEnvelope env1{1, 4};
    auto p1 = reinterpret_cast<const uint8_t*>(&env1);
    buf.insert(buf.end(), p1, p1 + sizeof(env1));
    uint32_t val = 42;
    auto pv = reinterpret_cast<const uint8_t*>(&val);
    buf.insert(buf.end(), pv, pv + 4);

    // Message type 99: unknown future type
    MessageEnvelope env2{99, 8};
    auto p2 = reinterpret_cast<const uint8_t*>(&env2);
    buf.insert(buf.end(), p2, p2 + sizeof(env2));
    uint64_t future_data = 0xDEADBEEF;
    auto pf = reinterpret_cast<const uint8_t*>(&future_data);
    buf.insert(buf.end(), pf, pf + 8);
}

void read_messages(const uint8_t* data, size_t len) {
    size_t pos = 0;
    while (pos + sizeof(MessageEnvelope) <= len) {
        MessageEnvelope env;
        std::memcpy(&env, data + pos, sizeof(env));
        pos += sizeof(env);

        if (env.message_type == 1) {
            uint32_t val;
            std::memcpy(&val, data + pos, 4);
            std::cout << "Known message: value=" << val << "\n";
        } else {
            std::cout << "Unknown type " << env.message_type
                      << ", skipping " << env.payload_length << " bytes\n";
        }
        pos += env.payload_length;  // Always advance by payload_length
    }
}

int main() {
    std::vector<uint8_t> buf;
    write_messages(buf);
    read_messages(buf.data(), buf.size());
    // Output:
    //   Known message: value=42
    //   Unknown type 99, skipping 8 bytes
}
```

The key line is `pos += env.payload_length` outside the `if/else`. The reader always advances past the payload, whether it understood the message or not. This is what makes the stream robust to future message types.

### Q2: Show why reusing a deprecated field tag leads to data corruption

This is the most important thing to understand about wire format evolution. When you recycle a tag, old readers that were written when that tag meant something else will interpret the new bytes using the old type. The result is not an error - it is silent garbage.

```cpp
#include <cstdint>
#include <iostream>

// WRONG: Reusing tag 2
// v1: tag 2 = age (uint32_t)
// v3: tag 2 = temperature (float)  <- REUSED! DANGEROUS!
//
// A v1 reader encountering v3 data would interpret the float bytes
// as a uint32_t - silent data corruption!

void demonstrate_tag_reuse_danger() {
    float temperature = 98.6f;
    uint32_t misinterpreted;
    static_assert(sizeof(float) == sizeof(uint32_t));
    std::memcpy(&misinterpreted, &temperature, sizeof(float));

    std::cout << "Temperature (float): " << temperature << "\n";
    std::cout << "Misread as uint32 (age): " << misinterpreted << "\n";
    // Output:
    //   Temperature (float): 98.6
    //   Misread as uint32 (age): 1120379084  <- nonsense!
}

// CORRECT approach: deprecated tags are reserved forever
// v1: tag 2 = age (uint32_t)
// v3: tag 2 = RESERVED (do not use)
//     tag 5 = temperature (float)  <- NEW tag number

int main() {
    demonstrate_tag_reuse_danger();
}
```

The number `1120379084` is just the bit pattern of `98.6f` reinterpreted as a 32-bit integer. There is no error, no exception - just a plausible-looking but completely wrong number stored in a field called "age". This kind of corruption can persist in databases for months before anyone notices.

### Q3: Implement a simple schema fingerprint for version detection

A schema fingerprint gives the reader a fast way to detect whether it is speaking the same schema as the writer, before attempting to parse anything. Here is a simple FNV-1a based implementation:

```cpp
#include <cstdint>
#include <string>
#include <iostream>
#include <numeric>

// A schema fingerprint hashes the set of (tag, type) pairs.
// Readers compare fingerprints to detect incompatible schemas.

uint32_t fnv1a_hash(const void* data, size_t len) {
    auto bytes = static_cast<const uint8_t*>(data);
    uint32_t hash = 2166136261u;
    for (size_t i = 0; i < len; ++i) {
        hash ^= bytes[i];
        hash *= 16777619u;
    }
    return hash;
}

struct FieldDef {
    uint16_t tag;
    uint16_t type;  // 0=uint32, 1=string, 2=float, ...
};

uint32_t compute_schema_fingerprint(const FieldDef* fields, size_t count) {
    uint32_t hash = 2166136261u;
    for (size_t i = 0; i < count; ++i) {
        auto p = reinterpret_cast<const uint8_t*>(&fields[i]);
        for (size_t j = 0; j < sizeof(FieldDef); ++j) {
            hash ^= p[j];
            hash *= 16777619u;
        }
    }
    return hash;
}

int main() {
    FieldDef schema_v1[] = {{1, 1}, {2, 0}};         // name(str), age(u32)
    FieldDef schema_v2[] = {{1, 1}, {2, 0}, {3, 1}}; // + email(str)

    auto fp1 = compute_schema_fingerprint(schema_v1, 2);
    auto fp2 = compute_schema_fingerprint(schema_v2, 3);

    std::cout << "v1 fingerprint: 0x" << std::hex << fp1 << "\n";
    std::cout << "v2 fingerprint: 0x" << std::hex << fp2 << "\n";
    std::cout << "Match: " << (fp1 == fp2 ? "yes" : "no") << "\n";
    // Different fingerprints - schemas differ
    // Reader can decide: attempt compatible read or reject
}
```

Different fingerprints mean "these schemas are not identical." The reader can then decide: attempt a compatible read (if it is willing to handle missing or extra fields), or reject the message and report a version mismatch. Either decision is better than silently parsing with the wrong schema.

---

## Notes

- **Never reuse field tag IDs** - this is the single most important rule for binary protocol evolution.
- **TLV (Type-Length-Value)** is the simplest approach; Protocol Buffers and FlatBuffers use more efficient varint encodings.
- **Endianness**: always specify wire byte order (little-endian is common on modern hardware).
- **Length-prefix every variable-length field** so unknown fields can be skipped cleanly.
- Consider using established formats (Protobuf, FlatBuffers, Cap'n Proto) instead of rolling your own.
