# Implement a binary protocol framer with length-prefixed messages

**Category:** Networking and IO  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/from_chars>  

---

## Topic Overview

TCP is a byte stream, not a message stream. When you send 100 bytes, the receiver might get them as a single 100-byte chunk, or as two 50-byte chunks, or as 100 individual bytes. There is no concept of a message boundary at the TCP level. Your protocol has to impose that structure itself, and the mechanism for doing so is called **framing**.

Length-prefix framing is the simplest and most common approach: you prepend a fixed-size header (typically 4 bytes) that tells the receiver how many bytes follow. The receiver reads the header, learns the payload length, and then reads exactly that many bytes. This is binary-safe (works for any payload content), O(1) to parse, and straightforward to implement. The alternatives - delimiter-based framing (e.g., newline-terminated) and TLV (Type-Length-Value) - have their uses but come with extra complexity.

The diagram below shows both the wire format and the partial-read challenge, which is the part that catches new developers off guard. TCP can deliver your data in chunks that do not align with your message boundaries. A 4-byte header might arrive as 2 bytes in one `recv` call and 2 bytes in the next. Your framer must accumulate bytes until it has enough to make progress.

```cpp
Wire format:
  +--------+--------+--------+--------+---...---+
  | len[0] | len[1] | len[2] | len[3] | payload |
  +--------+--------+--------+--------+---...---+
  |<--- 4 bytes (big-endian) --->|<-- len bytes ->|

Partial read handling:
  recv: [len0 len1]           -> need 2 more header bytes
  recv: [len2 len3 pay0 pay1] -> header complete, need (len-2) payload bytes
  recv: [pay2 ... payN]       -> message complete, dispatch!
```

| Framing method | Pros | Cons |
| --- | --- | --- |
| Length-prefix (4B) | O(1) parse, binary-safe | Fixed header overhead |
| Delimiter (\n) | Simple for text | Must escape delimiter in payload |
| TLV (Type-Length-Value) | Self-describing | More complex parsing |
| Varint length (protobuf) | Compact for small msgs | Variable header size |

---

## Self-Assessment

### Q1: Write a framer with 4-byte big-endian length header

The `Framer` class below uses the simplest possible strategy: it appends all incoming bytes to an internal buffer and then peels off complete messages in a loop. The `feed` function returns however many complete messages became available from the new data. Watch how it handles the partial delivery in `main` - the message arrives split across three calls, and the framer assembles it transparently.

```cpp
#include <iostream>
#include <vector>
#include <cstdint>
#include <cstring>
#include <cassert>

// Encode: prepend 4-byte big-endian length header
std::vector<uint8_t> frame_message(const uint8_t* data, uint32_t len) {
    std::vector<uint8_t> frame(4 + len);
    // Big-endian length header (network byte order)
    frame[0] = static_cast<uint8_t>((len >> 24) & 0xFF);
    frame[1] = static_cast<uint8_t>((len >> 16) & 0xFF);
    frame[2] = static_cast<uint8_t>((len >> 8)  & 0xFF);
    frame[3] = static_cast<uint8_t>((len >> 0)  & 0xFF);
    std::memcpy(frame.data() + 4, data, len);
    return frame;
}

// Decode: extract length from 4-byte big-endian header
uint32_t decode_length(const uint8_t* header) {
    return (static_cast<uint32_t>(header[0]) << 24)
         | (static_cast<uint32_t>(header[1]) << 16)
         | (static_cast<uint32_t>(header[2]) << 8)
         | (static_cast<uint32_t>(header[3]) << 0);
}

// Stateful framer: accumulates bytes from partial reads
class Framer {
public:
    static constexpr uint32_t MAX_MSG_SIZE = 16 * 1024 * 1024; // 16 MB safety limit

    // Feed incoming bytes. Returns complete messages.
    std::vector<std::vector<uint8_t>> feed(const uint8_t* data, size_t len) {
        std::vector<std::vector<uint8_t>> messages;
        buf_.insert(buf_.end(), data, data + len);

        while (true) {
            if (buf_.size() < 4) break;  // need header

            uint32_t msg_len = decode_length(buf_.data());
            if (msg_len > MAX_MSG_SIZE) {
                throw std::runtime_error("Message too large");
            }
            if (buf_.size() < 4 + msg_len) break;  // need more payload

            // Complete message!
            messages.emplace_back(buf_.begin() + 4,
                                  buf_.begin() + 4 + msg_len);
            buf_.erase(buf_.begin(), buf_.begin() + 4 + msg_len);
        }
        return messages;
    }

private:
    std::vector<uint8_t> buf_;
};

int main() {
    // Create two framed messages
    std::string msg1 = "Hello";
    std::string msg2 = "World!";
    auto f1 = frame_message(reinterpret_cast<const uint8_t*>(msg1.data()), msg1.size());
    auto f2 = frame_message(reinterpret_cast<const uint8_t*>(msg2.data()), msg2.size());

    // Simulate partial delivery (split across recv calls)
    Framer framer;

    // Feed first 3 bytes (partial header)
    auto msgs = framer.feed(f1.data(), 3);
    std::cout << "After 3 bytes: " << msgs.size() << " messages\n";  // 0

    // Feed rest of msg1 + first 2 bytes of msg2
    std::vector<uint8_t> chunk;
    chunk.insert(chunk.end(), f1.begin() + 3, f1.end());
    chunk.insert(chunk.end(), f2.begin(), f2.begin() + 2);
    msgs = framer.feed(chunk.data(), chunk.size());
    std::cout << "After more bytes: " << msgs.size() << " messages\n";  // 1
    std::cout << "  msg: " << std::string(msgs[0].begin(), msgs[0].end()) << '\n';  // Hello

    // Feed rest of msg2
    msgs = framer.feed(f2.data() + 2, f2.size() - 2);
    std::cout << "After rest: " << msgs.size() << " messages\n";  // 1
    std::cout << "  msg: " << std::string(msgs[0].begin(), msgs[0].end()) << '\n';  // World!
}
// Output:
//   After 3 bytes: 0 messages
//   After more bytes: 1 messages
//     msg: Hello
//   After rest: 1 messages
//     msg: World!
```

The `MAX_MSG_SIZE` guard is not optional in production code. Without it, a malicious or buggy peer could send a length field of, say, 2 GB, and your framer would attempt to allocate that much memory before receiving a single payload byte. Always validate the length field before acting on it.

### Q2: Handle partial reads correctly

The `Framer` in Q1 uses a simple append-to-vector approach that is easy to understand but has a performance cost: every `erase` from the front of a `std::vector` is O(n) because it shifts all remaining bytes. The `EfficientFramer` below avoids this by tracking state explicitly - it never copies bytes it has already processed. Instead, it remembers where it is in the current header or body and picks up exactly where it left off.

```cpp
#include <iostream>
#include <vector>
#include <cstdint>
#include <cstring>

// Production-quality framer with circular buffer for efficiency
class EfficientFramer {
public:
    enum class State { READING_HEADER, READING_BODY };

    struct Message {
        std::vector<uint8_t> payload;
    };

    // Feed bytes, extract complete messages
    // Key insight: track state explicitly to avoid re-scanning
    std::vector<Message> on_data(const uint8_t* data, size_t len) {
        std::vector<Message> result;
        size_t offset = 0;

        while (offset < len) {
            if (state_ == State::READING_HEADER) {
                // Accumulate header bytes (need 4 total)
                size_t need = 4 - header_bytes_;
                size_t avail = len - offset;
                size_t take = std::min(need, avail);
                std::memcpy(header_ + header_bytes_, data + offset, take);
                header_bytes_ += take;
                offset += take;

                if (header_bytes_ == 4) {
                    body_len_ = (static_cast<uint32_t>(header_[0]) << 24)
                              | (static_cast<uint32_t>(header_[1]) << 16)
                              | (static_cast<uint32_t>(header_[2]) << 8)
                              | (static_cast<uint32_t>(header_[3]));

                    if (body_len_ > MAX_SIZE) {
                        throw std::runtime_error("Oversized message");
                    }
                    body_.clear();
                    body_.reserve(body_len_);
                    state_ = State::READING_BODY;
                }
            } else {
                // Accumulate body bytes
                size_t need = body_len_ - body_.size();
                size_t avail = len - offset;
                size_t take = std::min(need, avail);
                body_.insert(body_.end(), data + offset, data + offset + take);
                offset += take;

                if (body_.size() == body_len_) {
                    result.push_back(Message{std::move(body_)});
                    // Reset for next message
                    state_ = State::READING_HEADER;
                    header_bytes_ = 0;
                    body_.clear();
                }
            }
        }
        return result;
    }

private:
    static constexpr uint32_t MAX_SIZE = 16 * 1024 * 1024;
    State state_ = State::READING_HEADER;
    uint8_t header_[4]{};
    size_t header_bytes_ = 0;
    uint32_t body_len_ = 0;
    std::vector<uint8_t> body_;
};

int main() {
    EfficientFramer framer;

    // Simulate byte-at-a-time delivery (worst case for partial reads)
    uint8_t wire[] = {
        0, 0, 0, 3, 'H', 'i', '!',            // msg1: "Hi!"
        0, 0, 0, 5, 'W', 'o', 'r', 'l', 'd'   // msg2: "World"
    };

    for (size_t i = 0; i < sizeof(wire); ++i) {
        auto msgs = framer.on_data(&wire[i], 1);
        for (auto& m : msgs) {
            std::string s(m.payload.begin(), m.payload.end());
            std::cout << "Complete message: \"" << s << "\"\n";
        }
    }
    // Output:
    //   Complete message: "Hi!"
    //   Complete message: "World"
}
```

The byte-at-a-time loop in `main` is the stress test for partial read handling. If your framer can handle one byte at a time correctly, it can handle any real-world delivery pattern. This is a good test to add to your unit tests.

### Q3: Decode length field with bit manipulation (no htons)

You have three practical options for encoding and decoding the 4-byte big-endian length header. This example shows all three and explains why the manual shift approach is preferred in most C++ codebases - it is portable, `constexpr`, and compiles down to the same instructions as the platform-specific alternatives.

```cpp
#include <iostream>
#include <cstdint>
#include <cstring>
#include <array>
#include <bit>  // C++20 std::endian

// Method 1: Manual shift (always correct, no endianness dependency)
uint32_t decode_be32_shift(const uint8_t* p) {
    return (static_cast<uint32_t>(p[0]) << 24)
         | (static_cast<uint32_t>(p[1]) << 16)
         | (static_cast<uint32_t>(p[2]) << 8)
         | (static_cast<uint32_t>(p[3]));
}

void encode_be32_shift(uint8_t* p, uint32_t val) {
    p[0] = static_cast<uint8_t>((val >> 24) & 0xFF);
    p[1] = static_cast<uint8_t>((val >> 16) & 0xFF);
    p[2] = static_cast<uint8_t>((val >> 8)  & 0xFF);
    p[3] = static_cast<uint8_t>((val)       & 0xFF);
}

// Method 2: memcpy + byte swap (avoids strict aliasing issues)
uint32_t decode_be32_memcpy(const uint8_t* p) {
    uint32_t val;
    std::memcpy(&val, p, 4);  // safe: memcpy has no alignment requirement
    // Convert from big-endian to host
    if constexpr (std::endian::native == std::endian::little) {
        // Byte swap: GCC/Clang compile this to bswap instruction
        val = ((val >> 24) & 0x000000FF)
            | ((val >> 8)  & 0x0000FF00)
            | ((val << 8)  & 0x00FF0000)
            | ((val << 24) & 0xFF000000);
    }
    return val;
}

// Method 3: C++23 std::byteswap (cleanest)
#if __cpp_lib_byteswap >= 202110L
#include <bit>
uint32_t decode_be32_cpp23(const uint8_t* p) {
    uint32_t val;
    std::memcpy(&val, p, 4);
    if constexpr (std::endian::native == std::endian::little)
        val = std::byteswap(val);
    return val;
}
#endif

int main() {
    // Encode 0x00001234 (4660) in big-endian
    uint8_t buf[4];
    encode_be32_shift(buf, 4660);

    std::cout << "Wire bytes: ";
    for (int i = 0; i < 4; ++i)
        std::cout << std::hex << static_cast<int>(buf[i]) << " ";
    std::cout << '\n';  // 0 0 12 34

    // Decode all three ways
    std::cout << std::dec;
    std::cout << "Shift:  " << decode_be32_shift(buf)  << '\n';  // 4660
    std::cout << "Memcpy: " << decode_be32_memcpy(buf)  << '\n';  // 4660

    // Why NOT use ntohl/htons:
    //   1. Requires <arpa/inet.h> (POSIX) or <winsock2.h> (Windows)
    //   2. Not constexpr
    //   3. Bit shifts are portable, constexpr, and compile to same instructions
}
```

---

## Notes

- Always validate the length field against a maximum (prevent allocation attacks).
- Big-endian is traditional "network byte order" but little-endian is becoming common (protobuf uses varint).
- For production: consider using `std::span<const uint8_t>` instead of raw pointers.
- The `EfficientFramer` (Q2) avoids buffer copying by tracking state explicitly.
- For zero-copy: use iovec with separate header/payload buffers (see scatter-gather I/O).
