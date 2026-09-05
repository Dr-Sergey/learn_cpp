# Design logging and telemetry architecture for production C++ systems

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Logging and telemetry** are the eyes and ears of production systems. Logging records discrete events (errors, state changes); telemetry collects continuous metrics (latency, throughput, resource usage). A good architecture makes both low-overhead, structured, and configurable without code changes. In C++ the challenge is achieving this with minimal performance impact - the last thing you want is your observability layer becoming a bottleneck.

### Logging vs Metrics vs Tracing

Think of these three as complementary tools that answer different questions. Logs tell you *what happened*, metrics tell you *how much* and *how often*, and traces tell you *where time went* across a distributed request.

| Type | Data | Cardinality | Overhead | Use Case |
| --- | --- | --- | --- | --- |
| **Logging** | Text/structured events | High | Medium | Debugging, audit |
| **Metrics** | Numeric counters/gauges | Low | Very low | Dashboards, alerts |
| **Tracing** | Request flow across services | Medium | Medium | Distributed debugging |

---

## Self-Assessment

### Q1: Design a structured logging architecture

Structured logging means attaching key-value fields to every log entry instead of baking everything into a free-text message. That way, a log aggregation system (like Elasticsearch or Splunk) can query `level=ERROR AND service=payments` instead of trying to grep through strings. The architecture below shows a `Logger` that routes `LogEntry` objects to one or more sinks - a console sink for humans, a JSON sink for machines.

Notice the `should_log()` fast path: we check the log level *before* constructing the message string. String formatting is surprisingly expensive, and you'll call these macros thousands of times per second in production.

**Answer:**

```cpp
#include <string>
#include <chrono>
#include <unordered_map>
#include <functional>
#include <vector>
#include <mutex>
#include <sstream>
#include <iostream>

// === Log levels ===
enum class LogLevel : int {
    Trace = 0, Debug = 1, Info = 2, Warn = 3, Error = 4, Fatal = 5
};

// === Structured log entry ===
struct LogEntry {
    LogLevel level;
    std::string message;
    std::string logger_name;
    std::chrono::system_clock::time_point timestamp;
    std::unordered_map<std::string, std::string> fields;  // Structured data
    std::string file;
    int line;
    std::thread::id thread_id;
};

// === Log sink interface ===
class ILogSink {
public:
    virtual ~ILogSink() = default;
    virtual void write(const LogEntry& entry) = 0;
    virtual void flush() = 0;
};

// === Console sink (human-readable) ===
class ConsoleSink : public ILogSink {
public:
    void write(const LogEntry& entry) override {
        std::lock_guard lock(mutex_);
        auto t = std::chrono::system_clock::to_time_t(entry.timestamp);
        std::cerr << std::put_time(std::localtime(&t), "%T")
                  << " [" << to_string(entry.level) << "] "
                  << entry.message;
        for (const auto& [k, v] : entry.fields)
            std::cerr << " " << k << "=" << v;
        std::cerr << "\n";
    }
    void flush() override { std::cerr.flush(); }
private:
    std::mutex mutex_;
    static const char* to_string(LogLevel l) {
        constexpr const char* names[] = {
            "TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"
        };
        return names[static_cast<int>(l)];
    }
};

// === JSON sink (machine-readable, for log aggregation) ===
class JsonFileSink : public ILogSink {
public:
    explicit JsonFileSink(const std::string& path)
        : file_(path, std::ios::app) {}

    void write(const LogEntry& entry) override {
        std::lock_guard lock(mutex_);
        file_ << R"({"level":")"
              << static_cast<int>(entry.level)
              << R"(","msg":")"
              << escape(entry.message) << '"';
        for (const auto& [k, v] : entry.fields)
            file_ << ",\"" << k << "\":\"" << escape(v) << '"';
        file_ << "}\n";
    }
    void flush() override { file_.flush(); }
private:
    std::string escape(const std::string& s) { /* JSON escape */ return s; }
    std::ofstream file_;
    std::mutex mutex_;
};

// === Logger ===
class Logger {
public:
    explicit Logger(std::string name) : name_(std::move(name)) {}

    void add_sink(std::shared_ptr<ILogSink> sink) {
        sinks_.push_back(std::move(sink));
    }

    void set_level(LogLevel level) { min_level_ = level; }

    // Fast path: check level before constructing message
    bool should_log(LogLevel level) const {
        return level >= min_level_;
    }

    void log(LogLevel level, const std::string& msg,
             std::unordered_map<std::string, std::string> fields = {},
             const char* file = "", int line = 0) {
        if (!should_log(level)) return;

        LogEntry entry{
            level, msg, name_,
            std::chrono::system_clock::now(),
            std::move(fields), file, line,
            std::this_thread::get_id()
        };

        for (auto& sink : sinks_)
            sink->write(entry);
    }

private:
    std::string name_;
    std::vector<std::shared_ptr<ILogSink>> sinks_;
    LogLevel min_level_ = LogLevel::Info;
};

// === Convenience macros (capture file/line) ===
#define LOG_INFO(logger, msg, ...) \
    if ((logger).should_log(LogLevel::Info)) \
        (logger).log(LogLevel::Info, msg, ##__VA_ARGS__, __FILE__, __LINE__)

#define LOG_ERROR(logger, msg, ...) \
    if ((logger).should_log(LogLevel::Error)) \
        (logger).log(LogLevel::Error, msg, ##__VA_ARGS__, __FILE__, __LINE__)
```

The macros capture `__FILE__` and `__LINE__` automatically, which saves you from threading that context through every call site. The `if (should_log(...))` guard in the macro body is the key optimization - the entire message construction is skipped when the level is filtered out.

### Q2: Implement a metrics collection system

Metrics are fundamentally different from logs: you want counters and gauges that are updated thousands of times per second with near-zero overhead. The trick is using `std::atomic` with `memory_order_relaxed` on the hot path - we only need eventual consistency here, not strict ordering. The `Histogram` type lets you track latency distributions (how many requests took 0-5ms, 5-10ms, etc.) rather than just averages, which is far more useful in production.

The `ScopedTimer` wraps RAII around latency measurement - it starts a clock on construction and records the elapsed time to a `Histogram` on destruction, so you can't accidentally forget to record the measurement even if the function throws.

**Answer:**

```cpp
#include <atomic>
#include <chrono>

// === Metric types ===
class Counter {
public:
    void increment(int64_t n = 1) {
        value_.fetch_add(n, std::memory_order_relaxed);
    }
    int64_t value() const {
        return value_.load(std::memory_order_relaxed);
    }
private:
    std::atomic<int64_t> value_{0};
};

class Gauge {
public:
    void set(double v) {
        value_.store(v, std::memory_order_relaxed);
    }
    void increment(double n = 1.0) {
        auto old = value_.load();
        while (!value_.compare_exchange_weak(old, old + n)) {}
    }
    double value() const {
        return value_.load(std::memory_order_relaxed);
    }
private:
    std::atomic<double> value_{0.0};
};

class Histogram {
public:
    explicit Histogram(std::vector<double> bucket_bounds)
        : bounds_(std::move(bucket_bounds)),
          buckets_(bounds_.size() + 1) {}

    void observe(double value) {
        count_.fetch_add(1, std::memory_order_relaxed);
        for (size_t i = 0; i < bounds_.size(); ++i) {
            if (value <= bounds_[i]) {
                buckets_[i].fetch_add(1, std::memory_order_relaxed);
                return;
            }
        }
        buckets_.back().fetch_add(1, std::memory_order_relaxed);
    }

private:
    std::vector<double> bounds_;
    std::vector<std::atomic<int64_t>> buckets_;
    std::atomic<int64_t> count_{0};
};

// === Scoped timer for latency measurement ===
class ScopedTimer {
public:
    ScopedTimer(Histogram& histogram)
        : histogram_(histogram),
          start_(std::chrono::steady_clock::now()) {}

    ~ScopedTimer() {
        auto elapsed = std::chrono::steady_clock::now() - start_;
        auto ms = std::chrono::duration<double, std::milli>(elapsed).count();
        histogram_.observe(ms);
    }

private:
    Histogram& histogram_;
    std::chrono::steady_clock::time_point start_;
};

// === Metrics registry ===
class MetricsRegistry {
public:
    Counter& counter(const std::string& name) {
        std::lock_guard lock(mutex_);
        return counters_[name];
    }
    Gauge& gauge(const std::string& name) {
        std::lock_guard lock(mutex_);
        return gauges_[name];
    }

    // Export for Prometheus / monitoring system
    std::string export_prometheus() const {
        std::ostringstream out;
        std::lock_guard lock(mutex_);
        for (const auto& [name, c] : counters_)
            out << name << " " << c.value() << "\n";
        for (const auto& [name, g] : gauges_)
            out << name << " " << g.value() << "\n";
        return out.str();
    }

private:
    std::unordered_map<std::string, Counter> counters_;
    std::unordered_map<std::string, Gauge> gauges_;
    mutable std::mutex mutex_;
};

// === Usage ===
MetricsRegistry& metrics() {
    static MetricsRegistry instance;
    return instance;
}

void handle_request() {
    metrics().counter("http_requests_total").increment();
    static Histogram latency({1, 5, 10, 25, 50, 100, 250, 500, 1000});
    ScopedTimer timer(latency);
    // ... process request ...
}
```

The `export_prometheus()` method produces text in the Prometheus exposition format, which tools like Grafana can scrape directly. The registry lookup uses a mutex, but that only happens on first access - `handle_request()` holds a `static Histogram` reference so subsequent calls go straight to the atomic update, no locking needed.

### Q3: Implement distributed tracing with context propagation

Distributed tracing answers the question: "When a user request came in, what did it actually do, and how long did each step take?" A `TraceContext` carries a `trace_id` (unique per request, same across all services) and a `span_id` (unique per operation within that request). Child spans inherit the parent's `trace_id` so everything stitches together in a trace viewer.

The clever part is using a thread-local pointer to propagate context automatically - functions don't need to accept a context parameter explicitly, and the `Span` RAII object saves and restores the previous context on destruction so nested spans work naturally.

**Answer:**

```cpp
#include <random>

// === Trace context ===
struct TraceContext {
    uint64_t trace_id;
    uint64_t span_id;
    uint64_t parent_span_id;

    static TraceContext create() {
        static thread_local std::mt19937_64 rng(std::random_device{}());
        return {rng(), rng(), 0};
    }

    TraceContext child() const {
        static thread_local std::mt19937_64 rng(std::random_device{}());
        return {trace_id, rng(), span_id};
    }

    std::string to_header() const {
        return std::to_string(trace_id) + "-" + std::to_string(span_id);
    }
};

// === Thread-local context propagation ===
thread_local TraceContext* current_trace = nullptr;

class Span {
public:
    Span(const std::string& name, TraceContext ctx)
        : name_(name), ctx_(ctx),
          start_(std::chrono::steady_clock::now()),
          previous_(current_trace) {
        current_trace = &ctx_;  // Set as current
    }

    ~Span() {
        auto elapsed = std::chrono::steady_clock::now() - start_;
        auto us = std::chrono::duration_cast<
            std::chrono::microseconds>(elapsed).count();

        // Report span to tracing backend
        report_span(name_, ctx_, us);

        current_trace = previous_;  // Restore previous
    }

    TraceContext context() const { return ctx_; }

private:
    std::string name_;
    TraceContext ctx_;
    std::chrono::steady_clock::time_point start_;
    TraceContext* previous_;
};

// === Usage ===
void handle_http_request() {
    auto ctx = TraceContext::create();
    Span root("handle_request", ctx);

    {
        Span db_span("query_database", ctx.child());
        // ... database query ...
    }

    {
        Span cache_span("check_cache", ctx.child());
        // ... cache lookup ...
    }
}
```

Each `Span` creation calls `ctx.child()` to produce a new span ID that knows its parent - this is what lets a trace viewer reconstruct the call tree. When the `Span` destructor fires, it reports the duration and restores whatever context was active before, so you can safely nest spans arbitrarily deep.

---

## Notes

- **Check log level before constructing the message** (`should_log()`) - avoids expensive string formatting.
- Use **structured logging** (key=value fields) for machine parsing, not just free-text messages.
- Metrics should use **atomic operations** and be lock-free on the hot path.
- `ScopedTimer` is the easiest way to measure latency - RAII guarantees measurement even on exceptions.
- **Thread-local** trace context avoids passing context through every function signature.
- Production logging libraries: spdlog (fast, header-only), Boost.Log, Google glog.
- Export metrics in Prometheus format for easy integration with Grafana dashboards.
