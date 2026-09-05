# Design contract tests for API boundaries between modules

**Category:** Testing in Practice  
**Standard:** C++26  

---

## Topic Overview

**Contract testing** verifies that modules interact correctly at their boundaries. The key idea is this: instead of mocking everything and hoping the real implementations behave the same way, you define the expected behavior of an interface as a reusable test suite and then run that same suite against every implementation. This catches integration bugs early, without requiring a full system test running against all the real infrastructure.

The mental model is a legal contract. The interface says "I promise to behave this way." The contract tests prove whether a given implementation actually keeps that promise.

### Testing Level Comparison

It helps to see where contract tests sit relative to other testing strategies:

| Level | Scope | Speed | Catches |
| --- | --- | --- | --- |
| Unit test | Single function/class | Very fast | Logic bugs |
| **Contract test** | **Interface boundary** | **Fast** | **API misuse, protocol violations** |
| Integration test | Multiple components | Slow | Wiring bugs |
| System test | Full application | Very slow | End-to-end failures |

Contract tests are faster than integration tests because they can use in-process implementations. They are more rigorous than unit tests because they verify behavior at the public boundary, not internal logic.

### Contract Test Patterns

Here is the structural relationship the pattern creates:

```cpp
 Consumer         Contract              Provider
 +--------+      +---------+           +---------+
 | Module |----->| IStore  |<----------| SQLite  |
 |   A    |      | (iface) |           | Store   |
 +--------+      +---------+           +---------+
                      ^
                      |
                 Contract Tests:
                 - Store/retrieve roundtrip
                 - Error on missing key
                 - Thread safety guarantees
```

The contract test suite depends only on the interface - never on any specific implementation. Each implementation is plugged in separately.

---

## Self-Assessment

### Q1: Define contract tests for an interface with multiple implementations

**Answer:**

The key technique here is Google Test's typed test suite. You write the tests once against the interface type, then register each implementation separately. The test infrastructure runs the full suite for every registered implementation automatically.

```cpp
#include <gtest/gtest.h>
#include <memory>
#include <string>
#include <optional>
#include <stdexcept>

// === Interface (contract definition) ===
class IKeyValueStore {
public:
    virtual ~IKeyValueStore() = default;
    virtual void put(const std::string& key, const std::string& value) = 0;
    virtual std::optional<std::string> get(const std::string& key) const = 0;
    virtual bool remove(const std::string& key) = 0;
    virtual size_t size() const = 0;
    virtual void clear() = 0;
};

// === Contract test fixture (parameterized by implementation) ===
template<typename Factory>
class KeyValueStoreContractTest : public ::testing::Test {
protected:
    void SetUp() override {
        store_ = Factory::create();
    }

    std::unique_ptr<IKeyValueStore> store_;
};

TYPED_TEST_SUITE_P(KeyValueStoreContractTest);

// === Contract: put then get returns the value ===
TYPED_TEST_P(KeyValueStoreContractTest, PutThenGetReturnsValue) {
    this->store_->put("key1", "value1");
    auto result = this->store_->get("key1");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(*result, "value1");
}

// === Contract: get on missing key returns nullopt ===
TYPED_TEST_P(KeyValueStoreContractTest, GetMissingKeyReturnsNullopt) {
    auto result = this->store_->get("nonexistent");
    EXPECT_FALSE(result.has_value());
}

// === Contract: put overwrites existing value ===
TYPED_TEST_P(KeyValueStoreContractTest, PutOverwritesExisting) {
    this->store_->put("key", "v1");
    this->store_->put("key", "v2");
    EXPECT_EQ(this->store_->get("key").value(), "v2");
}

// === Contract: remove returns true for existing key ===
TYPED_TEST_P(KeyValueStoreContractTest, RemoveExistingReturnsTrue) {
    this->store_->put("key", "value");
    EXPECT_TRUE(this->store_->remove("key"));
    EXPECT_FALSE(this->store_->get("key").has_value());
}

// === Contract: remove returns false for missing key ===
TYPED_TEST_P(KeyValueStoreContractTest, RemoveMissingReturnsFalse) {
    EXPECT_FALSE(this->store_->remove("nonexistent"));
}

// === Contract: size tracks entries ===
TYPED_TEST_P(KeyValueStoreContractTest, SizeTracksEntries) {
    EXPECT_EQ(this->store_->size(), 0);
    this->store_->put("a", "1");
    EXPECT_EQ(this->store_->size(), 1);
    this->store_->put("b", "2");
    EXPECT_EQ(this->store_->size(), 2);
    this->store_->remove("a");
    EXPECT_EQ(this->store_->size(), 1);
}

// === Contract: clear removes all entries ===
TYPED_TEST_P(KeyValueStoreContractTest, ClearRemovesAll) {
    this->store_->put("a", "1");
    this->store_->put("b", "2");
    this->store_->clear();
    EXPECT_EQ(this->store_->size(), 0);
}

REGISTER_TYPED_TEST_SUITE_P(KeyValueStoreContractTest,
    PutThenGetReturnsValue,
    GetMissingKeyReturnsNullopt,
    PutOverwritesExisting,
    RemoveExistingReturnsTrue,
    RemoveMissingReturnsFalse,
    SizeTracksEntries,
    ClearRemovesAll);
```

Notice that `REGISTER_TYPED_TEST_SUITE_P` just collects the tests - it does not instantiate them yet. Instantiation happens in the next step with `INSTANTIATE_TYPED_TEST_SUITE_P`, once for each implementation you want to verify.

### Q2: Apply contract tests to multiple implementations

**Answer:**

Here are two concrete implementations and the factory types that connect them to the contract suite. The registration at the bottom is the only code you add when a new implementation arrives.

```cpp
#include <unordered_map>

// === Implementation 1: In-memory store ===
class InMemoryStore : public IKeyValueStore {
public:
    void put(const std::string& key, const std::string& value) override {
        data_[key] = value;
    }
    std::optional<std::string> get(const std::string& key) const override {
        auto it = data_.find(key);
        if (it == data_.end()) return std::nullopt;
        return it->second;
    }
    bool remove(const std::string& key) override {
        return data_.erase(key) > 0;
    }
    size_t size() const override { return data_.size(); }
    void clear() override { data_.clear(); }
private:
    std::unordered_map<std::string, std::string> data_;
};

// === Implementation 2: File-backed store (simplified) ===
class FileBackedStore : public IKeyValueStore {
public:
    explicit FileBackedStore(const std::string& path) : path_(path) {}
    void put(const std::string& key, const std::string& value) override {
        cache_[key] = value;
        // In real code: persist to file
    }
    std::optional<std::string> get(const std::string& key) const override {
        auto it = cache_.find(key);
        return it != cache_.end() ? std::optional{it->second} : std::nullopt;
    }
    bool remove(const std::string& key) override {
        return cache_.erase(key) > 0;
    }
    size_t size() const override { return cache_.size(); }
    void clear() override { cache_.clear(); }
private:
    std::string path_;
    std::unordered_map<std::string, std::string> cache_;
};

// === Factory types for parameterized testing ===
struct InMemoryFactory {
    static std::unique_ptr<IKeyValueStore> create() {
        return std::make_unique<InMemoryStore>();
    }
};

struct FileBackedFactory {
    static std::unique_ptr<IKeyValueStore> create() {
        return std::make_unique<FileBackedStore>("/tmp/test_store.db");
    }
};

// === Instantiate contract tests for BOTH implementations ===
INSTANTIATE_TYPED_TEST_SUITE_P(InMemory, KeyValueStoreContractTest, InMemoryFactory);
INSTANTIATE_TYPED_TEST_SUITE_P(FileBacked, KeyValueStoreContractTest, FileBackedFactory);

// Both implementations must pass ALL contract tests.
// If FileBackedStore fails PutOverwritesExisting, we know it violates the contract.
```

When `FileBackedStore` fails `PutOverwritesExisting`, the failure message will say `FileBacked/KeyValueStoreContractTest.PutOverwritesExisting` - immediately telling you which implementation broke which clause of the contract.

### Q3: Contract tests for async/callback-based interfaces

**Answer:**

Async interfaces are trickier because you cannot just call a method and check the return value - you have to wait for a callback to fire. The pattern here uses `std::promise` and `std::future` to bridge from callback-style to synchronous assertion code, with an explicit timeout so a test never hangs forever.

```cpp
#include <gtest/gtest.h>
#include <future>
#include <chrono>
#include <functional>

// === Async interface ===
class IMessageBroker {
public:
    virtual ~IMessageBroker() = default;
    using Callback = std::function<void(const std::string& topic,
                                         const std::string& payload)>;
    virtual void publish(const std::string& topic,
                         const std::string& payload) = 0;
    virtual void subscribe(const std::string& topic, Callback cb) = 0;
    virtual void unsubscribe(const std::string& topic) = 0;
};

// === Contract test for async interfaces ===
template<typename Factory>
class MessageBrokerContractTest : public ::testing::Test {
protected:
    void SetUp() override { broker_ = Factory::create(); }
    std::unique_ptr<IMessageBroker> broker_;
};

TYPED_TEST_SUITE_P(MessageBrokerContractTest);

// Contract: subscriber receives published messages
TYPED_TEST_P(MessageBrokerContractTest, SubscriberReceivesMessages) {
    std::promise<std::string> received;
    auto future = received.get_future();

    this->broker_->subscribe("test-topic", [&](auto, auto payload) {
        received.set_value(payload);
    });

    this->broker_->publish("test-topic", "hello");

    ASSERT_EQ(future.wait_for(std::chrono::seconds(5)),
              std::future_status::ready);
    EXPECT_EQ(future.get(), "hello");
}

// Contract: unsubscribed topics don't deliver
TYPED_TEST_P(MessageBrokerContractTest, UnsubscribeStopsDelivery) {
    std::atomic<int> count{0};

    this->broker_->subscribe("topic", [&](auto, auto) { count++; });
    this->broker_->publish("topic", "msg1");
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    this->broker_->unsubscribe("topic");
    this->broker_->publish("topic", "msg2");
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    EXPECT_EQ(count.load(), 1);
}

// Contract: different topics are independent
TYPED_TEST_P(MessageBrokerContractTest, TopicsAreIndependent) {
    std::atomic<int> topic_a_count{0};
    std::atomic<int> topic_b_count{0};

    this->broker_->subscribe("A", [&](auto, auto) { topic_a_count++; });
    this->broker_->subscribe("B", [&](auto, auto) { topic_b_count++; });

    this->broker_->publish("A", "msg");
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    EXPECT_EQ(topic_a_count.load(), 1);
    EXPECT_EQ(topic_b_count.load(), 0);
}

REGISTER_TYPED_TEST_SUITE_P(MessageBrokerContractTest,
    SubscriberReceivesMessages,
    UnsubscribeStopsDelivery,
    TopicsAreIndependent);
```

The five-second timeout in `wait_for` is important. Without it, a broken implementation that never delivers a message would hang the test runner indefinitely. The timeout converts a hang into a clean `ASSERT_EQ` failure.

---

## Notes

- Contract tests define *what* an interface must do, not *how* - they are implementation-agnostic by design.
- Use `TYPED_TEST_SUITE_P` to write the tests once and run them against all implementations.
- Adding a new implementation? Just register it with `INSTANTIATE_TYPED_TEST_SUITE_P` - all contract tests run automatically with no changes to the test code.
- Contract tests complement unit tests: unit tests verify internal logic, contract tests verify observable behavior at public boundaries.
- For async interfaces, use `std::promise`/`std::future` with explicit timeouts to avoid hanging tests.
- A good contract test suite becomes the definitive, executable specification for an interface.
- When a contract test fails for one implementation, the fix belongs in that implementation - not in the test.
