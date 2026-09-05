# Use the Template Method pattern to define algorithm skeletons in base classes

**Category:** Modern OOP Patterns  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Template_method_pattern>  

---

## Topic Overview

The **Template Method pattern** defines an algorithm's structure in a base class and lets derived classes override specific steps without changing the overall sequence. In C++, this is typically implemented using the **Non-Virtual Interface (NVI) idiom**: a public non-virtual method calls private/protected virtual hooks.

It is worth stressing upfront: "Template Method" is a GoF design pattern name. It has nothing to do with C++ templates. The name refers to the idea that the base class method is a *template* for an algorithm - it lays out the skeleton and leaves the details for derived classes to fill in.

```cpp
┌──────────────────────────────────────┐
│  DataProcessor (base)                │
│                                      │
│  public (NON-VIRTUAL):               │
│    void process() {                  │
│      validate();    <- hook 1        │
│      transform();   <- hook 2        │
│      output();      <- hook 3        │
│    }                                 │
│                                      │
│  private (VIRTUAL):                  │
│    virtual void validate() = 0;      │
│    virtual void transform() = 0;     │
│    virtual void output() = 0;        │
└──────────────────────────────────────┘
         ^                  ^
         |                  |
┌────────┴──────┐  ┌───────┴───────┐
│  CsvProcessor │  │ JsonProcessor │
│  validate()   │  │  validate()   │
│  transform()  │  │  transform()  │
│  output()     │  │  output()     │
└───────────────┘  └───────────────┘
```

### Key Properties

| Property | Description |
| --- | --- |
| **Inversion of control** | Base class controls flow; derived provides specifics |
| **Hollywood Principle** | "Don't call us, we'll call you" |
| **NVI benefit** | Base can enforce pre/post conditions around hooks |
| **Open/Closed** | Open for extension (new derived), closed for modification (algorithm structure) |

---

## Self-Assessment

### Q1: Write a base class with a non-virtual public method that calls virtual private hooks

The public `generate()` method is the skeleton - it decides the order of steps. The three private virtual methods are the hooks that each subclass customizes. Notice that `HtmlReport` and `TextReport` cannot reorder the steps; they can only replace individual steps:

```cpp
#include <iostream>
#include <string>
#include <vector>

class Report {
public:
    // Template Method -- NON-VIRTUAL, defines the algorithm skeleton
    void generate() {
        auto data = gather_data();
        auto formatted = format(data);
        output(formatted);
    }

    virtual ~Report() = default;

private:
    // Virtual hooks -- derived classes customize these
    virtual std::vector<std::string> gather_data() = 0;
    virtual std::string format(const std::vector<std::string>& data) = 0;
    virtual void output(const std::string& result) = 0;
};

class HtmlReport : Report {  // private inheritance is fine here
public:
    using Report::generate;  // expose generate()

private:
    std::vector<std::string> gather_data() override {
        return {"Alice: 95", "Bob: 87", "Charlie: 92"};
    }

    std::string format(const std::vector<std::string>& data) override {
        std::string html = "<table>\n";
        for (const auto& row : data)
            html += "  <tr><td>" + row + "</td></tr>\n";
        html += "</table>";
        return html;
    }

    void output(const std::string& result) override {
        std::cout << "=== HTML Report ===\n" << result << "\n";
    }
};

class TextReport : Report {
public:
    using Report::generate;

private:
    std::vector<std::string> gather_data() override {
        return {"Alice: 95", "Bob: 87"};
    }

    std::string format(const std::vector<std::string>& data) override {
        std::string text;
        for (const auto& row : data)
            text += "- " + row + "\n";
        return text;
    }

    void output(const std::string& result) override {
        std::cout << "=== Text Report ===\n" << result;
    }
};

int main() {
    HtmlReport html;
    html.generate();  // calls: gather_data -> format -> output

    std::cout << "\n";

    TextReport text;
    text.generate();  // same algorithm, different hooks
}
// Expected output:
//   === HTML Report ===
//   <table>
//     <tr><td>Alice: 95</td></tr>
//     <tr><td>Bob: 87</td></tr>
//     <tr><td>Charlie: 92</td></tr>
//   </table>
//
//   === Text Report ===
//   - Alice: 95
//   - Bob: 87
```

---

### Q2: Show how NVI enforces pre/post conditions in the base class

This is where NVI really earns its keep. The derived class cannot bypass validation, cannot skip the timing measurement, and cannot mess up the commit/rollback order - those are all baked into `execute()` in the base class and the derived class simply cannot reach them:

```cpp
#include <iostream>
#include <string>
#include <chrono>
#include <stdexcept>

class Transaction {
public:
    // NVI Template Method -- enforces invariants around the hooks
    void execute() {
        // PRE-CONDITION: validate before doing anything
        if (!validate()) {
            std::cout << "  [REJECTED] Validation failed!\n";
            return;
        }
        std::cout << "  [OK] Validation passed.\n";

        // PRE-CONDITION: log start time
        auto start = std::chrono::steady_clock::now();

        // HOOK: derived-class-specific work
        do_execute();

        // POST-CONDITION: log elapsed time
        auto elapsed = std::chrono::steady_clock::now() - start;
        auto ms = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
        std::cout << "  [TIMING] Completed in " << ms << " us.\n";

        // POST-CONDITION: commit or rollback
        commit();
        std::cout << "  [COMMITTED]\n";
    }

    virtual ~Transaction() = default;

protected:
    // Optional hook -- derived can override (default: always valid)
    virtual bool validate() { return true; }
    virtual void commit() { /* default: no-op */ }

private:
    // Required hook -- derived MUST implement
    virtual void do_execute() = 0;
};

class TransferMoney : public Transaction {
    double amount_;
public:
    explicit TransferMoney(double amt) : amount_(amt) {}

protected:
    bool validate() override {
        return amount_ > 0 && amount_ <= 10000;  // limit check
    }

    void commit() override {
        std::cout << "  [DB] Transfer of $" << amount_ << " recorded.\n";
    }

private:
    void do_execute() override {
        std::cout << "  Transferring $" << amount_ << "...\n";
    }
};

int main() {
    std::cout << "Valid transfer:\n";
    TransferMoney t1(500.0);
    t1.execute();

    std::cout << "\nInvalid transfer:\n";
    TransferMoney t2(-100.0);
    t2.execute();  // rejected by validate()!
}
// Expected output:
//   Valid transfer:
//     [OK] Validation passed.
//     Transferring $500...
//     [TIMING] Completed in <N> us.
//     [DB] Transfer of $500 recorded.
//     [COMMITTED]
//
//   Invalid transfer:
//     [REJECTED] Validation failed!
```

**NVI enforcements the derived class cannot bypass:**

1. Validation always runs before `do_execute()`
2. Timing always wraps the hook
3. Commit runs only after successful execution
4. Derived classes cannot change the order or skip steps

---

### Q3: Compare Template Method with Strategy pattern for the same extensibility requirement

Both patterns solve "how do I let the caller customize one step of an algorithm?" The choice comes down to whether you need tight access to the base class state and whether the algorithm skeleton should be locked down. Template Method uses inheritance; Strategy uses composition:

```cpp
#include <iostream>
#include <functional>
#include <memory>
#include <string>
#include <algorithm>
#include <utility>
#include <vector>

// === TEMPLATE METHOD approach ===
class Sorter {
public:
    void sort(std::vector<int>& data) {
        std::cout << "Preparing data...\n";
        do_sort(data);
        std::cout << "Sort complete. Size: " << data.size() << "\n";
    }
    virtual ~Sorter() = default;
private:
    virtual void do_sort(std::vector<int>& data) = 0;
};

class BubbleSorter : public Sorter {
    void do_sort(std::vector<int>& data) override {
        // Bubble sort
        for (size_t i = 0; i < data.size(); ++i)
            for (size_t j = 0; j + 1 < data.size() - i; ++j)
                if (data[j] > data[j+1])
                    std::swap(data[j], data[j+1]);
        std::cout << "  (bubble sort used)\n";
    }
};

// === STRATEGY approach ===
class SorterStrategy {
    std::function<void(std::vector<int>&)> strategy_;
public:
    explicit SorterStrategy(std::function<void(std::vector<int>&)> s)
        : strategy_(std::move(s)) {}

    void sort(std::vector<int>& data) {
        std::cout << "Preparing data...\n";
        strategy_(data);
        std::cout << "Sort complete. Size: " << data.size() << "\n";
    }
};

int main() {
    std::vector<int> data1 = {5, 3, 8, 1, 2};

    // Template Method: requires subclassing
    BubbleSorter bs;
    bs.sort(data1);

    std::cout << "\n";

    // Strategy: uses composition + lambda
    std::vector<int> data2 = {5, 3, 8, 1, 2};
    SorterStrategy ss([](std::vector<int>& d) {
        std::sort(d.begin(), d.end());
        std::cout << "  (std::sort strategy used)\n";
    });
    ss.sort(data2);

    // Strategy can be CHANGED at runtime:
    // ss = SorterStrategy(another_algorithm);
}
// Expected output:
//   Preparing data...
//     (bubble sort used)
//   Sort complete. Size: 5
//
//   Preparing data...
//     (std::sort strategy used)
//   Sort complete. Size: 5
```

Both produce the same output. The difference is flexibility: Strategy lets you swap the algorithm at runtime without creating a new class. Template Method locks in the structure but gives derived classes access to protected members.

**Comparison:**

| Aspect | Template Method | Strategy |
| --- | --- | --- |
| **Mechanism** | Inheritance (virtual override) | Composition (`std::function`, interface) |
| **Coupling** | Tight - derived is bound to base | Loose - strategy is injected |
| **Runtime swap** | No (type is fixed at construction) | Yes (swap strategy any time) |
| **Number of classes** | One per variant (BubbleSorter, QuickSorter...) | Zero - use lambdas |
| **Access to base state** | Full (protected members) | Limited (only what's passed) |
| **Best for** | Algorithm skeletons with many hooks | Single variation point |
| **NVI/invariants** | Natural (base controls flow) | Must be enforced manually |

---

## Notes

- **Template Method != C++ templates.** The name refers to the GoF design pattern, not the language feature.
- **Protected vs Private hooks:** Use `private` virtual for hooks that should never be called directly. Use `protected` virtual when derived classes need to call the base implementation (`Base::do_step()`).
- **Default hooks:** Provide non-pure virtual hooks with reasonable defaults - derived classes only override what they need.
- **Combine both:** Use Template Method for the algorithm skeleton and inject Strategy objects for specific steps: `process() { validate(); strategy_(data); commit(); }`
- **C++20 alternative:** Use concepts to constrain hook types at compile time instead of runtime virtual dispatch.
