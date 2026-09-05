# Write effective unit tests with Arrange-Act-Assert pattern

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Arrange-Act-Assert (AAA)** is the fundamental unit test structure: set up preconditions, execute the behavior under test, then verify outcomes. Clear AAA structure makes tests readable, maintainable, and debuggable. Each test should test one logical behavior.

The reason AAA matters beyond just organization is that it enforces a discipline: if you cannot clearly identify one "Act" step, your test is probably doing too much. When a test fails, the three-section structure tells you immediately whether the problem is in your test setup, in the code under test, or in your expectation.

### AAA Structure

Here is the template. Notice the three labeled regions - keeping them visually distinct in your code makes the test much easier to understand months later.

```cpp
TEST(Suite, TestName) {
    // Arrange - set up the world
    //   Create objects, configure mocks, prepare inputs

    // Act - do the one thing being tested
    //   Call the function/method under test

    // Assert - verify the outcome
    //   Check return values, state changes, mock interactions
}
```

### Test Quality Checklist

| Property | Description | Anti-pattern |
| --- | --- | --- |
| **Fast** | Milliseconds, no I/O | Database calls in unit tests |
| **Isolated** | No shared state between tests | Static variables modified by tests |
| **Repeatable** | Same result every run | Tests depending on system time |
| **Self-validating** | Pass/fail without human inspection | Tests that print output to verify manually |
| **Thorough** | Cover happy path, edge cases, errors | Only testing the obvious case |

---

## Self-Assessment

### Q1: Write well-structured AAA tests for a real component

**Answer:**

The `ShoppingCart` below is the production code. Read through the tests after it and notice how each one has a single, clear action and assertions that only verify what that action should produce. The `ASSERT_TRUE` on line where we check `found.has_value()` is important - if the item is not in the cart, there is nothing meaningful to check about its quantity, so we abort the test with `ASSERT_` rather than continuing with `EXPECT_`.

```cpp
#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <optional>
#include <stdexcept>

// === Production code ===
class ShoppingCart {
public:
    struct Item {
        std::string name;
        double price;
        int quantity;
    };

    void add(const std::string& name, double price, int qty = 1) {
        if (price < 0) throw std::invalid_argument("Negative price");
        if (qty <= 0) throw std::invalid_argument("Non-positive quantity");
        for (auto& item : items_) {
            if (item.name == name) {
                item.quantity += qty;
                return;
            }
        }
        items_.push_back({name, price, qty});
    }

    bool remove(const std::string& name) {
        auto it = std::find_if(items_.begin(), items_.end(),
            [&](const Item& i) { return i.name == name; });
        if (it == items_.end()) return false;
        items_.erase(it);
        return true;
    }

    double total() const {
        double sum = 0;
        for (const auto& item : items_)
            sum += item.price * item.quantity;
        return sum;
    }

    double total_with_discount(double pct) const {
        return total() * (1.0 - pct / 100.0);
    }

    size_t item_count() const { return items_.size(); }
    bool empty() const { return items_.empty(); }

    std::optional<Item> find(const std::string& name) const {
        for (const auto& item : items_)
            if (item.name == name) return item;
        return std::nullopt;
    }

private:
    std::vector<Item> items_;
};

// === Tests following AAA pattern ===

// --- Happy path tests ---

TEST(ShoppingCartTest, AddItemIncreasesTotal) {
    // Arrange
    ShoppingCart cart;

    // Act
    cart.add("Widget", 9.99);

    // Assert
    EXPECT_EQ(cart.item_count(), 1);
    EXPECT_DOUBLE_EQ(cart.total(), 9.99);
}

TEST(ShoppingCartTest, AddDuplicateItemIncreasesQuantity) {
    // Arrange
    ShoppingCart cart;
    cart.add("Widget", 9.99, 2);

    // Act
    cart.add("Widget", 9.99, 3);

    // Assert - still one item type, but quantity is 5
    EXPECT_EQ(cart.item_count(), 1);
    auto found = cart.find("Widget");
    ASSERT_TRUE(found.has_value());   // ASSERT: abort if null (can't continue)
    EXPECT_EQ(found->quantity, 5);
    EXPECT_DOUBLE_EQ(cart.total(), 9.99 * 5);
}

TEST(ShoppingCartTest, RemoveExistingItemReturnsTrue) {
    // Arrange
    ShoppingCart cart;
    cart.add("Widget", 9.99);
    cart.add("Gadget", 19.99);

    // Act
    bool removed = cart.remove("Widget");

    // Assert
    EXPECT_TRUE(removed);
    EXPECT_EQ(cart.item_count(), 1);
    EXPECT_FALSE(cart.find("Widget").has_value());
}

// --- Edge case tests ---

TEST(ShoppingCartTest, EmptyCartHasZeroTotal) {
    // Arrange
    ShoppingCart cart;

    // Act & Assert (trivial act)
    EXPECT_TRUE(cart.empty());
    EXPECT_DOUBLE_EQ(cart.total(), 0.0);
}

TEST(ShoppingCartTest, RemoveNonexistentItemReturnsFalse) {
    // Arrange
    ShoppingCart cart;
    cart.add("Widget", 9.99);

    // Act
    bool removed = cart.remove("NonExistent");

    // Assert
    EXPECT_FALSE(removed);
    EXPECT_EQ(cart.item_count(), 1);  // Nothing changed
}

TEST(ShoppingCartTest, DiscountAppliedCorrectly) {
    // Arrange
    ShoppingCart cart;
    cart.add("Widget", 100.0);

    // Act
    double discounted = cart.total_with_discount(25.0);  // 25% off

    // Assert
    EXPECT_DOUBLE_EQ(discounted, 75.0);
}

// --- Error handling tests ---

TEST(ShoppingCartTest, NegativePriceThrows) {
    // Arrange
    ShoppingCart cart;

    // Act & Assert
    EXPECT_THROW(cart.add("Bad", -5.0), std::invalid_argument);
    EXPECT_TRUE(cart.empty());  // Cart unchanged after error
}

TEST(ShoppingCartTest, ZeroQuantityThrows) {
    // Arrange
    ShoppingCart cart;

    // Act & Assert
    EXPECT_THROW(cart.add("Bad", 5.0, 0), std::invalid_argument);
}
```

Notice `EXPECT_TRUE(cart.empty())` after the throw test. This is verifying that the exception left the cart in a clean state - testing error paths is not just about the exception type, it is also about checking that your code did not half-modify state before throwing.

### Q2: What makes a good vs bad unit test? Show anti-patterns

**Answer:**

The anti-patterns below are all real mistakes that appear in production test suites. Each one makes tests harder to maintain and debug without providing any extra coverage.

```cpp
// === ANTI-PATTERN 1: Multiple acts in one test ===
// BAD - if add() breaks, the remove() assertion confusingly fails too
TEST(Bad, TestEverything) {
    ShoppingCart cart;
    cart.add("A", 10.0);
    EXPECT_EQ(cart.item_count(), 1);  // Testing add AND count
    cart.add("B", 20.0);
    EXPECT_DOUBLE_EQ(cart.total(), 30.0);  // Testing add AND total
    cart.remove("A");
    EXPECT_DOUBLE_EQ(cart.total(), 20.0);  // Testing remove AND total
}
// GOOD: Split into separate tests, each testing one behavior


// === ANTI-PATTERN 2: Testing implementation, not behavior ===
// BAD - coupled to internal data structure (vector)
TEST(Bad, InternalState) {
    ShoppingCart cart;
    cart.add("A", 10.0);
    // Don't test internals! If we change vector to map, test breaks.
    // EXPECT_EQ(cart.items_[0].name, "A");  // Accessing private!
}
// GOOD: Test through public interface
TEST(Good, BehaviorBased) {
    ShoppingCart cart;
    cart.add("A", 10.0);
    EXPECT_TRUE(cart.find("A").has_value());  // Tests observable behavior
}


// === ANTI-PATTERN 3: Test names that don't describe the behavior ===
// BAD:
TEST(Cart, Test1) { /* ... */ }
TEST(Cart, TestAdd) { /* ... */ }  // What about add?

// GOOD: Name describes scenario AND expected outcome
TEST(ShoppingCart, AddDuplicateItem_IncreasesQuantity_NotItemCount) { /* ... */ }
TEST(ShoppingCart, RemoveLastItem_CartBecomesEmpty) { /* ... */ }


// === ANTI-PATTERN 4: Shared mutable state between tests ===
// BAD:
static ShoppingCart shared_cart;  // Tests depend on execution order!
TEST(Bad, First) { shared_cart.add("A", 10.0); }
TEST(Bad, Second) {
    // Depends on First running beforehand - fragile!
    EXPECT_EQ(shared_cart.item_count(), 1);
}
// GOOD: Each test creates its own instance (fixtures help with DRY)


// === ANTI-PATTERN 5: Overly precise assertions ===
// BAD - breaks if formatting changes:
// EXPECT_EQ(cart.summary(), "Cart: 1 items, total $9.99");
// GOOD - test semantics, not formatting:
// EXPECT_THAT(cart.summary(), HasSubstr("9.99"));
// EXPECT_EQ(cart.total(), 9.99);
```

The reason anti-pattern 3 (bad test names) matters more than it looks: when a test fails in CI at 2am, the test name is the first thing anyone reads. A name like `Test1` tells you nothing. A name like `AddDuplicateItem_IncreasesQuantity_NotItemCount` tells you immediately what broke and what the expected behavior is.

**Test naming conventions (pick one, be consistent):**

- `MethodName_Scenario_ExpectedResult` - `Add_DuplicateItem_IncreasesQuantity`
- `Should_ExpectedBehavior_When_Condition` - `Should_IncreaseQuantity_When_DuplicateAdded`
- Google Test style: `SuiteName.TestName` - `ShoppingCartTest.AddDuplicateIncreasesQuantity`

### Q3: Show test fixtures and parameterized tests for DRY test code

**Answer:**

The fixture below eliminates the repetitive `cart.add(...)` setup from every single test. The parameterized test section shows how to drive the same assertion logic with a table of input/expected pairs - one parameterized test does the work of five or six hand-written tests.

```cpp
#include <gtest/gtest.h>
#include <tuple>

// === Test Fixture: shared setup, per-test isolation ===

class ShoppingCartTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Runs before EACH test - fresh cart every time
        cart.add("Laptop", 999.99);
        cart.add("Mouse", 29.99);
        cart.add("Keyboard", 79.99);
    }

    // void TearDown() override { }  // Optional: runs after each test

    ShoppingCart cart;  // Each test gets its own instance
};

TEST_F(ShoppingCartTest, InitialCartHasThreeItems) {
    EXPECT_EQ(cart.item_count(), 3);
}

TEST_F(ShoppingCartTest, TotalIsCorrect) {
    EXPECT_DOUBLE_EQ(cart.total(), 999.99 + 29.99 + 79.99);
}

TEST_F(ShoppingCartTest, RemoveItemReducesCount) {
    cart.remove("Mouse");
    EXPECT_EQ(cart.item_count(), 2);
}


// === Parameterized Tests: same logic, different data ===

struct DiscountTestCase {
    double price;
    double discount_pct;
    double expected;
    std::string description;
};

class DiscountTest : public ::testing::TestWithParam<DiscountTestCase> {};

TEST_P(DiscountTest, AppliesDiscountCorrectly) {
    // Arrange
    auto [price, discount, expected, desc] = GetParam();
    ShoppingCart cart;
    cart.add("Item", price);

    // Act
    double result = cart.total_with_discount(discount);

    // Assert
    EXPECT_NEAR(result, expected, 0.01) << "Failed for: " << desc;
}

INSTANTIATE_TEST_SUITE_P(
    DiscountCases, DiscountTest,
    ::testing::Values(
        DiscountTestCase{100.0, 0.0, 100.0, "no discount"},
        DiscountTestCase{100.0, 50.0, 50.0, "50% off"},
        DiscountTestCase{100.0, 100.0, 0.0, "100% off (free)"},
        DiscountTestCase{200.0, 25.0, 150.0, "25% off $200"},
        DiscountTestCase{0.0, 50.0, 0.0, "discount on zero"}
    )
);
```

The `<< "Failed for: " << desc` at the end of the assertion is worth noting - when a parameterized test fails, you want to know which parameter set triggered it. Adding context to your assertion messages makes CI failures much faster to diagnose.

---

## Notes

- **One logical assertion per test** - multiple `EXPECT_*` calls are fine if they verify the same behavior.
- Use `EXPECT_*` (non-fatal) by default; `ASSERT_*` (fatal) only when continuing is meaningless.
- `EXPECT_DOUBLE_EQ` checks within 4 ULP; `EXPECT_NEAR(a, b, epsilon)` for custom tolerance.
- Test names should read as specifications - if you removed the code, the test names should describe behavior.
- Each `TEST` and `TEST_F` creates a new test fixture instance - tests are always isolated.
- Aim for test pyramid: many unit tests (fast), fewer integration tests, few end-to-end tests.
