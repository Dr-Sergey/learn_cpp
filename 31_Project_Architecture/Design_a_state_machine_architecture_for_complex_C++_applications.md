# Design a state machine architecture for complex C++ applications

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**State machines** model systems where behavior depends on the current state and an incoming event. You encounter them everywhere: protocol parsers, UI workflows, device drivers, game logic, connection managers. The core promise is that transitions are explicit - you can look at the code and see every valid path. That makes impossible states stay impossible, and it makes testing dramatically easier.

Modern C++ gives you a particularly nice tool for this: `std::variant`-based state machines. The reason this approach stands out is compile-time exhaustiveness. If you forget to handle a (state, event) combination, the compiler tells you before the program runs. That's a much better feedback loop than discovering a missed case in production.

### State Machine Implementation Approaches

If the table feels like a lot, the key tradeoff is between flexibility and safety. `switch/enum` is fast to write but lets invalid combinations slip through silently. `std::variant + std::visit` is a bit more ceremony upfront but gives you the compiler as a correctness partner.

| Approach | Type Safety | Overhead | Extensibility |
| --- | --- | --- | --- |
| **switch/enum** | Low (no exhaustiveness) | Zero | Hard to extend |
| **State pattern (virtual)** | Medium | vtable | Open/closed |
| **std::variant + visit** | High (compile-time) | Small | Closed set |
| **Boost.SML** | Very high | Zero (compile-time) | DSL-defined |

---

## Self-Assessment

### Q1: Implement a variant-based state machine with compile-time safety

**Answer:**

Here's a connection state machine that models the full lifecycle of a network session - idle, connecting (with retries), connected, disconnecting, and error. Each state is its own struct so it can carry state-specific data. Notice how the `Transition` struct handles each (state, event) pair as a separate overload - this is what gives us exhaustiveness checking.

```cpp
#include <variant>
#include <string>
#include <iostream>
#include <optional>

// === States (each state can carry data) ===
struct Idle {};
struct Connecting { std::string address; int retries = 0; };
struct Connected  { int session_id; };
struct Disconnecting { std::string reason; };
struct Error { std::string message; int code; };

using State = std::variant<Idle, Connecting, Connected,
                            Disconnecting, Error>;

// === Events ===
struct Connect     { std::string address; };
struct ConnectOk   { int session_id; };
struct ConnectFail { std::string error; };
struct Disconnect  { std::string reason; };
struct DisconnectOk {};
struct Timeout {};
struct Reset {};

using Event = std::variant<Connect, ConnectOk, ConnectFail,
                            Disconnect, DisconnectOk, Timeout, Reset>;

// === Transition function ===
struct Transition {
    // Idle + Connect -> Connecting
    State operator()(Idle&, Connect& e) {
        return Connecting{e.address};
    }

    // Connecting + ConnectOk -> Connected
    State operator()(Connecting&, ConnectOk& e) {
        return Connected{e.session_id};
    }

    // Connecting + ConnectFail -> Error or retry
    State operator()(Connecting& s, ConnectFail& e) {
        if (s.retries < 3)
            return Connecting{s.address, s.retries + 1};
        return Error{e.error, -1};
    }

    // Connecting + Timeout -> retry or Error
    State operator()(Connecting& s, Timeout&) {
        if (s.retries < 3)
            return Connecting{s.address, s.retries + 1};
        return Error{"Connection timeout", -2};
    }

    // Connected + Disconnect -> Disconnecting
    State operator()(Connected&, Disconnect& e) {
        return Disconnecting{e.reason};
    }

    // Disconnecting + DisconnectOk -> Idle
    State operator()(Disconnecting&, DisconnectOk&) {
        return Idle{};
    }

    // Any state + Reset -> Idle
    State operator()(auto&, Reset&) {
        return Idle{};
    }

    // Default: remain in current state (invalid transition)
    State operator()(auto& state, auto&) {
        return state;  // No transition
    }
};

// === State Machine ===
class ConnectionStateMachine {
public:
    void process(Event event) {
        state_ = std::visit(
            [](auto& s, auto& e) -> State {
                return Transition{}(s, e);
            },
            state_, event);
    }

    const State& current() const { return state_; }

    std::string state_name() const {
        return std::visit([](const auto& s) -> std::string {
            using T = std::decay_t<decltype(s)>;
            if constexpr (std::is_same_v<T, Idle>)           return "Idle";
            if constexpr (std::is_same_v<T, Connecting>)     return "Connecting";
            if constexpr (std::is_same_v<T, Connected>)      return "Connected";
            if constexpr (std::is_same_v<T, Disconnecting>)  return "Disconnecting";
            if constexpr (std::is_same_v<T, Error>)          return "Error";
        }, state_);
    }

private:
    State state_ = Idle{};
};
```

The key design move here is that `process()` replaces the current state entirely - it doesn't mutate it in place. Each transition returns a brand new state value. That makes the state machine easy to reason about, because each state transition is a pure function from (old state, event) to new state.

### Q2: Test state machine transitions exhaustively

**Answer:**

The nice thing about a variant-based state machine is that the test code reads almost like a table of expected behavior. You push events in, check the resulting state name, and optionally inspect the state-specific data. This test suite covers valid transitions, the retry logic, and the catch-all for invalid inputs - that last one is important to confirm the machine does nothing when it shouldn't.

```cpp
#include <gtest/gtest.h>

TEST(ConnectionSM, IdleToConnecting) {
    ConnectionStateMachine sm;
    sm.process(Connect{"192.168.1.1"});
    EXPECT_EQ(sm.state_name(), "Connecting");
    EXPECT_EQ(std::get<Connecting>(sm.current()).address, "192.168.1.1");
}

TEST(ConnectionSM, ConnectingToConnected) {
    ConnectionStateMachine sm;
    sm.process(Connect{"host"});
    sm.process(ConnectOk{42});
    EXPECT_EQ(sm.state_name(), "Connected");
    EXPECT_EQ(std::get<Connected>(sm.current()).session_id, 42);
}

TEST(ConnectionSM, RetriesOnFailure) {
    ConnectionStateMachine sm;
    sm.process(Connect{"host"});

    for (int i = 0; i < 3; ++i) {
        sm.process(ConnectFail{"refused"});
        EXPECT_EQ(sm.state_name(), "Connecting");
        EXPECT_EQ(std::get<Connecting>(sm.current()).retries, i + 1);
    }

    // 4th failure -> Error
    sm.process(ConnectFail{"refused"});
    EXPECT_EQ(sm.state_name(), "Error");
}

TEST(ConnectionSM, ResetFromAnyState) {
    ConnectionStateMachine sm;
    sm.process(Connect{"host"});
    sm.process(ConnectOk{1});
    EXPECT_EQ(sm.state_name(), "Connected");

    sm.process(Reset{});
    EXPECT_EQ(sm.state_name(), "Idle");
}

TEST(ConnectionSM, InvalidTransitionIgnored) {
    ConnectionStateMachine sm;  // Idle
    sm.process(ConnectOk{1});   // Invalid: Idle + ConnectOk
    EXPECT_EQ(sm.state_name(), "Idle");  // Unchanged
}
```

Notice that `InvalidTransitionIgnored` is just as important as the happy-path tests. The catch-all overload in `Transition` should leave the machine unchanged - that's the contract, and it needs to be verified.

### Q3: Add entry/exit actions and transition guards

**Answer:**

A pure state machine only tracks which state you're in. Real applications also need to do things when entering or leaving a state - start a timer, flush a buffer, log an event, cancel pending requests. The `WithActions` wrapper adds that layer cleanly without touching the core state machine logic. It captures both the old and new state, then fires the appropriate callbacks at each boundary.

```cpp
// === State machine with actions and guards ===
template<typename StateMachine>
class WithActions {
public:
    void process(Event event) {
        auto old_state = sm_.current();

        // Exit action for old state
        std::visit([this](const auto& s) { on_exit(s); }, old_state);

        sm_.process(std::move(event));

        // Entry action for new state (only if state changed)
        if (old_state.index() != sm_.current().index()) {
            std::visit([this](const auto& s) { on_enter(s); },
                       sm_.current());
        }
    }

    const State& current() const { return sm_.current(); }

private:
    void on_enter(const Connecting& s) {
        std::cout << "Starting connection to " << s.address << "\n";
        // Start connection timer
    }
    void on_enter(const Connected& s) {
        std::cout << "Connected, session=" << s.session_id << "\n";
    }
    void on_enter(const Error& s) {
        std::cout << "Error: " << s.message << "\n";
        // Log, alert, trigger recovery
    }
    void on_enter(const auto&) {}  // Default: no action

    void on_exit(const Connected&) {
        std::cout << "Leaving connected state\n";
        // Flush buffers, cancel pending requests
    }
    void on_exit(const auto&) {}  // Default: no action

    StateMachine sm_;
};

// Usage:
// WithActions<ConnectionStateMachine> sm;
// sm.process(Connect{"host"});  // Prints: Starting connection to host
// sm.process(ConnectOk{1});     // Prints: Connected, session=1
```

The check `old_state.index() != sm_.current().index()` is a compact way to ask "did the state actually change?" - `index()` returns which alternative of the variant is currently active. If the machine stayed in the same state (due to the catch-all), entry actions don't fire, which is the correct behavior.

---

## Notes

- `std::variant` + `std::visit` gives **compile-time exhaustiveness** - a missing transition overload is a compile error, not a runtime surprise.
- Each state struct can carry its own data, such as the address and retry count in `Connecting` or the session ID in `Connected`.
- The `auto& state, auto& event` catch-all handles invalid transitions gracefully by leaving the machine unchanged.
- Test every valid transition AND every invalid one - the catch-all contract must be verified separately.
- Entry and exit actions are the right place for resource acquisition and cleanup at state boundaries.
- For large state machines with more than 15 states, consider Boost.SML for a more declarative DSL-based approach.
- Hierarchical state machines, where a state contains a nested state machine, are an extension of this pattern known as HSM.
