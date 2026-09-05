# Generate documentation with Doxygen, Standardese, or hdoc

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://www.doxygen.nl/>  

---

## Topic Overview

API documentation should be generated from source code comments. The key insight here is that docs live right beside the code they describe, so when the code changes the docs are right there to update too - they don't drift off into a separate wiki or README that nobody touches.

### Doxygen (Industry Standard)

Doxygen is the workhorse of C++ documentation. You annotate your declarations with specially formatted comments, run the tool, and out comes a browsable HTML reference. Here's what a well-documented function and class look like:

```cpp
/// @brief Calculate the area of a circle.
/// @param radius The radius of the circle. Must be non-negative.
/// @return The area as a double.
/// @throws std::invalid_argument if radius is negative.
/// @note Uses M_PI from <cmath>.
///
/// @code
/// double a = circle_area(5.0);  // Returns ~78.54
/// @endcode
///
/// @see square_area(), ellipse_area()
/// @since v2.1
double circle_area(double radius);

/// @brief A thread-safe connection pool.
/// @tparam Connection The connection type (must be DefaultConstructible).
///
/// Usage:
/// @code
/// Pool<TcpConnection> pool(10);
/// auto conn = pool.acquire();
/// conn->send("hello");
/// pool.release(std::move(conn));
/// @endcode
template<typename Connection>
class Pool {
public:
    /// @brief Create a pool with the specified capacity.
    /// @param capacity Maximum number of connections.
    explicit Pool(size_t capacity);

    /// @brief Acquire a connection from the pool.
    /// @return A unique_ptr to a connection, or nullptr if pool is exhausted.
    [[nodiscard]] std::unique_ptr<Connection> acquire();

    /// @brief Return a connection to the pool.
    void release(std::unique_ptr<Connection> conn);
};
```

Notice how every `@param` tags a parameter name, `@return` says what comes back, `@throws` names the exception and the condition - and there's an embedded `@code` example so a reader never has to guess how to call it. The `@see` tag cross-links related functions, and `@since` tells users which library version introduced the API.

### Doxyfile Configuration

Doxygen is controlled by a configuration file (the "Doxyfile"). You only need a handful of keys to get started. Here's a minimal but useful set:

```ini
# Doxyfile essentials
PROJECT_NAME     = "MyLib"
OUTPUT_DIRECTORY = docs/
INPUT            = include/ src/
RECURSIVE        = YES
EXTRACT_ALL      = NO        # Only document commented entities
GENERATE_HTML    = YES
GENERATE_LATEX   = NO
USE_MDFILE_AS_MAINPAGE = README.md
WARN_IF_UNDOCUMENTED = YES  # Enforce documentation coverage
```

The most important CI-friendly setting is `WARN_IF_UNDOCUMENTED = YES`. With that on, Doxygen prints a warning for every public entity that lacks a doc comment, which means you can fail a build when documentation coverage drops.

---

## Self-Assessment

### Q1: What are the most important Doxygen tags for API documentation

The tags you'll reach for on almost every function are `@brief` (one-line summary), `@param` (documents each parameter), `@return` (describes what the function returns), and `@throws` (lists exceptions and when they fire). Beyond those, `@pre` and `@post` document preconditions and postconditions - essential for contracts - `@note` flags things the caller should keep in mind, and `@deprecated` marks APIs that are being phased out with a message pointing to the replacement.

### Q2: Compare Doxygen vs Standardese vs hdoc

All three tools generate API docs from annotated C++ source, but they each have a different sweet spot. The table below captures the key differences:

| Tool | Language | Output | C++20 Support |
| --- | --- | --- | --- |
| Doxygen | C/C++/Java | HTML, LaTeX, XML | Partial |
| Standardese | C++ only | Markdown, HTML | Better |
| hdoc | C++ only | HTML (modern) | Good |

Doxygen is the most established and works for mixed C/C++ codebases. Standardese generates documentation that deliberately mirrors the style of the C++ standard itself, which is excellent for library authors targeting that audience. hdoc produces modern, searchable HTML and has better C++20 feature coverage if you need concepts and modules documented properly.

### Q3: How to enforce documentation coverage in CI

The idea is simple: if Doxygen prints a warning about an undocumented symbol, treat that warning as a build failure. Here's the one-liner that does it:

```bash
# Run Doxygen with WARN_IF_UNDOCUMENTED = YES
# Parse warning output and fail CI if any undocumented public API
doxygen Doxyfile 2>&1 | grep "warning:" && exit 1
```

If any warning line appears, `grep` exits 0 and the `exit 1` fires - the CI job fails. If there are no warnings, `grep` exits non-zero and the `exit 1` is never reached. It's a one-liner gate that keeps your public API fully documented.

---

## Notes

- Document the contract, not just the types: preconditions, postconditions, exception guarantees, and thread safety are what callers actually need to know.
- Use `@code`/`@endcode` to embed a usage example in every public function - callers shouldn't have to hunt for sample code.
- Standardese generates documentation that reads like the C++ standard - excellent for library authors who want that formal, precise tone.
- hdoc produces modern, searchable HTML with full C++20 support, making it a strong choice for contemporary libraries.
