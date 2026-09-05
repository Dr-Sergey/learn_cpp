# Implement a type-safe discriminated union message bus with std::variant

**Category:** Design Patterns Modern Takes  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

A **variant-based message bus** uses `std::variant` as a closed set of message types. Unlike the `std::any`-based event bus, this approach catches invalid message types at **compile time** - you can't subscribe to or publish a type not in the variant.

The key trade-off here is openness vs. safety. An `std::any` bus lets you fire any type you want - convenient, but if you mistype a message struct name or forget to register a handler, nothing will warn you until runtime. A `std::variant` bus says up front: "here are the only messages this bus understands," and the compiler enforces that contract everywhere.

### Variant Bus vs Any-Based Bus

| Feature | `std::variant` bus | `std::any`/`type_index` bus |
| --- | --- | --- |
| Type safety | Compile-time (closed set) | Runtime (open set) |
| New message type | Must update variant typedef | Just publish any type |
| Invalid subscribe | Compile error | Runtime error or silent |
| Dispatch | `std::visit` (exhaustive) | `any_cast` (can throw) |
| Overhead | Zero allocation | `std::any` may heap-allocate |

---

## Self-Assessment

### Q1: Define Message = std::variant<Login, Logout, DataUpdate> and dispatch with std::visit

The `overloaded` helper at the top is a small but important piece of boilerplate you'll see everywhere variant-based dispatch is used. It merges multiple lambdas into a single callable that the compiler can pattern-match against each variant alternative.

```cpp
#include <iostream>
#include <string>
#include <variant>

// Message types
struct Login      { std::string username; std::string token; };
struct Logout     { std::string username; };
struct DataUpdate { int record_id; std::string new_value; };

// Closed set of messages - adding a type here updates all visitors
using Message = std::variant<Login, Logout, DataUpdate>;

// Overload helper (C++17 pattern)
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Dispatch with std::visit
void handle_message(const Message& msg) {
    std::visit(overloaded{
        [](const Login& e) {
            std::cout << "User logged in: " << e.username << '\n';
        },
        [](const Logout& e) {
            std::cout << "User logged out: " << e.username << '\n';
        },
        [](const DataUpdate& e) {
            std::cout << "Record " << e.record_id
                      << " updated to: " << e.new_value << '\n';
        }
    }, msg);
}

int main() {
    // Messages are type-safe - can only hold declared types
    Message m1 = Login{"alice", "tok_abc123"};
    Message m2 = DataUpdate{42, "new_email@example.com"};
    Message m3 = Logout{"alice"};

    handle_message(m1);  // User logged in: alice
    handle_message(m2);  // Record 42 updated to: new_email@example.com
    handle_message(m3);  // User logged out: alice

    // Message m4 = std::string{"invalid"};  // COMPILE ERROR: string not in variant
}
```

The commented-out last line is the whole point: `std::string` isn't in the `Message` variant, so the assignment simply won't compile. Compare that to an `std::any`-based bus where you'd publish the wrong type and silently get zero handlers called.

### Q2: Implement subscribe<T>(handler) and publish(msg) on a type-safe message bus

Here's where the design gets more interesting. Instead of one big `std::visit` in a global dispatch function, we want an actual bus object that different parts of the program can subscribe to. The key insight is using a `std::tuple` with one `vector<function>` slot per message type - that way `subscribe<T>()` can only compile if `T` is one of the bus's declared types.

```cpp
#include <iostream>
#include <string>
#include <variant>
#include <vector>
#include <functional>
#include <tuple>

// Message types
struct ChatMessage  { std::string from; std::string text; };
struct UserJoined   { std::string username; };
struct UserLeft     { std::string username; };

using Message = std::variant<ChatMessage, UserJoined, UserLeft>;

// Overload helper
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Type-safe message bus
template<typename... MsgTypes>
class MessageBus {
    // One handler vector per message type
    std::tuple<std::vector<std::function<void(const MsgTypes&)>>...> handlers_;

    template<typename T>
    auto& get_handlers() {
        return std::get<std::vector<std::function<void(const T&)>>>(handlers_);
    }

public:
    // subscribe<T>: only compiles if T is in MsgTypes...
    template<typename T>
    void subscribe(std::function<void(const T&)> handler) {
        get_handlers<T>().push_back(std::move(handler));
    }

    // publish: dispatches to handlers of the active variant alternative
    void publish(const std::variant<MsgTypes...>& msg) {
        std::visit(overloaded{
            [this](const MsgTypes& m) {
                for (auto& handler : get_handlers<MsgTypes>()) {
                    handler(m);
                }
            }...
        }, msg);
    }
};

int main() {
    MessageBus<ChatMessage, UserJoined, UserLeft> bus;

    bus.subscribe<ChatMessage>([](const ChatMessage& m) {
        std::cout << "[" << m.from << "]: " << m.text << '\n';
    });

    bus.subscribe<UserJoined>([](const UserJoined& m) {
        std::cout << "-> " << m.username << " joined the chat\n";
    });

    bus.subscribe<UserLeft>([](const UserLeft& m) {
        std::cout << "<- " << m.username << " left the chat\n";
    });

    bus.publish(UserJoined{"alice"});
    bus.publish(ChatMessage{"alice", "Hello everyone!"});
    bus.publish(UserJoined{"bob"});
    bus.publish(ChatMessage{"bob", "Hi Alice!"});
    bus.publish(UserLeft{"alice"});
}
```

**Output:**

```text
-> alice joined the chat
[alice]: Hello everyone!
-> bob joined the chat
[bob]: Hi Alice!
<- alice left the chat
```

### Q3: Show that subscribing to an unknown message type is a compile-time error

The compile-time enforcement comes from `std::get` on the tuple. If you try to get the slot for a type that isn't in the tuple's type list, `std::get` simply has no matching specialization to call - the compiler refuses with an error about no matching function. This is the mechanism that makes "accidentally subscribing to a wrong type" impossible to ship.

```cpp
#include <iostream>
#include <string>
#include <variant>
#include <vector>
#include <functional>
#include <tuple>

struct Ping { int seq; };
struct Pong { int seq; };
// struct Unknown { int x; };  // NOT in the message set

template<typename... MsgTypes>
class StrictBus {
    std::tuple<std::vector<std::function<void(const MsgTypes&)>>...> handlers_;

    template<typename T>
    auto& get_handlers() {
        // This fails at compile time if T is not in MsgTypes...
        // because std::get<vector<function<void(const T&)>>> won't find a match in the tuple
        return std::get<std::vector<std::function<void(const T&)>>>(handlers_);
    }
public:
    template<typename T>
    void subscribe(std::function<void(const T&)> h) {
        get_handlers<T>().push_back(std::move(h));
    }
};

int main() {
    StrictBus<Ping, Pong> bus;

    // Compiles - Ping is in the message set
    bus.subscribe<Ping>([](const Ping& p) {
        std::cout << "Ping #" << p.seq << '\n';
    });

    // Compiles - Pong is in the message set
    bus.subscribe<Pong>([](const Pong& p) {
        std::cout << "Pong #" << p.seq << '\n';
    });

    // COMPILE ERROR if uncommented:
    // bus.subscribe<Unknown>([](const Unknown& u) {
    //     std::cout << u.x << '\n';
    // });
    // Error: no matching call to std::get<vector<function<void(const Unknown&)>>>
    //        in tuple<vector<function<void(const Ping&)>>, vector<function<void(const Pong&)>>>
    //
    // The error happens because Unknown is not in the variant type list,
    // so std::get can't find it in the tuple -> compile-time safety!

    std::cout << "Type-safe bus: only declared message types accepted\n";
}
```

---

## Notes

- **Closed vs open:** variant-based buses have a fixed message set (add types = recompile). Use an `std::any`-based bus for open/plugin architectures where the set of messages isn't known at compile time.
- The **`overloaded` helper** is the standard C++17 pattern for multi-lambda visitors - it's likely to become a standard library utility in a future standard.
- **Zero overhead:** `std::visit` on a variant compiles to a jump table or if-else chain - no heap allocation at dispatch time.
- **Exhaustiveness:** if you add a new message type to the variant, every `std::visit` call that doesn't handle it becomes a compile error, forcing you to update all dispatch sites.
