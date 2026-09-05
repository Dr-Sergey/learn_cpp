# Build a Compile-Time Finite State Machine Using Template Specialization

**Category:** Compile Time Programming  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/partial_specialization>  

---

## Topic Overview

### What Is a Compile-Time FSM

A **finite state machine** (FSM) modeled at compile time encodes everything the runtime version normally would - states, events, and transitions - but entirely in the type system. Concretely:

- **States** as types (empty structs)
- **Events** as types (empty structs)
- **Transitions** as template specializations mapping `(State, Event) -> NextState`

The payoff is that the compiler verifies all transitions exist at compile time. If you try to take a transition that was never defined, you get a compile error rather than a runtime bug. This is the "make illegal states unrepresentable" idea taken to its logical extreme.

```cpp
         ┌─────────┐   CoinInserted   ┌──────────┐
         │  Locked  │ ───────────────► │ Unlocked │
         │  (state) │                  │  (state) │
         └────┬─────┘ ◄─────────────── └──────────┘
              │         PersonPassed
              │
              │ PersonPassed (stays Locked)
              └──────────────┐
                             ▼
                         (self-loop)
```

### Pattern

The key insight is leaving the primary template undefined. An undefined primary template means "no specialization exists for this combination" - and trying to use it becomes a hard compile error. You then explicitly define only the transitions you actually want.

```cpp
// States as types
struct Locked {};
struct Unlocked {};

// Events as types
struct CoinInserted {};
struct PersonPassed {};

// Primary template: undefined transition → compile error
template <typename State, typename Event>
struct Transition;  // no definition!

// Define valid transitions via specialization
template <> struct Transition<Locked, CoinInserted>    { using next = Unlocked; };
template <> struct Transition<Unlocked, PersonPassed>  { using next = Locked; };
template <> struct Transition<Locked, PersonPassed>    { using next = Locked; };
```

Notice that `Transition<Locked, PersonPassed>` maps back to `Locked`. That's how you encode a self-loop: you simply make the `next` type the same as the current state.

---

## Self-Assessment

### Q1: Encode states as types and transitions as template specializations: `Transition<State, Event>::next`

Here's a more complete FSM for a media player (Idle/Running/Paused/Stopped). The interesting part to study is how `FSM<State>::step<Event>` chains transitions - each call just resolves `NextState<State, Event>` at compile time, so the entire sequence of states is computed by the compiler before any code runs.

```cpp
#include <iostream>
#include <type_traits>

// === States ===
struct Idle       { static constexpr const char* name = "Idle"; };
struct Running    { static constexpr const char* name = "Running"; };
struct Paused     { static constexpr const char* name = "Paused"; };
struct Stopped    { static constexpr const char* name = "Stopped"; };

// === Events ===
struct Start      { static constexpr const char* name = "Start"; };
struct Pause      { static constexpr const char* name = "Pause"; };
struct Resume     { static constexpr const char* name = "Resume"; };
struct Stop       { static constexpr const char* name = "Stop"; };
struct Reset      { static constexpr const char* name = "Reset"; };

// === Primary template: undefined (will cause compile error for invalid transitions) ===
template <typename State, typename Event>
struct Transition;  // deliberately undefined

// === Valid transitions (defined via template specialization) ===
template <> struct Transition<Idle, Start>       { using next = Running; };
template <> struct Transition<Running, Pause>    { using next = Paused; };
template <> struct Transition<Running, Stop>     { using next = Stopped; };
template <> struct Transition<Paused, Resume>    { using next = Running; };
template <> struct Transition<Paused, Stop>      { using next = Stopped; };
template <> struct Transition<Stopped, Reset>    { using next = Idle; };

// === Helper alias ===
template <typename State, typename Event>
using NextState = typename Transition<State, Event>::next;

// === Compile-time FSM executor ===
template <typename State>
struct FSM {
    template <typename Event>
    using step = FSM<NextState<State, Event>>;

    static void print() { std::cout << "State: " << State::name << "\n"; }
};

int main() {
    std::cout << "=== Compile-Time FSM ===\n\n";

    // Walk through valid transitions at compile time
    using S0 = FSM<Idle>;
    S0::print();

    using S1 = S0::step<Start>;
    S1::print();  // Running

    using S2 = S1::step<Pause>;
    S2::print();  // Paused

    using S3 = S2::step<Resume>;
    S3::print();  // Running

    using S4 = S3::step<Stop>;
    S4::print();  // Stopped

    using S5 = S4::step<Reset>;
    S5::print();  // Idle

    // Verify with static_assert
    static_assert(std::is_same_v<NextState<Idle, Start>, Running>);
    static_assert(std::is_same_v<NextState<Running, Pause>, Paused>);
    static_assert(std::is_same_v<NextState<Paused, Stop>, Stopped>);
    static_assert(std::is_same_v<NextState<Stopped, Reset>, Idle>);
    std::cout << "\nAll transitions verified at compile time\n";

    return 0;
}
```

**Expected output:**

```text
=== Compile-Time FSM ===

State: Idle
State: Running
State: Paused
State: Running
State: Stopped
State: Idle

All transitions verified at compile time
```

### Q2: Implement a `static_assert` that fires if an undefined transition is taken

The undefined-primary-template approach gives you a compile error, but the message can be cryptic. A cleaner alternative is to give the primary template a definition that returns a sentinel `InvalidTransition` type, then use `static_assert` to emit a human-readable message whenever that sentinel shows up.

```cpp
#include <iostream>
#include <type_traits>

// States and Events
struct On  { static constexpr const char* name = "On"; };
struct Off { static constexpr const char* name = "Off"; };

struct Toggle {};
struct Shutdown {};

// === Transition with has_transition detection ===
template <typename State, typename Event, typename = void>
struct has_transition : std::false_type {};

template <typename State, typename Event>
struct has_transition<State, Event,
    std::void_t<typename Transition<State, Event>::next>>
    : std::true_type {};

// But we can't use the undefined primary template easily.
// Better approach: use a sentinel type for invalid transitions.

struct InvalidTransition {};

// Primary template: returns InvalidTransition (not undefined)
template <typename State, typename Event>
struct SafeTransition {
    using next = InvalidTransition;
};

// Valid transitions
template <> struct SafeTransition<Off, Toggle>    { using next = On; };
template <> struct SafeTransition<On, Toggle>     { using next = Off; };
template <> struct SafeTransition<On, Shutdown>   { using next = Off; };

template <typename State, typename Event>
using SafeNext = typename SafeTransition<State, Event>::next;

// === static_assert fires for invalid transitions ===
template <typename State, typename Event>
constexpr void validate_transition() {
    static_assert(!std::is_same_v<SafeNext<State, Event>, InvalidTransition>,
                  "ERROR: Undefined FSM transition! "
                  "Check your State + Event combination.");
}

int main() {
    // Valid transitions - static_assert passes
    validate_transition<Off, Toggle>();    // Off + Toggle -> On
    validate_transition<On, Toggle>();     // On + Toggle -> Off
    validate_transition<On, Shutdown>();   // On + Shutdown -> Off

    // Invalid transition - uncomment to see static_assert fire:
    // validate_transition<Off, Shutdown>();
    // ERROR: "Undefined FSM transition! Check your State + Event combination."

    static_assert(std::is_same_v<SafeNext<Off, Toggle>, On>);
    static_assert(std::is_same_v<SafeNext<On, Toggle>, Off>);
    static_assert(std::is_same_v<SafeNext<Off, Shutdown>, InvalidTransition>);

    std::cout << "All valid transitions pass static_assert\n";
    std::cout << "Invalid transitions would cause compile error\n";

    return 0;
}
```

### Q3: Compare this TMP approach with a `constexpr` switch-based FSM

Both approaches get you zero runtime overhead and compile-time validation - they just get there differently. The table below captures the trade-offs. The short version: TMP is richer and more type-level, `constexpr` switch is far easier to read and debug.

| Aspect | TMP (Template Specialization) | `constexpr` switch-based |
| --- | --- | --- |
| **State representation** | Types (`struct Idle {}`) | Enum values (`enum State { Idle, Running }`) |
| **Transitions** | Template specializations | `constexpr` function with `switch` |
| **Invalid transition** | Compile error (missing specialization) | Needs explicit handling (return error) |
| **Readability** | Complex - many specializations | Simple - familiar switch syntax |
| **Adding states** | Add structs + specializations | Add enum value + switch case |
| **Runtime overhead** | Zero (all resolved at compile time) | Zero (constexpr / optimized away) |
| **Error messages** | Cryptic template errors | Clear "missing case" warnings |
| **Best for** | Pure compile-time FSMs, type-level protocols | Runtime FSMs with compile-time validation |

Here's the `constexpr` switch version for comparison. Notice how much flatter and more readable the transition logic looks:

```cpp
#include <iostream>
#include <stdexcept>

// === constexpr switch-based FSM ===
enum class State { Idle, Running, Paused, Stopped };
enum class Event { Start, Pause, Resume, Stop, Reset };

constexpr const char* to_string(State s) {
    switch (s) {
        case State::Idle:    return "Idle";
        case State::Running: return "Running";
        case State::Paused:  return "Paused";
        case State::Stopped: return "Stopped";
    }
    return "Unknown";
}

constexpr State transition(State state, Event event) {
    switch (state) {
        case State::Idle:
            if (event == Event::Start) return State::Running;
            break;
        case State::Running:
            if (event == Event::Pause) return State::Paused;
            if (event == Event::Stop) return State::Stopped;
            break;
        case State::Paused:
            if (event == Event::Resume) return State::Running;
            if (event == Event::Stop) return State::Stopped;
            break;
        case State::Stopped:
            if (event == Event::Reset) return State::Idle;
            break;
    }
    throw std::logic_error("Invalid transition");  // compile-time error if used in constexpr
}

// Compile-time verification
static_assert(transition(State::Idle, Event::Start) == State::Running);
static_assert(transition(State::Running, Event::Pause) == State::Paused);
static_assert(transition(State::Paused, Event::Resume) == State::Running);
static_assert(transition(State::Stopped, Event::Reset) == State::Idle);

// Invalid: would fail at compile time:
// static_assert(transition(State::Idle, Event::Stop) == State::Stopped);  // ERROR!

int main() {
    std::cout << "=== constexpr switch-based FSM ===\n\n";

    State current = State::Idle;
    std::cout << to_string(current) << "\n";

    current = transition(current, Event::Start);
    std::cout << to_string(current) << "\n";  // Running

    current = transition(current, Event::Pause);
    std::cout << to_string(current) << "\n";  // Paused

    current = transition(current, Event::Stop);
    std::cout << to_string(current) << "\n";  // Stopped

    current = transition(current, Event::Reset);
    std::cout << to_string(current) << "\n";  // Idle

    std::cout << "\nAdvantage over TMP: same FSM works at runtime AND compile time.\n";
    std::cout << "TMP advantage: states carry associated types/data via specialization.\n";

    return 0;
}
```

---

## Notes

- TMP FSMs are powerful for **protocol enforcement at compile time** (e.g., ensuring a connection is opened before sending data).
- `constexpr` switch FSMs are simpler and work at both compile time and runtime.
- The TMP approach shines when transitions carry **type-level data** (e.g., a different type per state).
- C++20 concepts can constrain FSM transitions for better error messages.
- Production FSMs: consider Boost.SML or Boost.MSM for full-featured state machine libraries.
- Combine both approaches: use enums for runtime state and template specialization for compile-time validation.
