# Implement a type-safe state machine with std::variant and transition tables

**Category:** Design Patterns Modern Takes  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

This second state machine file focuses on **transition tables as data** - encoding transitions in a constexpr array rather than inline in `std::visit` lambdas. It also demonstrates how adding a new state creates compile errors in incomplete visitors.

The distinction is worth understanding. In the code-driven approach, each transition is a lambda body embedded inside a big `std::visit` call - easy to write, and good when transitions involve complex logic. In the data-driven approach, the table of `{from, event, to}` tuples lives in one place you can read like a chart. The code is simpler, and you can even validate transitions with `static_assert` at compile time.

### Data-Driven vs Code-Driven Transitions

```cpp
Code-driven (visit lambdas):
  Transitions are spread across lambda bodies

  + Easy for complex transition logic (guards, actions)
  - Hard to visualize all transitions at a glance

Data-driven (transition table):
  constexpr array of {from, event, to, guard, action}

  + All transitions visible in one table
  + Easy to add/remove transitions
  - Less flexible for complex actions
```

---

## Self-Assessment

### Q1: Encode states as variant alternatives and transitions as overloaded visit lambdas

A motor controller is a nice example here because it has a natural state that carries data - a `Running` state needs to remember the speed, and a `Paused` state needs to remember what speed to resume at. Enums can't hold that data; variant alternatives can.

```cpp
#include <iostream>
#include <variant>
#include <string>

// States
struct Idle     { };
struct Running  { int speed; };
struct Paused   { int saved_speed; };
struct Stopped  { std::string reason; };

using State = std::variant<Idle, Running, Paused, Stopped>;

// Events
struct Start { int speed; };
struct Pause {};
struct Resume {};
struct Stop  { std::string reason; };

using Event = std::variant<Start, Pause, Resume, Stop>;

// Overload helper
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Transitions encoded as overloaded visit lambdas
State next_state(const State& state, const Event& event) {
    return std::visit(overloaded{
        [](const Idle&, const Start& e) -> State {
            std::cout << "Starting at speed " << e.speed << "\n";
            return Running{e.speed};
        },
        [](const Running& s, const Pause&) -> State {
            std::cout << "Pausing (was speed " << s.speed << ")\n";
            return Paused{s.speed};
        },
        [](const Running&, const Stop& e) -> State {
            std::cout << "Stopping: " << e.reason << "\n";
            return Stopped{e.reason};
        },
        [](const Paused& s, const Resume&) -> State {
            std::cout << "Resuming at speed " << s.saved_speed << "\n";
            return Running{s.saved_speed};
        },
        [](const Paused&, const Stop& e) -> State {
            std::cout << "Stopping from pause: " << e.reason << "\n";
            return Stopped{e.reason};
        },
        // Catch-all for invalid transitions
        [](const auto& s, const auto&) -> State {
            std::cout << "Invalid transition\n";
            return s;
        }
    }, state, event);
}

int main() {
    State state = Idle{};

    state = next_state(state, Start{100});    // -> Running{100}
    state = next_state(state, Pause{});        // -> Paused{100}
    state = next_state(state, Start{50});      // Invalid (can't start from paused)
    state = next_state(state, Resume{});       // -> Running{100}
    state = next_state(state, Stop{"done"});   // -> Stopped{"done"}
}
```

Notice that `Paused{s.speed}` saves the speed in the state type itself. When `Resume` fires, it reads `s.saved_speed` right back out. No external variable needed - the state carries its own context.

### Q2: Build a transition table as a constexpr array of (from_state, event, to_state) triples

Here the entire set of valid transitions is declared as a `constexpr` array up front. The `find_transition` function then searches it at runtime (or compile time with `static_assert`). The nice part is that reading this table tells you everything about which transitions exist - no need to trace through lambda bodies.

```cpp
#include <iostream>
#include <variant>
#include <array>
#include <optional>
#include <functional>
#include <string_view>

// States as enum indices (for the table) and types (for the variant)
enum class StateID { Locked, Unlocked, Open, COUNT };
enum class EventID { Lock, Unlock, Push, Close, COUNT };

// Transition Table (compile-time data)
struct Transition {
    StateID from;
    EventID event;
    StateID to;
    std::string_view action;  // Description of what happens
};

constexpr auto transition_table = std::array{
    Transition{StateID::Locked,   EventID::Unlock, StateID::Unlocked, "Key turns, door unlocks"},
    Transition{StateID::Unlocked, EventID::Lock,   StateID::Locked,   "Door locks"},
    Transition{StateID::Unlocked, EventID::Push,   StateID::Open,     "Door swings open"},
    Transition{StateID::Open,     EventID::Close,  StateID::Unlocked, "Door closes"},
};

// Lookup: O(N) scan, but N is tiny and constexpr-friendly
constexpr std::optional<Transition> find_transition(StateID from, EventID event) {
    for (const auto& t : transition_table) {
        if (t.from == from && t.event == event)
            return t;
    }
    return std::nullopt;
}

// Runtime state machine using the table
constexpr std::string_view state_name(StateID s) {
    constexpr std::string_view names[] = {"Locked", "Unlocked", "Open"};
    return names[static_cast<int>(s)];
}

constexpr std::string_view event_name(EventID e) {
    constexpr std::string_view names[] = {"Lock", "Unlock", "Push", "Close"};
    return names[static_cast<int>(e)];
}

class DoorFSM {
    StateID state_ = StateID::Locked;
public:
    bool process(EventID event) {
        auto t = find_transition(state_, event);
        if (t) {
            std::cout << state_name(state_) << " + " << event_name(event)
                      << " -> " << state_name(t->to) << " (" << t->action << ")\n";
            state_ = t->to;
            return true;
        }
        std::cout << state_name(state_) << " + " << event_name(event)
                  << " -> INVALID\n";
        return false;
    }
};

int main() {
    DoorFSM door;
    door.process(EventID::Push);     // Locked + Push -> INVALID
    door.process(EventID::Unlock);   // Locked + Unlock -> Unlocked
    door.process(EventID::Push);     // Unlocked + Push -> Open
    door.process(EventID::Close);    // Open + Close -> Unlocked
    door.process(EventID::Lock);     // Unlocked + Lock -> Locked

    // Compile-time validation:
    static_assert(find_transition(StateID::Locked, EventID::Unlock).has_value());
    static_assert(!find_transition(StateID::Locked, EventID::Push).has_value());
}
```

The `static_assert` lines at the bottom are a really useful trick - you can verify that specific transitions exist (or don't exist) at compile time, turning runtime bugs into build failures.

### Q3: Show that adding a new state forces a compile error in any incomplete visitor

The `auto&&` catch-all is the escape hatch that defeats exhaustiveness checking. This example shows both the safe version (explicit handlers, compile error if you miss one) and the loose version (catch-all, which silently handles anything new). Prefer the first unless you genuinely want to ignore unknown states.

```cpp
#include <iostream>
#include <variant>

struct Off {};
struct On  {};
struct Standby {};  // NEW STATE - forces all visitors to be updated

using DeviceState = std::variant<Off, On, Standby>;

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// This visitor is COMPLETE - handles all 3 states:
void print_state(const DeviceState& s) {
    std::visit(overloaded{
        [](const Off&)     { std::cout << "Device is OFF\n"; },
        [](const On&)      { std::cout << "Device is ON\n"; },
        [](const Standby&) { std::cout << "Device is in STANDBY\n"; },
        // If Standby line is removed: COMPILE ERROR
        // "no matching function for call to 'overloaded<...>::operator()(const Standby&)'"
    }, s);
}

// This visitor uses auto&& catch-all - no error but loses exhaustiveness:
void print_state_loose(const DeviceState& s) {
    std::visit(overloaded{
        [](const Off&) { std::cout << "OFF\n"; },
        [](const On&)  { std::cout << "ON\n"; },
        [](const auto&) { std::cout << "OTHER\n"; },  // Catches Standby silently
        // WARNING: auto&& suppresses the compile error - use with caution
    }, s);
}

int main() {
    DeviceState s = Standby{};
    print_state(s);       // Device is in STANDBY
    print_state_loose(s); // OTHER
}
```

**Key insight:** avoid `auto&&` catch-all in visitors if you want the compiler to force updates when new states are added.

---

## Notes

- **Transition tables as data** are great for visualization and code generation - you can auto-generate state diagrams from the table, which is useful for documentation and debugging.
- **Guards:** add a `bool(*guard)()` field to the `Transition` struct for conditional transitions that only fire when a precondition holds.
- **Actions:** add a `void(*action)()` field to execute side effects on transitions without cluttering the main state logic.
- **`static_assert`** with `find_transition()` validates transition existence at compile time, turning runtime errors into build errors.
