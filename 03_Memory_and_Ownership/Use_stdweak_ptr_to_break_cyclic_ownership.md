# Use std::weak_ptr to Break Cyclic Ownership

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/weak_ptr>  

---

## Topic Overview

### The Cyclic Ownership Problem

When two objects hold `shared_ptr` to each other, neither object's reference count ever reaches zero - which means neither is ever destroyed. This is a straightforward memory leak, and it is silent: no assertion fires, no crash, the objects just never go away.

```cpp
  A --shared_ptr--> B
  ^                 |
  +--shared_ptr-----+   <- CYCLE: neither A nor B can be destroyed
```

### Solution: `weak_ptr`

`weak_ptr` is a **non-owning** observer of a `shared_ptr`-managed object. It does **not** increase the reference count. One direction of the cycle becomes a `weak_ptr`, which means destroying the external reference to `A` really does drop `A`'s count to zero.

```cpp
  A --shared_ptr--> B
  ^                 |
  +---weak_ptr------+   <- No ownership cycle: B doesn't keep A alive
```

| Feature | `shared_ptr` | `weak_ptr` | Raw pointer |
| --- | --- | --- | --- |
| Keeps object alive | Yes | No | No |
| Detects expiry | N/A | Yes (`expired()`) | No (dangling) |
| Safe access | Direct | Via `lock()` | Unsafe |
| Affects use_count | Yes | No | No |
| Keeps control block | Yes | Yes | No |

### Key API

You create a `weak_ptr` from a `shared_ptr`, then use `lock()` whenever you want to actually use the object. `lock()` either gives you a valid `shared_ptr` (and briefly bumps the ref count for that access) or returns `nullptr` if the object is already gone.

```cpp
std::shared_ptr<T> sp = std::make_shared<T>(...);
std::weak_ptr<T> wp = sp;          // Non-owning observer

wp.expired();                       // true if object destroyed
auto locked = wp.lock();            // Returns shared_ptr<T> or nullptr
wp.use_count();                     // Number of shared_ptr owners
wp.reset();                         // Release observation
```

---

## Self-Assessment

### Q1: Demonstrate a parent-child tree where parent holds `shared_ptr<Child>` and child holds `weak_ptr<Parent>`

Parent-child trees are the canonical place where cycles appear. The parent owns the children (so they live as long as the parent), but children only need to *find* their parent occasionally - that is the weak observation direction.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Child;  // Forward declaration

struct Parent : std::enable_shared_from_this<Parent> {
    std::string name;
    std::vector<std::shared_ptr<Child>> children;  // Parent OWNS children

    Parent(std::string n) : name(std::move(n)) {
        std::cout << "  [+] Parent '" << name << "' created\n";
    }
    ~Parent() {
        std::cout << "  [-] Parent '" << name << "' destroyed\n";
    }

    void add_child(std::shared_ptr<Child> child);
    void print_tree(int depth = 0) const;
};

struct Child {
    std::string name;
    std::weak_ptr<Parent> parent;  // Child does NOT own parent (breaks cycle!)
    std::vector<std::shared_ptr<Child>> children;

    Child(std::string n) : name(std::move(n)) {
        std::cout << "  [+] Child '" << name << "' created\n";
    }
    ~Child() {
        std::cout << "  [-] Child '" << name << "' destroyed\n";
    }

    std::string parent_name() const {
        if (auto p = parent.lock()) {
            return p->name;
        }
        return "(parent expired)";
    }
};

void Parent::add_child(std::shared_ptr<Child> child) {
    child->parent = shared_from_this();  // weak_ptr to this parent
    children.push_back(std::move(child));
}

void Parent::print_tree(int depth) const {
    std::string indent(depth * 2, ' ');
    std::cout << indent << "Parent: " << name << "\n";
    for (const auto& c : children) {
        std::cout << indent << "  Child: " << c->name
                  << " (parent=" << c->parent_name() << ")\n";
    }
}

int main() {
    std::cout << "=== Parent-Child Tree with weak_ptr ===\n\n";

    {
        auto root = std::make_shared<Parent>("Root");
        auto c1 = std::make_shared<Child>("Alice");
        auto c2 = std::make_shared<Child>("Bob");

        root->add_child(c1);
        root->add_child(c2);

        root->print_tree();

        std::cout << "\nReference counts:\n";
        std::cout << "  root: " << root.use_count() << "\n";  // 1 (only main)
        std::cout << "  c1:   " << c1.use_count() << "\n";    // 2 (main + parent)
        std::cout << "  c2:   " << c2.use_count() << "\n";    // 2 (main + parent)

        std::cout << "\nLeaving scope - all objects destroyed:\n";
    }
    // Output: Parent destroyed, then children - NO LEAK

    std::cout << "\n=== Contrast: with shared_ptr back-link (LEAK!) ===\n";
    std::cout << "If Child held shared_ptr<Parent> instead of weak_ptr<Parent>,\n";
    std::cout << "root's use_count would be 3 (main + Alice + Bob),\n";
    std::cout << "and destroying root in main would leave use_count=2 -> never freed!\n";

    return 0;
}
// Expected output:
//   [+] Parent 'Root' created
//   [+] Child 'Alice' created
//   [+] Child 'Bob' created
//   Parent: Root
//     Child: Alice (parent=Root)
//     Child: Bob (parent=Root)
//   Reference counts:
//     root: 1
//     c1: 2
//     c2: 2
//   Leaving scope - all objects destroyed:
//   [-] Parent 'Root' destroyed
//   [-] Child 'Alice' destroyed
//   [-] Child 'Bob' destroyed
```

With `weak_ptr` on the back-link, `root`'s use count stays at 1. When `root` goes out of scope the count drops to zero, `~Parent` runs, the children vector is cleared, and the children are destroyed in turn.

### Q2: Use `weak_ptr::lock()` safely and handle the case where the pointed-to object has been destroyed

The reason this trips people up is the temptation to check `expired()` first and then call `lock()`. That is a race condition - the object could expire between those two calls. Always use `lock()` directly and check whether the result is null.

```cpp
#include <iostream>
#include <memory>
#include <string>

struct Session {
    std::string id;
    Session(std::string i) : id(std::move(i)) {
        std::cout << "  Session '" << id << "' started\n";
    }
    ~Session() {
        std::cout << "  Session '" << id << "' ended\n";
    }
    void process() {
        std::cout << "  Processing on session '" << id << "'\n";
    }
};

// An observer that holds a weak_ptr - does not keep the session alive
class SessionObserver {
    std::weak_ptr<Session> session_;
public:
    SessionObserver(std::weak_ptr<Session> s) : session_(std::move(s)) {}

    bool try_use() {
        // lock() returns shared_ptr - extends lifetime during use
        if (auto sp = session_.lock()) {
            sp->process();
            std::cout << "  (use_count during lock: " << sp.use_count() << ")\n";
            return true;
        } else {
            std::cout << "  Session expired - cannot use\n";
            return false;
        }
    }

    bool is_alive() const {
        return !session_.expired();
    }
};

int main() {
    std::cout << "=== weak_ptr::lock() safe access ===\n\n";

    SessionObserver observer(std::weak_ptr<Session>{});  // Empty initially

    {
        auto session = std::make_shared<Session>("ABC-123");
        observer = SessionObserver(session);  // Observe the session

        std::cout << "\nSession is alive: " << std::boolalpha << observer.is_alive() << "\n";
        observer.try_use();  // Works - session exists

        std::cout << "\nDestroying session...\n";
    }
    // Session destroyed here

    std::cout << "\nSession is alive: " << std::boolalpha << observer.is_alive() << "\n";
    observer.try_use();  // Safely handles expired session

    // IMPORTANT: Never do this:
    // if (!wp.expired()) {
    //     auto sp = wp.lock();  // RACE: object might expire between check and lock!
    //     sp->use();            // CRASH if sp is null
    // }
    // CORRECT: Always check the result of lock() directly

    return 0;
}
// Expected output:
//   Session 'ABC-123' started
//   Session is alive: true
//   Processing on session 'ABC-123'
//   (use_count during lock: 2)
//   Destroying session...
//   Session 'ABC-123' ended
//   Session is alive: false
//   Session expired - cannot use
```

Notice that during the `lock()` call the use_count bumps to 2 - the observer temporarily holds a real `shared_ptr` for the duration of the access, keeping the session alive for that window. When the `if` block ends, the temporary `shared_ptr` is destroyed and the count drops back.

### Q3: Show the overhead of `weak_ptr` compared to raw pointer observers

`weak_ptr` is not free. It is the same size as `shared_ptr` (two pointers: one to the object, one to the control block), and `lock()` involves atomic operations. If the object lifetime is strictly controlled and you can guarantee a raw pointer won't dangle, a raw pointer observer has lower cost. This benchmark makes the difference concrete.

```cpp
#include <iostream>
#include <memory>
#include <chrono>
#include <vector>

struct Data {
    int value = 42;
};

int main() {
    std::cout << "=== weak_ptr vs Raw Pointer Overhead ===\n\n";

    // --- Size overhead ---
    std::cout << "--- Size Comparison ---\n";
    std::cout << "sizeof(Data*)             = " << sizeof(Data*) << " bytes\n";
    std::cout << "sizeof(shared_ptr<Data>)  = " << sizeof(std::shared_ptr<Data>) << " bytes\n";
    std::cout << "sizeof(weak_ptr<Data>)    = " << sizeof(std::weak_ptr<Data>) << " bytes\n";
    // weak_ptr stores: object pointer + control-block pointer (same as shared_ptr)

    std::cout << "\n--- Memory Layout ---\n";
    std::cout << "shared_ptr = [ptr to object | ptr to control block]\n";
    std::cout << "weak_ptr   = [ptr to object | ptr to control block]  (same layout!)\n";
    std::cout << "raw ptr    = [ptr to object]                          (half the size)\n";

    std::cout << "\n--- Control Block Lifetime ---\n";
    std::cout << "Control block holds: use_count + weak_count\n";
    std::cout << "Object destroyed when: use_count == 0\n";
    std::cout << "Control block freed when: use_count == 0 AND weak_count == 0\n";
    std::cout << "-> weak_ptr keeps the control block alive even after object is gone!\n";

    // --- Performance overhead ---
    std::cout << "\n--- Performance Comparison ---\n";

    constexpr int N = 10'000'000;
    auto sp = std::make_shared<Data>();
    Data* raw = sp.get();
    std::weak_ptr<Data> wp = sp;

    // Raw pointer access
    volatile int sink = 0;
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        sink = raw->value;
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // weak_ptr lock + access
    auto t3 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        if (auto locked = wp.lock()) {
            sink = locked->value;
        }
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    auto raw_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    auto weak_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t4 - t3).count();

    std::cout << "  Raw pointer:   " << raw_ms << " ms (" << N << " accesses)\n";
    std::cout << "  weak_ptr lock: " << weak_ms << " ms (" << N << " accesses)\n";
    std::cout << "  (lock() involves atomic ref-count increment/decrement)\n";

    std::cout << "\n--- When to Use Each ---\n";
    std::cout << "  weak_ptr:\n";
    std::cout << "    + Safely detects expired objects\n";
    std::cout << "    + Thread-safe (lock is atomic)\n";
    std::cout << "    + Breaks ownership cycles\n";
    std::cout << "    - 2x size of raw pointer\n";
    std::cout << "    - lock() has atomic overhead\n";
    std::cout << "    - Keeps control block alive\n";
    std::cout << "  Raw pointer observer:\n";
    std::cout << "    + Minimal size (1 pointer)\n";
    std::cout << "    + Zero access overhead\n";
    std::cout << "    - Cannot detect dangling (UB if used after free)\n";
    std::cout << "    - Only safe when lifetime is guaranteed by design\n";

    return 0;
}
// Expected output (varies by platform):
//   sizeof(Data*)             = 8 bytes
//   sizeof(shared_ptr<Data>)  = 16 bytes
//   sizeof(weak_ptr<Data>)    = 16 bytes
//   ...
//   Raw pointer:   ~15 ms
//   weak_ptr lock: ~120 ms
```

The atomic increment/decrement in `lock()` is the source of the speed difference. When you need the safety of detecting expiry and the code isn't on a hot path, `weak_ptr` is the right tool. In inner loops where the lifetime is controlled by other means, a raw observer pointer avoids the overhead.

---

## Notes

- **Use `weak_ptr` to break cycles** - the "child -> parent" direction should be `weak_ptr`.
- **Always use `lock()`** to access the object - never rely on `expired()` then dereference (TOCTOU race).
- `weak_ptr` keeps the **control block** alive (not the object) - this matters with `make_shared` where object and control block share a single allocation.
- **Thread safety:** `lock()` is atomic and safe to call from multiple threads.
- Common patterns: observer pattern, caches, back-references in trees/graphs.
