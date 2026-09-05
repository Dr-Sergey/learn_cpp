# Use the State pattern with std::variant for finite state machines

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

A **finite state machine (FSM)** modeled with `std::variant` represents each state as a distinct type and transitions as functions returning the next state. Unlike OOP-style State pattern (virtual methods + heap allocation), the variant approach is:

- **Value-based** - no heap, no pointers
- **Exhaustive** - the compiler forces you to handle every state x event combination
- **Cache-friendly** - variant stores states inline

The reason this is appealing is that the classic OOP State pattern requires a separate class per state, a virtual base, heap allocation for each state object, and no compile-time guarantee that you've handled every combination. With `std::variant`, each state is just a plain struct, the current state is stored by value, and `std::visit` ensures at compile time that you haven't forgotten any state.

```cpp
┌─────────┐  connect()   ┌──────────┐  ack()   ┌──────────────┐
│  Closed  │─────────────│  Listen   │──────────│  Established  │
└─────────┘              └──────────┘          └──────────────┘
     ^                                               |
     |              close()                           |
     └────────────────────────────────────────────────┘
```

### State Pattern: OOP vs Variant

| Feature | Virtual State Pattern | Variant State Pattern |
| --- | --- | --- |
| State storage | Heap-allocated (unique_ptr) | Inline (stack) |
| Dispatch | Virtual call (runtime) | `std::visit` (compile-time table) |
| New state | Open - add derived class | Closed - add to variant |
| New event | Must add to interface | Add overload to handler |
| Exhaustiveness | No compile-time guarantee | Yes - compiler catch |

---

## Self-Assessment

### Q1: Model a TCP connection state machine using `std::variant<Closed, Listen, SynReceived, Established>`

Each state is a plain struct that can carry state-specific data. The FSM itself is just a `using` alias around `std::variant`. Transitions are free functions that take the current state plus an event and return the next state:

```cpp
#include <iostream>
#include <string>
#include <variant>

// --- States (each is a simple type) ---
struct Closed      { };
struct Listen      { int port; };
struct SynReceived { std::string client_ip; };
struct Established { std::string client_ip; int port; };

// The state machine is just a variant!
using TcpState = std::variant<Closed, Listen, SynReceived, Established>;

// --- Events ---
struct EvListen   { int port; };
struct EvConnect  { std::string ip; };
struct EvAck      { };
struct EvClose    { };

// --- Transition functions (per-state event handlers) ---
// Each returns the new TcpState.

// Handle events in Closed state
TcpState on_event(Closed, EvListen ev) {
    std::cout << "Closed -> Listen on port " << ev.port << "\n";
    return Listen{ev.port};
}

// Handle events in Listen state
TcpState on_event(Listen s, EvConnect ev) {
    std::cout << "Listen -> SynReceived from " << ev.ip << "\n";
    return SynReceived{ev.ip};
}

TcpState on_event(Listen, EvClose) {
    std::cout << "Listen -> Closed\n";
    return Closed{};
}

// Handle events in SynReceived state
TcpState on_event(SynReceived s, EvAck) {
    std::cout << "SynReceived -> Established with " << s.client_ip << "\n";
    return Established{s.client_ip, 80};
}

// Handle events in Established state
TcpState on_event(Established, EvClose) {
    std::cout << "Established -> Closed\n";
    return Closed{};
}

// --- Catch-all for invalid transitions ---
template <typename State, typename Event>
TcpState on_event(State, Event) {
    std::cout << "Invalid transition! Staying in current state.\n";
    return State{};  // remain
}

// --- Dispatch helper ---
template <typename Event>
void send_event(TcpState& state, Event ev) {
    state = std::visit(
        [&ev](auto& s) -> TcpState { return on_event(s, ev); },
        state
    );
}

// --- Pretty-print current state ---
void print_state(const TcpState& state) {
    std::visit([](const auto& s) {
        using T = std::decay_t<decltype(s)>;
        if constexpr (std::is_same_v<T, Closed>)
            std::cout << "  State: Closed\n";
        else if constexpr (std::is_same_v<T, Listen>)
            std::cout << "  State: Listen (port " << s.port << ")\n";
        else if constexpr (std::is_same_v<T, SynReceived>)
            std::cout << "  State: SynReceived (" << s.client_ip << ")\n";
        else if constexpr (std::is_same_v<T, Established>)
            std::cout << "  State: Established (" << s.client_ip << ")\n";
    }, state);
}

int main() {
    TcpState conn = Closed{};
    print_state(conn);

    send_event(conn, EvListen{8080});
    print_state(conn);

    send_event(conn, EvConnect{"192.168.1.1"});
    print_state(conn);

    send_event(conn, EvAck{});
    print_state(conn);

    send_event(conn, EvClose{});
    print_state(conn);

    // Invalid transition:
    send_event(conn, EvAck{});  // Closed + Ack -> invalid
}
// Expected output:
//   State: Closed
//   Closed -> Listen on port 8080
//   State: Listen (port 8080)
//   Listen -> SynReceived from 192.168.1.1
//   State: SynReceived (192.168.1.1)
//   SynReceived -> Established with 192.168.1.1
//   State: Established (192.168.1.1, port 80)
//   Established -> Closed
//   State: Closed
//   Invalid transition! Staying in current state.
```

The `send_event` helper visits the current state and calls the appropriate `on_event` overload. The template catch-all at the bottom handles invalid transitions gracefully without requiring you to enumerate every impossible combination.

---

### Q2: Use `std::visit` to dispatch events to state-specific handlers

This example uses a visitor struct with overloaded `operator()` instead of free functions. Visiting on *two* variants at once is what makes `std::visit` so powerful here - it dispatches on the state and the event simultaneously:

```cpp
#include <iostream>
#include <string>
#include <variant>

// Turnstile FSM: Locked <-> Unlocked
struct Locked   {};
struct Unlocked {};

using State = std::variant<Locked, Unlocked>;

struct InsertCoin {};
struct Push       {};

using Event = std::variant<InsertCoin, Push>;

// Transition table via overloaded visitor
struct Transition {
    // Locked + InsertCoin -> Unlocked
    State operator()(Locked, InsertCoin) const {
        std::cout << "Coin inserted. Unlocking.\n";
        return Unlocked{};
    }
    // Locked + Push -> Locked (blocked)
    State operator()(Locked, Push) const {
        std::cout << "Locked! Insert coin first.\n";
        return Locked{};
    }
    // Unlocked + Push -> Locked (pass through)
    State operator()(Unlocked, Push) const {
        std::cout << "Passing through. Locking.\n";
        return Locked{};
    }
    // Unlocked + InsertCoin -> Unlocked (thank you!)
    State operator()(Unlocked, InsertCoin) const {
        std::cout << "Already unlocked. Thank you!\n";
        return Unlocked{};
    }
};

// Double-visit: dispatch on BOTH state AND event
void process(State& state, const Event& event) {
    state = std::visit(Transition{}, state, event);
    // ^^^^^^^^^^^^^^^^ visits (State, Event) pair!
}

int main() {
    State turnstile = Locked{};

    process(turnstile, Push{});        // Locked + Push
    process(turnstile, InsertCoin{});  // Locked + Coin
    process(turnstile, InsertCoin{});  // Unlocked + Coin
    process(turnstile, Push{});        // Unlocked + Push
    process(turnstile, Push{});        // Locked + Push
}
// Expected output:
//   Locked! Insert coin first.
//   Coin inserted. Unlocking.
//   Already unlocked. Thank you!
//   Passing through. Locking.
//   Locked! Insert coin first.
```

**Key insight:** `std::visit(visitor, variant1, variant2)` dispatches on **both** variants simultaneously - this is the multi-variant visit that replaces double dispatch.

---

### Q3: Show that illegal transitions are compile errors when each state only handles its valid events

This is where the variant FSM really shines compared to a `switch`-based approach. With `switch` + enum, you can forget a case and the compiler might only give you a warning. With `std::visit` and no catch-all, a missing combination is a hard compile error:

```cpp
#include <iostream>
#include <variant>
#include <string>

// --- Door FSM with compile-time safety ---
struct Open   {};
struct Closed {};
struct Jammed {};

using DoorState = std::variant<Open, Closed, Jammed>;

// Events
struct EvOpen  {};
struct EvClose {};
struct EvJam   {};

// Transition table -- only VALID transitions are defined
struct DoorTransition {
    // Closed -> Open (valid)
    DoorState operator()(Closed, EvOpen) {
        std::cout << "Opening door.\n";
        return Open{};
    }
    // Open -> Closed (valid)
    DoorState operator()(Open, EvClose) {
        std::cout << "Closing door.\n";
        return Closed{};
    }
    // Any state -> Jammed
    DoorState operator()(auto, EvJam) {
        std::cout << "Door jammed!\n";
        return Jammed{};
    }

    // --- NO handler for (Jammed, EvOpen) or (Jammed, EvClose) ---
    // If someone tries to visit with those combinations,
    // the compiler will emit an error: "no matching function for call"

    // --- Catch remaining combinations as no-ops ---
    // COMMENT OUT this catch-all to see compile errors for unhandled transitions:
    // DoorState operator()(auto state, auto) {
    //     std::cout << "Invalid transition.\n";
    //     return state;
    // }
};

// To demonstrate compile-time safety, we use explicit overloads:
// If we define EVERY valid transition and rely on the compiler to
// reject the rest, adding a new state forces updating the table.

// With catch-all REMOVED and you try:
//   std::visit(DoorTransition{}, DoorState{Jammed{}}, EvOpen{});
// Compiler error:
//   error: no match for call to '(DoorTransition)(Jammed&, EvOpen&)'

// SAFE version with explicit catch-all for remaining:
struct SafeDoorTransition {
    DoorState operator()(Closed, EvOpen)  { return Open{}; }
    DoorState operator()(Open, EvClose)   { return Closed{}; }
    DoorState operator()(auto, EvJam)     { return Jammed{}; }
    // Everything else -> stay in current state
    DoorState operator()(auto s, auto) {
        std::cout << "No valid transition.\n";
        // Return current state -- requires wrapping:
        return DoorState{s};
    }
};

int main() {
    DoorState door = Closed{};

    auto dispatch = [](DoorState& s, auto event) {
        s = std::visit(SafeDoorTransition{}, s, std::variant<EvOpen, EvClose, EvJam>{event});
    };

    dispatch(door, EvOpen{});   // Closed -> Open
    dispatch(door, EvClose{});  // Open -> Closed
    dispatch(door, EvJam{});    // Closed -> Jammed
    dispatch(door, EvOpen{});   // Jammed -> No valid transition
}
// Expected output:
//   Opening door.
//   Closing door.
//   Door jammed!
//   No valid transition.
```

**How compile-time exhaustiveness works:**

1. `std::visit` requires the visitor to handle **every** combination of variant alternatives
2. If you don't provide a catch-all `operator()(auto, auto)`, missing combinations = **compile error**
3. When you add a new state to the variant (e.g., `Maintenance`), **every** visitor that lacks a handler for it fails to compile
4. This is the **closed set** advantage: the compiler knows all possible types at compile time

---

## Notes

- **SBO (Small Buffer Optimization):** `std::variant` stores the active state inline - no heap allocation. Size = `max(sizeof(State_i)) + discriminant`.
- **`std::monostate`:** Use as first variant alternative for default-constructible variant: `variant<monostate, State1, State2>`.
- **Performance:** `std::visit` generates a jump table - O(1) dispatch, comparable to virtual call but with no indirection through heap.
- **Hierarchical FSMs:** Nest variants: `using SubState = variant<A, B>; using TopState = variant<SubState, C, D>;`
- **Libraries:** Boost.SML and Boost.MSM provide more powerful FSM frameworks, but `variant` is sufficient for many cases.
- **vs. `switch` + enum:** The enum approach compiles, but nothing prevents forgetting a case. The variant approach forces exhaustive handling.
