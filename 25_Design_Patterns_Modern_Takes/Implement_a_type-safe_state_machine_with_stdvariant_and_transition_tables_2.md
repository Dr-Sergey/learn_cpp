# Extend a std::variant state machine with entry actions and extended state

**Category:** Design Patterns Modern Takes  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

This third state machine file focuses on **state data, entry/exit actions**, and ensuring compile-time safety when adding new states. It demonstrates a more complete FSM with guards and side effects on transitions.

The traditional FSM treats states as simple labels with no attached information. The extended version covered here treats states as types that carry context - so `Saving` doesn't just mean "we are saving," it means "we are saving *this filename* and have written *this many bytes* so far." That extra context lives inside the state itself, which is exactly what variant alternatives are good at.

### Extended State Machine Features

```cpp
Traditional FSM:             Extended FSM (this file):
  State = enum value          State = type with data
  Transition = state->state   Transition = state->state + guard + action
  No entry/exit actions       Entry/exit functions per state
  No state data               States carry context (connection info, etc.)
```

---

## Self-Assessment

### Q1: Encode states as types in a std::variant and transitions as a constexpr table of (state, event) -> state

A document editor is a good domain for this because the states have natural data - you need to know *which* file is being saved, and *how many bytes* have been written so far. Notice the guard in the `Editing + Save` lambda: it checks `s.modified` before transitioning. Guards fit naturally as `if` statements inside transition lambdas.

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <optional>

// States with data
struct Editing {
    std::string filename;
    bool modified = false;
};
struct Saving {
    std::string filename;
    int bytes_written = 0;
};
struct Saved {
    std::string filename;
    int total_bytes;
};
struct Error {
    std::string message;
};

using State = std::variant<Editing, Saving, Saved, Error>;

// Events
struct Modify   { std::string change; };
struct Save     {};
struct Progress { int bytes; };
struct Complete { int total; };
struct Fail     { std::string error; };

using Event = std::variant<Modify, Save, Progress, Complete, Fail>;

// Overload helper
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Transition function with state data
State process(const State& state, const Event& event) {
    return std::visit(overloaded{
        // Editing + Modify -> Editing (with modified flag)
        [](const Editing& s, const Modify& e) -> State {
            std::cout << "Modified: " << e.change << "\n";
            return Editing{s.filename, true};
        },
        // Editing + Save -> Saving (guard: must be modified)
        [](const Editing& s, const Save&) -> State {
            if (!s.modified) {
                std::cout << "Nothing to save\n";
                return s;  // Guard: don't transition if unmodified
            }
            std::cout << "Saving " << s.filename << "...\n";
            return Saving{s.filename, 0};
        },
        // Saving + Progress -> Saving (update bytes)
        [](const Saving& s, const Progress& e) -> State {
            std::cout << "Written " << e.bytes << " bytes\n";
            return Saving{s.filename, s.bytes_written + e.bytes};
        },
        // Saving + Complete -> Saved
        [](const Saving& s, const Complete& e) -> State {
            std::cout << "Saved " << s.filename << " (" << e.total << " bytes)\n";
            return Saved{s.filename, e.total};
        },
        // Saving + Fail -> Error
        [](const Saving&, const Fail& e) -> State {
            std::cout << "Save failed: " << e.error << "\n";
            return Error{e.error};
        },
        // Saved + Modify -> back to Editing
        [](const Saved& s, const Modify& e) -> State {
            std::cout << "Re-editing: " << e.change << "\n";
            return Editing{s.filename, true};
        },
        // Default: invalid
        [](const auto& s, const auto&) -> State {
            std::cout << "Invalid transition\n";
            return s;
        }
    }, state, event);
}

int main() {
    State state = Editing{"document.txt", false};

    state = process(state, Save{});                    // Nothing to save (guard)
    state = process(state, Modify{"added paragraph"}); // Modified
    state = process(state, Save{});                    // Saving...
    state = process(state, Progress{512});              // Written 512 bytes
    state = process(state, Complete{1024});             // Saved!
    state = process(state, Modify{"new edit"});         // Re-editing
}
```

The guard pattern (`if (!s.modified) return s;`) keeps the state in place without needing a special "no transition" sentinel. Returning the current state is idiomatic here.

### Q2: Use std::visit to dispatch the current state and apply the transition

Entry and exit actions are a common requirement in real state machines - you want to know when you're leaving a state (to clean up) and when you're entering a new one (to initialize). The trick is comparing `new_state.index()` with `state_.index()` to detect whether the state actually changed before firing those side effects.

```cpp
#include <iostream>
#include <variant>
#include <functional>
#include <vector>
#include <string>
#include <type_traits>
#include <utility>

// States
struct LoggedOut {};
struct LoggingIn { std::string username; };
struct LoggedIn  { std::string username; int session_id; };

using AuthState = std::variant<LoggedOut, LoggingIn, LoggedIn>;

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// State machine with entry/exit actions
class AuthMachine {
    AuthState state_ = LoggedOut{};

    // Entry actions - called when entering a new state
    void on_enter(const LoggedOut&)  { std::cout << "  [enter] Showing login screen\n"; }
    void on_enter(const LoggingIn& s){ std::cout << "  [enter] Verifying " << s.username << "...\n"; }
    void on_enter(const LoggedIn& s) { std::cout << "  [enter] Welcome " << s.username << " (session " << s.session_id << ")\n"; }

    // Exit actions - called when leaving a state
    void on_exit(const LoggedOut&)   { std::cout << "  [exit] Hiding login screen\n"; }
    void on_exit(const LoggingIn&)   { std::cout << "  [exit] Cancelling verification\n"; }
    void on_exit(const LoggedIn& s)  { std::cout << "  [exit] Cleaning up session " << s.session_id << "\n"; }

public:
    template<typename E>
    void process(const E& event) {
        auto new_state = std::visit([&](const auto& current) -> AuthState {
            using S = std::decay_t<decltype(current)>;

            if constexpr (std::is_same_v<S, LoggedOut> && std::is_same_v<E, std::string>) {
                return LoggingIn{event};
            } else if constexpr (std::is_same_v<S, LoggingIn> && std::is_same_v<E, int>) {
                return LoggedIn{current.username, event};
            } else if constexpr (std::is_same_v<S, LoggedIn> && std::is_same_v<E, std::monostate>) {
                return LoggedOut{};
            } else {
                return current;  // No valid transition
            }
        }, state_);

        // Fire exit/enter if state actually changed
        if (new_state.index() != state_.index()) {
            std::visit([this](const auto& s) { on_exit(s); }, state_);
            state_ = std::move(new_state);
            std::visit([this](const auto& s) { on_enter(s); }, state_);
        }
    }
};

int main() {
    AuthMachine auth;
    auth.process(std::string("alice"));   // LoggedOut -> LoggingIn
    auth.process(42);                      // LoggingIn -> LoggedIn
    auth.process(std::monostate{});        // LoggedIn -> LoggedOut
}
```

The `index()` comparison is the guard that prevents spurious entry/exit calls on invalid transitions (where the state stays the same). It's a clean pattern that keeps the side-effect logic separate from the transition logic.

### Q3: Show that adding a new state forces all transition table entries to be updated at compile time

The key takeaway from this example is the contrast between the two visitors. The exhaustive one gets a hard compile error if a state is unhandled. The one with `auto&&` compiles fine but gives you an unhelpful "OTHER" message for any new state - which is exactly the silent-miss bug that variants are supposed to prevent. The right default is the exhaustive visitor.

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <type_traits>

struct Draft     {};
struct Review    {};
struct Approved  {};
struct Published {};
// struct Archived {};  // Uncomment to force all visitors to update

using DocState = std::variant<Draft, Review, Approved, Published /* , Archived */>;

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Visitor WITHOUT auto&& catch-all
// If Archived is added to DocState, this FAILS to compile:
std::string describe(const DocState& s) {
    return std::visit(overloaded{
        [](const Draft&)     -> std::string { return "Document is being drafted"; },
        [](const Review&)    -> std::string { return "Document is under review"; },
        [](const Approved&)  -> std::string { return "Document approved, ready to publish"; },
        [](const Published&) -> std::string { return "Document is published"; },
        // UNCOMMENT when Archived is added:
        // [](const Archived&)  -> std::string { return "Document is archived"; },
        //
        // Without it: error: no matching function for overloaded::operator()(const Archived&)
    }, s);
}

// Best practice: static_assert for completeness
template<typename Variant, typename... Handlers>
constexpr bool all_alternatives_handled = (
    std::is_invocable_v<overloaded<Handlers...>,
        std::variant_alternative_t<0, Variant>> && ...
);
// Use this to validate at compile time that your visitor is complete

int main() {
    DocState s = Review{};
    std::cout << describe(s) << '\n';  // Document is under review
}
```

---

## Notes

- **States with data** (e.g., `Saving{filename, bytes}`) are the key advantage over enum-based FSMs - you don't need parallel arrays or external variables to remember what was happening inside a state.
- **Entry/exit actions** provide a clean place for side effects such as updating a UI or releasing resources, without cluttering the transition logic itself.
- **Guards** (conditional transitions) fit naturally as `if` checks inside transition lambdas - return the current state to indicate "no transition."
- **Avoid `auto&&` catch-all** if you want compile-time exhaustiveness - it silently handles new states and trades away the main safety benefit of using a variant.
