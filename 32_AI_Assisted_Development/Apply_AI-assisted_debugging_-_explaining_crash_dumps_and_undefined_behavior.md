# Apply AI-assisted debugging - explaining crash dumps and undefined behavior

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

Debugging in C++ often involves reading output that is genuinely hostile to humans - mangled symbol names, stack addresses with no context, sanitizer reports full of cross-referencing stack traces, and UBSan messages that cite the standard without explaining what that means for your code. AI assistants are surprisingly good at exactly this work. They can analyze **crash dumps**, **sanitizer output**, **core dumps**, and **undefined behavior reports**, decode mangled names, trace the execution path from a stack trace, identify root causes from ASan/UBSan/TSan output, and suggest concrete fixes. What used to take hours of documentation hunting and mental stack-trace tracing can often be diagnosed in minutes.

The table below shows where AI debugging assistance is strongest. The key insight is that AI is excellent when there is structured output to read (sanitizer reports are almost like a formatted description of the bug) but weaker when you need to reason about timing (heisenbugs) or when you lack symbol information (optimized builds).

### AI Debugging Assistance Effectiveness

| Debugging Task | AI Effectiveness | Key Benefit |
| --- | --- | --- |
| Crash stack trace analysis | Excellent | Decodes mangled names, explains flow |
| ASan report interpretation | Excellent | Identifies root cause + fix |
| TSan report analysis | Good | Explains data race + synchronization fix |
| UBSan output | Excellent | Cites standard, explains why it's UB |
| Segfault in optimized code | Medium | May need debug build context |
| Heisenbugs (race conditions) | Medium | Suggests instrumentation approach |
| Memory corruption | Medium | Identifies likely cause from symptoms |

---

## Self-Assessment

### Q1: Use AI to analyze sanitizer reports

**Answer:**

The most important thing to understand about using AI for sanitizer analysis is that you should paste the **full report** - both the error and the allocation/deallocation stack traces. The AI needs to correlate all three: where memory was allocated, where it was freed, and where it was accessed again. With all three in hand, it can reconstruct the lifetime story that led to the bug.

Here is an example prompt and what good AI analysis looks like in response:

```text
=== PROMPT: ASan heap-use-after-free ===

"Explain this AddressSanitizer report and suggest a fix:

=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address
0x60200000efb0 at pc 0x00000051b3ef bp 0x7ffed5b0e830

READ of size 8 at 0x60200000efb0 thread T0
    #0 0x51b3ee in std::__1::basic_string<...>::size()
    #1 0x51b256 in Logger::log(std::string const&)
             (/app/src/logger.cpp:42)
    #2 0x518f3a in OrderProcessor::process(Order const&)
             (/app/src/order_processor.cpp:87)
    #3 0x5183a2 in main (/app/src/main.cpp:23)

0x60200000efb0 is located 0 bytes inside of 32-byte region
freed by thread T0 here:
    #0 0x4c3a8f in operator delete(void*)
    #1 0x51a42e in std::__1::basic_string<...>::~basic_string()
    #2 0x51a3f1 in Config::~Config()
             (/app/src/config.cpp:15)
    #3 0x5184b2 in main (/app/src/main.cpp:21)

previously allocated by thread T0 here:
    #0 0x4c3683 in operator new(unsigned long)
    #1 0x51b891 in std::__1::basic_string<...>::__init(...)
    #2 0x51b6a3 in Config::Config(std::string)
             (/app/src/config.cpp:8)
    #3 0x518312 in main (/app/src/main.cpp:18)

Context: Logger holds a reference to Config's log_path string.
Config is destroyed before Logger."


=== EXPECTED AI ANALYSIS ===

"Root cause: Object lifetime ordering issue.

1. main.cpp:18 - Config constructed, allocates log_path string
2. Logger created with reference to Config::log_path
3. main.cpp:21 - Config destroyed, log_path string freed
4. main.cpp:23 - OrderProcessor::process calls Logger::log
5. Logger::log accesses freed log_path string -> USE-AFTER-FREE

Fix options:

1. Ensure Logger is destroyed BEFORE Config (reverse declaration
   order, or use explicit init/shutdown)
2. Logger stores a copy of log_path, not a reference
3. Use shared_ptr<Config> so Config outlives Logger

Recommended fix (option 1 - RAII ordering):"
```

Once the AI identifies the root cause - Config is destroyed while Logger still holds a reference to one of its strings - the fix becomes obvious. Here is the before and after:

```cpp
// BAD: Config destroyed before Logger
int main() {
    Config config("app.conf");     // Constructed first
    Logger logger(config.log_path()); // Holds reference to config
    // ... config destroyed first (reverse order)
    // logger.log() accesses freed memory!
}

// FIX: Reverse order or use shared_ptr
int main() {
    auto config = std::make_shared<Config>("app.conf");
    Logger logger(config);  // Logger holds shared_ptr
    // Config outlives Logger
}
```

The use-after-free evaporates because `shared_ptr` keeps `Config` alive for as long as anyone holds a reference to it. The reason this bug is so easy to miss in the first place is that the two objects are in different parts of the code - nothing at the call site signals that a lifetime dependency was created.

### Q2: Use AI to explain complex UB scenarios

**Answer:**

Signed integer overflow is one of the most dangerous categories of undefined behavior precisely because it often *appears* to work correctly in debug builds. The compiler is allowed to assume it never happens, and at `-O3` it uses that assumption to transform code in ways that can completely change behavior. Explaining why this is UB and why optimization level matters is exactly where AI adds value - it can cite the standard clause and walk through the compiler's reasoning.

The prompt pattern here is to paste both the UBSan report and the code, and ask specifically why the behavior differs between optimization levels:

```cpp
=== PROMPT: Explain this UBSan report ===

"This code produces different results with -O0 vs -O3.
UBSan reports the issue below. Explain the root cause
and why optimization level affects the behavior:

runtime error: signed integer overflow: 2147483647 + 1
cannot be represented in type 'int'
    #0 in compute_checksum(int const*, unsigned long)
      (/app/src/checksum.cpp:12)"
```

Here is the problematic code, the AI's explanation, and two concrete fixes:

```cpp
// The problematic code:
int compute_checksum(const int* data, size_t len) {
    int sum = 0;
    for (size_t i = 0; i < len; ++i) {
        sum += data[i];  // UB: signed integer overflow
    }
    return sum;
}

// AI explanation:
// Signed integer overflow is UNDEFINED BEHAVIOR in C++
// [expr.pre]/4: "If during evaluation an expression, the
// result is not mathematically defined or not in the range
// of representable values, the behavior is undefined."
//
// At -O0: Overflow wraps around (on x86, 2's complement)
// At -O3: Compiler assumes no UB, may optimize the loop
//   based on the assumption that sum never overflows.
//   This can eliminate bounds checks, reorder operations,
//   or produce completely different code.
//
// Fix: Use unsigned int (overflow is well-defined) or
// use __builtin_add_overflow for checked arithmetic.

// Fix option 1: unsigned arithmetic
uint32_t compute_checksum(const int* data, size_t len) {
    uint32_t sum = 0;
    for (size_t i = 0; i < len; ++i) {
        sum += static_cast<uint32_t>(data[i]);
    }
    return sum;
}

// Fix option 2: checked arithmetic
std::optional<int> compute_checksum_checked(
    const int* data, size_t len) {
    int sum = 0;
    for (size_t i = 0; i < len; ++i) {
        if (__builtin_add_overflow(sum, data[i], &sum))
            return std::nullopt;  // Overflow detected
    }
    return sum;
}
```

The reason this trips people up is that x86 integer arithmetic wraps at the hardware level, so the "broken" behavior at `-O0` actually produces the result you expect. The code passes all its tests with debug builds and then silently breaks in production. UBSan exists specifically to catch this class of bug before you ship it.

### Q3: Use AI to analyze ThreadSanitizer data race reports

**Answer:**

TSan reports are structured around showing you two conflicting accesses: the write and the read (or write and write) that happened concurrently without synchronization. AI is well-suited to interpreting these because once it identifies the two accesses it can reason about what synchronization mechanism is missing and present multiple fix options with different performance trade-offs.

Here is the kind of TSan report you would paste and what a useful AI response looks like:

```text
=== PROMPT: TSan data race report ===

"Explain this ThreadSanitizer report and show the fix:

WARNING: ThreadSanitizer: data race (pid=1234)
  Write of size 8 at 0x7f... by thread T2:
    #0 RequestHandler::update_stats()
             request_handler.cpp:45
    #1 RequestHandler::handle()
             request_handler.cpp:32

  Previous read of size 8 at 0x7f... by thread T1:
    #0 RequestHandler::get_stats() const
             request_handler.cpp:51
    #1 MonitorThread::report()
             monitor.cpp:18

  Thread T2 is the request handling thread.
  Thread T1 is the monitoring/reporting thread."
```

The problematic code and three fix options follow - notice how the AI suggests different synchronization strategies depending on how much contention you expect:

```cpp
// The problematic code:
class RequestHandler {
    struct Stats {
        uint64_t total_requests = 0;    // Shared mutable state
        uint64_t total_bytes = 0;
        double avg_latency_ms = 0.0;
    };

    Stats stats_;  // No synchronization!

public:
    void handle(const Request& req) {
        auto result = process(req);
        update_stats();  // Thread T2 writes stats_
    }

    void update_stats() {
        stats_.total_requests++;     // WRITE (T2)
        stats_.total_bytes += size;
        stats_.avg_latency_ms = /*...*/;
    }

    Stats get_stats() const {
        return stats_;  // READ (T1) - data race!
    }
};

// AI-suggested fix options:

// Fix 1: Mutex (simple, correct)
class RequestHandler {
    Stats stats_;
    mutable std::mutex mtx_;
public:
    void update_stats() {
        std::lock_guard lock(mtx_);
        stats_.total_requests++;
        stats_.total_bytes += size;
    }
    Stats get_stats() const {
        std::lock_guard lock(mtx_);
        return stats_;
    }
};

// Fix 2: Atomics (higher performance, lock-free)
class RequestHandler {
    std::atomic<uint64_t> total_requests_{0};
    std::atomic<uint64_t> total_bytes_{0};
    std::atomic<double> avg_latency_{0.0};
public:
    void update_stats(uint64_t bytes, double latency) {
        total_requests_.fetch_add(1, std::memory_order_relaxed);
        total_bytes_.fetch_add(bytes, std::memory_order_relaxed);
        avg_latency_.store(latency, std::memory_order_relaxed);
    }
    Stats get_stats() const {
        return {
            total_requests_.load(std::memory_order_relaxed),
            total_bytes_.load(std::memory_order_relaxed),
            avg_latency_.load(std::memory_order_relaxed)
        };
    }
};

// Fix 3: Snapshot with double-buffer (no contention)
class RequestHandler {
    Stats current_;  // Only written by handler thread
    std::atomic<Stats*> published_{nullptr};
public:
    void publish_snapshot() {
        auto* snap = new Stats(current_);
        auto* old = published_.exchange(snap, std::memory_order_release);
        delete old;
    }
    Stats get_stats() const {
        auto* snap = published_.load(std::memory_order_acquire);
        return snap ? *snap : Stats{};
    }
};
```

Fix 1 is always correct and easy to understand - start here. Fix 2 is better if the monitoring thread reads stats frequently and you want to avoid blocking the handler thread. Fix 3 eliminates contention entirely by separating the write path from the read path, at the cost of some complexity.

---

## Notes

- When pasting sanitizer output, **include the full report** - AI needs both the error and the allocation/free stack traces to reconstruct the lifetime story.
- For crash dumps, provide the **source code context** around the failing function so the AI can connect addresses to variable names.
- AI is especially good at **decoding mangled C++ names** in stack traces - just paste the raw output and it will demangle for you.
- Always **reproduce with a debug build** first (`-g -O0`) for accurate line numbers before analyzing the report.
- TSan reports show the **two conflicting accesses** - AI explains the race condition and what synchronization is missing.
- For intermittent bugs, ask AI to suggest **instrumentation** (logging, assertions) to catch the issue reproducibly.
- Combine AI analysis with sanitizers: AI explains the report, the sanitizer proves the bug is real.
