# Implement a type-safe state machine with std::variant and a transition table

**Category:** Design Patterns Modern Takes  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

A **variant-based state machine** encodes each state as a separate type in a `std::variant`. State transitions are dispatched via `std::visit`, and the compiler **enforces exhaustiveness** - if you add a new state, every transition handler must be updated or the code won't compile.

The reason this matters is that traditional enum-based state machines let you silently miss cases. Add a new enum value and the compiler might warn you (with `-Wswitch`), but it will still produce a binary. With a variant, the compiler turns that warning into a hard error - you cannot ship incomplete state handling.

### State Machine Structure

```cpp
States:  std::variant<Idle, Connecting, Connected, Disconnected>
Events:  std::variant<Connect, Established, Timeout, Disconnect>

Transition table:
  (Idle,         Connect)      -> Connecting
  (Connecting,   Established)  -> Connected
  (Connecting,   Timeout)      -> Disconnected
  (Connected,    Disconnect)   -> Disconnected
  (any,          any)          -> no change (invalid transition)
```

---

## Self-Assessment

### Q1: Encode states as variant types and transitions as a constexpr table of (from_state, event) -> to_state

The two-argument form of `std::visit` - `std::visit(visitor, state, event)` - is the elegant piece here. It simultaneously pattern-matches on the current state type AND the event type, so each lambda naturally expresses one (state, event) -> new_state transition. The catch-all lambda at the bottom handles every other combination by staying put.

```cpp
#include <iostream>
#include <string>
#include <variant>

// States (each is a distinct type)
struct Idle         { };
struct Connecting   { std::string host; int port; };
struct Connected    { std::string host; int port; int session_id; };
struct Disconnected { std::string reason; };

using State = std::variant<Idle, Connecting, Connected, Disconnected>;

// Events
struct Connect     { std::string host; int port; };
struct Established { int session_id; };
struct Timeout     { };
struct Disconnect  { std::string reason; };

using Event = std::variant<Connect, Established, Timeout, Disconnect>;

// Overload helper
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Transition function
State transition(const State& state, const Event& event) {
    return std::visit(overloaded{
        // Idle + Connect -> Connecting
        [](const Idle&, const Connect& e) -> State {
            std::cout << "Connecting to " << e.host << ":" << e.port << "...\n";
            return Connecting{e.host, e.port};
        },
        // Connecting + Established -> Connected
        [](const Connecting& s, const Established& e) -> State {
            std::cout << "Connected! Session: " << e.session_id << "\n";
            return Connected{s.host, s.port, e.session_id};
        },
        // Connecting + Timeout -> Disconnected
        [](const Connecting&, const Timeout&) -> State {
            std::cout << "Connection timed out\n";
            return Disconnected{"timeout"};
        },
        // Connected + Disconnect -> Disconnected
        [](const Connected&, const Disconnect& e) -> State {
            std::cout << "Disconnected: " << e.reason << "\n";
            return Disconnected{e.reason};
        },
        // All other combinations: stay in current state (invalid transition)
        [](const auto& current_state, const auto&) -> State {
            std::cout << "Invalid transition - staying in current state\n";
            return current_state;
        }
    }, state, event);
}

int main() {
    State state = Idle{};

    state = transition(state, Connect{"server.com", 8080});     // -> Connecting
    state = transition(state, Established{42});                  // -> Connected
    state = transition(state, Connect{"other.com", 9090});       // Invalid (already connected)
    state = transition(state, Disconnect{"user requested"});      // -> Disconnected
}
```

Each call to `transition` returns the next state as a value, so the state machine is effectively stateless from the function's perspective - easy to reason about and easy to test.

**Output:**

```text
Connecting to server.com:8080...
Connected! Session: 42
Invalid transition - staying in current state
Disconnected: user requested
```

### Q2: Use std::visit to dispatch the current state and validate that only legal transitions fire

A door is a great state machine example because the constraints are obvious - you can't push open a locked door, and you can't lock a door that's already open. Watch how the catch-all lambda makes illegal transitions a silent no-op instead of undefined behavior.

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <cassert>

// Door state machine
struct Locked   {};
struct Unlocked {};
struct Open     {};

using DoorState = std::variant<Locked, Unlocked, Open>;

struct Lock   {};
struct Unlock { int key_id; };
struct Push   {};
struct Close  {};

using DoorEvent = std::variant<Lock, Unlock, Push, Close>;

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

struct DoorMachine {
    DoorState state = Locked{};

    void process(const DoorEvent& event) {
        state = std::visit(overloaded{
            // Locked state transitions
            [](const Locked&, const Unlock& e) -> DoorState {
                std::cout << "Unlocked with key " << e.key_id << "\n";
                return Unlocked{};
            },
            // Unlocked state transitions
            [](const Unlocked&, const Lock&) -> DoorState {
                std::cout << "Locked\n";
                return Locked{};
            },
            [](const Unlocked&, const Push&) -> DoorState {
                std::cout << "Door opened\n";
                return Open{};
            },
            // Open state transitions
            [](const Open&, const Close&) -> DoorState {
                std::cout << "Door closed (now unlocked)\n";
                return Unlocked{};
            },
            // Default: invalid transitions
            [](const auto& s, const auto&) -> DoorState {
                std::cout << "Can't do that right now\n";
                return s;  // Stay in current state
            }
        }, state, event);
    }

    void print_state() const {
        std::visit(overloaded{
            [](const Locked&)   { std::cout << "  State: LOCKED\n"; },
            [](const Unlocked&) { std::cout << "  State: UNLOCKED\n"; },
            [](const Open&)     { std::cout << "  State: OPEN\n"; },
        }, state);
    }
};

int main() {
    DoorMachine door;
    door.print_state();                           // LOCKED

    door.process(Push{});                         // Can't do that (locked!)
    door.process(Unlock{42});                     // Unlocked with key 42
    door.print_state();                           // UNLOCKED

    door.process(Push{});                         // Door opened
    door.process(Push{});                         // Can't do that (already open)
    door.process(Close{});                        // Door closed
    door.print_state();                           // UNLOCKED
}
```

### Q3: Show that adding a new state forces the compiler to update all visit handlers

This is arguably the most important advantage of the variant approach over enums. Compare what happens when you add a new traffic light state in each design - with an enum you get a warning you might miss, with a variant you get an error you cannot ignore.

```cpp
#include <iostream>
#include <variant>

// Original states:
struct Red    {};
struct Yellow {};
struct Green  {};

// Adding a new state - uncomment to see compile errors:
// struct FlashingRed {};

// using Light = std::variant<Red, Yellow, Green, FlashingRed>;
using Light = std::variant<Red, Yellow, Green>;

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// This visitor handles all 3 states - compiles fine
std::string light_name(const Light& light) {
    return std::visit(overloaded{
        [](const Red&)    { return std::string("RED"); },
        [](const Yellow&) { return std::string("YELLOW"); },
        [](const Green&)  { return std::string("GREEN"); },
        // If FlashingRed is added to the variant but NOT here:
        // COMPILE ERROR: "no matching function for call to 'overloaded::operator()(const FlashingRed&)'"
        //
        // The compiler FORCES you to handle FlashingRed.
        // This is the key advantage over enum-based state machines
        // where a missing case is just a warning (easily ignored).
    }, light);
}

// Compare with enum approach (NOT type-safe):
enum class LightEnum { Red, Yellow, Green /*, FlashingRed */ };

std::string light_name_enum(LightEnum l) {
    switch (l) {
        case LightEnum::Red:    return "RED";
        case LightEnum::Yellow: return "YELLOW";
        case LightEnum::Green:  return "GREEN";
        // If FlashingRed is added: just a -Wswitch warning (optional!)
        // With variant: hard compile error (mandatory!)
    }
    return "UNKNOWN";
}

int main() {
    Light l = Green{};
    std::cout << light_name(l) << '\n';  // GREEN
}
```

The reason this trips people up is that `auto&&` catch-all lambdas silently absorb new states, which defeats the whole point. If you use a catch-all at the bottom, you're back to the same situation as an unhandled enum case. Only a complete, explicit visitor gets you the compile-time guarantee.

---

## Notes

- **Two-variant visit:** `std::visit(visitor, state, event)` dispatches on both state AND event type simultaneously - this is what makes clean transition tables possible.
- **Stateful states:** variant alternatives can hold data (unlike enums) - for example, `Connected{host, session_id}` carries the connection details right inside the state type.
- **No runtime overhead:** `std::visit` compiles to a jump table, matching `switch` performance.
- **Libraries:** Boost.SML and Boost.MSM provide production-grade variant-based state machines with action/guard support if you need something more sophisticated.
