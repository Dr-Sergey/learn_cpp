# Understand Exception Safety in Move-Only Types

**Category:** Error Handling  
**Standard:** C++11, C++14, C++17  
**Reference:** [cppreference - Move constructors](https://en.cppreference.com/w/cpp/language/move_constructor), [cppreference - std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept)  

---

## Topic Overview

Move-only types (`std::unique_ptr`, unique file handles, non-copyable RAII wrappers) present a fundamental exception safety challenge: **if a move constructor throws mid-operation, the source object is in a valid-but-unspecified state, and you cannot fall back to a copy.** This makes the strong exception guarantee significantly harder to provide.

| Guarantee | Promise | Challenge with move-only types |
| --- | --- | --- |
| **Basic** | No leaks, invariants hold | Achievable - move leaves source in valid state |
| **Strong** | Operation succeeds or state is unchanged | Hard - no copy to fall back on if move throws |
| **No-throw** | Never throws | Ideal - mark move ops `noexcept` |

The reason this trips people up is that the standard library itself relies on `noexcept` move operations. `std::vector::push_back` will call `std::move_if_noexcept`, which returns an lvalue reference (forcing a copy) if the move constructor is not `noexcept`. But for move-only types there is no copy to fall back on, so the program simply must use the potentially-throwing move. If it throws during reallocation, elements already moved from the old buffer are gone - you cannot restore the original vector.

The diagram below shows what happens in each case:

```cpp
// vector reallocation with noexcept move:
// [A][B][C]  -->  [A'][B'][C'][D]   Safe: each move is noexcept

// vector reallocation with throwing move (move-only type):
// [A][B][C]  -->  [A'][B'][???]     B already moved from source
//                                   source A, B are gone - cannot restore original vector
```

The reason the noexcept case is safe is that `std::vector` can detect it and use the copy constructor as a fallback. With move-only types that fallback does not exist, so the strong guarantee is simply unachievable - and that is not a bug, it is a documented limitation of the standard.

The key rules for move-only types:

1. **Always mark move constructors and move assignment `noexcept`** if possible. This is the single most impactful thing for exception safety.
2. If a move constructor *must* throw, document it clearly - callers may need a two-phase approach (allocate first, then move).
3. Consider wrapping throwing-move types in `std::optional` or a state flag so that "moved-from" is a defined, checkable state.

---

## Self-Assessment

### Q1: Demonstrate the `std::vector` reallocation problem with a throwing move-only type

This example deliberately makes the move constructor throw on the second call during a reallocation. Watch what happens to the vector's state afterward - this is the behavior the standard leaves implementation-defined when move-only types throw during reallocation.

```cpp
// vector_move_throw.cpp - C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o vector_move_throw vector_move_throw.cpp
#include <iostream>
#include <vector>
#include <string>
#include <stdexcept>

struct ThrowingWidget {
    std::string name;
    static int move_count;
    static int throw_after;  // throw on the Nth move

    ThrowingWidget(std::string n) : name(std::move(n)) {}

    // NOT noexcept - and not copyable
    ThrowingWidget(ThrowingWidget&& other) /* noexcept(false) implicit */ {
        ++move_count;
        if (throw_after > 0 && move_count >= throw_after) {
            throw std::runtime_error("move #" + std::to_string(move_count) + " failed");
        }
        name = std::move(other.name);
    }

    ThrowingWidget& operator=(ThrowingWidget&&) = default;
    ThrowingWidget(const ThrowingWidget&) = delete;
    ThrowingWidget& operator=(const ThrowingWidget&) = delete;
};

int ThrowingWidget::move_count   = 0;
int ThrowingWidget::throw_after  = 0;

int main() {
    // Phase 1: fill vector normally
    ThrowingWidget::throw_after = 0;  // no throwing yet
    std::vector<ThrowingWidget> vec;
    vec.reserve(2);  // capacity = 2
    vec.emplace_back("Alpha");
    vec.emplace_back("Beta");

    std::cout << "Before push_back: size=" << vec.size()
              << " cap=" << vec.capacity() << "\n";
    for (const auto& w : vec)
        std::cout << "  " << w.name << "\n";

    // Phase 2: trigger reallocation - move constructor will throw
    ThrowingWidget::move_count  = 0;
    ThrowingWidget::throw_after = 2;  // fail on 2nd move (during realloc)

    try {
        vec.push_back(ThrowingWidget("Gamma"));  // triggers reallocation
        std::cout << "push_back succeeded (unexpected)\n";
    } catch (const std::exception& ex) {
        std::cout << "\n*** Exception during reallocation: " << ex.what() << "\n";
        std::cout << "Vector state after exception:\n";
        std::cout << "  size=" << vec.size() << "\n";
        for (size_t i = 0; i < vec.size(); ++i) {
            std::cout << "  [" << i << "] name=\"" << vec[i].name << "\""
                      << (vec[i].name.empty() ? " (MOVED-FROM!)" : "") << "\n";
        }
        // Note: the vector may be in a partially-moved state.
        // The standard says behavior is implementation-defined for
        // move-only types with throwing moves during reallocation.
    }
}
// Typical output (implementation-dependent):
// Before push_back: size=2 cap=2
//   Alpha
//   Beta
//
// *** Exception during reallocation: move #2 failed
// Vector state after exception:
//   size=2
//   [0] name="" (MOVED-FROM!)
//   [1] name="Beta"
```

### Q2: Show the safe pattern - `noexcept` move + RAII for strong guarantee

The fix is straightforward: make the move `noexcept`. The `static_assert` lines at the bottom of the class definition are your safety net - they will give you a compile-time error if you accidentally introduce a noexcept-removing change later.

```cpp
// safe_move_only.cpp - C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o safe_move_only safe_move_only.cpp
#include <iostream>
#include <memory>
#include <vector>
#include <type_traits>

// A properly designed move-only type: noexcept move operations
class UniqueBuffer {
    std::unique_ptr<char[]> data_;
    size_t size_ = 0;

public:
    UniqueBuffer() = default;

    explicit UniqueBuffer(size_t n)
        : data_(std::make_unique<char[]>(n)), size_(n) {}

    // noexcept move - critical for exception safety in containers
    UniqueBuffer(UniqueBuffer&& other) noexcept
        : data_(std::move(other.data_)), size_(other.size_) {
        other.size_ = 0;
    }

    UniqueBuffer& operator=(UniqueBuffer&& other) noexcept {
        data_ = std::move(other.data_);
        size_ = other.size_;
        other.size_ = 0;
        return *this;
    }

    UniqueBuffer(const UniqueBuffer&) = delete;
    UniqueBuffer& operator=(const UniqueBuffer&) = delete;

    size_t size() const noexcept { return size_; }
    explicit operator bool() const noexcept { return data_ != nullptr; }
};

// Verify at compile time
static_assert(std::is_nothrow_move_constructible_v<UniqueBuffer>);
static_assert(std::is_nothrow_move_assignable_v<UniqueBuffer>);
static_assert(!std::is_copy_constructible_v<UniqueBuffer>);

int main() {
    std::vector<UniqueBuffer> buffers;

    // These push_backs trigger reallocations.
    // Because move is noexcept, vector uses move - safe and fast.
    for (int i = 1; i <= 10; ++i) {
        buffers.emplace_back(i * 1024);
    }

    std::cout << "Buffers: " << buffers.size() << "\n";
    for (size_t i = 0; i < buffers.size(); ++i) {
        std::cout << "  [" << i << "] size=" << buffers[i].size() << "\n";
    }

    // move_if_noexcept returns an rvalue reference (true move) because
    // UniqueBuffer's move constructor is noexcept
    UniqueBuffer& ref = buffers[0];
    auto&& val = std::move_if_noexcept(ref);  // rvalue ref - will move
    static_assert(std::is_rvalue_reference_v<decltype(val)>,
                  "Should be rvalue reference for noexcept movable type");
    std::cout << "move_if_noexcept yields rvalue ref: true\n";
}
// Output:
// Buffers: 10
//   [0] size=1024
//   [1] size=2048
//   ...
//   [9] size=10240
// move_if_noexcept yields rvalue ref: true
```

### Q3: Implement a strong-guarantee `assign_or_rollback` for a struct with move-only members

The strong guarantee pattern for move-only members is "construct-then-swap": build the new state completely (let it throw if it wants to), then commit it using a `noexcept` move. If construction throws, you never touched the original. If it succeeds, the `noexcept` swap commits the change safely.

```cpp
// assign_rollback.cpp - C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o assign_rollback assign_rollback.cpp
#include <iostream>
#include <memory>
#include <optional>
#include <stdexcept>
#include <string>
#include <utility>

// A resource that might throw during construction (but not during move)
class Connection {
    std::string endpoint_;
    int fd_;  // simulated file descriptor

public:
    Connection() : endpoint_("none"), fd_(-1) {}

    // Construction might throw (e.g., DNS failure)
    explicit Connection(std::string endpoint)
        : endpoint_(std::move(endpoint)), fd_(42) {
        if (endpoint_.find("bad") != std::string::npos) {
            throw std::runtime_error("connection failed: " + endpoint_);
        }
    }

    // Move is noexcept - critical
    Connection(Connection&& o) noexcept
        : endpoint_(std::move(o.endpoint_)), fd_(o.fd_) {
        o.fd_ = -1;
    }

    Connection& operator=(Connection&& o) noexcept {
        endpoint_ = std::move(o.endpoint_);
        fd_ = o.fd_;
        o.fd_ = -1;
        return *this;
    }

    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;

    const std::string& endpoint() const { return endpoint_; }
    int fd() const { return fd_; }
};

// Session owns a move-only Connection.
// We want strong guarantee on reconnect(): either fully updated or unchanged.
class Session {
    Connection conn_;
    std::string user_;

public:
    Session(Connection c, std::string user)
        : conn_(std::move(c)), user_(std::move(user)) {}

    // Strong guarantee: if new connection fails, session is unchanged
    void reconnect(const std::string& new_endpoint) {
        // Step 1: construct the new connection - may throw
        Connection new_conn(new_endpoint);

        // Step 2: swap into place - noexcept because Connection's move is noexcept
        // If step 1 threw, we never reach here - strong guarantee
        conn_ = std::move(new_conn);
    }

    void print() const {
        std::cout << "Session[user=" << user_
                  << ", endpoint=" << conn_.endpoint()
                  << ", fd=" << conn_.fd() << "]\n";
    }
};

int main() {
    Session session(Connection("server-a.com"), "alice");
    session.print();

    // Successful reconnect
    session.reconnect("server-b.com");
    session.print();

    // Failed reconnect - session must remain on server-b.com
    try {
        session.reconnect("bad-server.com");  // throws in Connection ctor
    } catch (const std::exception& ex) {
        std::cout << "Reconnect failed: " << ex.what() << "\n";
    }
    session.print();  // must still show server-b.com

    std::cout << "\nStrong guarantee maintained.\n";
}
// Output:
// Session[user=alice, endpoint=server-a.com, fd=42]
// Session[user=alice, endpoint=server-b.com, fd=42]
// Reconnect failed: connection failed: bad-server.com
// Session[user=alice, endpoint=server-b.com, fd=42]
//
// Strong guarantee maintained.
```

The key insight in `reconnect()` is that the risky work (constructing `new_conn`) happens on a local variable before anything in the `Session` is touched. Either the construction succeeds and we commit with a `noexcept` move, or it throws and the `Session` is completely unchanged.

---

## Notes

- **Rule of thumb:** always declare move constructors and move assignment operators `noexcept` for move-only types. The standard library depends on it.
- `std::move_if_noexcept` returns an lvalue reference (preventing move) when `T` is copyable and the move is potentially throwing. For move-only types, it returns an rvalue reference regardless - there is no copy to fall back on.
- `std::vector` reallocation of move-only types with throwing moves has **implementation-defined behavior** - elements may be permanently lost. This is not a standard violation; the strong guarantee is simply impossible.
- The strong guarantee pattern for move-only members is "construct-then-swap": build the new state entirely, then swap/move it into place using `noexcept` operations.
- `std::unique_ptr` itself has a `noexcept` move constructor. Problems arise when wrapping types that allocate or validate in their own move constructors.
- If your type truly cannot make its move constructor `noexcept`, consider using `std::optional<T>` as a member so the "empty" state is explicit and well-defined.
- Use `static_assert(std::is_nothrow_move_constructible_v<T>)` in your types to catch accidental `noexcept` loss early.
