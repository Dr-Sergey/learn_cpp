# Integrate TLS into C++ Networking with OpenSSL and Botan

**Category:** Networking and IO  
**Standard:** C++17, C++20, C++23  
**Reference:** [OpenSSL Wiki](https://wiki.openssl.org/index.php/SSL/TLS_Client), [Botan TLS](https://botan.randombit.net/handbook/api_ref/tls.html)  

---

## Topic Overview

Transport Layer Security (TLS) is the backbone of secure network communication. In C++, two dominant libraries exist: **OpenSSL** - the de facto standard with a C API that demands careful resource management - and **Botan** - a modern C++ library with RAII-native TLS classes. Senior developers must understand both because OpenSSL dominates production deployments while Botan offers superior ergonomics and auditability.

The fundamental challenge with OpenSSL in C++ is its C-style API: `SSL_CTX*`, `SSL*`, `BIO*` are raw pointers that leak if not freed correctly. Wrapping them in RAII types is non-negotiable. Beyond resource management, correct certificate verification, cipher suite selection, and protocol version pinning are critical - a single misconfiguration (e.g., not calling `SSL_CTX_set_verify`) silently disables peer verification, creating a false sense of security.

The table below gives you a quick comparison so you can decide which library fits your situation:

| Aspect | OpenSSL | Botan |
| --- | --- | --- |
| API style | C with manual resource mgmt | Modern C++ with RAII |
| Maturity | 25+ years, ubiquitous | Mature, used in EU smartcards |
| Async integration | Via BIO pairs or Asio SSL stream | Callback-based TLS::Channel |
| Certificate handling | X509_STORE, manual chain build | Certificate_Store, cleaner API |
| FIPS support | Yes (FIPS module) | Yes (via botan-fips) |
| Build complexity | Moderate | CMake-native, simpler |

Async TLS with Boost.Asio uses `boost::asio::ssl::stream<tcp::socket>`, which wraps OpenSSL internally. The handshake, read, and write operations each become async operations that the io_context drives. For Botan, you implement `Botan::TLS::Callbacks` and feed raw bytes from your transport into `Botan::TLS::Channel`, receiving decrypted data via callbacks - this decouples TLS from the transport layer entirely.

---

## Self-Assessment

### Q1: How do you wrap OpenSSL's SSL_CTX and SSL in RAII types and establish a TLS connection with proper certificate verification

The reason this trips people up is that OpenSSL's API was designed in C, so every object is a raw pointer with a matching free function. If you forget to call `SSL_CTX_free` or `SSL_free`, you have a leak. The solution is to define custom deleters and pair them with `std::unique_ptr`, which then handles cleanup automatically. Pay close attention to the two lines that set verification - they are the most commonly skipped and the most important.

```cpp
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/x509v3.h>
#include <memory>
#include <stdexcept>
#include <string>

// Custom deleters for OpenSSL types
struct SslCtxDeleter {
    void operator()(SSL_CTX* ctx) const { SSL_CTX_free(ctx); }
};
struct SslDeleter {
    void operator()(SSL* ssl) const { SSL_free(ssl); } // also frees attached BIO
};
struct BioDeleter {
    void operator()(BIO* bio) const { BIO_free_all(bio); }
};

using SslCtxPtr = std::unique_ptr<SSL_CTX, SslCtxDeleter>;
using SslPtr    = std::unique_ptr<SSL, SslDeleter>;

class TlsConnection {
    SslCtxPtr ctx_;
    SslPtr    ssl_;

public:
    explicit TlsConnection(const std::string& hostname, int port) {
        // Create context - TLS_client_method() negotiates highest available version
        ctx_.reset(SSL_CTX_new(TLS_client_method()));
        if (!ctx_) throw std::runtime_error("SSL_CTX_new failed");

        // Load system CA certificates
        if (!SSL_CTX_set_default_verify_paths(ctx_.get()))
            throw std::runtime_error("Failed to load CA certs");

        // CRITICAL: Enable certificate verification
        SSL_CTX_set_verify(ctx_.get(), SSL_VERIFY_PEER, nullptr);
        SSL_CTX_set_min_proto_version(ctx_.get(), TLS1_2_VERSION);

        // Create SSL object and connect
        ssl_.reset(SSL_new(ctx_.get()));
        if (!ssl_) throw std::runtime_error("SSL_new failed");

        // Set SNI hostname - required for virtual hosting
        SSL_set_tlsext_host_name(ssl_.get(), hostname.c_str());

        // Enable hostname verification (OpenSSL 1.1.0+)
        SSL_set1_host(ssl_.get(), hostname.c_str());

        // Create connected BIO
        auto addr = hostname + ":" + std::to_string(port);
        BIO* bio = BIO_new_connect(addr.c_str());
        if (BIO_do_connect(bio) <= 0)
            throw std::runtime_error("TCP connect failed");

        SSL_set_bio(ssl_.get(), bio, bio); // SSL takes ownership
        if (SSL_connect(ssl_.get()) <= 0)
            throw std::runtime_error("TLS handshake failed");
    }

    std::string read(std::size_t max_bytes = 4096) {
        std::string buf(max_bytes, '\0');
        int n = SSL_read(ssl_.get(), buf.data(), static_cast<int>(buf.size()));
        if (n <= 0) throw std::runtime_error("SSL_read error");
        buf.resize(static_cast<std::size_t>(n));
        return buf;
    }

    void write(std::string_view data) {
        int n = SSL_write(ssl_.get(), data.data(), static_cast<int>(data.size()));
        if (n <= 0) throw std::runtime_error("SSL_write error");
    }
};

// Usage:
// TlsConnection conn("example.com", 443);
// conn.write("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n");
// auto response = conn.read();
```

Notice that `SSL_set_bio` transfers ownership of the `BIO` to the `SSL` object, which means `SSL_free` cleans it up. That is why you do not need a separate `BioDeleter` in use here - you only need it if you create a BIO that you do not hand off to an SSL object.

### Q2: How does Botan's TLS::Channel decouple TLS processing from the transport layer via callbacks

Botan's design is fundamentally different from OpenSSL's. Instead of giving you a socket-aware TLS object, it gives you a pure TLS state machine. You feed raw bytes in, and TLS calls your callbacks with either encrypted bytes to send or decrypted application data to process. This means you can run TLS over any byte-stream transport - TCP, UDP, shared memory, a serial port - without changing the TLS code.

```cpp
#include <botan/tls_channel.h>
#include <botan/tls_callbacks.h>
#include <botan/tls_session_manager_memory.h>
#include <botan/tls_policy.h>
#include <botan/auto_rng.h>
#include <botan/certstor_system.h>
#include <vector>
#include <functional>
#include <iostream>

class MyTlsCallbacks : public Botan::TLS::Callbacks {
    // Function to send raw bytes over your transport (TCP, UDP, etc.)
    std::function<void(std::span<const uint8_t>)> send_fn_;
    std::vector<uint8_t> received_plaintext_;

public:
    explicit MyTlsCallbacks(
        std::function<void(std::span<const uint8_t>)> send_fn)
        : send_fn_(std::move(send_fn)) {}

    // Called when TLS has encrypted data to send over the wire
    void tls_emit_data(std::span<const uint8_t> data) override {
        send_fn_(data);
    }

    // Called when TLS has decrypted application data
    void tls_record_received(uint64_t /*seq*/, std::span<const uint8_t> data) override {
        received_plaintext_.insert(received_plaintext_.end(),
                                   data.begin(), data.end());
    }

    // Called when handshake completes
    void tls_session_established(const Botan::TLS::Session_Summary& session) override {
        std::cout << "TLS established: " << session.version().to_string()
                  << " cipher: " << session.ciphersuite().to_string() << "\n";
    }

    // Called on TLS alert
    void tls_alert(Botan::TLS::Alert alert) override {
        std::cerr << "TLS alert: " << alert.type_string() << "\n";
    }

    // Certificate verification - delegate to system store
    void tls_verify_cert_chain(
        const std::vector<Botan::X509_Certificate>& chain,
        const std::vector<std::optional<Botan::OCSP::Response>>& ocsp,
        const std::vector<Botan::Certificate_Store*>& trusted,
        Botan::Usage_Type usage,
        std::string_view hostname,
        const Botan::TLS::Policy& policy) override
    {
        // Default verification - throws on failure
        Botan::TLS::Callbacks::tls_verify_cert_chain(
            chain, ocsp, trusted, usage, hostname, policy);
    }

    std::vector<uint8_t> take_plaintext() { return std::exchange(received_plaintext_, {}); }
};

// Integration pattern:
// 1. Create MyTlsCallbacks with your TCP send function
// 2. Create Botan::TLS::Client channel with callbacks
// 3. When TCP data arrives, feed it to channel.received_data()
// 4. To send: call channel.send(plaintext_bytes)
// 5. Botan calls tls_emit_data() with ciphertext to transmit
```

The integration pattern at the bottom of the snippet is the mental model to hold. Your event loop calls `channel.received_data()` whenever bytes arrive from the network. Botan handles all the TLS record parsing internally and calls your `tls_record_received` callback when it has assembled a complete application-level record. You never touch ciphertext directly.

### Q3: How do you perform async TLS with Boost.Asio's ssl::stream including proper shutdown

With Boost.Asio, TLS is layered on top of a TCP socket using `ssl::stream<tcp::socket>`. The important thing to understand is that the handshake, reads, and writes are all async operations, meaning they interleave with other work in your io_context. The coroutine style makes the sequencing look synchronous even though it is not.

<!-- compile: needs third-party library header `boost/asio.hpp` -->
```cpp
#include <boost/asio.hpp>
#include <boost/asio/ssl.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <iostream>
#include <string>

namespace asio = boost::asio;
namespace ssl  = asio::ssl;
using tcp = asio::ip::tcp;

asio::awaitable<void> tls_client(asio::io_context& ioc,
                                  const std::string& host,
                                  const std::string& port) {
    // SSL context with system CA certs
    ssl::context ssl_ctx(ssl::context::tlsv12_client);
    ssl_ctx.set_default_verify_paths();
    ssl_ctx.set_verify_mode(ssl::verify_peer);

    // Create SSL stream over TCP
    ssl::stream<tcp::socket> stream(ioc, ssl_ctx);

    // Set SNI hostname - critical for shared hosting
    if (!SSL_set_tlsext_host_name(stream.native_handle(), host.c_str()))
        throw boost::system::system_error(
            boost::asio::error::invalid_argument);

    // Set hostname verification
    stream.set_verify_callback(ssl::host_name_verification(host));

    // Async DNS resolve + connect
    tcp::resolver resolver(ioc);
    auto endpoints = co_await resolver.async_resolve(
        host, port, asio::use_awaitable);
    co_await asio::async_connect(
        stream.lowest_layer(), endpoints, asio::use_awaitable);

    // Async TLS handshake
    co_await stream.async_handshake(
        ssl::stream_base::client, asio::use_awaitable);

    // Send HTTP request
    std::string request = "GET / HTTP/1.1\r\nHost: " + host +
                          "\r\nConnection: close\r\n\r\n";
    co_await asio::async_write(stream,
        asio::buffer(request), asio::use_awaitable);

    // Read response
    std::string response;
    char buf[1024];
    boost::system::error_code ec;
    while (true) {
        auto n = co_await stream.async_read_some(
            asio::buffer(buf), asio::redirect_error(asio::use_awaitable, ec));
        if (ec) break;
        response.append(buf, n);
    }
    std::cout << response.substr(0, 200) << "...\n";

    // Graceful TLS shutdown - send close_notify
    // Ignore errors: remote may close TCP before responding
    co_await stream.async_shutdown(
        asio::redirect_error(asio::use_awaitable, ec));
}

int main() {
    asio::io_context ioc;
    asio::co_spawn(ioc, tls_client(ioc, "example.com", "443"), asio::detached);
    ioc.run();
}

// Build: g++ -std=c++20 -lssl -lcrypto -lboost_system -pthread
```

The `redirect_error` on `async_shutdown` is deliberate. TLS shutdown requires both sides to send a `close_notify` alert before closing the TCP connection. In practice the remote end often closes the TCP connection first, which makes `async_shutdown` return an error. That error is harmless - you just ignore it rather than letting it propagate as an exception.

---

## Notes

- **Always set `SSL_VERIFY_PEER`** - OpenSSL defaults to `SSL_VERIFY_NONE`, silently accepting any certificate.
- **Always set SNI** via `SSL_set_tlsext_host_name` - without it, servers behind CDNs return the wrong certificate.
- **Pin minimum TLS version** to 1.2; TLS 1.0/1.1 are deprecated (RFC 8996).
- Botan's callback model makes it trivial to use TLS over non-TCP transports (e.g., DTLS over UDP, TLS over shared memory).
- For async Asio TLS, `async_shutdown` may fail if the peer closes TCP first - always use `redirect_error`.
- In production, consider certificate pinning for mobile/embedded clients and OCSP stapling for servers.
- OpenSSL's `SSL_CTX` is thread-safe for reads after creation; `SSL` objects are **not** thread-safe.
