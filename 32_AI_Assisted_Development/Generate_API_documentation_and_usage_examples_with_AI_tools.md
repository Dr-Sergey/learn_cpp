# Generate API documentation and usage examples with AI tools

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

Writing documentation is one of those tasks that most C++ developers know they should do and most do not enjoy doing. AI assistants have made it dramatically more practical: you paste a header file, describe what style of documentation you want, and get back complete Doxygen comments, usage examples, thread-safety notes, and exception specifications. That is not a small help - documentation for a non-trivial class can take an hour to write well by hand, and the AI can produce a solid first draft in seconds.

This is one of the highest-ROI uses of AI for C++. The tricky part is knowing where to trust the output and where to verify it carefully. The table below gives you that map.

### AI Documentation Generation Quality

| Documentation Type | AI Quality | Key Consideration |
| --- | --- | --- |
| Function/method docs (Doxygen) | Excellent | Verify parameter descriptions |
| Usage examples | Excellent | Test that examples compile |
| Class overview documentation | Good | May need domain context |
| Thread-safety annotations | Medium | Verify actual thread safety |
| Exception specifications | Good | Check against actual throws |
| Migration guides | Good | Needs version context |
| README generation | Excellent | Review for accuracy |

Thread-safety annotations are the one area to be careful about. The AI generates plausible-sounding notes, but whether the actual implementation is thread-safe requires reading the implementation - the header signature alone does not tell you.

---

## Self-Assessment

### Q1: Generate Doxygen documentation from headers

**Answer:**

The key to getting good Doxygen output is specifying the format requirements explicitly: parameter directions (`[in]`, `[out]`, `[in,out]`), which sections to include (`@pre`, `@post`, `@throws`, `@note`), and the grammatical style you want. Without these specifications, you get inconsistent output that you have to reformat anyway.

Here is a prompt that produces thorough, consistent documentation:

```cpp
=== PROMPT ===

"Generate Doxygen documentation for every public method
in this header. Include:

- Brief and detailed description
- @param with direction ([in], [out], [in,out])
- @return with all possible return values
- @throws with conditions
- @note for thread safety and performance
- @code ... @endcode with a usage example
- @pre and @post conditions

Style: imperative mood ('Returns' not 'This returns')."
```

Here is the input header and what the AI produces. Notice how it generates different documentation for the blocking `try_pop_for` vs the non-blocking `try_pop` - it infers the behavioral difference from the signature:

```cpp
// Input header:
template<typename T, typename Alloc = std::allocator<T>>
class ConcurrentQueue {
public:
    ConcurrentQueue();
    explicit ConcurrentQueue(size_t initial_capacity);
    ~ConcurrentQueue();

    bool push(const T& item);
    bool push(T&& item);
    bool try_pop(T& out);
    std::optional<T> try_pop();
    bool try_pop_for(T& out, std::chrono::milliseconds timeout);
    size_t size() const;
    bool empty() const;
    void clear();
};

// === AI-generated Doxygen ===

/**
 * @brief Thread-safe, lock-free multi-producer multi-consumer queue.
 *
 * Provides a bounded concurrent queue suitable for inter-thread
 * communication. Uses a ring buffer internally with atomic operations
 * for lock-free push/pop in the common case.
 *
 * @tparam T     Element type. Must be nothrow move-constructible.
 * @tparam Alloc Allocator type for internal storage (default: std::allocator).
 *
 * @note Thread-safe: all methods can be called concurrently from
 *       multiple threads without external synchronization.
 * @note Lock-free for push/try_pop when queue is neither full nor empty.
 *
 * @code
 * ConcurrentQueue<WorkItem> queue(1024);
 *
 * // Producer thread:
 * queue.push(WorkItem{"task-1", payload});
 *
 * // Consumer thread:
 * WorkItem item;
 * if (queue.try_pop(item)) {
 *     process(item);
 * }
 * @endcode
 */

/**
 * @brief Constructs an empty queue with default capacity (1024).
 * @post empty() == true && size() == 0
 */

/**
 * @brief Constructs an empty queue with the specified capacity.
 * @param[in] initial_capacity Maximum number of elements.
 *            Rounded up to the next power of 2 internally.
 * @pre initial_capacity > 0
 * @throws std::invalid_argument if initial_capacity is 0.
 * @throws std::bad_alloc if memory allocation fails.
 */

/**
 * @brief Enqueues an element by copy.
 * @param[in] item Element to enqueue.
 * @return true if successfully enqueued, false if queue is full.
 * @note Non-blocking. Returns immediately if queue is full.
 * @note Thread-safe: can be called from multiple producer threads.
 */

/**
 * @brief Attempts to dequeue an element without blocking.
 * @param[out] out Receives the dequeued element if successful.
 *             Not modified if queue is empty.
 * @return true if an element was dequeued, false if queue is empty.
 * @note Non-blocking. Returns immediately if queue is empty.
 */

/**
 * @brief Attempts to dequeue with a timeout.
 * @param[out] out Receives the dequeued element if successful.
 * @param[in] timeout Maximum time to wait for an element.
 * @return true if an element was dequeued within the timeout.
 * @note Blocking: may sleep up to @p timeout duration.
 * @note Uses condition variable internally (not lock-free).
 */
```

After getting the output, check that all `@throws` annotations match what the implementation actually throws, and that the thread-safety notes in `@note` match the actual synchronization strategy. The AI infers these from the header, which means it may be right or it may be guessing.

### Q2: Generate usage examples and tutorials

**Answer:**

Usage documentation is where AI output tends to be most reliable, because example code is generated from the API you provide and can be mechanically checked for consistency. The key is to ask for coverage at multiple levels: quick start, common patterns, error handling, and frequently asked questions.

Here is the kind of prompt that produces a complete usage guide:

```cpp
=== PROMPT ===

"Generate a comprehensive usage guide for this library
with code examples. Cover:

- Quick start (minimal working example)
- Common patterns
- Error handling
- Advanced usage
- Performance tips
- FAQ with code answers"
```

Here is what AI-generated usage documentation looks like in practice for an HTTP client library:

```cpp
#include <chrono>
#include <iostream>
#include <span>
// === AI-generated usage guide ===

// ## Quick Start
//
// ```cpp
// #include <my_lib/http_client.hpp>
//
// int main() {
//     http::Client client;
//     auto response = client.get("https://api.example.com/data");
//     if (response.ok()) {
//         std::cout << response.body() << '\n';
//     }
// }
// ```

// ## Common Patterns

// Pattern: POST with JSON body
http::Client client;
auto response = client.post("https://api.example.com/users",
    http::json_body({{"name", "Alice"}, {"age", 30}})
);

// Pattern: Add headers and authentication
client.set_default_header("Authorization", "Bearer " + token);
client.set_default_header("Content-Type", "application/json");

// Pattern: Timeout and retry
auto response = client.get("https://slow-api.example.com",
    http::Options{}
        .timeout(std::chrono::seconds{5})
        .retries(3)
        .retry_delay(std::chrono::milliseconds{500})
);

// ## Error Handling

auto response = client.get(url);
if (!response) {
    // Network error (DNS, connection refused, timeout)
    std::cerr << "Error: " << response.error().message() << '\n';
} else if (response.status() >= 400) {
    // HTTP error (4xx, 5xx)
    std::cerr << "HTTP " << response.status()
              << ": " << response.body() << '\n';
}

// ## FAQ

// Q: How to make concurrent requests?
auto [r1, r2, r3] = http::when_all(
    client.async_get("https://api1.example.com"),
    client.async_get("https://api2.example.com"),
    client.async_get("https://api3.example.com")
);

// Q: How to stream large responses?
client.get_stream(url, [](std::span<const char> chunk) {
    process_chunk(chunk);
    return true;  // Continue receiving
});
```

Always compile the generated examples before publishing them. AI-generated code looks right but may have small errors - a missing include, a wrong method name, an outdated API. Running them through the compiler is the only reliable check.

### Q3: Generate API changelog and migration docs

**Answer:**

One of the most tedious documentation tasks is writing a migration guide when an API changes between versions. AI can do this mechanically if you give it both header versions - it will diff them, categorize the changes, and generate before/after code examples. The prompt is straightforward:

```cpp
=== PROMPT ===

"Compare these two header versions and generate:

1. Changelog (what changed, added, removed, deprecated)
2. Migration guide with before/after code examples
3. Breaking changes list"
```

Here is what the AI-generated output looks like for a library that changed from exceptions to `std::expected` and updated its options API:

```markdown
## AI-Generated Changelog: v2.0 -> v3.0

### Breaking Changes

- `Client::get()` now returns `Expected<Response, Error>`
  instead of `Response` (was throwing on error)

- `Response::body()` returns `std::string_view` instead
  of `std::string` (zero-copy, but don't store it!)

- Removed: `Client::set_timeout(int ms)` ->
  use `Options::timeout(chrono::milliseconds)`

### Migration Guide

#### Error handling (BREAKING)
```

```cpp
// v2.0 (exceptions)
try {
    auto resp = client.get(url);
    use(resp.body());
} catch (const http::Error& e) {
    handle_error(e);
}

// v3.0 (Expected)
auto resp = client.get(url);
if (resp) {
    use(resp->body());
} else {
    handle_error(resp.error());
}
```

```cpp
// v2.0
client.set_timeout(5000);  // Removed

// v3.0
client.get(url, http::Options{}
    .timeout(std::chrono::seconds{5}));
```

The `string_view` change in `body()` is the kind of subtle breaking change that a human writer might forget to call out but that has real consequences: callers who stored the return value of `body()` now have a dangling reference. AI will flag it because it sees the type change in the diff.

---

## Notes

- Always **paste the actual header** - AI needs the real signatures to generate accurate documentation, not a description of what the class does.
- Ask AI to generate **compilable examples** and actually compile them before publishing - small errors are common.
- For Doxygen, specify the **style** you want (parameter direction annotations, brief vs detailed descriptions, naming conventions).
- AI is excellent at generating **README.md** files with badges, install instructions, and quick start sections.
- For large APIs, process **one class at a time** for better quality - too much context at once can degrade the output.
- Ask AI to identify **missing documentation** by comparing header declarations against existing doc comments.
- Use AI to keep docs current: "update these docs for the changes in this diff" works well as a maintenance workflow.
