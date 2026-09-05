# Design classes with single responsibility principle in C++

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The **Single Responsibility Principle (SRP)** states: *A class should have only one reason to change.* In practice, this means each class should own one coherent piece of domain logic. SRP violations produce "god classes" - large, fragile, untestable monsters that are painful to modify and nearly impossible to test in isolation.

The reason SRP matters so much is coupling. When a class mixes multiple concerns - say, parsing, validation, and database access - then a change to your email library touches the same class as a change to your SQL schema. Those concerns have nothing to do with each other, but now they're locked together. The fix is to separate them so each class has exactly one "axis of change."

### SRP Smell Detector

| Symptom | SRP Violation | Fix |
| --- | --- | --- |
| Class has 20+ member functions | Multiple responsibilities merged | Split into focused classes |
| `#include` list has 15+ headers | Too many dependencies | Extract collaborators |
| Class name contains "And" or "Manager" | Two concerns in one | Separate them |
| Changing logging breaks business logic | Cross-cutting concern coupled | Extract Logger |
| Can't test one feature without another | Tight coupling | Inject dependencies |

---

## Self-Assessment

### Q1: Refactor a god class into SRP-compliant components

The `UserManager` below is a classic god class: it parses JSON, validates emails, writes to a database, formats HTML, and sends email - all in one type. Any of those five things can change independently, which means this class has five reasons to change. The refactored version gives each concern its own class.

**Answer:**

```cpp
// BEFORE: God class - handles parsing, validation, storage, formatting
class UserManager {
    std::vector<User> users_;
    std::ofstream log_file_;
    sqlite3* db_;

public:
    User parseFromJson(const std::string& json) { /* parse JSON */ }
    bool validateEmail(const std::string& email) { /* regex check */ }
    void saveToDatabase(const User& u) { /* SQL INSERT */ }
    std::string formatAsHtml(const User& u) { /* HTML template */ }
    void logAction(const std::string& msg) { /* write to file */ }
    void sendWelcomeEmail(const User& u) { /* SMTP */ }
    // 20 more methods...
};

// AFTER: Each class has ONE reason to change
#include <string>
#include <vector>
#include <memory>
#include <optional>
#include <iostream>
#include <regex>

struct User {
    int id;
    std::string name;
    std::string email;
};

// Responsibility 1: Parsing
class UserParser {
public:
    std::optional<User> from_json(std::string_view json) const {
        // Parse JSON, return nullopt on failure
        User u;
        // ... parsing logic
        return u;
    }
};

// Responsibility 2: Validation
class UserValidator {
public:
    struct Result {
        bool valid;
        std::string error;
    };

    Result validate(const User& u) const {
        if (u.name.empty())
            return {false, "Name is required"};
        if (!std::regex_match(u.email, std::regex(R"([^@]+@[^@]+\.[^@]+)")))
            return {false, "Invalid email format"};
        return {true, {}};
    }
};

// Responsibility 3: Persistence
class UserRepository {
public:
    virtual ~UserRepository() = default;
    virtual void save(const User& u) = 0;
    virtual std::optional<User> find_by_id(int id) = 0;
    virtual std::vector<User> find_all() = 0;
};

class SqliteUserRepository : public UserRepository {
    // sqlite3* db_;
public:
    void save(const User& u) override { /* INSERT INTO users ... */ }
    std::optional<User> find_by_id(int id) override { return std::nullopt; }
    std::vector<User> find_all() override { return {}; }
};

// Responsibility 4: Notification
class NotificationService {
public:
    virtual ~NotificationService() = default;
    virtual void send_welcome(const User& u) = 0;
};

// Coordinator: Orchestrates but doesn't DO the work
class UserService {
    UserParser parser_;
    UserValidator validator_;
    std::unique_ptr<UserRepository> repo_;
    std::unique_ptr<NotificationService> notifier_;

public:
    UserService(std::unique_ptr<UserRepository> repo,
                std::unique_ptr<NotificationService> notifier)
        : repo_(std::move(repo)), notifier_(std::move(notifier)) {}

    bool register_user(std::string_view json) {
        auto user = parser_.from_json(json);
        if (!user) return false;

        auto validation = validator_.validate(*user);
        if (!validation.valid) {
            std::cerr << "Validation: " << validation.error << "\n";
            return false;
        }

        repo_->save(*user);
        notifier_->send_welcome(*user);
        return true;
    }
};
// Each class can now be tested independently with mocks
```

`UserService` is the coordinator - it knows the workflow, but it doesn't do any of the actual work. Its one reason to change is "the registration workflow changes." The validator's one reason to change is "the validation rules change." If you add a new validation rule, you touch exactly one class. That's the payoff.

### Q2: Explain how to identify SRP violations in existing code

You don't always see a god class immediately. Here are five concrete detection techniques to apply to any unfamiliar codebase.

**Answer:**

```cpp
// TECHNIQUE 1: Count the #includes - more than 8-10 suggests too many concerns
#include <string>      // Data
#include <vector>      // Data
#include <fstream>     // File I/O  <- different concern
#include <sqlite3.h>   // Database  <- different concern
#include <curl/curl.h> // Network   <- different concern
#include <openssl/evp.h> // Crypto  <- different concern
// -> This class does too many things

// TECHNIQUE 2: "Describe this class in one sentence without 'and'"
// BAD: "It parses AND validates AND stores AND formats users"
// GOOD: "It validates user data" - single concern

// TECHNIQUE 3: Group methods by what data they primarily touch
class Report {
    // Group A: data fetching
    void fetchFromDb();          // Uses db_
    void fetchFromApi();         // Uses http_client_

    // Group B: formatting
    std::string toHtml();        // Uses template_
    std::string toPdf();         // Uses pdf_renderer_

    // Group C: delivery
    void emailReport();          // Uses smtp_
    void uploadToS3();           // Uses aws_client_

    // -> Three groups = THREE classes: ReportFetcher, ReportFormatter, ReportDelivery
};

// TECHNIQUE 4: If mocking requires mocking unrelated things, SRP is violated
// "To test email sending, I had to set up a database mock" -> SRP violation

// TECHNIQUE 5: Change impact analysis
// "Changing the PDF library breaks the database tests" -> coupled concerns
```

Technique 3 is especially useful in practice: if you can divide a class's methods into natural groups that each touch a different subset of the member variables, those groups are your new classes. The methods that use `db_` have nothing to say about the methods that use `smtp_` - they should be separate types.

### Q3: Show SRP applied to an embedded system with hardware abstraction

SRP isn't just for enterprise software. In embedded systems it has a concrete extra benefit: by isolating the control logic from the hardware, you can test the logic on your development machine without any hardware attached at all.

**Answer:**

```cpp
#include <cstdint>
#include <functional>
#include <array>

// Embedded example: Thermostat system

// SRP class 1: Reads temperature hardware
class TemperatureReader {
    volatile uint32_t* adc_register_;
    static constexpr double SCALE = 0.01;
    static constexpr double OFFSET = -40.0;
public:
    explicit TemperatureReader(uint32_t* adc_reg) : adc_register_(adc_reg) {}
    double read_celsius() const {
        uint32_t raw = *adc_register_;
        return raw * SCALE + OFFSET;
    }
};

// SRP class 2: Implements control logic (testable without hardware!)
class ThermostatController {
    double setpoint_;
    double hysteresis_;
    bool heating_on_ = false;
public:
    ThermostatController(double setpoint, double hysteresis)
        : setpoint_(setpoint), hysteresis_(hysteresis) {}

    bool should_heat(double current_temp) {
        if (current_temp < setpoint_ - hysteresis_)
            heating_on_ = true;
        else if (current_temp > setpoint_ + hysteresis_)
            heating_on_ = false;
        return heating_on_;
    }

    void set_setpoint(double sp) { setpoint_ = sp; }
};

// SRP class 3: Controls the heater actuator
class HeaterActuator {
    volatile uint32_t* gpio_register_;
    uint32_t pin_mask_;
public:
    HeaterActuator(uint32_t* gpio, uint32_t pin) : gpio_register_(gpio), pin_mask_(1u << pin) {}
    void set(bool on) {
        if (on) *gpio_register_ |= pin_mask_;
        else    *gpio_register_ &= ~pin_mask_;
    }
};

// SRP class 4: Logs data (separate from control logic)
class DataLogger {
public:
    void log(double temp, bool heating) {
        // Write to SD card, UART, or flash
    }
};

// Coordinator: wires components together
class ThermostatApp {
    TemperatureReader reader_;
    ThermostatController controller_;
    HeaterActuator heater_;
    DataLogger logger_;
public:
    ThermostatApp(uint32_t* adc, uint32_t* gpio, uint32_t pin,
                  double setpoint, double hyst)
        : reader_(adc), controller_(setpoint, hyst), heater_(gpio, pin) {}

    void tick() {
        double temp = reader_.read_celsius();
        bool heat = controller_.should_heat(temp);
        heater_.set(heat);
        logger_.log(temp, heat);
    }
};

// Test the controller WITHOUT hardware
void test_controller() {
    ThermostatController ctrl(22.0, 0.5);
    assert(ctrl.should_heat(20.0) == true);   // Below setpoint-hysteresis
    assert(ctrl.should_heat(21.0) == true);   // Still heating (hysteresis)
    assert(ctrl.should_heat(23.0) == false);  // Above setpoint+hysteresis
    assert(ctrl.should_heat(22.0) == false);  // Still off (hysteresis)
    assert(ctrl.should_heat(21.0) == false);  // Still in band
    assert(ctrl.should_heat(20.0) == true);   // Dropped below again
}
```

The `test_controller` function at the bottom runs entirely on the host, with no ADC or GPIO register in sight. Because `ThermostatController` has no hardware dependencies at all - its one job is implementing the hysteresis logic - it's completely testable in isolation. That's SRP paying off in the most tangible way possible.

---

## Notes

- SRP doesn't mean "one function per class" - it means one *reason to change*.
- A `UserService` that orchestrates parser+validator+repo is fine - its reason to change is "the workflow changes."
- In embedded: SRP enables testing control logic on host without hardware.
- Use dependency injection to wire SRP-compliant classes together.
- Cross-cutting concerns (logging, metrics) can use the Decorator or Observer pattern instead of polluting every class.
