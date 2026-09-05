# Use the Outbox pattern for reliable event publishing in C++ systems

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

The **Outbox pattern** solves a subtle but critical problem called the **dual-write problem**. Here's the scenario: your service updates its database (say, marking an order as created) and then publishes an event to a message broker (so other services can react). These are two separate operations. If the database write succeeds but the event publish fails, your data says the order exists but no other service knows about it. If the publish succeeds but the database write fails, other services react to an order that doesn't actually exist.

The Outbox pattern resolves this by never publishing events directly at all. Instead, you write the event into an "outbox" table in the same database transaction as your business data. A separate relay process then reads the outbox and forwards events to the message broker. Since the outbox write is in the same transaction as the business data, they either both succeed or both fail - atomically. The relay provides at-least-once delivery from there.

### Dual-Write Problem

| Scenario | DB Write | Event Publish | Result |
| --- | --- | --- | --- |
| Happy path | Yes | Yes | Consistent |
| Event fails | Yes | No | DB updated, no event - **inconsistent** |
| DB fails | No | Yes | Event sent, no data - **inconsistent** |
| Both fail | No | No | Consistent (nothing happened) |
| **Outbox** | Yes (data + event in one txn) | Async relay | **Always consistent** |

---

## Self-Assessment

### Q1: Implement the outbox pattern with a database transaction

**Answer:**

The central idea to watch for here is that both the business data INSERT and the outbox INSERT happen inside the same `txn`. They commit together or roll back together. That's what gives you atomicity.

```cpp
#include <string>
#include <vector>
#include <chrono>
#include <optional>

// === Outbox entry ===
struct OutboxEntry {
    int64_t id;
    std::string aggregate_type;  // e.g., "Order"
    std::string aggregate_id;    // e.g., order ID
    std::string event_type;      // e.g., "OrderCreated"
    std::string payload;         // JSON serialized event
    std::chrono::system_clock::time_point created_at;
    bool published = false;
};

// === Service writes business data + outbox in one transaction ===
class OrderService {
public:
    OrderService(Database& db) : db_(db) {}

    std::string create_order(const OrderRequest& req) {
        auto txn = db_.begin_transaction();

        try {
            // 1. Write business data
            Order order;
            order.id = generate_id();
            order.customer_id = req.customer_id;
            order.total = req.total;
            order.status = "pending";

            txn.execute(
                "INSERT INTO orders (id, customer_id, total, status) "
                "VALUES (?, ?, ?, ?)",
                order.id, order.customer_id, order.total, order.status);

            // 2. Write event to outbox (SAME transaction)
            std::string payload = serialize_event(
                OrderCreatedEvent{order.id, order.customer_id, order.total});

            txn.execute(
                "INSERT INTO outbox "
                "(aggregate_type, aggregate_id, event_type, payload) "
                "VALUES (?, ?, ?, ?)",
                "Order", order.id, "OrderCreated", payload);

            txn.commit();  // Both succeed or both fail
            return order.id;

        } catch (...) {
            txn.rollback();  // Both rolled back
            throw;
        }
    }

    void cancel_order(const std::string& order_id) {
        auto txn = db_.begin_transaction();

        txn.execute("UPDATE orders SET status = 'cancelled' WHERE id = ?",
                    order_id);

        txn.execute(
            "INSERT INTO outbox "
            "(aggregate_type, aggregate_id, event_type, payload) "
            "VALUES (?, ?, ?, ?)",
            "Order", order_id, "OrderCancelled",
            serialize_event(OrderCancelledEvent{order_id}));

        txn.commit();
    }

private:
    Database& db_;
};
```

After `txn.commit()` succeeds, you have guaranteed that both the order row and the outbox row exist in the database. Even if the process crashes immediately afterward, the outbox row survives and will be picked up on the next restart.

### Q2: Implement the outbox relay (publisher)

**Answer:**

The relay is a background process that polls the outbox for unpublished entries, forwards them to the message broker, and then marks them as published. The `FOR UPDATE SKIP LOCKED` clause in the query is important if you run multiple relay instances - it prevents two relays from picking up the same row simultaneously.

```cpp
// === Outbox relay: reads outbox and publishes to message broker ===
class OutboxRelay {
public:
    OutboxRelay(Database& db, MessageBroker& broker,
                std::chrono::milliseconds poll_interval = std::chrono::seconds(1))
        : db_(db), broker_(broker), poll_interval_(poll_interval) {}

    void start() {
        running_ = true;
        thread_ = std::jthread([this](std::stop_token st) {
            while (!st.stop_requested()) {
                process_batch();
                std::this_thread::sleep_for(poll_interval_);
            }
        });
    }

    void stop() {
        running_ = false;
        if (thread_.joinable()) thread_.request_stop();
    }

private:
    void process_batch() {
        // Fetch unpublished entries (with row locking)
        auto entries = db_.query<OutboxEntry>(
            "SELECT id, aggregate_type, aggregate_id, event_type, payload "
            "FROM outbox WHERE published = false "
            "ORDER BY id ASC LIMIT 100 FOR UPDATE SKIP LOCKED");

        for (const auto& entry : entries) {
            try {
                // Publish to message broker
                broker_.publish(
                    /*topic=*/entry.aggregate_type + "." + entry.event_type,
                    /*key=*/entry.aggregate_id,
                    /*payload=*/entry.payload,
                    /*headers=*/{{"event_id", std::to_string(entry.id)}}
                );

                // Mark as published
                db_.execute(
                    "UPDATE outbox SET published = true WHERE id = ?",
                    entry.id);

            } catch (const std::exception& e) {
                // Will retry on next poll (at-least-once)
                std::cerr << "Publish failed for outbox entry "
                          << entry.id << ": " << e.what() << "\n";
                break;  // Preserve ordering
            }
        }
    }

    Database& db_;
    MessageBroker& broker_;
    std::chrono::milliseconds poll_interval_;
    std::atomic<bool> running_{false};
    std::jthread thread_;
};

// === Outbox cleanup: purge old published entries ===
void cleanup_outbox(Database& db, std::chrono::hours retention) {
    auto cutoff = std::chrono::system_clock::now() - retention;
    db.execute(
        "DELETE FROM outbox WHERE published = true AND created_at < ?",
        cutoff);
}
```

Notice the `break` on publish failure. This is deliberate - it preserves event ordering. If entry 5 fails to publish, you don't skip it and publish entry 6, because downstream consumers might depend on receiving events in order. You stop the batch and retry from entry 5 on the next poll cycle.

### Q3: Idempotent consumer to handle duplicate events

**Answer:**

Because the relay provides at-least-once delivery (it might retry an entry that was actually published but not yet marked as such), consumers can receive the same event more than once. An idempotent consumer handles this gracefully by recording which event IDs it has already processed and skipping duplicates.

The key - and this is the same transactional trick as the producer side - is that recording "I processed this event" happens in the same transaction as the actual processing. That way, if the processing succeeds but the "mark as done" write fails, the event will be retried and the processing will be deduplicated correctly.

```cpp
// === Idempotent consumer: deduplicates at-least-once delivery ===
class IdempotentConsumer {
public:
    IdempotentConsumer(Database& db) : db_(db) {}

    // Returns true if this event was already processed
    bool is_duplicate(const std::string& event_id) {
        auto result = db_.query_scalar<int>(
            "SELECT COUNT(*) FROM processed_events WHERE event_id = ?",
            event_id);
        return result > 0;
    }

    // Process idempotently: skip if already done
    template<typename Event, typename Handler>
    void handle(const std::string& event_id, const Event& event,
                Handler&& handler) {
        if (is_duplicate(event_id)) {
            std::cout << "Skipping duplicate event: " << event_id << "\n";
            return;
        }

        auto txn = db_.begin_transaction();
        try {
            // Process the event
            handler(event, txn);

            // Record that we processed it (same transaction)
            txn.execute(
                "INSERT INTO processed_events (event_id, processed_at) "
                "VALUES (?, NOW())",
                event_id);

            txn.commit();
        } catch (...) {
            txn.rollback();
            throw;
        }
    }

private:
    Database& db_;
};

// === Consumer setup ===
void setup_consumer(MessageBroker& broker, IdempotentConsumer& consumer) {
    broker.subscribe("Order.OrderCreated",
        [&consumer](const Message& msg) {
            auto event_id = msg.header("event_id");
            auto event = deserialize<OrderCreatedEvent>(msg.payload());

            consumer.handle(event_id, event,
                [](const OrderCreatedEvent& e, Transaction& txn) {
                    // Process: e.g., send welcome email, update analytics
                    txn.execute(
                        "INSERT INTO order_analytics "
                        "(order_id, customer_id, amount) VALUES (?, ?, ?)",
                        e.order_id, e.customer_id, e.total);
                });
        });
}

// SQL schema:
// CREATE TABLE outbox (
//     id BIGSERIAL PRIMARY KEY,
//     aggregate_type VARCHAR(100) NOT NULL,
//     aggregate_id VARCHAR(100) NOT NULL,
//     event_type VARCHAR(100) NOT NULL,
//     payload JSONB NOT NULL,
//     published BOOLEAN DEFAULT false,
//     created_at TIMESTAMP DEFAULT NOW()
// );
// CREATE INDEX idx_outbox_unpublished ON outbox(published, id)
//     WHERE published = false;
//
// CREATE TABLE processed_events (
//     event_id VARCHAR(100) PRIMARY KEY,
//     processed_at TIMESTAMP DEFAULT NOW()
// );
```

The partial index on the outbox table (`WHERE published = false`) is important in production. The index only covers unpublished rows, so it stays small and fast even as the table grows with millions of historical entries.

---

## Notes

- The outbox guarantees **atomicity** between DB write and event: both happen in one transaction, so they can never get out of sync.
- The relay provides **at-least-once delivery** - consumers must be idempotent to handle the (rare) case where the same event arrives twice.
- **Polling** is the simplest relay implementation; **CDC (Change Data Capture)** via tools like Debezium or WAL tailing is more efficient for high-throughput systems since it reacts to changes in real time instead of checking on a timer.
- `FOR UPDATE SKIP LOCKED` allows multiple relay instances to run in parallel without conflicts - each row is processed by exactly one relay instance.
- Clean up old published entries periodically (a retention of 7 days is a common default) to keep the outbox table from growing without bound.
- The outbox table doubles as a persistent event log, which is useful for replaying events during debugging or after a consumer failure.
- An alternative for PostgreSQL is to use `LISTEN/NOTIFY` instead of polling the relay, which reduces latency significantly.
