# Use access specifiers strategically - public, protected, private semantics

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

Access specifiers (`public`, `protected`, `private`) are the primary mechanism for **encapsulation** in C++. Strategic use goes far beyond "make everything private" - it communicates design intent, defines extension points, and controls the coupling surface between classes.

### Access Level Semantics

| Specifier | Who Can Access | Design Intent |
| --- | --- | --- |
| **`public`** | Anyone | Stable API contract - hard to change without breaking clients |
| **`protected`** | Class + derived classes | Extension point for subclasses - semi-stable interface |
| **`private`** | Class only (+ friends) | Implementation detail - free to change |

### Inheritance Access

| Base Member | `public` Inheritance | `protected` Inheritance | `private` Inheritance |
| --- | --- | --- | --- |
| `public` | `public` | `protected` | `private` |
| `protected` | `protected` | `protected` | `private` |
| `private` | inaccessible | inaccessible | inaccessible |

### Decision Framework

Work through this top to bottom when you are not sure which specifier to use:

```cpp
Should this member be part of the public API?
├─ Yes -> public
└─ No
   ├─ Should derived classes customize it? -> protected (NVI pattern)
   └─ No -> private
         ├─ Does a specific non-member function need access? -> friend
         └─ No -> stay private
```

### Common Mistakes

- Making data members `protected` - this exposes your representation to all subclasses, which is just as bad as making it public.
- Using `public` inheritance to reuse implementation (should be `private` or composition).
- Over-using `friend` - it breaks encapsulation as much as making something `public`.
- Making everything `public` "for convenience" during development - this almost never gets cleaned up later.

---

## Self-Assessment

### Q1: Demonstrate strategic access specifier usage in a class hierarchy using the NVI pattern

The Non-Virtual Interface (NVI) pattern is the cleanest way to structure a polymorphic hierarchy. The base class exposes a `public` non-virtual function that controls the overall workflow, and defers the actual work to a `protected` or `private` virtual function that subclasses override. This means the base class always gets to run pre-conditions and post-conditions, and subclasses can only touch the part they are supposed to touch.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

// NVI: public non-virtual + private/protected virtual

class Document {
public:
    // PUBLIC: Stable API contract. Clients call these.
    // Non-virtual: the base class controls the workflow.

    std::string render() const {
        // Pre-condition / invariant check
        if (title_.empty())
            throw std::runtime_error("Document must have a title");

        std::string result = render_header();
        result += do_render_body();  // <= customization point
        result += render_footer();
        return result;
    }

    void set_title(const std::string& title) { title_ = title; }
    const std::string& title() const { return title_; }

    virtual ~Document() = default;

protected:
    // PROTECTED: available to subclasses for customization.
    // These are the "hooks" in the NVI pattern.

    virtual std::string do_render_body() const = 0;

    // Derived classes can read (but not write) title for rendering
    const std::string& get_title() const { return title_; }

private:
    // PRIVATE: implementation details. Free to change.

    std::string render_header() const {
        return "=== " + title_ + " ===\n";
    }

    std::string render_footer() const {
        return "---\n";
    }

    std::string title_;
};

class MarkdownDocument : public Document {
protected:
    std::string do_render_body() const override {
        std::string body;
        for (const auto& section : sections_)
            body += "## " + section + "\n";
        return body;
    }

public:
    void add_section(const std::string& s) { sections_.push_back(s); }

private:
    std::vector<std::string> sections_;
};

class HtmlDocument : public Document {
protected:
    std::string do_render_body() const override {
        return "<div>" + content_ + "</div>\n";
    }

public:
    void set_content(const std::string& c) { content_ = c; }

private:
    std::string content_;
};

int main() {
    MarkdownDocument md;
    md.set_title("Design Guide");
    md.add_section("Architecture");
    md.add_section("Implementation");
    std::cout << md.render();

    HtmlDocument html;
    html.set_title("Report");
    html.set_content("Quarterly results");
    std::cout << html.render();
}
```

**Access specifier rationale:**

- `public`: `render()`, `set_title()` - the client API, stable contract.
- `protected`: `do_render_body()` - customization hook for subclasses.
- `private`: `render_header()`, `render_footer()`, `title_` - implementation details.

### Q2: Explain private inheritance vs composition, and when to use `protected` inheritance

Private inheritance is often described as "implemented-in-terms-of" - it gives you the base class machinery without exposing the base class as part of the type. The table below shows how it compares to composition. The practical advice is simple: reach for composition first and only consider private inheritance when you genuinely need to override virtual functions of the base class, or when you need the Empty Base Optimization (EBO) to save memory.

| Aspect | Private Inheritance | Composition |
| --- | --- | --- |
| Relationship | "is-implemented-in-terms-of" | "has-a" |
| Base public members | Become private | Accessed via member |
| Virtual overrides | Can override base virtuals | Cannot |
| EBO (Empty Base Opt.) | Yes | No (member has size >= 1) |
| Coupling | Tight (access to protected) | Loose |
| Clarity | Confusing to readers | Clear and obvious |

```cpp
#include <iostream>
#include <mutex>

// Private inheritance: is-implemented-in-terms-of
// Use when you need to override virtual functions of the base
// or leverage Empty Base Optimization.

class Timer {
public:
    virtual ~Timer() = default;
    void start() {
        // ... starts timer, eventually calls:
        on_tick();
    }
protected:
    virtual void on_tick() = 0;  // Hook for subclasses
};

// Widget IS NOT a Timer to clients, but uses Timer's mechanism internally
class Widget : private Timer {
public:
    void begin_animation() {
        start();  // Calls Timer::start(), which calls our on_tick()
    }

protected:
    void on_tick() override {
        std::cout << "Widget animation frame\n";
    }
};

// Widget w;
// w.start();  // ERROR: start() is private through private inheritance
// w.begin_animation();  // OK: public Widget interface


// Protected inheritance: rare but useful
// Derived classes of Widget can also use Timer's protected interface

class AnimatedWidget : private Timer {
protected:
    // Re-expose start() to further subclasses only
    using Timer::start;
    void on_tick() override { std::cout << "Animated\n"; }
};

class FancyWidget : public AnimatedWidget {
public:
    void animate() {
        start();  // OK: AnimatedWidget made it protected
    }
};


// Empty Base Optimization (EBO)

struct EmptyAllocator {};  // sizeof == 1 as member, 0 as base

// With composition: sizeof(WithMember) = sizeof(int) + 1 + padding = 8
struct WithMember {
    int data;
    EmptyAllocator alloc;  // Takes at least 1 byte + padding
};

// With private inheritance: sizeof(WithBase) = sizeof(int) = 4
struct WithBase : private EmptyAllocator {
    int data;
};

// C++20 alternative: [[no_unique_address]]
struct Modern {
    int data;
    [[no_unique_address]] EmptyAllocator alloc;  // May be 0 bytes
};

int main() {
    Widget w;
    w.begin_animation();

    std::cout << "WithMember: " << sizeof(WithMember) << '\n';  // 8
    std::cout << "WithBase: " << sizeof(WithBase) << '\n';      // 4
    std::cout << "Modern: " << sizeof(Modern) << '\n';          // 4
}
```

**When to use each:**

- **Composition** (default): 95% of cases. Clearest, loosest coupling.
- **Private inheritance**: Need to override virtual functions of the "base" or need EBO.
- **Protected inheritance**: Need private inheritance but also want derived classes to access the base.

### Q3: Show `friend` usage patterns and when friendship is appropriate design

`friend` is sometimes treated as a dirty word in C++ design discussions, but it is actually the right answer in a handful of well-defined situations. The key is knowing which situations those are. The pattern table at the end of this example spells it out.

```cpp
#include <iostream>
#include <string>
#include <sstream>

// Pattern 1: Hidden friends for operators (ADL-safe)

class Temperature {
public:
    explicit Temperature(double celsius) : celsius_(celsius) {}

    // Hidden friend: found only via ADL, not polluting namespace
    friend bool operator==(Temperature a, Temperature b) {
        return a.celsius_ == b.celsius_;
    }

    friend bool operator<(Temperature a, Temperature b) {
        return a.celsius_ < b.celsius_;
    }

    friend std::ostream& operator<<(std::ostream& os, Temperature t) {
        return os << t.celsius_ << "C";
    }

private:
    double celsius_;
};

// Pattern 2: Factory friend (named constructor idiom)

class SecureConnection {
public:
    void send(const std::string& data) {
        std::cout << "Sending on fd " << fd_ << ": " << data << '\n';
    }

    // Factory is the only way to create this object
    friend class ConnectionFactory;

private:
    // Constructor is private - only factory can create
    SecureConnection(int fd, std::string cert)
        : fd_(fd), cert_(std::move(cert)) {}

    int fd_;
    std::string cert_;
};

class ConnectionFactory {
public:
    static SecureConnection create(const std::string& host) {
        // Validate, establish TLS, etc.
        int fd = 42;  // Simulated
        return SecureConnection(fd, "cert_for_" + host);
    }
};

// Pattern 3: Attorney-Client (limited friendship)
// Problem: friend gives access to EVERYTHING. Attorney limits it.

class Engine {
public:
    void start() { std::cout << "Engine started\n"; }
    int rpm() const { return rpm_; }

private:
    friend class EngineAttorney;  // Only the attorney, not Car directly
    void set_throttle(int pct) { rpm_ = pct * 60; }
    void emergency_shutdown() { rpm_ = 0; }
    int rpm_ = 0;
};

// Attorney exposes only specific private operations
class EngineAttorney {
    friend class Car;  // Only Car can use the attorney

    static void set_throttle(Engine& e, int pct) { e.set_throttle(pct); }
    // Note: emergency_shutdown is NOT exposed through the attorney
};

class Car {
    Engine engine_;
public:
    void accelerate(int pct) {
        EngineAttorney::set_throttle(engine_, pct);  // OK: via attorney
        // engine_.set_throttle(pct);  // ERROR: private
        // engine_.emergency_shutdown();  // ERROR: private
        std::cout << "RPM: " << engine_.rpm() << '\n';
    }
};

// When friendship is appropriate:
// 1. operator<< and operator>> (must be non-member)
// 2. Factory functions that need private constructor access
// 3. Tightly-coupled class pairs (e.g., iterator + container)
// 4. Attorney-Client for controlled access

// When friendship is a code smell:
// 1. Friend class in a different module -> tight cross-module coupling
// 2. Many friends -> the class interface is too restrictive
// 3. Using friend to "hack around" access -> redesign instead

int main() {
    // Hidden friends
    Temperature t1(100.0), t2(37.0);
    std::cout << t1 << " vs " << t2 << ": " << (t1 < t2 ? "less" : "greater or equal") << '\n';

    // Factory
    auto conn = ConnectionFactory::create("example.com");
    conn.send("Hello");

    // Attorney-Client
    Car car;
    car.accelerate(50);
}
```

---

## Notes

- Default access: `class` = `private`, `struct` = `public`; use `struct` for aggregates/PODs, `class` for encapsulated types.
- `friend` declarations are not transitive and not inherited.
- Prefer **hidden friends** for operators - they prevent unwanted implicit conversions via ADL.
- `protected` data members are almost always a mistake; use protected *functions* instead.
- Private inheritance is equivalent to composition + friendship to the base.
- C++20 `[[no_unique_address]]` often eliminates the need for private inheritance for EBO.
- Guideline: start everything `private`, promote to `protected`/`public` only when needed.
