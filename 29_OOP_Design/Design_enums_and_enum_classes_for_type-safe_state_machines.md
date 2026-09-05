# Design enums and enum classes for type-safe state machines

**Category:** OOP Design  
**Standard:** C++11  

---

## Topic Overview

**Enum classes** (scoped enumerations, introduced in C++11) are the foundation for type-safe state machines in modern C++. The reason you want them over the old-style `enum` is that they fix three real annoyances: they don't implicitly convert to integers (which causes subtle bugs), they don't leak their names into the enclosing scope (so `Red` doesn't accidentally collide with some other `Red`), and they let you specify an explicit underlying type for storage control. Combined with `std::variant`, exhaustive `switch` handling, and a compile-time transition table, they let you build finite state machines (FSMs) that the compiler can actually help you get right.

### Unscoped vs Scoped Enums

If the table feels like a lot, the key rows to remember are implicit int conversion (dangerous in unscoped enums, prohibited in scoped ones) and scope (scoped enums require `Color::Red`, which is more typing but eliminates name collisions).

| Feature | `enum Color { Red }` | `enum class Color { Red }` |
| --- | --- | --- |
| Scope | Enclosing namespace | `Color::Red` required |
| Implicit int conversion | Yes (dangerous) | No (explicit cast needed) |
| Forward declarable | Only with underlying type | Yes (default: `int`) |
| Underlying type control | Optional (C++11) | Optional (default: `int`) |
| Bitwise operators | Work implicitly | Must overload manually |
| `switch` completeness | Compiler may warn | Compiler may warn |

### State Machine Patterns

Here is the state diagram for the connection FSM used in Q1 below. Mentally walking through the arrows before reading the code will make the transition table much easier to follow.

```cpp
  ┌────────┐     connect()     ┌────────────┐    auth()     ┌───────────────┐
  │  Idle  │ ──────────► │ Connecting  │ ────────► │ Authenticated │
  └────────┘                 └──────┬─────┘                └───────┬───────┘
       ▲                      │ error()                     │ disconnect()
       │                ┌────▼────┐                         │
       └────────────────┤  Error  │◄─────────────────────────┘
        retry()         └─────────┘
```

---

## Self-Assessment

### Q1: Build a type-safe state machine using enum class with proper transition validation

The key idea here is separating the transition table from the FSM class itself. The table is a compile-time array of `{from, event, to}` triples, and a constexpr function does the lookup. This means the FSM class `handle()` method is just "look up the next state, apply it, return success/failure" - the policy lives in the table, not scattered through a big switch.

```cpp
#include <iostream>
#include <string>
#include <stdexcept>
#include <array>
#include <optional>

// === Enum-based FSM with transition table ===

enum class ConnectionState : uint8_t {
    Idle,
    Connecting,
    Connected,
    Authenticated,
    Disconnecting,
    Error,
    COUNT  // Sentinel for array sizing
};

enum class Event : uint8_t {
    Connect,
    Connected,
    Authenticate,
    Disconnect,
    Failure,
    Retry,
    COUNT
};

// String conversion for logging (no implicit int conversion!)
constexpr const char* to_string(ConnectionState s) {
    switch (s) {
        case ConnectionState::Idle:           return "Idle";
        case ConnectionState::Connecting:     return "Connecting";
        case ConnectionState::Connected:      return "Connected";
        case ConnectionState::Authenticated:  return "Authenticated";
        case ConnectionState::Disconnecting:  return "Disconnecting";
        case ConnectionState::Error:          return "Error";
        default:                              return "Unknown";
    }
}

constexpr const char* to_string(Event e) {
    switch (e) {
        case Event::Connect:      return "Connect";
        case Event::Connected:    return "Connected";
        case Event::Authenticate: return "Authenticate";
        case Event::Disconnect:   return "Disconnect";
        case Event::Failure:      return "Failure";
        case Event::Retry:        return "Retry";
        default:                  return "Unknown";
    }
}

// Compile-time transition table
using S = ConnectionState;
using E = Event;

struct Transition {
    S from;
    E event;
    S to;
};

constexpr std::array transitions = {
    Transition{S::Idle,          E::Connect,      S::Connecting},
    Transition{S::Connecting,    E::Connected,    S::Connected},
    Transition{S::Connected,     E::Authenticate, S::Authenticated},
    Transition{S::Authenticated, E::Disconnect,   S::Disconnecting},
    Transition{S::Connected,     E::Disconnect,   S::Disconnecting},
    Transition{S::Disconnecting, E::Disconnect,   S::Idle},  // completed
    Transition{S::Connecting,    E::Failure,      S::Error},
    Transition{S::Connected,     E::Failure,      S::Error},
    Transition{S::Authenticated, E::Failure,      S::Error},
    Transition{S::Error,         E::Retry,        S::Idle},
};

constexpr std::optional<S> next_state(S current, E event) {
    for (const auto& t : transitions) {
        if (t.from == current && t.event == event)
            return t.to;
    }
    return std::nullopt;  // Invalid transition
}

class ConnectionFSM {
public:
    bool handle(Event event) {
        auto next = next_state(state_, event);
        if (!next) {
            std::cerr << "Invalid transition: " << to_string(state_)
                      << " + " << to_string(event) << '\n';
            return false;
        }
        std::cout << to_string(state_) << " --[" << to_string(event)
                  << "]--> " << to_string(*next) << '\n';
        state_ = *next;
        return true;
    }

    ConnectionState state() const { return state_; }

private:
    ConnectionState state_ = ConnectionState::Idle;
};

int main() {
    ConnectionFSM fsm;

    fsm.handle(Event::Connect);       // Idle -> Connecting
    fsm.handle(Event::Connected);     // Connecting -> Connected
    fsm.handle(Event::Authenticate);  // Connected -> Authenticated
    fsm.handle(Event::Failure);       // Authenticated -> Error
    fsm.handle(Event::Retry);         // Error -> Idle

    // Invalid transition:
    fsm.handle(Event::Authenticate);  // Idle + Authenticate = invalid
}
```

Notice how adding a new event or state only requires touching the `transitions` array and the `to_string` helpers - the FSM class itself needs no changes. That's exactly the kind of separation you want in maintainable code.

### Q2: Explain how to add flags/bitmask support to enum classes safely

This is the trade-off you accept with `enum class`: no implicit bitwise operators, so you have to provide them yourself. The approach below uses a concept (`BitmaskEnum`) gated by an explicit opt-in trait (`EnableBitmask`), which means you don't accidentally give every enum bitwise operators - only the ones you declare as bitmasks.

```cpp
#include <type_traits>
#include <iostream>

// === Type-safe bitmask enum ===

enum class Permission : uint32_t {
    None    = 0,
    Read    = 1 << 0,
    Write   = 1 << 1,
    Execute = 1 << 2,
    Delete  = 1 << 3,
    Admin   = Read | Write | Execute | Delete,  // Composite
};

// SFINAE-gated operators: only enabled for types that opt in
template<typename E>
struct EnableBitmask : std::false_type {};

template<>
struct EnableBitmask<Permission> : std::true_type {};

template<typename E>
concept BitmaskEnum = EnableBitmask<E>::value && std::is_enum_v<E>;

template<BitmaskEnum E>
constexpr E operator|(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) | static_cast<U>(rhs));
}

template<BitmaskEnum E>
constexpr E operator&(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) & static_cast<U>(rhs));
}

template<BitmaskEnum E>
constexpr E operator~(E val) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(~static_cast<U>(val));
}

template<BitmaskEnum E>
constexpr E& operator|=(E& lhs, E rhs) { return lhs = lhs | rhs; }

template<BitmaskEnum E>
constexpr E& operator&=(E& lhs, E rhs) { return lhs = lhs & rhs; }

// Helper: test if flag is set
template<BitmaskEnum E>
constexpr bool has_flag(E value, E flag) {
    return (value & flag) == flag;
}

void check_access(Permission perms) {
    if (has_flag(perms, Permission::Read))
        std::cout << "Can read\n";
    if (has_flag(perms, Permission::Write))
        std::cout << "Can write\n";
    if (has_flag(perms, Permission::Execute))
        std::cout << "Can execute\n";
}

int main() {
    auto user_perms = Permission::Read | Permission::Write;
    check_access(user_perms);
    // Output: Can read\nCan write

    // This does NOT compile (type-safe!):
    // int x = user_perms;         // ERROR: no implicit conversion
    // user_perms = 3;             // ERROR: no implicit conversion
    // if (user_perms)             // ERROR: no implicit bool conversion
    // user_perms | 0x10;          // ERROR: no mixed int/enum
}
```

The boilerplate is a one-time cost per project. Once you have `EnableBitmask` and the operator templates in a header, opting any new enum into bitmask behavior is a single specialization.

**Key trade-offs:**

- Bitmask enums require boilerplate operators (one-time cost per project).
- Alternative: use `std::bitset<N>` for large flag sets (but loses named semantics).
- C++23 `std::to_underlying()` replaces `static_cast<underlying_type_t<E>>` verbosity.

### Q3: Implement a variant-based state machine with per-state data

The limitation of the enum-based FSM above is that every state shares the same flat data. What if `Connecting` needs to remember the host and port, but `Authenticated` needs a token? With an enum you end up with a struct that has fields for every state, most of which are garbage in any given state. `std::variant` solves this by making each state its own type that carries exactly the data it needs.

```cpp
#include <variant>
#include <string>
#include <iostream>
#include <optional>

// === States carry their own data ===
struct Idle {};

struct Connecting {
    std::string host;
    int port;
    int attempt = 1;
};

struct Connected {
    std::string host;
    int port;
    int socket_fd;
};

struct Authenticated {
    std::string host;
    int socket_fd;
    std::string user;
    std::string token;
};

struct Error {
    std::string message;
    int last_errno;
};

using State = std::variant<Idle, Connecting, Connected, Authenticated, Error>;

// === Events ===
struct ConnectEvent { std::string host; int port; };
struct ConnectedEvent { int socket_fd; };
struct AuthEvent { std::string user; std::string password; };
struct AuthSuccessEvent { std::string token; };
struct DisconnectEvent {};
struct ErrorEvent { std::string message; int code; };
struct RetryEvent {};

// === Transition function using std::visit + overload pattern ===
template<class... Ts> struct overload : Ts... { using Ts::operator()...; };

State on_event(const State& state, const ConnectEvent& evt) {
    return std::visit(overload{
        [&](const Idle&) -> State {
            std::cout << "Connecting to " << evt.host << ":" << evt.port << "\n";
            return Connecting{evt.host, evt.port};
        },
        [](const auto& s) -> State {
            std::cout << "Cannot connect: not in Idle state\n";
            return s;  // No transition
        }
    }, state);
}

State on_event(const State& state, const ConnectedEvent& evt) {
    return std::visit(overload{
        [&](const Connecting& s) -> State {
            std::cout << "Connected (fd=" << evt.socket_fd << ")\n";
            return Connected{s.host, s.port, evt.socket_fd};
        },
        [](const auto& s) -> State { return s; }
    }, state);
}

State on_event(const State& state, const ErrorEvent& evt) {
    return std::visit(overload{
        [&](const Idle&) -> State { return Idle{}; },  // ignore in Idle
        [&](const auto&) -> State {
            std::cout << "Error: " << evt.message << " (" << evt.code << ")\n";
            return Error{evt.message, evt.code};
        }
    }, state);
}

State on_event(const State& state, const RetryEvent&) {
    return std::visit(overload{
        [](const Error& e) -> State {
            std::cout << "Retrying after error: " << e.message << "\n";
            return Idle{};
        },
        [](const auto& s) -> State { return s; }
    }, state);
}

// === Usage ===

void print_state(const State& s) {
    std::visit(overload{
        [](const Idle&)            { std::cout << "  State: Idle\n"; },
        [](const Connecting& c)    { std::cout << "  State: Connecting to " << c.host << "\n"; },
        [](const Connected& c)     { std::cout << "  State: Connected (fd=" << c.socket_fd << ")\n"; },
        [](const Authenticated& a) { std::cout << "  State: Authenticated as " << a.user << "\n"; },
        [](const Error& e)         { std::cout << "  State: Error - " << e.message << "\n"; },
    }, s);
}

int main() {
    State state = Idle{};
    print_state(state);

    state = on_event(state, ConnectEvent{"db.example.com", 5432});
    print_state(state);

    state = on_event(state, ErrorEvent{"Connection refused", -1});
    print_state(state);

    state = on_event(state, RetryEvent{});
    print_state(state);

    state = on_event(state, ConnectEvent{"db.example.com", 5432});
    state = on_event(state, ConnectedEvent{42});
    print_state(state);
}
```

When you add a new state to the variant, every `std::visit` call that doesn't handle it becomes a compile error. That's the compiler telling you exactly where you forgot to update your logic - far better than a runtime `default:` case silently doing nothing.

**Advantages of variant-based FSM over enum-based:**

- Each state carries its own data - no unused fields.
- Compiler enforces exhaustive handling of all states.
- Adding a new state causes compile errors everywhere it's unhandled.
- No invalid state+data combinations possible.

---

## Notes

- Always prefer `enum class` over `enum` - the scoping and type safety are worth the verbosity.
- Use `static_assert(std::is_same_v<std::underlying_type_t<E>, uint8_t>)` to enforce size.
- C++23's `std::to_underlying()` cleans up casts: `std::to_underlying(Permission::Read)` -> `1`.
- For complex FSMs, consider libraries like Boost.SML (State Machine Language) or Boost.Statechart.
- Variant-based FSMs excel when states have different data; enum-based FSMs excel for simple flag-like states.
- `-Wswitch-enum` (GCC/Clang) warns on missing cases - enable it for FSM enums.
