# Apply the TDD workflow in C++ with GoogleTest

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/catchorg/Catch2>  

---

## Topic Overview

This file focuses on **TDD with Google Test** and how TDD reveals awkward APIs before they're written. (See also file #680 for Catch2-based TDD and the red-green-refactor cycle.)

The core insight behind TDD is that writing the test first changes how you design code. When you have to write a test for something that doesn't exist yet, you're forced to think like a user of that code. If calling the API feels awkward in a test, that's a signal the design needs work - and you find that out before you've committed to an implementation.

### Google Test Quick Reference

Here are the macros you'll reach for most often. The key distinction to remember is `EXPECT_` vs `ASSERT_`: an `EXPECT_` failure keeps the test running so you see all the failures at once, while an `ASSERT_` stops immediately - useful when later steps depend on an earlier one passing.

| Macro                              | Purpose                              |
| --- | --- |
| `TEST(Suite, Name)`               | Define a test case                   |
| `TEST_F(Fixture, Name)`           | Test with shared setup/teardown      |
| `EXPECT_EQ(a, b)`                 | Non-fatal assertion (continues)      |
| `ASSERT_EQ(a, b)`                 | Fatal assertion (stops test)         |
| `EXPECT_THROW(expr, ExcType)`     | Expect exception                     |
| `EXPECT_THAT(val, matcher)`       | GMock matcher-based assertion        |

---

## Self-Assessment

### Q1: Write a failing test first, implement the minimal code to pass it, then refactor

**Answer:**

The example below builds a `RingBuffer` incrementally - one test per behavior, in the order you'd naturally discover them. Notice how each test block is clearly labeled with which TDD phase it corresponds to.

```cpp
// TDD with Google Test: building a RingBuffer

// RED: Write test before implementation
#include <gtest/gtest.h>
#include <stdexcept>

// Implementation will go in ring_buffer.hpp
template<typename T, size_t N>
class RingBuffer {
    T data_[N]{};
    size_t head_ = 0, tail_ = 0, size_ = 0;
public:
    void push(const T& val) {
        if (size_ == N) throw std::overflow_error("buffer full");
        data_[tail_] = val;
        tail_ = (tail_ + 1) % N;
        ++size_;
    }
    T pop() {
        if (size_ == 0) throw std::underflow_error("buffer empty");
        T val = data_[head_];
        head_ = (head_ + 1) % N;
        --size_;
        return val;
    }
    size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
    bool full() const { return size_ == N; }
};

// Iteration 1: basic push/pop
TEST(RingBuffer, PushAndPop) {
    RingBuffer<int, 4> rb;
    ASSERT_TRUE(rb.empty());
    ASSERT_EQ(rb.size(), 0);

    rb.push(10);
    ASSERT_EQ(rb.size(), 1);
    ASSERT_FALSE(rb.empty());

    ASSERT_EQ(rb.pop(), 10);
    ASSERT_TRUE(rb.empty());
}

// Iteration 2: FIFO order
TEST(RingBuffer, FIFOOrder) {
    RingBuffer<int, 4> rb;
    rb.push(1);
    rb.push(2);
    rb.push(3);

    EXPECT_EQ(rb.pop(), 1);  // First in = first out
    EXPECT_EQ(rb.pop(), 2);
    EXPECT_EQ(rb.pop(), 3);
}

// Iteration 3: wrap-around
TEST(RingBuffer, WrapAround) {
    RingBuffer<int, 3> rb;
    rb.push(1); rb.push(2); rb.push(3);  // Full
    rb.pop();                              // Remove 1, head advances
    rb.push(4);                            // 4 wraps to slot 0

    EXPECT_EQ(rb.pop(), 2);
    EXPECT_EQ(rb.pop(), 3);
    EXPECT_EQ(rb.pop(), 4);  // Wrapped element
}

// Iteration 4: error cases
TEST(RingBuffer, OverflowThrows) {
    RingBuffer<int, 2> rb;
    rb.push(1);
    rb.push(2);
    EXPECT_THROW(rb.push(3), std::overflow_error);
}

TEST(RingBuffer, UnderflowThrows) {
    RingBuffer<int, 2> rb;
    EXPECT_THROW(rb.pop(), std::underflow_error);
}
```

Each iteration added exactly one new behavior: first basic push/pop, then ordering, then the trickiest part (wrap-around), then error handling. This incremental approach means you always have a green baseline to come back to if something breaks.

### Q2: Show the red-green-refactor cycle using Google Test for a simple stack class

**Answer:**

This example is worth reading carefully because it shows the refactor step in action. After getting all the tests green with straightforward implementations, there's a duplication: both `top()` and `pop()` need to check whether the stack is empty. The refactor extracts that into `ensure_not_empty()`. The test suite then tells you immediately whether the refactor was safe.

```cpp
#include <gtest/gtest.h>
#include <vector>
#include <stdexcept>

// RED -> GREEN -> REFACTOR: Building a Stack

template<typename T>
class Stack {
    std::vector<T> data_;

    void ensure_not_empty() const {
        if (data_.empty())
            throw std::underflow_error("stack is empty");
    }

public:
    void push(T val) { data_.push_back(std::move(val)); }

    T pop() {
        ensure_not_empty();
        T val = std::move(data_.back());
        data_.pop_back();
        return val;
    }

    const T& top() const {
        ensure_not_empty();
        return data_.back();
    }

    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }
};

// Test suite - written BEFORE the implementation

class StackTest : public ::testing::Test {
protected:
    Stack<int> stack;

    void SetUp() override {
        // Fresh empty stack for each test
    }
};

// RED cycle 1: empty stack behavior
TEST_F(StackTest, NewStackIsEmpty) {
    EXPECT_TRUE(stack.empty());
    EXPECT_EQ(stack.size(), 0);
}

// RED cycle 2: push increases size
TEST_F(StackTest, PushIncreasesSize) {
    stack.push(42);
    EXPECT_EQ(stack.size(), 1);
    EXPECT_FALSE(stack.empty());
}

// RED cycle 3: LIFO order
TEST_F(StackTest, PopReturnsLastPushed) {
    stack.push(1);
    stack.push(2);
    stack.push(3);
    EXPECT_EQ(stack.pop(), 3);
    EXPECT_EQ(stack.pop(), 2);
    EXPECT_EQ(stack.pop(), 1);
}

// RED cycle 4: top without removing
TEST_F(StackTest, TopDoesNotRemove) {
    stack.push(99);
    EXPECT_EQ(stack.top(), 99);
    EXPECT_EQ(stack.size(), 1);  // Still there
}

// RED cycle 5: error handling
TEST_F(StackTest, PopOnEmptyThrows) {
    EXPECT_THROW(stack.pop(), std::underflow_error);
}

TEST_F(StackTest, TopOnEmptyThrows) {
    EXPECT_THROW(stack.top(), std::underflow_error);
}

// REFACTOR: extracted ensure_not_empty() to eliminate duplication
// between top() and pop() - tests still pass
```

The refactor comment at the bottom is the whole point: you changed the internal structure but the tests didn't need to change at all. That's what a good refactor looks like - behavior unchanged, code cleaner.

### Q3: Explain how TDD drives interface design: tests reveal awkward APIs before they are written

**Answer:**

This is probably the most underappreciated benefit of TDD. When you write the test first, you're designing the API from the perspective of someone who has to use it. If the test feels awkward to write - lots of setup, weird calling conventions, hard to express what you're checking - that's your API telling you something is off.

The example below shows this in action with a URL parser. Writing the test first led to three design decisions: use a `Url` struct with named fields (instead of positional returns), use `std::optional` for failure (instead of exceptions), and expose a free function (not a class with state). All of those came from asking "what would be natural to write in a test?"

```cpp
// Example: TDD reveals a bad API

// ATTEMPT 1: Start with a test for a URL parser
// RED: Write what you WANT the API to look like:

#include <gtest/gtest.h>
#include <string>
#include <optional>

// This test reveals the ideal API from the USER'S perspective:
// TEST(UrlParser, ParsesSimpleUrl) {
//     auto result = parse_url("https://example.com:8080/path?q=1");
//     ASSERT_TRUE(result.has_value());
//     EXPECT_EQ(result->scheme, "https");
//     EXPECT_EQ(result->host, "example.com");
//     EXPECT_EQ(result->port, 8080);
//     EXPECT_EQ(result->path, "/path");
//     EXPECT_EQ(result->query, "q=1");
// }

// The test naturally drove us to design:
// - A Url struct with named fields (not positional returns)
// - std::optional return (not exceptions for invalid input)
// - Free function parse_url() (not a class with setState())

struct Url {
    std::string scheme;
    std::string host;
    int port = 0;
    std::string path;
    std::string query;
};

std::optional<Url> parse_url(const std::string& raw) {
    // Simplified parser
    auto scheme_end = raw.find("://");
    if (scheme_end == std::string::npos) return std::nullopt;

    Url url;
    url.scheme = raw.substr(0, scheme_end);
    auto rest = raw.substr(scheme_end + 3);

    auto port_start = rest.find(':');
    auto path_start = rest.find('/');
    auto query_start = rest.find('?');

    if (port_start != std::string::npos && port_start < path_start) {
        url.host = rest.substr(0, port_start);
        auto port_end = (path_start != std::string::npos) ? path_start : rest.size();
        url.port = std::stoi(rest.substr(port_start + 1, port_end - port_start - 1));
    } else {
        url.host = rest.substr(0, path_start);
    }

    if (path_start != std::string::npos) {
        auto pend = (query_start != std::string::npos) ? query_start : rest.size();
        url.path = rest.substr(path_start, pend - path_start);
    }
    if (query_start != std::string::npos) {
        url.query = rest.substr(query_start + 1);
    }

    return url;
}

TEST(UrlParser, ParsesSimpleUrl) {
    auto result = parse_url("https://example.com:8080/path?q=1");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->scheme, "https");
    EXPECT_EQ(result->host, "example.com");
    EXPECT_EQ(result->port, 8080);
    EXPECT_EQ(result->path, "/path");
    EXPECT_EQ(result->query, "q=1");
}

TEST(UrlParser, InvalidUrlReturnsNullopt) {
    EXPECT_FALSE(parse_url("not-a-url").has_value());
    // TDD drove us toward optional instead of exceptions -
    // because writing EXPECT_THROW for "bad data" felt wrong
    // when it's just "not parseable", not an error
}

TEST(UrlParser, MinimalUrl) {
    auto result = parse_url("http://localhost");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->scheme, "http");
    EXPECT_EQ(result->host, "localhost");
    EXPECT_EQ(result->port, 0);  // Default when not specified
}

// Key insight: if the TEST is ugly, the API is ugly.
// TDD forces you to USE the API before implementing it.
```

Notice how the invalid-URL test pushed the design toward `std::optional` rather than exceptions. Writing `EXPECT_THROW` for malformed input felt wrong - a bad URL isn't an exceptional event, it's just data that doesn't parse. The test pushed back on a design choice before any implementation existed. That's TDD working exactly as intended.

---

## Notes

- Google Test: `find_package(GTest REQUIRED)` + `target_link_libraries(tests GTest::gtest_main)`
- `TEST_F` with a fixture avoids repeating setup code across test cases
- TDD is most valuable for **new features** - retrofitting tests onto existing code is harder (but still worthwhile)
- Aim for **<1 second** test suite execution for tight feedback loops
- Combine with CI: `ctest --output-on-failure` runs all tests and reports failures
