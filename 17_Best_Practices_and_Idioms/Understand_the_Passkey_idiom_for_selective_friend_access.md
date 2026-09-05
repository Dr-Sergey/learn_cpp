# Understand the Passkey idiom for selective friend access

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Passkey>  

---

## Topic Overview

The **Passkey idiom** restricts who can call a public function by requiring a token object that only authorized classes can create.

The problem with a plain `friend` declaration is that it is all-or-nothing: once you declare a class as a friend, it has access to everything private. The Passkey idiom gives you surgical precision - you keep the method public, but you add a parameter that acts as a key. Only the class that holds the matching key can actually call the function.

### The Problem

Here is the blunt-instrument approach that many people reach for first:

```cpp
class Server {
    friend class Admin;  // gives Admin access to EVERYTHING private
};
// Too much power! Admin can access ALL private members.
```

### The Solution: Passkey

With Passkey, `shutdown` and `restart` are public methods, but calling them requires a `Passkey<Admin>` token - which only `Admin` can construct:

```cpp
class Server {
public:
    void shutdown(Passkey<Admin>);  // public, but only Admin can call it
    void restart(Passkey<Admin>);   // because only Admin can create Passkey<Admin>
    int status() const;             // anyone can call (no passkey needed)
};
```

---

## Self-Assessment

### Q1: Implement a `Passkey<T>` that only `T` can construct

The whole mechanism rests on one trick: `Passkey<T>` has a private constructor, and its only friend is `T`. So `T` can create the passkey, and nobody else can - not even types that know about `Passkey`.

```cpp
#include <iostream>
#include <string>

// The Passkey: only T can construct it
template<typename T>
class Passkey {
    friend T;  // only T can access private constructor
    Passkey() {}                          // private!
    Passkey(const Passkey&) = default;    // private!
};

// Server has restricted methods
class Admin;  // forward declaration

class Server {
public:
    void shutdown(Passkey<Admin>) {
        std::cout << "Server shutting down...\n";
    }

    void restart(Passkey<Admin>) {
        std::cout << "Server restarting...\n";
    }

    std::string status() const {  // no passkey -- anyone can call
        return "running";
    }
};

class Admin {
public:
    void shutdown_server(Server& s) {
        s.shutdown(Passkey<Admin>{});  // OK: Admin can create Passkey<Admin>
    }

    void restart_server(Server& s) {
        s.restart(Passkey<Admin>{});   // OK
    }
};

class User {
public:
    void try_shutdown(Server& s) {
        // s.shutdown(Passkey<Admin>{});  // ERROR: Passkey<Admin> ctor is private!
        // s.shutdown({});                // ERROR: can't construct Passkey<Admin>
        std::cout << "User cannot shutdown. Status: " << s.status() << '\n';
    }
};

int main() {
    Server server;
    Admin admin;
    User user;

    user.try_shutdown(server);     // Can only check status
    admin.shutdown_server(server); // Can shut down
    admin.restart_server(server);  // Can restart
}
// Expected output:
// User cannot shutdown. Status: running
// Server shutting down...
// Server restarting...
```

Notice that `User` cannot even call `s.shutdown({})` with a brace-init - there is no implicit conversion path available. The enforcement is airtight at compile time.

### Q2: Show that Passkey prevents accidental access without fully private constructors

You can also use Passkey to restrict *construction* of an object while keeping the constructor technically public. This is handy when `std::make_unique` or similar helpers need to call the constructor, but you don't want arbitrary code doing the same.

```cpp
#include <iostream>

template<typename T>
class Passkey {
    friend T;
    Passkey() {}
    Passkey(const Passkey&) = default;
};

// Factory is the only one allowed to create Widget
class Factory;

class Widget {
    int id_;
public:
    // Public constructor, but requires Passkey<Factory>
    Widget(int id, Passkey<Factory>) : id_(id) {
        std::cout << "Widget " << id_ << " created\n";
    }

    void use() const { std::cout << "Using widget " << id_ << '\n'; }
};

class Factory {
public:
    Widget create(int id) {
        return Widget(id, Passkey<Factory>{});  // OK: Factory can create passkey
    }
};

int main() {
    Factory f;
    Widget w = f.create(42);  // OK
    w.use();

    // Widget w2(99, Passkey<Factory>{});  // ERROR: can't create Passkey<Factory>
    // Widget w3(99, {});                  // ERROR: can't brace-init Passkey<Factory>
    std::cout << "Direct construction blocked!\n";
}
// Expected output:
// Widget 42 created
// Using widget 42
// Direct construction blocked!
```

This is a common pattern when you want a factory to be the single point of creation but still need the constructor accessible for value-construction inside the factory itself.

### Q3: Compare Passkey with the Attorney-Client idiom

There is another idiom that solves a related but different problem: the Attorney-Client idiom. It uses an intermediate class to expose *specific private members* to specific callers, rather than restricting who can call a *public* method.

```cpp
#include <iostream>

// Client has private internals
class Client {
    int secret_ = 42;
    void private_method() { std::cout << "Private method!\n"; }

    friend class Attorney;  // Attorney is the only friend
};

// Attorney exposes ONLY what's needed to specific users
class Attorney {
public:
    // Only expose secret_, not private_method()
    static int get_secret(const Client& c) { return c.secret_; }
};

int main() {
    Client c;
    std::cout << "Secret: " << Attorney::get_secret(c) << '\n';
    // Attorney chose to expose secret_ but NOT private_method()
}
// Expected output:
// Secret: 42
```

The Attorney approach requires a dedicated middleman class for each "client" you want to grant access to. Passkey scales more naturally - you just add a `Passkey<NewCaller>` parameter to each function you want to open up.

**Comparison:**

| Feature | Passkey | Attorney-Client |
| --- | --- | --- |
| Restricts who calls | Public functions | Private functions (via Attorney) |
| Granularity | Per-function (each takes a passkey) | Per-member (Attorney chooses what to expose) |
| Coupling | Low (only need forward decl) | Medium (Attorney friends Client) |
| Template? | Yes (`Passkey<T>`) | No |
| C++ version | Works in C++11+ | Works in C++98+ |
| Use case | "Only X should call this public method" | "Expose specific private members to specific users" |

---

## Notes

- Passkey is a zero-cost abstraction - the empty Passkey object is optimized away.
- Can combine multiple passkeys: `void method(Passkey<A>, Passkey<B>)` - both A and B must authorize.
- In C++20, you can use concepts to constrain passkey construction even further.
- Passkey is better than `friend` because `friend` grants access to ALL private members.
