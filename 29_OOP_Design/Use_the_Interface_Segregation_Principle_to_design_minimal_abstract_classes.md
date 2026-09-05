# Use the Interface Segregation Principle to design minimal abstract classes

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**Interface Segregation Principle (ISP):** Clients should not be forced to depend on interfaces they don't use. In C++, this means splitting "fat" abstract base classes into small, focused interfaces. The underlying insight is simple: if you have a `ReadOnlyDocument` that is forced to implement `write()` and `fax()` just because they're on the one big interface, something has gone wrong in the design. The runtime behavior (throwing exceptions for unsupported operations) is a tell-tale sign that the interface is too broad.

The reason this trips people up is that fat interfaces feel convenient at first. You write one `IDocument` with every operation you can think of, and all document types inherit from it. Then reality arrives: a read-only report cannot write, a simple text file cannot fax, and your codebase fills up with `throw std::runtime_error("Not supported!")` stubs. ISP says: if a type can't meaningfully implement an operation, that operation should not be on the type's interface.

### Fat vs Segregated Interfaces

```cpp
FAT interface (violation):              SEGREGATED interfaces:
+---------------------+               +----------+ +----------+ +----------+
| IDevice              |               | IReadable| | IWritable| | ISeekable|
| - read()            |     ->        | - read() | | - write()| | - seek() |
| - write()           |               +----------+ +----------+ +----------+
| - seek()            |
| - flush()           |               +----------+
| - get_status()      |               |IFlushable|
| - set_baudrate()    |               | - flush()|
| - calibrate()       |               +----------+
+---------------------+
Read-only devices must implement write()=no-op -> violation
```

---

## Self-Assessment

### Q1: Refactor a fat interface into segregated interfaces

Look at the `BAD` section first: `ReadOnlyDocument` is forced to implement `write()`, `fax()`, and `encrypt()` - and all it can do is throw at runtime. That's a design problem masquerading as a runtime error. The `GOOD` section fixes this by giving each capability its own interface. Now classes implement exactly the capabilities they actually have, and the compiler catches misuse at compile time:

**Answer:**

```cpp
#include <cstdint>
#include <span>
#include <string>
#include <memory>
#include <vector>

// BAD: Fat interface - forces unnecessary implementations
class IDocument {
public:
    virtual ~IDocument() = default;
    virtual std::string read() = 0;
    virtual void write(std::string_view content) = 0;
    virtual void print() = 0;
    virtual void fax() = 0;              // Not all documents can be faxed!
    virtual void encrypt() = 0;           // Not all need encryption!
    virtual void export_pdf() = 0;        // Not all can export to PDF!
};

// ReadOnlyDocument is FORCED to implement write(), fax(), encrypt()...
class ReadOnlyDocument : public IDocument {
    void write(std::string_view) override { throw std::runtime_error("Read-only!"); }
    void fax() override { throw std::runtime_error("Not supported!"); }
    void encrypt() override { throw std::runtime_error("Not needed!"); }
    // These throw at RUNTIME - should be caught at COMPILE TIME
};

// GOOD: Segregated interfaces
class IReadable {
public:
    virtual ~IReadable() = default;
    virtual std::string read() const = 0;
};

class IWritable {
public:
    virtual ~IWritable() = default;
    virtual void write(std::string_view content) = 0;
};

class IPrintable {
public:
    virtual ~IPrintable() = default;
    virtual void print() const = 0;
};

class IEncryptable {
public:
    virtual ~IEncryptable() = default;
    virtual void encrypt(std::span<const uint8_t> key) = 0;
    virtual void decrypt(std::span<const uint8_t> key) = 0;
};

class IExportable {
public:
    virtual ~IExportable() = default;
    virtual std::vector<uint8_t> export_pdf() const = 0;
};

// Now each class implements ONLY what it supports:
class TextDocument : public IReadable, public IWritable, public IPrintable {
    std::string content_;
public:
    std::string read() const override { return content_; }
    void write(std::string_view c) override { content_ = c; }
    void print() const override { /* send to printer */ }
};

class SecureDocument : public IReadable, public IWritable, public IEncryptable {
    std::string content_;
    bool encrypted_ = false;
public:
    std::string read() const override { return content_; }
    void write(std::string_view c) override { content_ = c; }
    void encrypt(std::span<const uint8_t> key) override { encrypted_ = true; }
    void decrypt(std::span<const uint8_t> key) override { encrypted_ = false; }
};

class ReadOnlyReport : public IReadable, public IPrintable, public IExportable {
    std::string data_;
public:
    explicit ReadOnlyReport(std::string data) : data_(std::move(data)) {}
    std::string read() const override { return data_; }
    void print() const override { /* print report */ }
    std::vector<uint8_t> export_pdf() const override { return {}; }
    // NO write, NO encrypt - compile-time enforcement!
};

// Functions accept only the interface they need:
void backup(const IReadable& doc) {       // Only needs to read
    auto content = doc.read();
}

void edit(IWritable& doc) {               // Only needs to write
    doc.write("Updated content");
}

void secure_and_store(IReadable& r, IEncryptable& e) {
    auto data = r.read();
    uint8_t key[] = {1,2,3,4};
    e.encrypt(key);
}
```

Notice that `backup` only takes an `IReadable`. It doesn't know whether the document is also writable or encryptable, and it doesn't need to. Each function declares its dependencies exactly, which also makes the code much easier to test - you can mock just `IReadable` without dragging in the full document implementation.

### Q2: Explain ISP in the context of embedded C++ hardware interfaces

Embedded systems make a particularly strong case for ISP. A simple thermistor has one job: read a temperature. If it's forced to implement `calibrate()`, `enable_interrupt()`, and `sleep()` just because a fat sensor interface demands them, the firmware is littered with stub implementations that silently do nothing or throw errors. The segregated version means a function that calls `sleep()` on all power-managed devices simply cannot be passed a thermistor - the compiler enforces it. Here is the contrast:

**Answer:**

```cpp
#include <cstdint>

// BAD: One fat sensor interface
class ISensor {
public:
    virtual ~ISensor() = default;
    virtual double read() = 0;
    virtual void calibrate(double ref) = 0;  // Some sensors can't be calibrated!
    virtual void set_range(int range) = 0;   // Some have fixed range!
    virtual void enable_interrupt() = 0;      // Some are polled only!
    virtual void sleep() = 0;                 // Some can't sleep!
};

// GOOD: Segregated sensor interfaces
class IReadable_Sensor {
public:
    virtual ~IReadable_Sensor() = default;
    virtual double read() = 0;
};

class ICalibratable {
public:
    virtual ~ICalibratable() = default;
    virtual bool calibrate(double reference) = 0;
};

class IConfigurable {
public:
    virtual ~IConfigurable() = default;
    virtual void set_range(int range) = 0;
    virtual int get_range() const = 0;
};

class IInterruptDriven {
public:
    virtual ~IInterruptDriven() = default;
    using Callback = void(*)(double value, void* ctx);
    virtual void enable_interrupt(Callback cb, void* ctx) = 0;
    virtual void disable_interrupt() = 0;
};

class IPowerManaged {
public:
    virtual ~IPowerManaged() = default;
    virtual void sleep() = 0;
    virtual void wake() = 0;
    virtual double power_consumption_mw() const = 0;
};

// Concrete sensors implement only applicable interfaces:
class SimpleThermistor : public IReadable_Sensor {
    volatile uint32_t* adc_;
public:
    explicit SimpleThermistor(uint32_t* adc) : adc_(adc) {}
    double read() override { return (*adc_) * 0.01 - 40.0; }
    // No calibration, no interrupts, no power management
};

class AccelerometerIMU : public IReadable_Sensor,
                         public ICalibratable,
                         public IConfigurable,
                         public IInterruptDriven,
                         public IPowerManaged {
    int range_ = 2;
public:
    double read() override { return 0.0; /* read SPI */ }
    bool calibrate(double ref) override { return true; }
    void set_range(int r) override { range_ = r; }
    int get_range() const override { return range_; }
    void enable_interrupt(Callback cb, void* ctx) override { /* configure EXTI */ }
    void disable_interrupt() override { /* disable EXTI */ }
    void sleep() override { /* set low-power mode register */ }
    void wake() override { /* clear low-power bit */ }
    double power_consumption_mw() const override { return 0.5; }
};

// Power management code only depends on IPowerManaged:
void enter_low_power(std::vector<IPowerManaged*>& devices) {
    for (auto* d : devices) d->sleep();
}
// SimpleThermistor is NOT passed here - compile-time safety
```

The `enter_low_power` function is a perfect example of ISP in action: it only holds pointers to `IPowerManaged`. You physically cannot pass a `SimpleThermistor` to it, even by mistake. That safety is free - it comes from the design, not from runtime checks.

### Q3: Show ISP with concepts (C++20) as an alternative to abstract classes

C++20 concepts give you ISP with zero overhead - no vtable, no indirection, no pointer. A concept is just a compile-time check on what operations a type supports. Functions constrained by concepts communicate their requirements clearly and the compiler enforces them, just like interfaces do, but with no runtime cost. The concept-based approach also works with types that never heard of your interfaces - they just need to have the right operations:

**Answer:**

```cpp
#include <concepts>
#include <string>
#include <iostream>
#include <span>
#include <string_view>
#include <utility>
#include <vector>

// ISP with concepts - zero-overhead, compile-time interfaces

template<typename T>
concept Readable = requires(const T& t) {
    { t.read() } -> std::convertible_to<std::string>;
};

template<typename T>
concept Writable = requires(T& t, std::string_view sv) {
    { t.write(sv) };
};

template<typename T>
concept Serializable = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::vector<uint8_t>>;
    { T::deserialize(std::declval<std::span<const uint8_t>>()) } -> std::same_as<T>;
};

// Functions constrained to exactly the interface they need:
void backup(const Readable auto& doc) {
    auto content = doc.read();
    // Only requires read() - nothing else
}

void update(Writable auto& doc) {
    doc.write("new content");
}

void sync(Readable auto& r, Writable auto& w) {
    w.write(r.read());
}

// Concrete types - no inheritance needed!
struct PlainTextFile {
    std::string content;
    std::string read() const { return content; }
    void write(std::string_view s) { content = s; }
};

struct ConfigFile {
    std::string data;
    std::string read() const { return data; }
    // No write() - it's read-only
};

int main() {
    PlainTextFile f{"hello"};
    ConfigFile c{"key=value"};

    backup(f);   // OK: PlainTextFile satisfies Readable
    backup(c);   // OK: ConfigFile satisfies Readable
    update(f);   // OK: PlainTextFile satisfies Writable
    // update(c); // COMPILE ERROR: ConfigFile doesn't satisfy Writable
    return 0;
}
```

The concept-based approach has one big advantage over virtual interfaces: completely unrelated types that just happen to have the right methods will satisfy the concept. You don't need to modify `PlainTextFile` to opt it into `Readable` - if it has a `read()` method returning something string-like, it just works. This is sometimes called duck typing enforced at compile time, and it is one of the nicest things about concepts.

---

## Notes

- ISP in C++ means preferring many small abstract base classes with 2 to 4 methods over one large monolithic one - smaller interfaces are easier to implement correctly and easier to compose.
- With C++20 concepts, ISP is enforced at compile time with zero overhead - no vtable, no virtual dispatch, just a constraint check during template instantiation.
- Fat interfaces force implementers to throw `std::runtime_error("Not supported")` - that's a design smell, not a feature. The capability should not be on the interface at all if some implementers can't provide it.
- In embedded systems, ISP lets you write power management, calibration, and interrupt-handling code that accepts only devices that actually have those capabilities - making unsupported operations a compile error rather than a silent no-op.
- Multiple inheritance of pure interfaces is perfectly fine in C++ (unlike some other languages) - there's no diamond problem when the interfaces carry no data and no shared state.
