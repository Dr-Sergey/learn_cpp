# Implement WebSocket Communication in C++ with Boost.Beast

**Category:** Networking and IO  
**Standard:** C++17, C++20  
**Reference:** [Boost.Beast WebSocket](https://www.boost.org/doc/libs/release/libs/beast/doc/html/beast/using_websocket.html), [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)  

---

## Topic Overview

WebSocket provides full-duplex communication over a single TCP connection, which makes it the natural choice for anything real-time: trading systems, telemetry dashboards, multiplayer game backends, IoT gateways. Unlike HTTP, where every interaction is a request followed by a response, WebSocket lets both sides send messages independently at any time, without waiting for the other side to ask first.

Boost.Beast's WebSocket implementation sits on top of Asio and provides both synchronous and asynchronous APIs. You get full control over frames, ping/pong, and close handshakes.

A WebSocket connection starts its life as an ordinary HTTP connection. The client sends an HTTP Upgrade request; the server responds with `101 Switching Protocols`. Beast handles this transparently - you call `ws.handshake()` on the client side or `ws.accept()` on the server side, and Beast takes care of all the header negotiation. After the handshake, the connection is fully symmetric: both sides can send and receive at will.

The table below lists the frame types you will encounter. Most of the time you will only deal with text and binary frames, but understanding ping/pong and close frames matters when you are debugging keepalive issues or trying to shut down cleanly.

| Frame Type   | Opcode | Beast API                          | Use Case                           |
| --- | --- | --- | --- |
| Text        | 0x1    | `ws.text(true); ws.write(...)`     | JSON messages, chat                |
| Binary      | 0x2    | `ws.binary(true); ws.write(...)`   | Protobuf, images, binary protocols |
| Ping        | 0x9    | `ws.ping(payload)`                 | Keepalive, latency measurement     |
| Pong        | 0xA    | Auto-sent by Beast on ping receipt | Required by RFC 6455               |
| Close       | 0x8    | `ws.close(reason)`                 | Graceful shutdown                  |

Here is the connection lifecycle at a glance:

```cpp
WebSocket Lifecycle:
┌────────┐  HTTP Upgrade   ┌────────┐
│ Client │────────────────>│ Server │
│        │<────────────────│        │
│        │  101 Switching  │        │
│        │                 │        │
│        │<===============>│        │  Full-duplex frames
│        │  text/binary    │        │
│        │                 │        │
│        │───close─────────>        │
│        │<──close─────────│        │  Close handshake
└────────┘                 └────────┘
```

One subtlety worth knowing up front: Beast automatically handles control frames (ping/pong) during read operations. When a ping arrives while you are in the middle of a `read`, Beast sends the pong and then continues waiting for the next data frame. You do not have to do anything special. The close handshake is similar - calling `ws.close()` sends a close frame and then waits for the peer's close frame. The reason this trips people up is when both sides initiate close simultaneously. Beast handles this correctly, but you need to make sure both sides actually complete the handshake (by continuing to call `read` after sending close) - if one side just drops the TCP connection, the other side will see an error, not a clean close.

---

## Self-Assessment

### Q1: Implement an async WebSocket server with C++20 coroutines that broadcasts messages to all connected clients

The server manages a `ClientHub` - a thread-safe registry of all currently connected WebSocket streams. When any client sends a message, the hub broadcasts it to everyone. The `handle_session` coroutine runs per connection: it accepts the WebSocket upgrade, registers with the hub, then enters a read loop. When the loop exits (either gracefully via a close frame or via an error), the guard lambda ensures the client is removed from the hub.

<!-- compile: needs third-party library header `boost/beast/core.hpp` -->
```cpp
#include <boost/beast/core.hpp>
#include <boost/beast/websocket.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <boost/asio/steady_timer.hpp>
#include <iostream>
#include <memory>
#include <set>
#include <mutex>
#include <string>

namespace beast = boost::beast;
namespace ws    = beast::websocket;
namespace asio  = boost::asio;
using tcp = asio::ip::tcp;

// Thread-safe client registry for broadcast
class ClientHub {
    mutable std::mutex mu_;
    std::set<ws::stream<tcp::socket>*> clients_;

public:
    void join(ws::stream<tcp::socket>* client) {
        std::lock_guard lock(mu_);
        clients_.insert(client);
    }

    void leave(ws::stream<tcp::socket>* client) {
        std::lock_guard lock(mu_);
        clients_.erase(client);
    }

    void broadcast(const std::string& msg) {
        std::lock_guard lock(mu_);
        for (auto* client : clients_) {
            beast::error_code ec;
            client->text(true);
            client->write(asio::buffer(msg), ec);
            // Ignore write errors - client may have disconnected
        }
    }
};

asio::awaitable<void>
handle_session(tcp::socket socket, ClientHub& hub) {
    ws::stream<tcp::socket> stream(std::move(socket));

    // Configure WebSocket settings before accept
    stream.set_option(ws::stream_base::timeout::suggested(
        beast::role_type::server));
    stream.set_option(ws::stream_base::decorator(
        [](ws::response_type& res) {
            res.set(beast::http::field::server, "Beast-WS/1.0");
        }));

    // Accept the WebSocket upgrade
    co_await stream.async_accept(asio::use_awaitable);

    hub.join(&stream);
    auto guard = [&] { hub.leave(&stream); };

    try {
        beast::flat_buffer buffer;
        while (true) {
            // Read a message
            auto bytes = co_await stream.async_read(
                buffer, asio::use_awaitable);

            // Get message as string
            auto msg = beast::buffers_to_string(buffer.data());
            buffer.consume(bytes);

            std::cout << "Received: " << msg << "\n";

            // Broadcast to all clients
            hub.broadcast(msg);
        }
    } catch (const boost::system::system_error& e) {
        if (e.code() != ws::error::closed)
            std::cerr << "Session error: " << e.what() << "\n";
    }

    guard();
}

asio::awaitable<void> listener(tcp::endpoint ep, ClientHub& hub) {
    auto executor = co_await asio::this_coro::executor;
    tcp::acceptor acceptor(executor, ep);

    while (true) {
        auto socket = co_await acceptor.async_accept(asio::use_awaitable);
        asio::co_spawn(executor,
            handle_session(std::move(socket), hub),
            asio::detached);
    }
}

int main() {
    asio::io_context ioc;
    ClientHub hub;

    asio::co_spawn(ioc,
        listener(tcp::endpoint(tcp::v4(), 9001), hub),
        asio::detached);

    std::cout << "WebSocket server on ws://localhost:9001\n";
    ioc.run();
}
```

Note the `guard` lambda in `handle_session`. It calls `hub.leave` when the function exits, regardless of whether that exit is due to a normal close or an exception. This is the RAII pattern applied to cleanup logic - without something like this, disconnected clients would stay in the hub indefinitely and cause write errors on every subsequent broadcast.

### Q2: Implement an async WebSocket client with automatic reconnection and ping/pong keepalive

A robust WebSocket client needs two things beyond the basic connect-and-read loop: keepalive pings to prevent NAT/firewall timeouts, and automatic reconnection with exponential backoff when the connection drops. Beast handles pings for you when you set `keep_alive_pings = true`. The reconnection logic in `run_with_reconnect` is a standard exponential backoff pattern - start at 500ms, double each attempt, cap at 30 seconds.

<!-- compile: needs third-party library header `boost/beast/core.hpp` -->
```cpp
#include <boost/beast/core.hpp>
#include <boost/beast/websocket.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <boost/asio/steady_timer.hpp>
#include <iostream>
#include <string>
#include <functional>
#include <chrono>

namespace beast = boost::beast;
namespace ws    = beast::websocket;
namespace asio  = boost::asio;
using tcp = asio::ip::tcp;

class WsClient {
    asio::io_context& ioc_;
    std::string host_;
    std::string port_;
    std::function<void(std::string)> on_message_;

public:
    WsClient(asio::io_context& ioc,
             std::string host, std::string port,
             std::function<void(std::string)> on_message)
        : ioc_(ioc), host_(std::move(host)), port_(std::move(port)),
          on_message_(std::move(on_message)) {}

    void start() {
        asio::co_spawn(ioc_, run_with_reconnect(), asio::detached);
    }

private:
    asio::awaitable<void> run_with_reconnect() {
        int backoff_ms = 500;
        constexpr int max_backoff_ms = 30'000;

        while (true) {
            try {
                co_await run_session();
            } catch (const std::exception& e) {
                std::cerr << "Connection lost: " << e.what() << "\n";
            }

            // Exponential backoff with cap
            std::cout << "Reconnecting in " << backoff_ms << " ms...\n";
            asio::steady_timer timer(ioc_,
                std::chrono::milliseconds(backoff_ms));
            co_await timer.async_wait(asio::use_awaitable);

            backoff_ms = std::min(backoff_ms * 2, max_backoff_ms);
        }
    }

    asio::awaitable<void> run_session() {
        // Resolve and connect
        tcp::resolver resolver(ioc_);
        auto endpoints = co_await resolver.async_resolve(
            host_, port_, asio::use_awaitable);

        ws::stream<tcp::socket> stream(ioc_);

        auto ep = co_await asio::async_connect(
            beast::get_lowest_layer(stream),
            endpoints, asio::use_awaitable);

        // Configure timeouts with automatic ping
        ws::stream_base::timeout opt{};
        opt.handshake_timeout = std::chrono::seconds(5);
        opt.idle_timeout      = std::chrono::seconds(30);
        opt.keep_alive_pings  = true;  // Beast sends pings automatically
        stream.set_option(opt);

        // WebSocket handshake
        auto host = host_ + ":" + std::to_string(ep.port());
        co_await stream.async_handshake(host, "/",
                                         asio::use_awaitable);

        std::cout << "Connected to " << host << "\n";

        // Read loop
        beast::flat_buffer buffer;
        while (true) {
            auto bytes = co_await stream.async_read(
                buffer, asio::use_awaitable);

            auto msg = beast::buffers_to_string(buffer.data());
            buffer.consume(bytes);

            on_message_(std::move(msg));
        }
    }
};

int main() {
    asio::io_context ioc;

    WsClient client(ioc, "localhost", "9001",
        [](std::string msg) {
            std::cout << "Received: " << msg << "\n";
        });

    client.start();
    ioc.run();
}
```

### Q3: How do you handle binary frames, message fragmentation, and graceful shutdown correctly

This example brings together several practical patterns: a typed binary message with serialize/deserialize, explicit fragmentation for large payloads, graceful close with timeout protection, and the control callback for observing pings and pongs. Read through each function - they work together to show a complete production-quality handling story.

```cpp
#include <boost/beast/core.hpp>
#include <boost/beast/websocket.hpp>
#include <boost/asio.hpp>
#include <vector>
#include <cstdint>
#include <iostream>

namespace beast = boost::beast;
namespace ws    = beast::websocket;
namespace asio  = boost::asio;
using tcp = asio::ip::tcp;

// Binary message protocol: [4-byte type][payload]
struct BinaryMessage {
    uint32_t type;
    std::vector<uint8_t> payload;

    std::vector<uint8_t> serialize() const {
        std::vector<uint8_t> data(sizeof(type) + payload.size());
        std::memcpy(data.data(), &type, sizeof(type));
        std::memcpy(data.data() + sizeof(type),
                    payload.data(), payload.size());
        return data;
    }

    static BinaryMessage deserialize(const void* data, std::size_t len) {
        BinaryMessage msg;
        if (len < sizeof(msg.type))
            throw std::runtime_error("Message too short");
        std::memcpy(&msg.type, data, sizeof(msg.type));
        auto* payload_start = static_cast<const uint8_t*>(data) + sizeof(msg.type);
        msg.payload.assign(payload_start, payload_start + len - sizeof(msg.type));
        return msg;
    }
};

// Send binary message
void send_binary(ws::stream<tcp::socket>& stream,
                 const BinaryMessage& msg) {
    auto data = msg.serialize();
    stream.binary(true);  // Set binary mode
    stream.write(asio::buffer(data));
}

// Read any frame type (text or binary)
void read_message(ws::stream<tcp::socket>& stream) {
    beast::flat_buffer buffer;
    stream.read(buffer);

    if (stream.got_binary()) {
        // Binary frame
        auto data = static_cast<const uint8_t*>(buffer.data().data());
        auto size = buffer.data().size();
        auto msg = BinaryMessage::deserialize(data, size);
        std::cout << "Binary msg type=" << msg.type
                  << " payload_size=" << msg.payload.size() << "\n";
    } else {
        // Text frame
        auto text = beast::buffers_to_string(buffer.data());
        std::cout << "Text: " << text << "\n";
    }
}

// Write large message with explicit fragmentation
void send_fragmented(ws::stream<tcp::socket>& stream,
                     const std::vector<uint8_t>& large_data,
                     std::size_t fragment_size = 16384) {
    stream.binary(true);

    std::size_t offset = 0;
    bool first = true;
    while (offset < large_data.size()) {
        auto chunk_size = std::min(fragment_size,
                                    large_data.size() - offset);
        bool is_last = (offset + chunk_size >= large_data.size());

        // write_some with fin=false continues the message
        // write_some with fin=true completes it
        stream.write_some(!is_last,  // fin = false means "more fragments"
            asio::buffer(large_data.data() + offset, chunk_size));

        offset += chunk_size;
        first = false;
    }
    // Beast reassembles fragments on the read side automatically
}

// Graceful close handshake
void graceful_close(ws::stream<tcp::socket>& stream) {
    beast::error_code ec;

    // Send close frame with reason
    stream.close(ws::close_code::normal, ec);
    if (ec) {
        // Close may fail if peer already closed TCP
        std::cerr << "Close error: " << ec.message() << "\n";
        return;
    }

    // After close(), Beast has already completed the handshake:
    // 1. Sent our close frame
    // 2. Read peer's close frame (or timed out)
    // The TCP connection can now be shut down
}

// Async close with timeout protection
asio::awaitable<void>
async_graceful_close(ws::stream<tcp::socket>& stream) {
    // Set a short timeout for the close handshake
    ws::stream_base::timeout opt{};
    opt.handshake_timeout = std::chrono::seconds(5);
    opt.idle_timeout      = std::chrono::seconds(5);
    opt.keep_alive_pings  = false;
    stream.set_option(opt);

    boost::system::error_code ec;
    co_await stream.async_close(
        ws::close_code::normal,
        asio::redirect_error(asio::use_awaitable, ec));

    if (ec && ec != ws::error::closed) {
        std::cerr << "Async close error: " << ec.message() << "\n";
    }
}

// Handling ping/pong manually (usually unnecessary - Beast auto-responds)
void setup_custom_ping_handler(ws::stream<tcp::socket>& stream) {
    // Beast automatically responds to pings with pongs.
    // Use control_callback to observe or log them:
    stream.control_callback(
        [](ws::frame_type kind, beast::string_view payload) {
            switch (kind) {
                case ws::frame_type::ping:
                    std::cout << "Got ping: " << payload << "\n";
                    break;
                case ws::frame_type::pong:
                    std::cout << "Got pong: " << payload << "\n";
                    break;
                case ws::frame_type::close:
                    std::cout << "Got close frame\n";
                    break;
            }
        });
}
```

---

## Notes

- Beast **automatically replies to pings with pongs** during `read()` - you don't need to handle them manually unless you want to log or measure latency.
- Set `stream_base::timeout::keep_alive_pings = true` for automatic periodic pings - essential for NAT/firewall keepalive.
- `write_some(fin, buffer)` enables explicit fragmentation; the receiving side's `read()` reassembles fragments transparently.
- The close handshake is **bidirectional**: calling `close()` sends a close frame and waits for the peer's close frame. If both sides initiate simultaneously, Beast handles it correctly.
- For text frames, Beast validates UTF-8 by default. Send `binary(true)` for non-UTF-8 to avoid validation overhead and errors.
- In production broadcast scenarios, avoid synchronous `write()` to all clients in a loop - one slow client blocks all. Use per-client write queues with `async_write`.
- WebSocket compression (`permessage-deflate`) can be enabled via `stream.set_option(ws::stream_base::decorator(...))` but adds CPU overhead - benchmark before enabling.
