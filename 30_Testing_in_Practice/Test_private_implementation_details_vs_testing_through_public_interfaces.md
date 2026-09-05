# Test private implementation details vs testing through public interfaces

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

One of the most debated testing questions you'll encounter: should you test private methods directly, or only through the public interface? The answer is almost always **test through the public interface**, but understanding when and why exceptions exist will make you a better test designer - and a better class designer.

The reason this matters is that tests are a form of contract. When you test through the public interface, you're saying "I promise this behavior is stable." When you test a private method directly, you're coupling your tests to an internal implementation decision. If you later refactor - say, merging two private methods into one - those tests break even though the behavior is identical. That's the worst kind of false positive.

### Decision Matrix

| Situation | Recommendation | Why |
| --- | --- | --- |
| Simple private helper | **Test via public** | It's an implementation detail - may change |
| Complex algorithm in private method | **Test via public**, but thoroughly | Cover all code paths through public API |
| Private method is really a separate concern | **Extract to its own class**, test directly | It's hiding a missing abstraction |
| Template metaprogramming internals | **Test the public result** | Testing TMP internals is fragile |
| Legacy code with no public entry point | **Add friend test fixture as last resort** | Pragmatic, mark as tech debt |

### Why Testing Internals Is Usually Wrong

Here's the mental model. Your class has a public interface on the outside and private helpers on the inside. Tests belong on the outside - they verify what the class promises to the world, not how it delivers on that promise.

```cpp
Public Interface (stable contract):
┌────────────────────────────────────┐
│  process(input) -> output           │  <- Test THIS
│    ├── validate(input)  [private]  │  <- Don't test this directly
│    ├── transform(data)  [private]  │  <- Don't test this directly
│    └── format(result)   [private]  │  <- Don't test this directly
└────────────────────────────────────┘

If you test validate() directly:

- Refactoring (e.g., merging validate + transform) breaks tests
- Tests don't prove the PUBLIC behavior works
- Tests are coupled to implementation, not behavior
```

---

## Self-Assessment

### Q1: Show how to test complex behavior through the public interface

The `MarkdownParser` below has four private methods - `split_lines`, `extract_title`, `extract_sections`, `extract_code_blocks` - any of which you might be tempted to test directly. Don't. Every one of them is fully exercised by driving `parse()` with appropriate inputs. The tests at the bottom cover all four private implementations without ever naming them.

```cpp
#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <algorithm>

// === Production code with complex private methods ===
class MarkdownParser {
public:
    struct Document {
        std::string title;
        std::vector<std::string> sections;
        std::vector<std::string> code_blocks;
    };

    // PUBLIC API: this is what we test
    Document parse(const std::string& markdown) {
        auto lines = split_lines(markdown);      // private
        auto title = extract_title(lines);        // private
        auto sections = extract_sections(lines);  // private
        auto code = extract_code_blocks(lines);   // private
        return {title, sections, code};
    }

private:
    // These are implementation details - we DON'T test them directly
    std::vector<std::string> split_lines(const std::string& text) {
        std::vector<std::string> lines;
        std::string line;
        for (char c : text) {
            if (c == '\n') {
                lines.push_back(line);
                line.clear();
            } else {
                line += c;
            }
        }
        if (!line.empty()) lines.push_back(line);
        return lines;
    }

    std::string extract_title(const std::vector<std::string>& lines) {
        for (const auto& line : lines)
            if (line.starts_with("# "))
                return line.substr(2);
        return "";
    }

    std::vector<std::string> extract_sections(const std::vector<std::string>& lines) {
        std::vector<std::string> sections;
        for (const auto& line : lines)
            if (line.starts_with("## "))
                sections.push_back(line.substr(3));
        return sections;
    }

    std::vector<std::string> extract_code_blocks(const std::vector<std::string>& lines) {
        std::vector<std::string> blocks;
        bool in_block = false;
        std::string current;
        for (const auto& line : lines) {
            if (line.starts_with("```")) {
                if (in_block) {
                    blocks.push_back(current);
                    current.clear();
                }
                in_block = !in_block;
            } else if (in_block) {
                current += line + "\n";
            }
        }
        return blocks;
    }
};

// === Tests: ALL through the public parse() method ===

TEST(MarkdownParserTest, ExtractsTitle) {
    MarkdownParser parser;
    auto doc = parser.parse("# My Title\nSome content");
    EXPECT_EQ(doc.title, "My Title");
}

TEST(MarkdownParserTest, ExtractsSections) {
    MarkdownParser parser;
    auto doc = parser.parse("# Title\n## Section 1\ntext\n## Section 2");
    ASSERT_EQ(doc.sections.size(), 2);
    EXPECT_EQ(doc.sections[0], "Section 1");
    EXPECT_EQ(doc.sections[1], "Section 2");
}

TEST(MarkdownParserTest, ExtractsCodeBlocks) {
    MarkdownParser parser;
    auto doc = parser.parse("# Title\n```\nint x = 42;\n```");
    ASSERT_EQ(doc.code_blocks.size(), 1);
    EXPECT_EQ(doc.code_blocks[0], "int x = 42;\n");
}

TEST(MarkdownParserTest, EmptyInput) {
    MarkdownParser parser;
    auto doc = parser.parse("");
    EXPECT_TRUE(doc.title.empty());
    EXPECT_TRUE(doc.sections.empty());
    EXPECT_TRUE(doc.code_blocks.empty());
}

TEST(MarkdownParserTest, NoTitle) {
    MarkdownParser parser;
    auto doc = parser.parse("Just plain text\n## Section");
    EXPECT_TRUE(doc.title.empty());
    EXPECT_EQ(doc.sections.size(), 1);
}

// We fully test split_lines, extract_title, etc.
// through parse() without coupling to their names.
// If we refactor internals, these tests still pass.
```

If you later decide to merge `split_lines` and `extract_title` into a single pass, or switch to a regex-based implementation, none of these tests need to change. That's the payoff.

### Q2: When is testing private methods justified, and what are the approaches

Sometimes the urge to test a private method is a signal - it's telling you that the private method has grown into something with its own identity that deserves to be its own class. The first approach (extracting a class) is almost always the right move when you reach that point.

```cpp
// === Approach 1 (RECOMMENDED): Extract class ===
// If a private method is complex enough to need its own tests,
// it's telling you it should be its own class.

// BEFORE: complex private method
/*
class OrderProcessor {
private:
    double calculate_tax(double amount, const Address& addr) {
        // 50 lines of complex tax logic with edge cases
    }
};
*/

// AFTER: extracted, independently testable
class TaxCalculator {
public:
    double calculate(double amount, const std::string& state) {
        // Complex tax logic
        if (state == "OR") return 0;  // Oregon: no sales tax
        if (state == "CA") return amount * 0.0725;
        return amount * 0.05;
    }
};

class OrderProcessor {
public:
    explicit OrderProcessor(TaxCalculator& tax) : tax_(tax) {}
    // Now uses tax_.calculate() instead of private method
private:
    TaxCalculator& tax_;
};

// Direct unit test of the extracted class:
TEST(TaxCalculatorTest, OregonHasNoSalesTax) {
    TaxCalculator calc;
    EXPECT_DOUBLE_EQ(calc.calculate(100.0, "OR"), 0.0);
}


// === Approach 2 (LAST RESORT): Friend test class ===

class Widget {
    friend class WidgetInternalTest;  // Only for testing
private:
    int compute_checksum(const std::vector<uint8_t>& data) {
        int sum = 0;
        for (auto b : data) sum = (sum + b) % 256;
        return sum;
    }
};

class WidgetInternalTest : public ::testing::Test {
protected:
    Widget widget_;  // Can access private members
};

TEST_F(WidgetInternalTest, ChecksumComputation) {
    std::vector<uint8_t> data = {1, 2, 3};
    EXPECT_EQ(widget_.compute_checksum(data), 6);
}
// WARNING: This test is coupled to the private API.
// Mark it as tech debt to be refactored.


// === Approach 3 (PREPROCESSOR HACK - AVOID) ===
// Some codebases use:
// #define private public
// #include "widget.h"
// This is undefined behavior and breaks with modern compilers.
// NEVER do this.
```

**Guidelines:**

1. If you feel the need to test a private method, consider extracting it as a class first.
2. If extraction is impractical (legacy code with no safe refactoring path), use `friend` as a tactical compromise.
3. Never use `#define private public` - it causes undefined behavior and will break with modern compilers.
4. If you can't achieve the coverage you need through the public API, the design probably needs refactoring.

### Q3: Show strategies for testing internal state transitions without exposing internals

State machines are the classic case where developers reach for private access. "I need to verify the state is `Connected`," they say - and then they expose a `state_` getter or use `#define private public`. You don't need to do that. The state only matters because it affects behavior, so test the behavior.

```cpp
#include <gtest/gtest.h>
#include <string>

// === Strategy: Test observable effects, not internal state ===

class Connection {
public:
    bool connect(const std::string& host) {
        // Transitions: Disconnected -> Connecting -> Connected
        // Don't test internal state_ variable directly!
        host_ = host;
        state_ = State::Connected;
        return true;
    }

    bool send(const std::string& data) {
        if (state_ != State::Connected) return false;
        sent_bytes_ += data.size();
        return true;
    }

    void disconnect() {
        state_ = State::Disconnected;
    }

    // Observable queries - THESE are what we test against
    bool is_connected() const { return state_ == State::Connected; }
    size_t bytes_sent() const { return sent_bytes_; }

private:
    enum class State { Disconnected, Connecting, Connected };
    State state_ = State::Disconnected;
    std::string host_;
    size_t sent_bytes_ = 0;
};

// Test state transitions through OBSERVABLE EFFECTS

TEST(ConnectionTest, SendFailsWhenDisconnected) {
    Connection conn;
    EXPECT_FALSE(conn.send("data"));  // Tests internal state indirectly
}

TEST(ConnectionTest, SendSucceedsWhenConnected) {
    Connection conn;
    conn.connect("host");
    EXPECT_TRUE(conn.send("data"));
    EXPECT_EQ(conn.bytes_sent(), 4);
}

TEST(ConnectionTest, SendFailsAfterDisconnect) {
    Connection conn;
    conn.connect("host");
    conn.disconnect();
    EXPECT_FALSE(conn.send("data"));  // Verifies state reset
}

TEST(ConnectionTest, IsConnectedReflectsState) {
    Connection conn;
    EXPECT_FALSE(conn.is_connected());

    conn.connect("host");
    EXPECT_TRUE(conn.is_connected());

    conn.disconnect();
    EXPECT_FALSE(conn.is_connected());
}

// We fully verify the state machine without ever accessing
// the private State enum or state_ variable.
```

The key design move here is adding `is_connected()` and `bytes_sent()` as observable queries. These aren't exposing internals - they're expressing the public contract of the class. Adding queries like these is almost always the right answer when you're tempted to expose private state for testing purposes.

---

## Notes

- The default rule is: test through public interface only. Tests prove behavior, not implementation.
- Private methods that are "too complex" to test indirectly are missing abstractions that want to be extracted as their own class.
- `friend class TestFixture` is a tactical compromise for legacy code, not a design pattern to reach for first.
- The need to test internals often signals an SRP violation - the class is doing too many distinct things.
- Add observable queries (`is_connected()`, `size()`) to make state testable without exposing private members directly.
- Remember the golden rule: if you refactor internals and tests break, those tests were testing implementation, not behavior. That's the sign you went too deep.
- Code coverage tools can confirm that private methods are exercised through public API tests - use them to verify your public-interface tests are reaching all the private code paths.
