# Use gRPC for High-Performance RPC in C++

**Category:** Networking and IO  
**Standard:** C++17, C++20  
**Reference:** [gRPC C++ Docs](https://grpc.io/docs/languages/cpp/), [gRPC GitHub](https://github.com/grpc/grpc)  

---

## Topic Overview

gRPC is Google's high-performance RPC framework built on HTTP/2 and Protocol Buffers. In C++, it provides both synchronous and asynchronous APIs, with the async `CompletionQueue` model offering maximum throughput for server workloads. Understanding gRPC at a senior level means knowing when to use sync vs async, how streaming RPCs work, and how to propagate deadlines and errors correctly.

The gRPC C++ stack has three API tiers: the **sync API** (simple, one-thread-per-RPC), the **async API** (CompletionQueue-based, complex but high-performance), and the newer **callback API** (simpler async with lambdas). For most production servers handling thousands of concurrent RPCs, the async or callback API is necessary - the sync API blocks a thread per active RPC.

Here is a quick reference for the four RPC patterns. Pick the one that matches the shape of your data flow:

| RPC Type | Client | Server | Use Case |
| --- | --- | --- | --- |
| Unary | Send 1, Recv 1 | Recv 1, Send 1 | Simple request/response |
| Server streaming | Send 1, Recv N | Recv 1, Send N | Large result sets, live feeds |
| Client streaming | Send N, Recv 1 | Recv N, Send 1 | File upload, aggregation |
| Bidirectional | Send N, Recv M | Recv N, Send M | Chat, real-time sync |

The diagram below shows how the `.proto` file feeds the `protoc` compiler, which generates C++ stubs that your sync, async, or callback server can then implement:

```cpp
+----------+   .proto    +-----------+   protoc    +--------------+
|  Schema  |------------>|  protoc   |------------>|  Generated   |
|  (.proto)|             |  compiler |             |  C++ stubs   |
+----------+             +-----------+             |  .grpc.pb.h  |
                                                   |  .grpc.pb.cc |
                                                   +------+-------+
                                                          |
                                          +---------------+--------------+
                                          v               v              v
                                    Sync Server     Async Server    Callback Server
```

Deadlines are a first-class concept in gRPC: every RPC should have a deadline. Without one, a stuck server leaks client resources indefinitely. Deadlines propagate automatically through gRPC interceptors when you forward the `ServerContext` metadata, enabling end-to-end timeout enforcement across microservice chains.

---

## Self-Assessment

### Q1: Define a gRPC service with protobuf and implement a sync server and client with proper error handling

Start with the `.proto` schema. Everything flows from here - the service definition, the message types, and the streaming pattern are all declared in this file before you write a single line of C++.

```protobuf
// keyvalue.proto
syntax = "proto3";
package kvstore;

service KeyValueStore {
    rpc Get(GetRequest) returns (GetResponse);
    rpc Put(PutRequest) returns (PutResponse);
    rpc ListKeys(ListKeysRequest) returns (stream KeyEntry); // server streaming
}

message GetRequest  { string key = 1; }
message GetResponse { string value = 1; bool found = 2; }

message PutRequest  { string key = 1; string value = 2; }
message PutResponse { bool success = 1; }

message ListKeysRequest { string prefix = 1; }
message KeyEntry { string key = 1; string value = 2; }
```

Now here is the sync server implementation. Notice how each RPC handler checks `ctx->IsCancelled()` - this is good practice because the client may have timed out or disconnected before your handler even starts processing. The `shared_mutex` lets `Get` and `ListKeys` run concurrently while `Put` gets exclusive access.

```cpp
// server.cpp - Sync gRPC server
#include <grpcpp/grpcpp.h>
#include "keyvalue.grpc.pb.h"
#include <unordered_map>
#include <shared_mutex>
#include <iostream>

class KVStoreServiceImpl final : public kvstore::KeyValueStore::Service {
    std::unordered_map<std::string, std::string> store_;
    mutable std::shared_mutex mu_;

public:
    grpc::Status Get(grpc::ServerContext* ctx,
                     const kvstore::GetRequest* req,
                     kvstore::GetResponse* resp) override {
        // Check if client already cancelled
        if (ctx->IsCancelled()) {
            return grpc::Status(grpc::CANCELLED, "Client cancelled");
        }

        std::shared_lock lock(mu_);
        auto it = store_.find(req->key());
        if (it != store_.end()) {
            resp->set_value(it->second);
            resp->set_found(true);
        } else {
            resp->set_found(false);
            return grpc::Status(grpc::NOT_FOUND,
                                "Key not found: " + req->key());
        }
        return grpc::Status::OK;
    }

    grpc::Status Put(grpc::ServerContext* /*ctx*/,
                     const kvstore::PutRequest* req,
                     kvstore::PutResponse* resp) override {
        std::unique_lock lock(mu_);
        store_[req->key()] = req->value();
        resp->set_success(true);
        return grpc::Status::OK;
    }

    grpc::Status ListKeys(grpc::ServerContext* ctx,
                          const kvstore::ListKeysRequest* req,
                          grpc::ServerWriter<kvstore::KeyEntry>* writer) override {
        std::shared_lock lock(mu_);
        for (const auto& [k, v] : store_) {
            if (ctx->IsCancelled()) return grpc::Status::CANCELLED;
            if (k.starts_with(req->prefix())) {
                kvstore::KeyEntry entry;
                entry.set_key(k);
                entry.set_value(v);
                writer->Write(entry);
            }
        }
        return grpc::Status::OK;
    }
};

void RunServer() {
    std::string addr("0.0.0.0:50051");
    KVStoreServiceImpl service;

    grpc::ServerBuilder builder;
    builder.AddListeningPort(addr, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    builder.SetMaxReceiveMessageSize(4 * 1024 * 1024); // 4 MB

    auto server = builder.BuildAndStart();
    std::cout << "Server listening on " << addr << "\n";
    server->Wait();
}
```

### Q2: Implement a gRPC client with deadline propagation and proper error status handling

The most important habit to build here is always setting a deadline on `ClientContext` before making any RPC call. Without a deadline you are trusting the server to behave correctly forever - which in distributed systems it will not always do. After the call, inspect the status code carefully; each gRPC error code implies a different recovery strategy.

<!-- compile: needs third-party library header `grpcpp/grpcpp.h` -->
```cpp
// client.cpp - gRPC client with deadlines and error handling
#include <grpcpp/grpcpp.h>
#include "keyvalue.grpc.pb.h"
#include <chrono>
#include <iostream>

class KVClient {
    std::unique_ptr<kvstore::KeyValueStore::Stub> stub_;

public:
    explicit KVClient(std::shared_ptr<grpc::Channel> channel)
        : stub_(kvstore::KeyValueStore::NewStub(channel)) {}

    std::optional<std::string> Get(const std::string& key,
                                    std::chrono::milliseconds timeout) {
        kvstore::GetRequest req;
        req.set_key(key);

        kvstore::GetResponse resp;
        grpc::ClientContext ctx;

        // Set deadline - ALWAYS set a deadline in production
        ctx.set_deadline(std::chrono::system_clock::now() + timeout);

        // Add metadata for tracing
        ctx.AddMetadata("x-request-id", "client-123");

        grpc::Status status = stub_->Get(&ctx, req, &resp);

        if (status.ok() && resp.found()) {
            return resp.value();
        }

        // Detailed error handling by status code
        switch (status.error_code()) {
            case grpc::DEADLINE_EXCEEDED:
                std::cerr << "Timeout getting key: " << key << "\n";
                break;
            case grpc::NOT_FOUND:
                std::cerr << "Key not found: " << key << "\n";
                break;
            case grpc::UNAVAILABLE:
                std::cerr << "Server unavailable, retry with backoff\n";
                break;
            default:
                std::cerr << "RPC failed [" << status.error_code()
                          << "]: " << status.error_message() << "\n";
        }
        return std::nullopt;
    }

    void ListKeys(const std::string& prefix) {
        kvstore::ListKeysRequest req;
        req.set_prefix(prefix);

        grpc::ClientContext ctx;
        ctx.set_deadline(std::chrono::system_clock::now() +
                         std::chrono::seconds(10));

        auto reader = stub_->ListKeys(&ctx, req);

        kvstore::KeyEntry entry;
        while (reader->Read(&entry)) {
            std::cout << entry.key() << " = " << entry.value() << "\n";
        }

        grpc::Status status = reader->Finish();
        if (!status.ok()) {
            std::cerr << "ListKeys failed: " << status.error_message() << "\n";
        }
    }
};

int main() {
    auto channel = grpc::CreateChannel(
        "localhost:50051", grpc::InsecureChannelCredentials());

    // Wait for channel to be ready with timeout
    if (!channel->WaitForConnected(
            gpr_time_add(gpr_now(GPR_CLOCK_REALTIME),
                         gpr_time_from_seconds(5, GPR_TIMESPAN)))) {
        std::cerr << "Failed to connect to server\n";
        return 1;
    }

    KVClient client(channel);
    client.Get("mykey", std::chrono::milliseconds(500));
    client.ListKeys("user_");
}
```

For streaming RPCs like `ListKeys`, always call `reader->Finish()` after the `Read` loop completes. The stream may have ended because the server finished normally, or because of an error. You only find out which by inspecting the status returned by `Finish`.

### Q3: How do you implement a gRPC server interceptor for logging and metrics

Interceptors are the right place for cross-cutting concerns like logging, metrics, and authentication - not inside individual RPC handlers. Each interceptor wraps the RPC lifecycle and must call `methods->Proceed()` to pass control to the next stage. Forgetting that call will silently hang the RPC.

```cpp
// interceptor.cpp - gRPC server interceptor for observability
#include <grpcpp/grpcpp.h>
#include <grpcpp/support/server_interceptor.h>
#include <chrono>
#include <iostream>

class LoggingInterceptor : public grpc::experimental::Interceptor {
    grpc::experimental::ServerRpcInfo* info_;
    std::chrono::steady_clock::time_point start_;

public:
    explicit LoggingInterceptor(grpc::experimental::ServerRpcInfo* info)
        : info_(info), start_(std::chrono::steady_clock::now()) {}

    void Intercept(grpc::experimental::InterceptorBatchMethods* methods) override {
        // PRE_SEND_STATUS: RPC is about to complete
        if (methods->QueryInterceptionHookPoint(
                grpc::experimental::InterceptionHookPoint::PRE_SEND_STATUS)) {
            auto elapsed = std::chrono::steady_clock::now() - start_;
            auto ms = std::chrono::duration_cast<
                std::chrono::microseconds>(elapsed).count();

            // Log method name and latency
            std::cout << "[RPC] " << info_->method()
                      << " | " << ms << "us"
                      << " | type=" << static_cast<int>(info_->type())
                      << "\n";
        }

        // MUST call Proceed() to continue the RPC pipeline
        methods->Proceed();
    }
};

class LoggingInterceptorFactory
    : public grpc::experimental::ServerInterceptorFactoryInterface {
public:
    grpc::experimental::Interceptor* CreateServerInterceptor(
        grpc::experimental::ServerRpcInfo* info) override {
        return new LoggingInterceptor(info);
    }
};

// Register:
// grpc::ServerBuilder builder;
// std::vector<std::unique_ptr<
//     grpc::experimental::ServerInterceptorFactoryInterface>> creators;
// creators.push_back(std::make_unique<LoggingInterceptorFactory>());
// builder.experimental().SetInterceptorCreators(std::move(creators));
```

The factory pattern here is how gRPC creates a fresh interceptor instance per RPC. That is why `start_` can be a member variable - each RPC gets its own `LoggingInterceptor` object, so there is no shared state to worry about.

---

## Notes

- **Always set deadlines** on client RPCs - unset deadlines mean infinite wait, leaking resources in failure scenarios.
- gRPC status codes map to HTTP codes but are not identical; `NOT_FOUND` is gRPC 5, HTTP 404.
- The sync API creates one thread per RPC - unsuitable for servers handling >1000 concurrent RPCs.
- Use `grpc::ChannelArguments` to tune keepalive: `GRPC_ARG_KEEPALIVE_TIME_MS` prevents idle connections from being killed by load balancers.
- Interceptors are the correct place for cross-cutting concerns (auth, logging, metrics), not middleware in handlers.
- For bidirectional streaming, both `Read` and `Write` can be called concurrently from different threads, but each individually is not thread-safe.
- Use `grpc_cli` tool for ad-hoc testing: `grpc_cli call localhost:50051 kvstore.KeyValueStore.Get "key: 'foo'"`.
