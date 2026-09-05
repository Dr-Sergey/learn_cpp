# Build HTTP Servers and Clients with Boost.Beast and cpp-httplib

**Category:** Networking and IO  
**Standard:** C++17, C++20  
**Reference:** [Boost.Beast Docs](https://www.boost.org/doc/libs/release/libs/beast/doc/html/index.html), [cpp-httplib](https://github.com/yhirose/cpp-httplib)  

---

## Topic Overview

C++ has no standard HTTP library, so you have to pick one. The two most common choices are **Boost.Beast** and **cpp-httplib**, and they have almost nothing in common beyond both speaking HTTP.

**Boost.Beast** is built on top of Boost.Asio and gives you full control over async I/O. It exposes HTTP as first-class objects - `http::request` and `http::response` - with interchangeable body types (string, file, empty). What it does *not* give you is routing, middleware, or JSON handling. You build all of that yourself, on top of the primitives Beast provides. That is the trade-off: maximum control, maximum responsibility.

**cpp-httplib** takes the opposite approach. It is a single header file with batteries included - routing with regex patterns, multipart form parsing, built-in gzip, logging hooks. The catch is that it is synchronous, using one thread per connection. That is perfectly fine for internal tools and REST APIs with modest load, but it will not scale to tens of thousands of concurrent connections.

The table below summarises the key differences. If the choice is not obvious from your requirements, a useful heuristic is: if you are writing a production service that expects many concurrent clients or needs WebSocket support, choose Beast. If you need an HTTP endpoint in 50 lines with no build system changes, choose cpp-httplib.

| Feature                | Boost.Beast                    | cpp-httplib                  |
| --- | --- | --- |
| Async support          | Full (Asio coroutines, callbacks) | None (thread-per-connection) |
| Header-only            | Yes (with Boost)              | Yes (single header)          |
| TLS support            | Via Asio SSL                  | Via OpenSSL (optional)       |
| HTTP/2                 | No (HTTP/1.1 only)            | No                           |
| Routing                | Manual                        | Built-in regex routes        |
| WebSocket              | Yes                           | No                           |
| Dependencies           | Boost                         | None (or OpenSSL for HTTPS)  |
| Ideal for              | High-perf async servers       | Internal tools, prototypes   |

Here is a sketch of how each library is structured internally. Beast revolves around an `io_context` and strands to manage concurrency; cpp-httplib wraps everything in a `Server` object with a built-in thread pool and router.

```cpp
Beast Architecture:                cpp-httplib Architecture:
┌──────────────┐                   ┌──────────────────┐
│  io_context  │                   │  httplib::Server  │
│  ┌────────┐  │                   │  ┌────────────┐  │
│  │ strand  │  │                   │  │  Router    │  │
│  │┌──────┐│  │                   │  │  Handler[] │  │
│  ││stream││  │                   │  └────────────┘  │
│  │└──────┘│  │                   │  Thread Pool     │
│  └────────┘  │                   └──────────────────┘
└──────────────┘
```

---

## Self-Assessment

### Q1: Build an async HTTP server with Boost.Beast and C++20 coroutines that handles GET and POST with JSON

The server below uses C++20 coroutines (`co_await`) to handle each connection asynchronously. Pay attention to the `handle_request` function, which acts as the router - it inspects the method and target and returns an appropriate response object. The `handle_session` coroutine drives the HTTP request/response loop for a single connection, and the `listener` coroutine accepts new connections and spawns a session for each one.

<!-- compile: needs third-party library header `boost/beast/core.hpp` -->
```cpp
#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <iostream>
#include <string>

namespace beast = boost::beast;
namespace http  = beast::http;
namespace asio  = boost::asio;
using tcp = asio::ip::tcp;

// Simple router: match method + target
http::response<http::string_body>
handle_request(http::request<http::string_body>&& req) {
    auto const make_response = [&](http::status status,
                                    std::string body,
                                    std::string content_type = "application/json") {
        http::response<http::string_body> res{status, req.version()};
        res.set(http::field::server, "Beast-Server/1.0");
        res.set(http::field::content_type, content_type);
        res.keep_alive(req.keep_alive());
        res.body() = std::move(body);
        res.prepare_payload();
        return res;
    };

    // Route: GET /health
    if (req.method() == http::verb::get && req.target() == "/health") {
        return make_response(http::status::ok, R"({"status":"ok"})");
    }

    // Route: POST /echo
    if (req.method() == http::verb::post && req.target() == "/echo") {
        return make_response(http::status::ok, std::string(req.body()));
    }

    // Route: GET /
    if (req.method() == http::verb::get && req.target() == "/") {
        return make_response(http::status::ok,
            R"({"message":"Hello from Beast"})");
    }

    return make_response(http::status::not_found,
        R"({"error":"Not found"})");
}

asio::awaitable<void> handle_session(tcp::socket socket) {
    beast::flat_buffer buffer;
    try {
        while (true) {
            // Read request
            http::request<http::string_body> req;
            co_await http::async_read(socket, buffer, req,
                                       asio::use_awaitable);

            // Handle and send response
            auto resp = handle_request(std::move(req));
            bool keep_alive = resp.keep_alive();

            co_await http::async_write(socket, resp,
                                        asio::use_awaitable);

            if (!keep_alive) break;
        }
    } catch (const std::exception& e) {
        // Connection closed or error - normal for HTTP
    }
    beast::error_code ec;
    socket.shutdown(tcp::socket::shutdown_send, ec);
}

asio::awaitable<void> listener(tcp::endpoint endpoint) {
    auto executor = co_await asio::this_coro::executor;
    tcp::acceptor acceptor(executor, endpoint);
    acceptor.set_option(asio::socket_base::reuse_address(true));

    while (true) {
        auto socket = co_await acceptor.async_accept(asio::use_awaitable);
        asio::co_spawn(executor, handle_session(std::move(socket)),
                       asio::detached);
    }
}

int main() {
    asio::io_context ioc{1};
    asio::co_spawn(ioc,
        listener(tcp::endpoint(tcp::v4(), 8080)),
        asio::detached);
    std::cout << "Listening on :8080\n";
    ioc.run();
}
```

Notice that Beast does not provide a router - `handle_request` is a plain function that you write yourself. In a larger project you would typically replace this with a `std::unordered_map` keyed on method+target, or a proper trie for prefix matching.

### Q2: Build the equivalent simple HTTP server and client with cpp-httplib in minimal code

Here is how the same kind of service looks in cpp-httplib. The amount of boilerplate drops dramatically because the library handles routing, threading, and request parsing for you. The server below also demonstrates a pre-routing handler (useful as a logging middleware) and a thread-safe in-memory key-value store.

<!-- compile: needs third-party library header `httplib.h` -->
```cpp
// server.cpp - cpp-httplib with routing, logging middleware, and JSON
#include "httplib.h"  // single header
#include <iostream>
#include <string>
#include <mutex>
#include <unordered_map>

int main() {
    httplib::Server svr;

    // Thread-safe in-memory store
    std::unordered_map<std::string, std::string> store;
    std::mutex mu;

    // Logging middleware - runs before every handler
    svr.set_pre_routing_handler([](const httplib::Request& req,
                                    httplib::Response& /*res*/) {
        std::cout << req.method << " " << req.path << "\n";
        return httplib::Server::HandlerResponse::Unhandled; // continue
    });

    // GET /health
    svr.Get("/health", [](const httplib::Request&, httplib::Response& res) {
        res.set_content(R"({"status":"ok"})", "application/json");
    });

    // GET /items/:id - path parameter
    svr.Get(R"(/items/(\w+))", [&](const httplib::Request& req,
                                     httplib::Response& res) {
        auto id = req.matches[1].str();
        std::lock_guard lock(mu);
        if (auto it = store.find(id); it != store.end()) {
            res.set_content(
                R"({"id":")" + id + R"(","value":")" + it->second + R"("})",
                "application/json");
        } else {
            res.status = 404;
            res.set_content(R"({"error":"not found"})", "application/json");
        }
    });

    // POST /items/:id
    svr.Post(R"(/items/(\w+))", [&](const httplib::Request& req,
                                      httplib::Response& res) {
        auto id = req.matches[1].str();
        {
            std::lock_guard lock(mu);
            store[id] = req.body;
        }
        res.status = 201;
        res.set_content(R"({"created":")" + id + R"("})",
                        "application/json");
    });

    // Error handler
    svr.set_error_handler([](const httplib::Request&, httplib::Response& res) {
        res.set_content(
            R"({"error":")" + std::to_string(res.status) + R"("})",
            "application/json");
    });

    std::cout << "Serving on http://localhost:8080\n";
    svr.listen("0.0.0.0", 8080);
}
```

The client side is equally concise. You construct a `Client`, set timeouts, and call `Get`, `Post`, etc. The result is a `std::optional`-like object - always check it before reading the status or body, because a null result means the connection itself failed.

<!-- compile: needs third-party library header `httplib.h` -->
```cpp
// client.cpp - cpp-httplib client
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("http://localhost:8080");
    cli.set_connection_timeout(5);  // seconds
    cli.set_read_timeout(5);

    // POST
    auto post_res = cli.Post("/items/foo", "bar_value", "text/plain");
    if (post_res && post_res->status == 201) {
        std::cout << "Created: " << post_res->body << "\n";
    }

    // GET
    auto get_res = cli.Get("/items/foo");
    if (get_res && get_res->status == 200) {
        std::cout << "Got: " << get_res->body << "\n";
    }

    // GET with headers
    httplib::Headers headers = {{"Authorization", "Bearer token123"}};
    auto auth_res = cli.Get("/health", headers);
    if (auth_res) {
        std::cout << "Health: " << auth_res->body << "\n";
    }
}
```

### Q3: How do you serve static files and handle multipart file uploads with Beast

Serving static files is a case where Beast's `file_body` body type really shines. Rather than reading a file into a string and then writing that string to the socket, `file_body` uses the OS-native `sendfile()` on Linux for a zero-copy path - the file data never has to pass through your process's memory. The code below shows both static file serving with path sanitization and a basic multipart body extraction. The path sanitization part is worth reading carefully: directory traversal (`../`) attacks are real and need to be blocked explicitly.

```cpp
#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/asio.hpp>
#include <filesystem>
#include <fstream>
#include <iostream>

namespace beast = boost::beast;
namespace http  = beast::http;
namespace fs    = std::filesystem;

// MIME type lookup
std::string_view mime_type(std::string_view path) {
    if (path.ends_with(".html")) return "text/html";
    if (path.ends_with(".css"))  return "text/css";
    if (path.ends_with(".js"))   return "application/javascript";
    if (path.ends_with(".json")) return "application/json";
    if (path.ends_with(".png"))  return "image/png";
    if (path.ends_with(".jpg"))  return "image/jpeg";
    return "application/octet-stream";
}

// Serve a static file using Beast's file_body for zero-copy sendfile
http::response<http::file_body>
serve_file(const http::request<http::string_body>& req,
           const fs::path& doc_root) {
    // Sanitize path - prevent directory traversal
    std::string target(req.target());
    if (target.empty() || target[0] != '/' ||
        target.find("..") != std::string::npos) {
        throw std::runtime_error("Bad target");
    }

    fs::path file_path = doc_root / target.substr(1);
    if (fs::is_directory(file_path))
        file_path /= "index.html";

    // Canonicalize and verify it's under doc_root
    auto canonical = fs::weakly_canonical(file_path);
    auto root_canonical = fs::weakly_canonical(doc_root);
    if (canonical.string().find(root_canonical.string()) != 0)
        throw std::runtime_error("Path traversal attempt");

    http::response<http::file_body> res{http::status::ok, req.version()};
    res.set(http::field::content_type, mime_type(file_path.string()));

    beast::error_code ec;
    res.body().open(file_path.string().c_str(),
                    beast::file_mode::scan, ec);
    if (ec) throw std::runtime_error("File not found");

    res.prepare_payload();
    res.keep_alive(req.keep_alive());
    return res;
}

// Parse multipart boundary from Content-Type
std::string extract_boundary(std::string_view content_type) {
    auto pos = content_type.find("boundary=");
    if (pos == std::string_view::npos) return {};
    auto boundary = content_type.substr(pos + 9);
    // Remove surrounding quotes if present
    if (boundary.starts_with("\"") && boundary.ends_with("\""))
        boundary = boundary.substr(1, boundary.size() - 2);
    return std::string(boundary);
}

// Minimal multipart body extraction (production: use a proper parser)
void handle_upload(const http::request<http::string_body>& req,
                   const fs::path& upload_dir) {
    auto ct = std::string(req[http::field::content_type]);
    auto boundary = extract_boundary(ct);
    if (boundary.empty()) return;

    // Find file data between boundaries (simplified)
    const auto& body = req.body();
    std::string delim = "--" + boundary;
    auto start = body.find("\r\n\r\n");
    auto end   = body.rfind(delim);
    if (start != std::string::npos && end != std::string::npos) {
        start += 4; // skip \r\n\r\n
        auto data = body.substr(start, end - start - 2);
        std::ofstream out(upload_dir / "uploaded_file",
                          std::ios::binary);
        out.write(data.data(), static_cast<std::streamsize>(data.size()));
    }
}
```

The double-check using `weakly_canonical` is important. A simple `target.find("..")` check alone can be bypassed with URL-encoded sequences in some configurations, so always resolve to a canonical path and verify the result is still within your document root.

---

## Notes

- Beast does not provide routing - build a `std::unordered_map<string, Handler>` or use a trie for path matching.
- Beast's `file_body` uses OS-native `sendfile()` on Linux for zero-copy file serving - much faster than reading into a string.
- cpp-httplib's regex routing uses `std::regex` - avoid complex patterns in hot paths due to `std::regex` compilation overhead.
- For JSON, pair Beast with `nlohmann/json` or `simdjson`; cpp-httplib has no built-in JSON but integrates trivially.
- Always sanitize file paths to prevent directory traversal (`..`); canonicalize and verify the resolved path is within the document root.
- cpp-httplib supports gzip compression with `svr.set_compress(true)` when compiled with zlib.
- For HTTP/2, neither Beast nor cpp-httplib supports it - consider `nghttp2` or use a reverse proxy (nginx) in front.
