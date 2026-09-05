# Build observable systems - metrics, tracing, and health checks in C++

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Observability** is the ability to understand a system's internal state from its external outputs. The three pillars are **metrics** (numeric measurements over time), **tracing** (request flow through components), and **logs** (discrete events). **Health checks** report component readiness. Together they enable proactive monitoring, debugging production issues, and capacity planning.

The intuition behind why you need all three: metrics tell you *something is wrong* ("error rate spiked at 14:32"), tracing tells you *where it is wrong* ("the payment service is the bottleneck"), and logs tell you *what exactly happened* ("transaction id X failed because of Y"). Each pillar answers a different question, and you need all of them to be effective.

### Three Pillars of Observability

| Pillar | Answers | Example | Format |
| --- | --- | --- | --- |
| **Metrics** | "How much?" | Request rate, error %, latency P99 | Prometheus/StatsD |
| **Tracing** | "What path?" | Request through 5 services | OpenTelemetry/Jaeger |
| **Logging** | "What happened?" | Error details, state changes | JSON/structured |
| **Health** | "Is it working?" | Ready/live/startup probes | HTTP /health |

---

## Self-Assessment

### Q1: Implement health check endpoints

**Answer:**

A health check system has three parts: a result type (healthy/degraded/unhealthy), individual check functions that test specific dependencies, and an aggregator that runs all checks and rolls up an overall status. The JSON output is what Kubernetes and load balancers will consume:

```cpp
#include <string>
#include <vector>
#include <functional>
#include <chrono>

// === Health check result ===
enum class HealthStatus { Healthy, Degraded, Unhealthy };

struct HealthCheckResult {
    std::string name;
    HealthStatus status;
    std::string message;
    std::chrono::milliseconds duration;
};

struct SystemHealth {
    HealthStatus overall;
    std::vector<HealthCheckResult> checks;
    std::chrono::system_clock::time_point timestamp;

    std::string to_json() const {
        std::string json = R"({"status":")"

            + status_string(overall) + R"(","checks":[)";

        for (size_t i = 0; i < checks.size(); ++i) {
            if (i > 0) json += ",";
            json += R"({"name":")"

                + checks[i].name + R"(","status":")"
                + status_string(checks[i].status)
                + R"(","message":")"
                + checks[i].message
                + R"(","duration_ms":)"
                + std::to_string(checks[i].duration.count()) + "}";

        }
        json += "]}";
        return json;
    }

private:
    static std::string status_string(HealthStatus s) {
        switch (s) {
            case HealthStatus::Healthy: return "healthy";
            case HealthStatus::Degraded: return "degraded";
            case HealthStatus::Unhealthy: return "unhealthy";
        }
        return "unknown";
    }
};

// === Health checker ===
class HealthChecker {
public:
    using Check = std::function<HealthCheckResult()>;

    void add_check(std::string name, Check check) {
        checks_.push_back({std::move(name), std::move(check)});
    }

    SystemHealth check_all() const {
        SystemHealth health;
        health.timestamp = std::chrono::system_clock::now();
        health.overall = HealthStatus::Healthy;

        for (const auto& [name, check_fn] : checks_) {
            auto start = std::chrono::steady_clock::now();
            auto result = check_fn();
            result.duration = std::chrono::duration_cast<
                std::chrono::milliseconds>(
                    std::chrono::steady_clock::now() - start);
            result.name = name;

            if (result.status == HealthStatus::Unhealthy)
                health.overall = HealthStatus::Unhealthy;
            else if (result.status == HealthStatus::Degraded &&
                     health.overall == HealthStatus::Healthy)
                health.overall = HealthStatus::Degraded;

            health.checks.push_back(std::move(result));
        }
        return health;
    }

private:
    struct Entry {
        std::string name;
        Check check;
    };
    std::vector<Entry> checks_;
};

// === Register health checks ===
void setup_health(HealthChecker& hc, Database& db, Cache& cache) {
    hc.add_check("database", [&db]() -> HealthCheckResult {
        try {
            db.execute("SELECT 1");
            return {"", HealthStatus::Healthy, "Connected", {}};
        } catch (const std::exception& e) {
            return {"", HealthStatus::Unhealthy, e.what(), {}};
        }
    });

    hc.add_check("cache", [&cache]() -> HealthCheckResult {
        if (cache.ping())
            return {"", HealthStatus::Healthy, "OK", {}};
        return {"", HealthStatus::Degraded, "Cache unreachable", {}};
    });

    hc.add_check("disk_space", []() -> HealthCheckResult {
        auto free = get_free_disk_space("/");
        if (free < 100'000'000)  // <100MB
            return {"", HealthStatus::Unhealthy, "Low disk", {}};
        if (free < 1'000'000'000)  // <1GB
            return {"", HealthStatus::Degraded, "Disk space low", {}};
        return {"", HealthStatus::Healthy, "OK", {}};
    });
}

// Expose via HTTP:
// GET /health -> health_checker.check_all().to_json()
// HTTP 200 = healthy, 503 = unhealthy
```

The `Degraded` status is the interesting middle ground. A degraded service is still working but something is wrong (cache is down, disk is getting full). Load balancers can still route traffic to it, but you know to investigate. `Unhealthy` means stop routing traffic immediately.

### Q2: Build an integrated observability facade

**Answer:**

Rather than scattering metric increments and log calls throughout your codebase, a unified `Observability` facade gives you one object to pass around. It automatically attaches the current trace context to log messages, which is what lets you correlate a log entry to the exact distributed trace that generated it:

```cpp
// === Unified observability interface ===
class Observability {
public:
    // Metrics
    void counter_inc(const std::string& name, int64_t n = 1) {
        metrics_.counter(name).increment(n);
    }
    void gauge_set(const std::string& name, double value) {
        metrics_.gauge(name).set(value);
    }
    void histogram_observe(const std::string& name, double value) {
        get_histogram(name).observe(value);
    }

    // Tracing
    auto start_span(const std::string& name) {
        auto ctx = current_trace
            ? current_trace->child()
            : TraceContext::create();
        return Span(name, ctx);
    }

    // Logging (structured)
    void log_info(const std::string& msg,
                   std::unordered_map<std::string, std::string> fields = {}) {
        // Auto-attach trace context
        if (current_trace) {
            fields["trace_id"] = std::to_string(current_trace->trace_id);
            fields["span_id"] = std::to_string(current_trace->span_id);
        }
        logger_.log(LogLevel::Info, msg, std::move(fields));
    }

    // Health
    SystemHealth health() { return health_.check_all(); }

    // Export all metrics (for /metrics endpoint)
    std::string export_metrics() { return metrics_.export_prometheus(); }

private:
    MetricsRegistry metrics_;
    Logger logger_{"app"};
    HealthChecker health_;

    Histogram& get_histogram(const std::string& name) {
        static std::unordered_map<std::string, Histogram> histograms;
        auto it = histograms.find(name);
        if (it == histograms.end()) {
            it = histograms.emplace(name,
                Histogram({1, 5, 10, 25, 50, 100, 250, 500, 1000})).first;
        }
        return it->second;
    }
};

// === Usage in request handler ===
void handle(Observability& obs, const Request& req) {
    obs.counter_inc("requests_total");
    auto span = obs.start_span("handle_request");

    auto start = std::chrono::steady_clock::now();
    try {
        auto result = process(req);
        obs.log_info("Request processed",
            {{"path", req.path}, {"status", "200"}});
    } catch (const std::exception& e) {
        obs.counter_inc("errors_total");
        obs.log_info("Request failed",
            {{"path", req.path}, {"error", e.what()}});
    }

    auto ms = std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();
    obs.histogram_observe("request_duration_ms", ms);
}
```

The histogram bucket sizes (1, 5, 10, 25, ... ms) are chosen to align with common SLA thresholds. When you query P99 latency in Grafana, those buckets determine resolution. Plan them up front.

### Q3: Kubernetes-ready liveness and readiness probes

**Answer:**

Kubernetes uses three distinct probe types, and understanding the difference matters: a liveness failure causes a pod restart; a readiness failure removes the pod from the load balancer but does not restart it; a startup failure prevents liveness/readiness checks from running until startup completes. Here is how to implement all three:

```cpp
// === Probe types for container orchestration ===
class ProbeManager {
public:
    enum class ProbeType { Liveness, Readiness, Startup };

    // Liveness: is the process alive? (restart if false)
    bool is_live() const {
        return !deadlocked_ && !fatal_error_;
    }

    // Readiness: can the process accept traffic? (remove from LB if false)
    bool is_ready() const {
        return is_live() && db_connected_ && cache_connected_
            && initialized_;
    }

    // Startup: has initial setup completed? (don't check liveness yet)
    bool is_started() const {
        return initialized_;
    }

    // Called by subsystems
    void set_initialized() { initialized_ = true; }
    void set_db_connected(bool v) { db_connected_ = v; }
    void set_cache_connected(bool v) { cache_connected_ = v; }
    void set_deadlocked() { deadlocked_ = true; }
    void set_fatal_error() { fatal_error_ = true; }

    // HTTP handler registration
    void register_routes(HttpServer& server) {
        server.get("/health/live", [this](auto&, auto& res) {
            res.status = is_live() ? 200 : 503;
            res.body = is_live() ? R"({"status":"live"})" :
                                   R"({"status":"dead"})";
        });
        server.get("/health/ready", [this](auto&, auto& res) {
            res.status = is_ready() ? 200 : 503;
            res.body = is_ready() ? R"({"status":"ready"})" :
                                    R"({"status":"not ready"})";
        });
        server.get("/health/startup", [this](auto&, auto& res) {
            res.status = is_started() ? 200 : 503;
        });
    }

private:
    std::atomic<bool> initialized_{false};
    std::atomic<bool> db_connected_{false};
    std::atomic<bool> cache_connected_{false};
    std::atomic<bool> deadlocked_{false};
    std::atomic<bool> fatal_error_{false};
};
```

The `atomic<bool>` flags are important here because subsystem threads will be calling `set_db_connected` and `set_cache_connected` concurrently with Kubernetes polling the `/health/ready` endpoint. The atomics make those concurrent reads and writes safe without needing a mutex on the hot path.

---

## Notes

- **Metrics are cheap** (atomic increments); use them liberally for counters, gauges, and histograms.
- **Trace IDs in logs** correlate log entries with distributed traces - this is essential for debugging production issues in microservices.
- Health checks must be **fast** (under 1 second): do not run expensive queries in health endpoints.
- Kubernetes probes: liveness (restart if dead), readiness (stop traffic if unready), startup (grace period for slow boot).
- Use spdlog or similar for structured logging; opentelemetry-cpp for traces and metrics.
- Export metrics in Prometheus exposition format for Grafana dashboards.
- A **ScopedTimer** combined with a histogram is the easiest way to instrument latency everywhere in your codebase.
