# Design event-driven architectures with message queues in C++

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Event-driven architecture (EDA)** decouples producers from consumers through asynchronous event dispatch. A component publishes an event without knowing who handles it. Subscribers react independently when they see an event they care about. The result is loose coupling: you can add a new subscriber without touching the code that publishes the event.

Combined with **message queues**, EDA also gives you buffering and back-pressure. A producer can keep pushing events even if the consumer is temporarily slow - the queue absorbs the difference. In C++ this spans a wide range: in-process event buses using `std::function` and `std::any`, cross-thread queues using mutexes and condition variables, and inter-process queues backed by ZeroMQ, RabbitMQ, or Kafka.

### Event-Driven vs Request-Response

| Aspect | Request-Response | Event-Driven |
| --- | --- | --- |
| Coupling | Caller knows callee | Publisher blind to consumers |
| Flow | Synchronous | Asynchronous |
| Scaling | Bounded by callee | Independent scaling |
| Ordering | Request order | Event ordering guarantees needed |
| Failure | Cascading | Isolated (queue buffers) |

---

## Self-Assessment

### Q1: Implement a type-safe in-process event bus

**Answer:**

The `EventBus` uses `std::type_index` as the key and `std::any` to store typed event values behind a type-erased handler. The template subscribe and publish methods recover the type at the boundaries. Because the handlers are called outside the lock, a subscriber that publishes another event does not deadlock.

```cpp
#include <functional>
#include <unordered_map>
#include <typeindex>
#include <vector>
#include <memory>
#include <any>
#include <mutex>

// === Type-safe Event Bus ===
class EventBus {
public:
    // Subscribe to events of type E
    template<typename E>
    size_t subscribe(std::function<void(const E&)> handler) {
        std::lock_guard lock(mutex_);
        auto id = next_id_++;
        auto wrapper = [handler](const std::any& e) {
            handler(std::any_cast<const E&>(e));
        };
        handlers_[typeid(E)].push_back({id, std::move(wrapper)});
        return id;
    }

    // Unsubscribe by ID
    void unsubscribe(size_t id) {
        std::lock_guard lock(mutex_);
        for (auto& [type, entries] : handlers_) {
            entries.erase(
                std::remove_if(entries.begin(), entries.end(),
                    [id](const Entry& e) { return e.id == id; }),
                entries.end());
        }
    }

    // Publish event to all subscribers
    template<typename E>
    void publish(const E& event) {
        std::vector<std::function<void(const std::any&)>> to_call;
        {
            std::lock_guard lock(mutex_);
            auto it = handlers_.find(typeid(E));
            if (it != handlers_.end())
                for (auto& entry : it->second)
                    to_call.push_back(entry.handler);
        }
        // Call outside lock to avoid deadlocks
        for (auto& fn : to_call)
            fn(std::any(event));
    }

private:
    struct Entry {
        size_t id;
        std::function<void(const std::any&)> handler;
    };
    std::unordered_map<std::type_index, std::vector<Entry>> handlers_;
    std::mutex mutex_;
    size_t next_id_ = 1;
};

// === Events ===
struct OrderPlaced {
    int order_id;
    double total;
    std::string customer;
};

struct PaymentProcessed {
    int order_id;
    bool success;
};

struct OrderShipped {
    int order_id;
    std::string tracking;
};

// === Subscribing ===
void setup(EventBus& bus) {
    // Payment service reacts to orders
    bus.subscribe<OrderPlaced>([&bus](const OrderPlaced& e) {
        bool ok = process_payment(e.total);
        bus.publish(PaymentProcessed{e.order_id, ok});
    });

    // Shipping reacts to successful payments
    bus.subscribe<PaymentProcessed>([&bus](const PaymentProcessed& e) {
        if (e.success) {
            auto tracking = ship_order(e.order_id);
            bus.publish(OrderShipped{e.order_id, tracking});
        }
    });

    // Notification reacts to shipping
    bus.subscribe<OrderShipped>([](const OrderShipped& e) {
        send_email("Your order " + std::to_string(e.order_id)
                   + " shipped: " + e.tracking);
    });
}
```

Notice how the payment handler publishes `PaymentProcessed` after processing, and the shipping handler in turn publishes `OrderShipped`. Each service only knows about the events it cares about - no service calls another service directly. This chain of reactions is the characteristic pattern of event-driven systems.

### Q2: Add an async message queue with worker threads

**Answer:**

The synchronous event bus dispatches events on the caller's thread. For anything heavier - or when you need to decouple production rate from consumption rate - you want a queue and a worker pool. The `MessageQueue` below is thread-safe with a condition variable so workers sleep rather than spin. `AsyncEventProcessor` wraps the queue and gives you configurable concurrency.

```cpp
#include <queue>
#include <thread>
#include <condition_variable>
#include <atomic>

// === Thread-safe message queue ===
template<typename T>
class MessageQueue {
public:
    void push(T msg) {
        {
            std::lock_guard lock(mutex_);
            queue_.push(std::move(msg));
        }
        cv_.notify_one();
    }

    T pop() {
        std::unique_lock lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty() || stopped_; });
        if (stopped_ && queue_.empty())
            throw std::runtime_error("Queue stopped");
        T msg = std::move(queue_.front());
        queue_.pop();
        return msg;
    }

    bool try_pop(T& msg, std::chrono::milliseconds timeout) {
        std::unique_lock lock(mutex_);
        if (!cv_.wait_for(lock, timeout,
                [this] { return !queue_.empty() || stopped_; }))
            return false;
        if (stopped_ && queue_.empty()) return false;
        msg = std::move(queue_.front());
        queue_.pop();
        return true;
    }

    void stop() {
        {
            std::lock_guard lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }

    size_t size() const {
        std::lock_guard lock(mutex_);
        return queue_.size();
    }

private:
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;
};

// === Async Event Processor with worker pool ===
class AsyncEventProcessor {
public:
    using Task = std::function<void()>;

    explicit AsyncEventProcessor(int workers = 4)
        : workers_(workers) {
        for (int i = 0; i < workers; ++i) {
            threads_.emplace_back([this] { worker_loop(); });
        }
    }

    template<typename E>
    void enqueue(E event, std::function<void(const E&)> handler) {
        queue_.push([event = std::move(event),
                     handler = std::move(handler)]() {
            handler(event);
        });
    }

    ~AsyncEventProcessor() {
        queue_.stop();
        for (auto& t : threads_) t.join();
    }

private:
    void worker_loop() {
        while (true) {
            try {
                auto task = queue_.pop();
                task();
            } catch (...) {
                break;  // Queue stopped
            }
        }
    }

    MessageQueue<Task> queue_;
    std::vector<std::jthread> threads_;
    int workers_;
};
```

The `stop()` and join pattern in the destructor is important: when the processor is destroyed, it signals the queue to stop, which wakes all waiting workers. Each worker catches the exception from `pop()` and exits cleanly. If you skip this and just destroy the threads, you get undefined behavior.

### Q3: Implement event sourcing with replay

**Answer:**

Event sourcing is an architectural pattern where you store every event that ever happened, rather than storing only the current state. The current state is computed by replaying events from the beginning. This gives you a complete audit trail and the ability to rebuild state from scratch if something goes wrong. The reason this trips people up is that it requires a mental shift: the event log is the source of truth, and the current state is just a derived view of it.

```cpp
// === Event Store: persist all events for replay ===
class EventStore {
public:
    struct StoredEvent {
        size_t sequence;
        std::string type;
        std::any data;
        std::chrono::system_clock::time_point timestamp;
    };

    template<typename E>
    void append(const E& event) {
        events_.push_back({
            next_seq_++,
            typeid(E).name(),
            std::any(event),
            std::chrono::system_clock::now()
        });
    }

    // Replay all events through handler
    template<typename E>
    void replay(std::function<void(const E&)> handler) const {
        for (const auto& evt : events_) {
            if (evt.type == typeid(E).name()) {
                handler(std::any_cast<const E&>(evt.data));
            }
        }
    }

    // Replay from a specific sequence number
    std::vector<StoredEvent> since(size_t seq) const {
        std::vector<StoredEvent> result;
        for (const auto& evt : events_)
            if (evt.sequence > seq)
                result.push_back(evt);
        return result;
    }

    size_t latest_sequence() const {
        return events_.empty() ? 0 : events_.back().sequence;
    }

private:
    std::vector<StoredEvent> events_;
    size_t next_seq_ = 1;
};

// === Rebuild state from events ===
class OrderAggregate {
public:
    void apply(const OrderPlaced& e) {
        id_ = e.order_id;
        total_ = e.total;
        status_ = "placed";
    }
    void apply(const PaymentProcessed& e) {
        status_ = e.success ? "paid" : "payment_failed";
    }
    void apply(const OrderShipped& e) {
        tracking_ = e.tracking;
        status_ = "shipped";
    }

    // Rebuild from event store:
    // EventStore store;
    // OrderAggregate order;
    // store.replay<OrderPlaced>([&](auto& e) { order.apply(e); });
    // store.replay<PaymentProcessed>([&](auto& e) { order.apply(e); });

    std::string status() const { return status_; }

private:
    int id_ = 0;
    double total_ = 0;
    std::string status_;
    std::string tracking_;
};
```

The `since(seq)` method enables incremental replay - if you have a checkpoint at sequence 500 and want to bring it up to date, you only replay events from 501 onward. That is essential for production event stores where replaying from the beginning would be too slow.

---

## Notes

- Use an in-process **event bus** for fast, same-process communication between components.
- Use a **message queue** when you need cross-thread or cross-process communication with buffering and back-pressure.
- Always copy the subscriber list before invoking handlers - a handler might subscribe or unsubscribe during dispatch, which would invalidate the iteration.
- Event ordering matters in multi-producer scenarios: use sequence numbers and enforce a single-writer per aggregate.
- Event sourcing gives you a complete audit trail and state rebuild capability, but it increases storage requirements.
- For high throughput, consider lock-free SPSC (single-producer, single-consumer) queues to eliminate mutex contention entirely.
- Real-world queue choices: ZeroMQ for in-process/IPC/TCP, Redis Streams for lightweight distributed queues, Apache Kafka for high-throughput distributed event streaming.
