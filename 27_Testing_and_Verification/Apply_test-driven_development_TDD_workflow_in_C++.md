# Apply test-driven development (TDD) workflow in C++

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/catchorg/Catch2>  

---

## Topic Overview

**Test-Driven Development (TDD)** is a workflow where you write a **failing test first**, then write the **minimal code** to make it pass, then **refactor**. This "Red-Green-Refactor" cycle produces code that is testable by design, with complete test coverage from the start.

The reason TDD works is subtle but important: it changes the order in which you think. Instead of implementing something and then figuring out how to test it, you start by asking "how do I want to use this?" That question shapes the design. Code written test-first tends to have smaller functions, cleaner interfaces, and fewer hidden dependencies - not because of discipline, but because those properties naturally make code easier to test.

### The Red-Green-Refactor Cycle

The diagram below captures the rhythm of TDD. Each lap around the loop adds one new behavior. The key discipline is staying in the green phase as much as possible: never write more production code than what's needed to make the current failing test pass.

```cpp
    +---- RED -----+
    | Write a test  |
    | that FAILS    |
    +------+--------+
           |
           v
    +---- GREEN ---+
    | Write MINIMAL|
    | code to pass |
    +------+-------+
           |
           v
    +-- REFACTOR --+
    | Clean up code|
    | tests still  |
    | pass         |
    +------+-------+
           |
           +---> repeat
```

### Catch2 Quick Reference

Catch2 uses a slightly different style than Google Test - tests are free functions with string names rather than macro-generated classes. The macros below are all you need for most situations. `REQUIRE` stops the test on failure; `CHECK` lets it continue so you see all failures at once.

| Macro                         | Purpose                                 |
| --- | --- |
| `TEST_CASE("name")`          | Define a test case                      |
| `SECTION("name")`            | Subsection within a test case           |
| `REQUIRE(expr)`              | Assert (test fails if false)            |
| `CHECK(expr)`                | Assert (reports failure, continues)     |
| `REQUIRE_THROWS_AS(expr, T)` | Assert exception type thrown            |

---

## Self-Assessment

### Q1: Write a failing test with Catch2 REQUIRE before implementing the function it tests

**Answer:**

Here's the simplest possible TDD example: a single function with a forward declaration so the file compiles, but no definition yet. The compile error itself is the "RED" state - the test cannot even run because the function doesn't exist.

```cpp
// Step 1: RED - Write the test FIRST
// File: test_calculator.cpp
#include <catch2/catch_test_macros.hpp>

// Forward declaration - function doesn't exist yet!
int add(int a, int b);

TEST_CASE("add() computes the sum of two integers") {
    REQUIRE(add(2, 3) == 5);
    REQUIRE(add(-1, 1) == 0);
    REQUIRE(add(0, 0) == 0);
    REQUIRE(add(-5, -3) == -8);
    REQUIRE(add(INT_MAX, 0) == INT_MAX);
}
// COMPILE ERROR: add() is not defined - this is the "RED" state

// Step 2: GREEN - Minimal implementation
// File: calculator.hpp
int add(int a, int b) {
    return a + b;  // Simplest code that makes all tests pass
}
// TESTS PASS - this is the "GREEN" state

// Step 3: No refactor needed (already clean)
// In more complex cases, you'd clean up duplicated code here while
// ensuring tests still pass.
```

The "minimal implementation" rule is not about writing bad code on purpose - it's about not getting ahead of yourself. You implement exactly what the current test demands, nothing more. The next test will demand more, and you'll grow the implementation naturally.

### Q2: Follow the red-green-refactor cycle: failing test, minimal implementation, clean refactor

**Answer:**

This example builds a `Stack<T>` incrementally across three iterations. Each iteration follows the same pattern: write a failing test, add the minimum code to make it pass, then look for cleanup opportunities. The refactor at the end is the payoff: duplicated guard logic gets extracted into a helper, and the tests tell you instantly whether the cleanup was safe.

```cpp
// Full TDD cycle for a Stack<T> class

// Iteration 1: push and size

// RED: Write failing test
#include <catch2/catch_test_macros.hpp>
#include <stdexcept>

// Forward declare
template<typename T> class Stack;

TEST_CASE("Stack push and size") {
    Stack<int> s;
    REQUIRE(s.size() == 0);
    REQUIRE(s.empty());

    s.push(42);
    REQUIRE(s.size() == 1);
    REQUIRE(!s.empty());
}
// -> FAILS: Stack doesn't exist

// GREEN: Minimal implementation
#include <vector>

template<typename T>
class Stack {
    std::vector<T> data_;
public:
    void push(const T& val) { data_.push_back(val); }
    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }
};
// -> PASSES

// Iteration 2: pop and top

// RED: Add new tests (existing tests still run)
TEST_CASE("Stack pop and top") {
    Stack<int> s;
    s.push(10);
    s.push(20);

    REQUIRE(s.top() == 20);

    s.pop();
    REQUIRE(s.top() == 10);
    REQUIRE(s.size() == 1);
}

TEST_CASE("Stack pop on empty throws") {
    Stack<int> s;
    REQUIRE_THROWS_AS(s.pop(), std::underflow_error);
    REQUIRE_THROWS_AS(s.top(), std::underflow_error);
}
// -> FAILS: top() and pop() don't exist

// GREEN: Add methods
// (add to Stack class):
//   const T& top() const {
//       if (empty()) throw std::underflow_error("top on empty stack");
//       return data_.back();
//   }
//   void pop() {
//       if (empty()) throw std::underflow_error("pop on empty stack");
//       data_.pop_back();
//   }
// -> PASSES

// Iteration 3: REFACTOR
// Notice: the emptiness check is duplicated in top() and pop().
// Extract a helper:
//   void check_not_empty() const {
//       if (empty()) throw std::underflow_error("stack is empty");
//   }
// Then top() and pop() call check_not_empty().
// Run tests -> still PASS -> refactor is safe.
```

Notice that the existing tests kept running during iteration 2. That's intentional - you never want to break the green state for existing behavior while adding new behavior. The tests become a safety net that grows with the code.

### Q3: Show how TDD drives better API design: the test forces a clean, testable interface

**Answer:**

This is the most important lesson from TDD. When you write tests first, untestable designs reveal themselves immediately because the tests become painful to write. The contrast below is striking: `BadUserService` has hard-wired dependencies that make it nearly impossible to test without spinning up a real database and email server. `UserService`, designed test-first, has clean interfaces and injected dependencies - and every behavior can be verified in milliseconds with fakes.

```cpp
// Without TDD: untestable design
// Developer writes implementation first, tests as afterthought

class BadUserService {
public:
    bool register_user(const std::string& name) {
        // Directly connects to database - untestable without real DB!
        auto conn = DatabasePool::get_connection();  // Hard dependency
        conn.execute("INSERT INTO users ...");
        EmailService::send_welcome(name);            // Hard dependency
        Logger::log("User registered");              // Hard dependency
        return true;
    }
};
// Testing this requires: running DB, email server, log infrastructure
// -> Slow, flaky, rarely tested

// With TDD: testable design emerges naturally
// Writing the TEST FIRST forces you to design injectable interfaces

// Step 1: Write the test - you're "designing the API from the consumer side"
#include <catch2/catch_test_macros.hpp>
#include <string>
#include <vector>
#include <functional>

struct UserRepository {
    virtual ~UserRepository() = default;
    virtual bool save(const std::string& name) = 0;
};

struct Notifier {
    virtual ~Notifier() = default;
    virtual void notify(const std::string& msg) = 0;
};

class UserService {
    UserRepository& repo_;
    Notifier& notifier_;
public:
    UserService(UserRepository& r, Notifier& n) : repo_(r), notifier_(n) {}

    bool register_user(const std::string& name) {
        if (name.empty()) return false;
        if (!repo_.save(name)) return false;
        notifier_.notify("Welcome " + name);
        return true;
    }
};

// Step 2: Test with fakes - no real DB or email needed
struct FakeRepo : UserRepository {
    std::vector<std::string> saved;
    bool should_fail = false;
    bool save(const std::string& name) override {
        if (should_fail) return false;
        saved.push_back(name);
        return true;
    }
};

struct FakeNotifier : Notifier {
    std::vector<std::string> messages;
    void notify(const std::string& msg) override {
        messages.push_back(msg);
    }
};

TEST_CASE("UserService registers user and notifies") {
    FakeRepo repo;
    FakeNotifier notifier;
    UserService svc(repo, notifier);

    REQUIRE(svc.register_user("Alice"));
    REQUIRE(repo.saved.size() == 1);
    REQUIRE(repo.saved[0] == "Alice");
    REQUIRE(notifier.messages[0] == "Welcome Alice");
}

TEST_CASE("UserService rejects empty name") {
    FakeRepo repo;
    FakeNotifier notifier;
    UserService svc(repo, notifier);

    REQUIRE_FALSE(svc.register_user(""));
    REQUIRE(repo.saved.empty());
}

TEST_CASE("UserService handles DB failure") {
    FakeRepo repo;
    repo.should_fail = true;
    FakeNotifier notifier;
    UserService svc(repo, notifier);

    REQUIRE_FALSE(svc.register_user("Bob"));
    REQUIRE(notifier.messages.empty());  // No notification if save failed
}

// TDD forced us to:
// 1. Define clean interfaces (UserRepository, Notifier)
// 2. Use dependency injection (constructor params)
// 3. Separate concerns (save vs notify vs validation)
// 4. Handle edge cases (empty name, DB failure)
// -> The TEST drove the DESIGN
```

The "DB failure" test case is worth pointing out specifically. When you write the test first and ask "what should happen if saving fails?" you naturally discover that the notifier should not be called. That's a behavioral requirement that might never have been written down otherwise - TDD surfaced it because the test demanded an answer.

---

## Notes

- Build system: `find_package(Catch2 3 REQUIRED)` + `target_link_libraries(tests Catch2::Catch2WithMain)`
- Run single tests: `./tests "Stack push and size"` (Catch2 supports name-based filtering)
- TDD works best for **pure logic** (calculations, data transformations, state machines)
- For I/O-bound code, TDD naturally pushes toward dependency injection - this is a feature, not overhead
- Combine with sanitizers: `cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined"` catches memory bugs TDD alone misses
