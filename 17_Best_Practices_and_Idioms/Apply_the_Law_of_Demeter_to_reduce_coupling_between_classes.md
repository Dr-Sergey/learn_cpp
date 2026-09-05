# Apply the Law of Demeter to reduce coupling between classes

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Law_of_Demeter>  

---

## Topic Overview

The **Law of Demeter** (LoD), also known as the **principle of least knowledge**, states: *a method should only call methods on objects it directly knows about*. Don't "reach through" an object to access its internal collaborators.

### The Rule

A method `f()` of class `C` can call methods on:

1. `C` itself (`this->method()`)
2. `f`'s parameters
3. Objects created within `f`
4. `C`'s direct member objects

**NOT:** objects returned by other calls ("train wrecks").

Here's the difference visualised. The violation chains through three layers of indirection; the fix asks only one object for what you need:

```cpp
Violation:        order.getCustomer().getAddress().getCity()
                  \___/  \_________/  \________/  \_____/
                  you      reach       reach       reach
                  know     through     through     through

Fixed:            order.getShippingCity()
                  \___/  \_____________/
                  you    directly ask
```

The "train wreck" name is fitting: each `.` is another car in a chain, and if any one of them changes, the whole chain derails.

---

## Self-Assessment

### Q1: Identify a chain of calls `a.b().c().d()` and explain why it creates tight coupling

The bad version below reaches through three types to get a single city string. That means this one function is coupled to `Order`, `Customer`, *and* `Address` - three classes that must all stay stable for it to compile:

```cpp
#include <iostream>
#include <string>

// PROBLEM: deep coupling through method chains
class Address {
public:
    std::string city() const { return city_; }
    std::string city_ = "Seattle";
};

class Customer {
public:
    const Address& address() const { return addr_; }
    Address addr_;
};

class Order {
public:
    const Customer& customer() const { return cust_; }
    Customer cust_;
};

void print_shipping_label_bad(const Order& order) {
    // Train wreck: knows about Order -> Customer -> Address -> city
    std::string city = order.customer().address().city();
    //                 ^------- violation: 3 levels deep
    std::cout << "Ship to: " << city << '\n';
}
// WHY IS THIS BAD?
// - print_shipping_label_bad depends on Order, Customer, AND Address
// - If Address changes (e.g., city() -> city_name()), this code breaks
// - If Customer stores address differently, this code breaks
// - 3 classes coupled instead of 1

int main() {
    Order order;
    print_shipping_label_bad(order);
}
// Expected output:
// Ship to: Seattle
```

Any time someone refactors `Customer` to store addresses differently - or renames a method on `Address` - this function breaks even though it's supposed to be about printing a shipping label, not about the internals of customer data management.

### Q2: Refactor to expose a higher-level method that hides the chain

The fix is to push the navigation responsibility down into the types that own the data. `Customer` knows about `Address`, so `Customer` gets `shipping_city()`. `Order` knows about `Customer`, so `Order` gets `shipping_city()` too. Each class only delegates one level down:

```cpp
#include <iostream>
#include <string>

class Address {
public:
    std::string city() const { return city_; }
    std::string city_ = "Seattle";
};

class Customer {
public:
    // Customer wraps Address details
    std::string shipping_city() const { return addr_.city(); }
private:
    Address addr_;
};

class Order {
public:
    // Order exposes what callers actually need
    std::string shipping_city() const { return cust_.shipping_city(); }
private:
    Customer cust_;
};

void print_shipping_label_good(const Order& order) {
    // Only talks to Order directly
    std::string city = order.shipping_city();
    std::cout << "Ship to: " << city << '\n';
}
// Now:
// - print_shipping_label_good depends only on Order
// - Customer's internal structure can change freely
// - Address could be replaced entirely - print_shipping_label is unaffected

int main() {
    Order order;
    print_shipping_label_good(order);
}
// Expected output:
// Ship to: Seattle
```

Now `print_shipping_label_good` depends on exactly one type: `Order`. You can change everything about how customers store addresses and this function won't even need recompiling.

### Q3: Explain when chaining (builder pattern, fluent interface) is intentional and acceptable

**Chaining is NOT a LoD violation when the calls are on the *same* object:**

```cpp
#include <iostream>
#include <string>
#include <sstream>

// ACCEPTABLE: builder pattern - every call returns *this
class QueryBuilder {
    std::string table_, where_, order_;
public:
    QueryBuilder& from(std::string t) { table_ = t; return *this; }
    QueryBuilder& where(std::string w) { where_ = w; return *this; }
    QueryBuilder& order_by(std::string o) { order_ = o; return *this; }

    std::string build() const {
        return "SELECT * FROM " + table_ +
               " WHERE " + where_ +
               " ORDER BY " + order_;
    }
};

int main() {
    // Every method returns the SAME object - no reaching through
    auto query = QueryBuilder()
        .from("users")
        .where("age > 18")    // returns *this, not a different object
        .order_by("name")
        .build();

    std::cout << query << '\n';

    // Also acceptable: std::ostream chaining
    std::ostringstream oss;
    oss << "Hello" << ' ' << "World";  // each << returns the same stream
    std::cout << oss.str() << '\n';
}
// Expected output:
// SELECT * FROM users WHERE age > 18 ORDER BY name
// Hello World
```

The reason builder chains are fine is that every call in the chain returns the *same* object - there's no "reaching through." You're not navigating into collaborators; you're reconfiguring the builder itself. The Law of Demeter is about coupling to foreign objects, not about the number of method calls you write on one line.

**When chaining is OK vs not:**

| Pattern | OK? | Reason |
| --- | --- | --- |
| `builder.a().b().c()` - returns `*this` | Yes | Same object throughout |
| `stream << a << b << c` - returns `stream&` | Yes | Same object throughout |
| `order.customer().address().city()` | No | Different objects at each level |
| `ranges::view \| filter \| transform` | Yes | DSL, each step builds a view |

---

## Notes

- LoD is a **guideline**, not a rigid law. Applying it everywhere can lead to bloated wrapper methods.
- Data objects (DTOs, structs) are generally exempt - their purpose IS direct data access.
- The real goal is **minimal coupling**: code should depend on as few types as possible.
- In C++ templates, LoD is less applicable - templates work with any type that satisfies concepts.
