# Use AI to generate unit tests and test fixtures for C++ code

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

Writing tests is important but genuinely tedious, and it's one of the best uses of AI in a C++ workflow. Given a class or function, AI can identify test scenarios - including edge cases, error paths, and boundary conditions - that developers often miss when they're mentally close to the implementation. The key to getting good results is specificity in your prompt: tell the AI which framework you're using, what aspects of the code to focus on, and what coverage level you want.

### AI Test Generation Effectiveness

Not all test types are created equal. AI excels at structured, predictable test patterns but struggles anywhere human judgment about concurrent behavior or system context is required.

| Test Type | AI Quality | Human Review Needed |
| --- | --- | --- |
| Happy-path unit tests | Excellent | Low |
| Edge case identification | Good | Medium (may miss domain-specific) |
| Error path testing | Good | Medium |
| Concurrency tests | Weak | High |
| Performance/benchmark tests | Medium | Medium |
| Integration tests | Medium | High (needs system context) |
| Fuzz test harnesses | Good | Low |

---

## Self-Assessment

### Q1: Generate comprehensive unit tests from a class

**Answer:**

The prompt below asks for a `TEST_F` fixture-based suite, which is the right choice when multiple tests share setup logic. Notice that it explicitly lists the categories to cover - without that list, AI tends to generate only happy-path tests.

```cpp
=== PROMPT ===

"Generate GoogleTest unit tests for this C++ class.
Cover:

- Normal operation
- Edge cases (empty, single element, max capacity)
- Error conditions
- Move semantics
- Exception safety

Use TEST_F with a fixture for setup/teardown.
Each test should be independent and self-documenting.
```

Here's the class interface being tested, followed by the AI-generated suite. The fixture creates a fresh buffer for each test, which is what makes the tests independent - no test depends on state left by a previous one.

```cpp
// === The class to test ===
template<typename T>
class RingBuffer {
public:
    explicit RingBuffer(size_t capacity);
    bool push(const T& item);
    bool push(T&& item);
    std::optional<T> pop();
    std::optional<T> peek() const;
    size_t size() const;
    size_t capacity() const;
    bool empty() const;
    bool full() const;
    void clear();
};

// === AI-generated test suite ===
class RingBufferTest : public ::testing::Test {
protected:
    void SetUp() override {
        buf_ = std::make_unique<RingBuffer<int>>(5);
    }
    std::unique_ptr<RingBuffer<int>> buf_;
};

// --- Construction ---
TEST_F(RingBufferTest, ConstructedEmpty) {
    EXPECT_TRUE(buf_->empty());
    EXPECT_FALSE(buf_->full());
    EXPECT_EQ(buf_->size(), 0);
    EXPECT_EQ(buf_->capacity(), 5);
}

// --- Push/Pop ---
TEST_F(RingBufferTest, PushPopSingle) {
    EXPECT_TRUE(buf_->push(42));
    EXPECT_EQ(buf_->size(), 1);
    auto val = buf_->pop();
    ASSERT_TRUE(val.has_value());
    EXPECT_EQ(*val, 42);
    EXPECT_TRUE(buf_->empty());
}

TEST_F(RingBufferTest, PushPopFIFO) {
    buf_->push(1); buf_->push(2); buf_->push(3);
    EXPECT_EQ(buf_->pop(), 1);
    EXPECT_EQ(buf_->pop(), 2);
    EXPECT_EQ(buf_->pop(), 3);
}

TEST_F(RingBufferTest, PushToFull) {
    for (int i = 0; i < 5; ++i)
        EXPECT_TRUE(buf_->push(i));
    EXPECT_TRUE(buf_->full());
    EXPECT_FALSE(buf_->push(99));  // Should fail
}

// --- Edge cases ---
TEST_F(RingBufferTest, PopFromEmpty) {
    auto val = buf_->pop();
    EXPECT_FALSE(val.has_value());
}

TEST_F(RingBufferTest, PeekDoesNotRemove) {
    buf_->push(42);
    auto peek1 = buf_->peek();
    auto peek2 = buf_->peek();
    EXPECT_EQ(peek1, peek2);
    EXPECT_EQ(buf_->size(), 1);
}

TEST_F(RingBufferTest, WrapAround) {
    // Fill then drain then fill to trigger wrap-around
    for (int i = 0; i < 5; ++i) buf_->push(i);
    for (int i = 0; i < 3; ++i) buf_->pop();
    for (int i = 10; i < 13; ++i) EXPECT_TRUE(buf_->push(i));

    EXPECT_EQ(buf_->pop(), 3);
    EXPECT_EQ(buf_->pop(), 4);
    EXPECT_EQ(buf_->pop(), 10);
    EXPECT_EQ(buf_->pop(), 11);
    EXPECT_EQ(buf_->pop(), 12);
}

TEST_F(RingBufferTest, ClearResetsState) {
    buf_->push(1); buf_->push(2);
    buf_->clear();
    EXPECT_TRUE(buf_->empty());
    EXPECT_EQ(buf_->size(), 0);
    EXPECT_TRUE(buf_->push(99));  // Can push after clear
}

// --- Move semantics ---
TEST_F(RingBufferTest, PushMoveOnly) {
    RingBuffer<std::unique_ptr<int>> mbuf(3);
    mbuf.push(std::make_unique<int>(42));
    auto val = mbuf.pop();
    ASSERT_NE(val, std::nullopt);
    EXPECT_EQ(**val, 42);
}
```

The `WrapAround` test is the most interesting one here - a ring buffer's whole purpose is that it wraps around correctly, and that's easy to miss if you only test with a fresh empty buffer. Always check that AI includes this kind of state-transition test, not just isolated single-operation tests.

### Q2: Generate test fixtures with mock objects

**Answer:**

When the class under test has dependencies (database, email service, HTTP client), you need mocks to test it in isolation. AI can generate both the mock classes and the tests, as long as you include the interface definitions in the prompt. Without those interfaces, AI has to guess and will usually guess wrong.

```cpp
=== PROMPT ===

"Generate GoogleTest + GoogleMock tests for OrderService.
Mock out the database and email dependencies.
Test each method with success and failure scenarios."
```

The interfaces to mock come first, then the generated mocks and fixture. Notice that the fixture creates fresh mock instances in `SetUp()` - this ensures each test starts with a clean mock that has no preloaded expectations from a previous test.

```cpp
// === Interfaces to mock ===
class IOrderRepository {
public:
    virtual ~IOrderRepository() = default;
    virtual std::optional<Order> find(const std::string& id) = 0;
    virtual bool save(const Order& order) = 0;
    virtual bool remove(const std::string& id) = 0;
};

class IEmailService {
public:
    virtual ~IEmailService() = default;
    virtual bool send(const std::string& to, const std::string& subject,
                      const std::string& body) = 0;
};

// === AI-generated mocks ===
class MockOrderRepo : public IOrderRepository {
public:
    MOCK_METHOD(std::optional<Order>, find, (const std::string&), (override));
    MOCK_METHOD(bool, save, (const Order&), (override));
    MOCK_METHOD(bool, remove, (const std::string&), (override));
};

class MockEmailService : public IEmailService {
public:
    MOCK_METHOD(bool, send, (const std::string&, const std::string&,
                              const std::string&), (override));
};

// === AI-generated fixture ===
class OrderServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        repo_ = std::make_shared<MockOrderRepo>();
        email_ = std::make_shared<MockEmailService>();
        service_ = std::make_unique<OrderService>(repo_, email_);
    }

    std::shared_ptr<MockOrderRepo> repo_;
    std::shared_ptr<MockEmailService> email_;
    std::unique_ptr<OrderService> service_;

    Order sample_order() {
        return Order{"ORD-001", "customer@test.com", 99.99};
    }
};

// === Tests ===
TEST_F(OrderServiceTest, CreateOrder_Success) {
    EXPECT_CALL(*repo_, save(::testing::_))
        .WillOnce(::testing::Return(true));
    EXPECT_CALL(*email_, send("customer@test.com", ::testing::_, ::testing::_))
        .WillOnce(::testing::Return(true));

    auto result = service_->create_order("customer@test.com", 99.99);
    ASSERT_TRUE(result.has_value());
    EXPECT_FALSE(result->id.empty());
}

TEST_F(OrderServiceTest, CreateOrder_DbFailure) {
    EXPECT_CALL(*repo_, save(::testing::_))
        .WillOnce(::testing::Return(false));
    EXPECT_CALL(*email_, send(::testing::_, ::testing::_, ::testing::_))
        .Times(0);  // Email should NOT be sent on save failure

    auto result = service_->create_order("customer@test.com", 99.99);
    EXPECT_FALSE(result.has_value());
}

TEST_F(OrderServiceTest, CancelOrder_NotFound) {
    EXPECT_CALL(*repo_, find("ORD-999"))
        .WillOnce(::testing::Return(std::nullopt));

    auto result = service_->cancel_order("ORD-999");
    EXPECT_FALSE(result);
}

TEST_F(OrderServiceTest, CancelOrder_EmailFailure_StillCancels) {
    EXPECT_CALL(*repo_, find("ORD-001"))
        .WillOnce(::testing::Return(sample_order()));
    EXPECT_CALL(*repo_, save(::testing::_))
        .WillOnce(::testing::Return(true));
    EXPECT_CALL(*email_, send(::testing::_, ::testing::_, ::testing::_))
        .WillOnce(::testing::Return(false));  // Email fails

    auto result = service_->cancel_order("ORD-001");
    EXPECT_TRUE(result);  // Order still cancelled despite email failure
}
```

The last test - `CancelOrder_EmailFailure_StillCancels` - is the kind of test that's easy to forget but important to have. It documents a specific design decision: a failed email notification should not roll back the cancellation. AI generates this kind of scenario reliably when you ask for "failure scenarios" in the prompt.

### Q3: Generate edge case and boundary tests

**Answer:**

Edge case tests are where AI really earns its keep. When you're writing `find_median` you're thinking about the algorithm, not about whether `INT_MIN` and `INT_MAX` in the same vector will overflow during averaging. AI thinks about these systematically.

```cpp
=== PROMPT ===

"For this function, generate tests for ALL edge cases:

- Empty input
- Single element
- Boundary values (INT_MIN, INT_MAX, 0)
- Duplicate values
- Already sorted / reverse sorted
- Large input (performance sanity check)
```

Here's the function being tested and the AI-generated suite. The performance test at the end is especially useful - it acts as a smoke test that the implementation isn't accidentally O(n^2).

```cpp
// Function to test:
int find_median(std::vector<int>& data);

// AI-generated edge case tests:
TEST(FindMedianTest, EmptyVector_Throws) {
    std::vector<int> empty;
    EXPECT_THROW(find_median(empty), std::invalid_argument);
}

TEST(FindMedianTest, SingleElement) {
    std::vector<int> v{42};
    EXPECT_EQ(find_median(v), 42);
}

TEST(FindMedianTest, TwoElements) {
    std::vector<int> v{1, 3};
    EXPECT_EQ(find_median(v), 2);  // Average for even count
}

TEST(FindMedianTest, OddCount) {
    std::vector<int> v{3, 1, 2};
    EXPECT_EQ(find_median(v), 2);
}

TEST(FindMedianTest, EvenCount) {
    std::vector<int> v{4, 1, 3, 2};
    EXPECT_EQ(find_median(v), 2);  // (2+3)/2 = 2 (integer division)
}

TEST(FindMedianTest, AllDuplicates) {
    std::vector<int> v{5, 5, 5, 5, 5};
    EXPECT_EQ(find_median(v), 5);
}

TEST(FindMedianTest, AlreadySorted) {
    std::vector<int> v{1, 2, 3, 4, 5};
    EXPECT_EQ(find_median(v), 3);
}

TEST(FindMedianTest, ReverseSorted) {
    std::vector<int> v{5, 4, 3, 2, 1};
    EXPECT_EQ(find_median(v), 3);
}

TEST(FindMedianTest, NegativeNumbers) {
    std::vector<int> v{-5, -1, -3};
    EXPECT_EQ(find_median(v), -3);
}

TEST(FindMedianTest, MixedSignedNumbers) {
    std::vector<int> v{-10, 0, 10};
    EXPECT_EQ(find_median(v), 0);
}

TEST(FindMedianTest, IntBoundaries) {
    std::vector<int> v{INT_MIN, 0, INT_MAX};
    EXPECT_EQ(find_median(v), 0);
}

TEST(FindMedianTest, LargeInput_Performance) {
    std::vector<int> v(1'000'000);
    std::iota(v.begin(), v.end(), 0);
    std::shuffle(v.begin(), v.end(), std::mt19937{42});

    auto start = std::chrono::steady_clock::now();
    auto result = find_median(v);
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    EXPECT_EQ(result, 499'999);
    EXPECT_LT(ms, 1000);  // Should complete in <1s
}
```

Always review the expected values in AI-generated tests. The AI may assert the correct behavior for a function it imagines you're writing, but if your `find_median` uses a different rounding convention for even-count vectors, some assertions will be wrong even though the implementation is correct. The tests document a contract; make sure you agree with that contract before you commit to it.

---

## Notes

- Prompt with the class interface, not just the class name - AI needs the method signatures to generate accurate tests.
- Ask AI to generate tests before you write the implementation (TDD workflow) - the tests then constrain what the implementation needs to do, which often clarifies the design.
- AI excels at edge cases - it systematically considers empty, single element, boundary, and overflow scenarios that are easy to overlook.
- Mock tests need interface definitions included in the prompt - without them, AI has to invent the interface and will likely get it wrong.
- Always review AI-generated test assertions - they may assert the wrong expected value based on an assumption about your function's behavior.
- Use parameterized tests (`TEST_P`) for data-driven testing of multiple inputs - ask AI to generate a `INSTANTIATE_TEST_SUITE_P` block alongside the test body.
- AI is weak at concurrency tests - it often generates tests that pass by luck rather than correctly synchronizing. Write those by hand.
