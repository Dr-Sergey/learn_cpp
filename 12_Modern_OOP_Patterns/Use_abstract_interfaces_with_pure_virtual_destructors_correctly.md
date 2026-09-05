# Use abstract interfaces with pure virtual destructors correctly

**Category:** Modern OOP Patterns  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/destructor>  

---

## Topic Overview

An **abstract interface** in C++ is a class with only pure virtual functions and a virtual destructor. A **pure virtual destructor** (`virtual ~Class() = 0;`) makes a class abstract even if all other functions have implementations. The key rule: a pure virtual destructor **must still have a body** (defined out-of-line) because derived class destructors implicitly call the base destructor.

This is one of those rules that seems contradictory until you understand the mechanism: "pure virtual" means "no derived class is optional in overriding it," but the *base destructor itself still runs* every time a derived object is destroyed. So the body must exist for the linker to find.

### Destructor and Polymorphic Deletion

The virtual destructor rule is about what happens when you `delete` through a base pointer. Without a virtual destructor, only the base destructor runs, and the derived class's resources are leaked.

```cpp
delete base_ptr;  where base_ptr points to Derived

WITH virtual destructor:           WITHOUT virtual destructor:
~Derived() called                  ~Base() called ONLY
  -> releases Derived resources       -> Derived resources LEAKED!
  -> then calls ~Base()               -> UNDEFINED BEHAVIOR
  -> releases Base resources
```

### When to Use What

| Scenario | Destructor Type |
| --- | --- |
| Deleted through base pointer | **public virtual** (or pure virtual) |
| Not deleted through base pointer (CRTP, mixin) | **protected non-virtual** |
| Concrete non-polymorphic class | Non-virtual (default) |

---

## Self-Assessment

### Q1: Show a memory leak when deleting a derived object through a base pointer without virtual destructor

The output here is the key part - notice that with the non-virtual destructor, only the base destructor message appears. The 1024-byte buffer allocated in `BadDerived` is never freed. No crash, no error - just a silent leak.

```cpp
#include <iostream>
#include <cstring>

// BAD: Non-virtual destructor in base class
class BadBase {
    int* data_;
public:
    BadBase() : data_(new int[10]) {
        std::cout << "  BadBase::BadBase() allocated 10 ints\n";
    }

    // BAD: Non-virtual destructor!
    ~BadBase() {
        delete[] data_;
        std::cout << "  BadBase::~BadBase() freed 10 ints\n";
    }
};

class BadDerived : public BadBase {
    char* buffer_;
public:
    BadDerived() : buffer_(new char[1024]) {
        std::cout << "  BadDerived::BadDerived() allocated 1024 chars\n";
    }

    ~BadDerived() {
        delete[] buffer_;
        std::cout << "  BadDerived::~BadDerived() freed 1024 chars\n";
    }
};

// GOOD: Virtual destructor in base class
class GoodBase {
    int* data_;
public:
    GoodBase() : data_(new int[10]) {
        std::cout << "  GoodBase::GoodBase() allocated 10 ints\n";
    }

    virtual ~GoodBase() {  // Virtual!
        delete[] data_;
        std::cout << "  GoodBase::~GoodBase() freed 10 ints\n";
    }
};

class GoodDerived : public GoodBase {
    char* buffer_;
public:
    GoodDerived() : buffer_(new char[1024]) {
        std::cout << "  GoodDerived::GoodDerived() allocated 1024 chars\n";
    }

    ~GoodDerived() override {
        delete[] buffer_;
        std::cout << "  GoodDerived::~GoodDerived() freed 1024 chars\n";
    }
};

int main() {
    std::cout << "=== BAD: Non-virtual destructor ===\n";
    {
        BadBase* ptr = new BadDerived();
        delete ptr;  // UNDEFINED BEHAVIOR!
        // Only BadBase::~BadBase() called
        // BadDerived::~BadDerived() NEVER called -> 1024 chars leaked!
    }

    std::cout << "\n=== GOOD: Virtual destructor ===\n";
    {
        GoodBase* ptr = new GoodDerived();
        delete ptr;  // Correct: calls ~GoodDerived() then ~GoodBase()
    }
}
// Expected output:
//   === BAD: Non-virtual destructor ===
//     BadBase::BadBase() allocated 10 ints
//     BadDerived::BadDerived() allocated 1024 chars
//     BadBase::~BadBase() freed 10 ints      <- ONLY base dtor! Leak!
//
//   === GOOD: Virtual destructor ===
//     GoodBase::GoodBase() allocated 10 ints
//     GoodDerived::GoodDerived() allocated 1024 chars
//     GoodDerived::~GoodDerived() freed 1024 chars  <- derived dtor called!
//     GoodBase::~GoodBase() freed 10 ints            <- then base dtor
```

The bad case doesn't crash in this example, which makes it particularly dangerous. Tools like AddressSanitizer or Valgrind will catch it, but it won't show up in normal execution.

---

### Q2: Write an abstract interface class with a pure virtual destructor provided out-of-line

The subtle rule here is that `= 0` on a destructor is not the same as `= delete`. The `= 0` means "this class is abstract and no concrete class can omit this destructor," but the body still runs on every destruction. So you define it out-of-line with an empty body (or any cleanup it needs).

```cpp
#include <iostream>
#include <memory>
#include <string>

// Abstract interface with pure virtual destructor
class ISerializable {
public:
    virtual std::string serialize() const = 0;
    virtual void deserialize(const std::string& data) = 0;

    // Pure virtual destructor - makes class abstract
    virtual ~ISerializable() = 0;
};

// MUST provide a body! Derived destructors call this implicitly.
ISerializable::~ISerializable() {
    std::cout << "  ISerializable::~ISerializable()\n";
}

// Concrete implementation
class User : public ISerializable {
    std::string name_;
    int age_;
public:
    User(std::string n, int a) : name_(std::move(n)), age_(a) {}

    ~User() override {
        std::cout << "  User::~User(" << name_ << ")\n";
    }

    std::string serialize() const override {
        return name_ + ":" + std::to_string(age_);
    }

    void deserialize(const std::string& data) override {
        auto pos = data.find(':');
        name_ = data.substr(0, pos);
        age_ = std::stoi(data.substr(pos + 1));
    }
};

int main() {
    // ISerializable s;  // ERROR: cannot instantiate abstract class

    auto user = std::make_unique<User>("Alice", 30);
    std::cout << "Serialized: " << user->serialize() << "\n";

    // Use through base pointer
    std::unique_ptr<ISerializable> iface = std::make_unique<User>("Bob", 25);
    std::cout << "Serialized: " << iface->serialize() << "\n";

    // Destruction chain:
    std::cout << "\nDestroying objects:\n";
    iface.reset();
    user.reset();
}
// Expected output:
//   Serialized: Alice:30
//   Serialized: Bob:25
//
//   Destroying objects:
//     User::~User(Bob)
//     ISerializable::~ISerializable()    <- pure virtual dtor body called!
//     User::~User(Alice)
//     ISerializable::~ISerializable()
```

**Why pure virtual destructor needs a body:**

```cpp
Derived::~Derived()
    -> ...derived cleanup...
    -> Base::~Base()         <- compiler-generated call
    -> ISerializable::~ISerializable()  <- LINKER ERROR if no body!
```

If you forget the out-of-line body, you get a linker error - not a compile error. The linker can't find the definition of `ISerializable::~ISerializable()` when it tries to generate the destruction chain for any concrete subclass.

---

### Q3: Explain when a protected non-virtual destructor is better than a public virtual destructor

If a class is never intended to be deleted through a base pointer - CRTP bases, mixins, policy classes - then a virtual destructor is unnecessary overhead. A protected non-virtual destructor gives you exactly the right contract: derived classes can destroy themselves normally, but nobody can call `delete base_ptr` on this type.

```cpp
#include <iostream>

// CRTP base - never deleted through Base*
template <typename Derived>
class Singleton {
protected:
    // Protected non-virtual: can't do "delete base_ptr"
    ~Singleton() {
        std::cout << "  ~Singleton()\n";
    }
    Singleton() = default;

public:
    // No virtual overhead - no vtable!
    static Derived& instance() {
        static Derived inst;
        return inst;
    }
};

class Logger : public Singleton<Logger> {
    friend class Singleton<Logger>;  // allow Singleton to construct
    Logger() { std::cout << "  Logger created\n"; }
public:
    ~Logger() { std::cout << "  ~Logger()\n"; }

    void log(const std::string& msg) {
        std::cout << "  [LOG] " << msg << "\n";
    }
};

// Mixin base - mixed into derived, never deleted polymorphically
class NonCopyable {
protected:
    NonCopyable() = default;
    ~NonCopyable() = default;  // protected non-virtual
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};

class Connection : public NonCopyable {
public:
    Connection() { std::cout << "  Connection opened\n"; }
    ~Connection() { std::cout << "  Connection closed\n"; }
};

int main() {
    // Logger is CRTP - never deleted through Singleton*
    Logger::instance().log("Hello");
    Logger::instance().log("World");

    // Connection is non-copyable mixin
    Connection conn;

    // NonCopyable* p = &conn;
    // delete p;  // ERROR: destructor is protected
}
// Expected output:
//   Logger created
//   [LOG] Hello
//   [LOG] World
//   Connection opened
//   Connection closed
```

**Decision matrix:**

| Question | Destructor Choice |
| --- | --- |
| Will objects be deleted through base pointer? | **public virtual** |
| Is the class a CRTP base, mixin, or policy? | **protected non-virtual** |
| Is the class `final` and concrete? | Non-virtual (or virtual, low cost) |
| Does the class have any virtual functions? | Add `virtual ~Dtor() = default;` |

---

## Notes

- **C++ Core Guideline C.35:** "A base class destructor should be either public and virtual, or protected and non-virtual."
- **Clang-Tidy** check `cppcoreguidelines-virtual-class-destructor` catches missing virtual destructors.
- **`-Wnon-virtual-dtor`** (GCC/Clang) warns when a class has virtual functions but a non-virtual destructor.
- **Cost of `virtual` destructor:** One vtable pointer per object (~8 bytes on 64-bit). For small objects this can be significant.
- **`= default` in the header** vs out-of-line definition: Pure virtual destructors REQUIRE out-of-line bodies. Regular virtual destructors can be `= default`.
- **Rule of thumb:** If a class has ANY virtual function, it should have a virtual destructor (or protected non-virtual if it's a CRTP/mixin base).
