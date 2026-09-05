# Implement hexagonal architecture (ports and adapters) in C++

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Hexagonal architecture** (also called ports and adapters) is a design strategy that keeps your core business logic completely isolated from the outside world - no database headers in your domain code, no HTTP types in your use cases, nothing. Instead, the core defines **ports**, which are pure abstract interfaces describing what it needs or offers. External systems plug in as **adapters** that implement those interfaces.

The reason this inverts the usual dependency direction is the key insight. In a traditional layered architecture, the business logic reaches down to call the database. In hexagonal, the database adapter reaches up to implement the core's interface. That means the database depends on the core, not the other way around. Swap out the database, and the core does not need to change at all.

### Architecture Diagram

```cpp
              +--- Driving Adapters ---+
              |  REST API  |  CLI  |   |
              +------+-----+---+------+
                     |         |
              +------v---------v------+
              |    Driving Ports       |
              |  (Input Interfaces)    |
              +------------------------+
              |                        |
              |    APPLICATION CORE    |
              |   (Domain + Use Cases) |
              |                        |
              +------------------------+
              |    Driven Ports         |
              |  (Output Interfaces)   |
              +------+---------+------+
                     |         |
              +------v---------v------+
              | SQLite  |  MQTT  | FS  |
              +--- Driven Adapters ---+
```

There are two kinds of ports, and understanding the difference is the key to placing things correctly:

| Concept | Role | Direction |
| --- | --- | --- |
| **Driving Port** | Interface the outside world calls | Outside -> Core |
| **Driving Adapter** | Implements external trigger (HTTP, CLI) | Translates to port call |
| **Driven Port** | Interface the core needs from outside | Core -> Outside |
| **Driven Adapter** | Implements external resource (DB, queue) | Implements port |

Driving ports are how the outside world talks to your core. Driven ports are what your core needs from the outside world. If you remember that split, the whole pattern clicks into place.

---

## Self-Assessment

### Q1: Implement hexagonal architecture with ports and adapters

Let's walk through a complete example with a user management service. The directory structure alone communicates the architecture - notice that `core/` has zero dependency on anything in `adapters/`. The dependency arrows all point inward.

**Answer:**

```cpp
project/
├── core/                    # Application core (no external deps)
│   ├── domain/
│   │   ├── user.h
│   │   └── user_id.h
│   ├── ports/
│   │   ├── driving/         # Input interfaces
│   │   │   └── i_user_service.h
│   │   └── driven/          # Output interfaces
│   │       ├── i_user_repository.h
│   │       └── i_notification_sender.h
│   └── use_cases/
│       └── user_service.h
├── adapters/
│   ├── driving/             # Input adapters
│   │   ├── rest_controller.h
│   │   └── cli_adapter.h
│   └── driven/              # Output adapters
│       ├── postgres_user_repo.h
│       ├── in_memory_user_repo.h
│       └── email_notification.h
└── main.cpp                 # Composition root
```

Now here is the code that fills in those files. The domain types are plain structs with no dependencies. The driven ports (`IUserRepository`, `INotificationSender`) are abstract interfaces that the core calls outward through. The driving port (`IUserService`) is the abstract interface external callers use to reach the core. The use case implementation (`UserService`) is where the actual business logic lives:

```cpp
// === core/domain/user.h ===
#pragma once
#include <string>

struct UserId {
    int value;
    bool operator==(const UserId&) const = default;
};

struct User {
    UserId id;
    std::string name;
    std::string email;
    bool active = true;
};

// === core/ports/driven/i_user_repository.h (Driven Port) ===
#pragma once
#include "core/domain/user.h"
#include <optional>
#include <vector>

class IUserRepository {
public:
    virtual ~IUserRepository() = default;
    virtual void save(const User& user) = 0;
    virtual std::optional<User> find_by_id(UserId id) = 0;
    virtual std::vector<User> find_all() = 0;
    virtual void remove(UserId id) = 0;
};

// === core/ports/driven/i_notification_sender.h (Driven Port) ===
#pragma once
#include <string>

class INotificationSender {
public:
    virtual ~INotificationSender() = default;
    virtual void send(const std::string& to, const std::string& message) = 0;
};

// === core/ports/driving/i_user_service.h (Driving Port) ===
#pragma once
#include "core/domain/user.h"
#include <optional>
#include <string>

class IUserService {
public:
    virtual ~IUserService() = default;
    virtual User create_user(const std::string& name,
                              const std::string& email) = 0;
    virtual std::optional<User> get_user(UserId id) = 0;
    virtual void deactivate_user(UserId id) = 0;
};

// === core/use_cases/user_service.h (Implementation of driving port) ===
#pragma once
#include "core/ports/driving/i_user_service.h"
#include "core/ports/driven/i_user_repository.h"
#include "core/ports/driven/i_notification_sender.h"

class UserService : public IUserService {
public:
    UserService(IUserRepository& repo, INotificationSender& notifier)
        : repo_(repo), notifier_(notifier) {}

    User create_user(const std::string& name,
                      const std::string& email) override {
        User user{UserId{next_id_++}, name, email, true};
        repo_.save(user);
        notifier_.send(email, "Welcome, " + name + "!");
        return user;
    }

    std::optional<User> get_user(UserId id) override {
        return repo_.find_by_id(id);
    }

    void deactivate_user(UserId id) override {
        auto user = repo_.find_by_id(id);
        if (!user) throw std::runtime_error("User not found");
        user->active = false;
        repo_.save(*user);
        notifier_.send(user->email, "Your account has been deactivated.");
    }

private:
    IUserRepository& repo_;
    INotificationSender& notifier_;
    int next_id_ = 1;
};
```

`UserService` only knows about `IUserRepository` and `INotificationSender`. It has no idea whether the repository is a Postgres database, an in-memory hash map, or a mock. That indirection is the whole payoff.

### Q2: Implement adapters and compose at the application entry point

The adapters live outside the core and implement the core's interfaces. The composition root - `main.cpp` - is the only place in the entire codebase that knows the concrete types. Every other file sees only interfaces.

**Answer:**

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// === adapters/driven/in_memory_user_repo.h ===
#pragma once
#include "core/ports/driven/i_user_repository.h"
#include <unordered_map>

class InMemoryUserRepo : public IUserRepository {
public:
    void save(const User& user) override { users_[user.id.value] = user; }

    std::optional<User> find_by_id(UserId id) override {
        auto it = users_.find(id.value);
        return it != users_.end() ? std::optional{it->second} : std::nullopt;
    }

    std::vector<User> find_all() override {
        std::vector<User> result;
        for (auto& [_, user] : users_) result.push_back(user);
        return result;
    }

    void remove(UserId id) override { users_.erase(id.value); }

private:
    std::unordered_map<int, User> users_;
};

// === adapters/driven/console_notification.h ===
#pragma once
#include "core/ports/driven/i_notification_sender.h"
#include <iostream>

class ConsoleNotification : public INotificationSender {
public:
    void send(const std::string& to, const std::string& msg) override {
        std::cout << "[Notify " << to << "] " << msg << "\n";
    }
};

// === main.cpp: Composition Root ===
#include "core/use_cases/user_service.h"
#include "adapters/driven/in_memory_user_repo.h"
#include "adapters/driven/console_notification.h"

int main() {
    // Wire up adapters - this is the only place that knows all concrete types
    InMemoryUserRepo repo;
    ConsoleNotification notifier;
    UserService service(repo, notifier);

    // Use only through the port interface
    IUserService& user_api = service;

    auto user = user_api.create_user("Alice", "alice@example.com");
    auto found = user_api.get_user(user.id);
}
```

When you later want to swap `InMemoryUserRepo` for a `PostgresUserRepo`, you create the new adapter file, change exactly one line in `main.cpp`, and nothing in the core touches. That is the whole promise of hexagonal architecture delivered in practice.

### Q3: Test the core without any adapters

This is where hexagonal architecture pays off most visibly. You can write a complete test of your business logic with zero real infrastructure. No database running, no email server, nothing - just mock adapters injected at construction.

**Answer:**

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "core/use_cases/user_service.h"

class MockRepo : public IUserRepository {
public:
    MOCK_METHOD(void, save, (const User&), (override));
    MOCK_METHOD(std::optional<User>, find_by_id, (UserId), (override));
    MOCK_METHOD(std::vector<User>, find_all, (), (override));
    MOCK_METHOD(void, remove, (UserId), (override));
};

class MockNotifier : public INotificationSender {
public:
    MOCK_METHOD(void, send, (const std::string&, const std::string&), (override));
};

TEST(UserServiceTest, CreateUserSavesAndNotifies) {
    MockRepo repo;
    MockNotifier notifier;
    UserService service(repo, notifier);

    EXPECT_CALL(repo, save(testing::_)).Times(1);
    EXPECT_CALL(notifier, send("alice@test.com", testing::HasSubstr("Welcome")));

    auto user = service.create_user("Alice", "alice@test.com");
    EXPECT_EQ(user.name, "Alice");
    EXPECT_TRUE(user.active);
}

TEST(UserServiceTest, DeactivateNotifiesUser) {
    MockRepo repo;
    MockNotifier notifier;
    UserService service(repo, notifier);

    User existing{UserId{1}, "Bob", "bob@test.com", true};
    EXPECT_CALL(repo, find_by_id(UserId{1})).WillOnce(testing::Return(existing));
    EXPECT_CALL(repo, save(testing::_));
    EXPECT_CALL(notifier, send("bob@test.com", testing::HasSubstr("deactivated")));

    service.deactivate_user(UserId{1});
}
```

These tests run in milliseconds and exercise every branch of the business logic. The mock objects verify not just that the code runs, but that it interacted with its dependencies in exactly the right way - saving to the repo, sending the right notification to the right address.

---

## Notes

- The core has zero external dependencies - no database, no network, no framework includes.
- Ports are interfaces defined in the core; adapters are implementations outside the core.
- The composition root (main.cpp) is the only place that knows all concrete types.
- Swapping from SQLite to PostgreSQL means writing a new driven adapter - the core doesn't change.
- Testing the core requires only mock adapters - no real databases or networks.
- Hexagonal is more flexible than layered: any external system is just an adapter.
- Common mistake: putting domain logic in adapters - keep business rules in the core.
