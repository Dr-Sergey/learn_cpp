# Apply the Mediator pattern to reduce coupling between subsystems

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

The **Mediator pattern** centralizes communication between subsystems so they do not reference each other directly. Instead of N×N direct dependencies, each subsystem communicates through a mediator, resulting in N×1 dependencies. This is essential in large C++ projects where subsystems (audio, physics, UI, networking) must interact without tight coupling.

The mental model that helps here: imagine a busy airport. Without a control tower, every pilot would have to talk directly to every other pilot before making a move. With a control tower (the mediator), each pilot only needs one communication channel. The control tower knows about all the planes; the planes know only about the control tower.

### Direct Coupling vs Mediator

| Aspect | Direct Coupling | Mediator |
| --- | --- | --- |
| Dependencies | N×(N-1) pairwise | N×1 (each to mediator) |
| Adding subsystem | Touch all related subsystems | Only touch mediator |
| Testing | Must mock all peers | Mock only mediator |
| Compile time | Headers pull in everything | Subsystems independent |
| Risk | Spaghetti dependencies | Mediator becomes god object |

---

## Self-Assessment

### Q1: Implement a mediator for subsystem communication

**Answer:**

The approach here is a type-safe event bus. Each subsystem subscribes to the event types it cares about and publishes the events it produces. Critically, no subsystem holds a reference to any other - they only hold a reference to the mediator:

```cpp
#include <string>
#include <unordered_map>
#include <functional>
#include <vector>
#include <any>
#include <typeindex>
#include <memory>

// === Event types ===
struct PlayerDiedEvent {
    int player_id;
    float x, y;
};

struct SoundRequestEvent {
    std::string sound_name;
    float volume;
};

struct UINotificationEvent {
    std::string message;
    int priority;
};

struct ScoreChangedEvent {
    int player_id;
    int new_score;
};

// === Mediator: type-safe event bus ===
class Mediator {
public:
    template<typename Event>
    using Handler = std::function<void(const Event&)>;

    template<typename Event>
    void subscribe(Handler<Event> handler) {
        auto& handlers = handlers_[typeid(Event)];
        handlers.push_back(
            [handler](const std::any& e) {
                handler(std::any_cast<const Event&>(e));
            });
    }

    template<typename Event>
    void publish(const Event& event) {
        auto it = handlers_.find(typeid(Event));
        if (it != handlers_.end()) {
            for (auto& handler : it->second) {
                handler(event);
            }
        }
    }

private:
    using AnyHandler = std::function<void(const std::any&)>;
    std::unordered_map<std::type_index, std::vector<AnyHandler>> handlers_;
};

// === Subsystems: no knowledge of each other ===
class AudioSystem {
public:
    explicit AudioSystem(Mediator& mediator) {
        mediator.subscribe<SoundRequestEvent>(
            [this](const SoundRequestEvent& e) {
                play_sound(e.sound_name, e.volume);
            });
        mediator.subscribe<PlayerDiedEvent>(
            [this](const PlayerDiedEvent&) {
                play_sound("death", 1.0f);
            });
    }
private:
    void play_sound(const std::string& name, float vol) {
        // Audio implementation
    }
};

class UISystem {
public:
    explicit UISystem(Mediator& mediator) : mediator_(mediator) {
        mediator.subscribe<UINotificationEvent>(
            [this](const UINotificationEvent& e) {
                show_notification(e.message, e.priority);
            });
        mediator.subscribe<ScoreChangedEvent>(
            [this](const ScoreChangedEvent& e) {
                update_scoreboard(e.player_id, e.new_score);
            });
    }
private:
    Mediator& mediator_;
    void show_notification(const std::string&, int) { }
    void update_scoreboard(int, int) { }
};

class GameLogic {
public:
    explicit GameLogic(Mediator& mediator) : mediator_(mediator) {}

    void kill_player(int id, float x, float y) {
        // Business logic
        // ...

        // Notify other subsystems via mediator (no direct deps)
        mediator_.publish(PlayerDiedEvent{id, x, y});
        mediator_.publish(UINotificationEvent{"Player killed!", 1});
        mediator_.publish(ScoreChangedEvent{id, 0});
    }

private:
    Mediator& mediator_;
};
```

Notice that `GameLogic` has no idea that `AudioSystem` or `UISystem` exist. It just fires events. The mediator takes care of delivery. You can add a new `ReplaySystem` that records `PlayerDiedEvent` without touching `GameLogic` at all - just subscribe it to the mediator.

### Q2: Priority-based and filtered event dispatch

**Answer:**

A basic event bus dispatches to all subscribers in registration order. In production systems you often need more control: a security audit handler should run before any business handler, and a VIP notification handler should only fire for certain customers. Here is how to add priority and filtering without breaking the subscriber interface:

```cpp
// === Advanced mediator with priority and filtering ===
class PriorityMediator {
public:
    template<typename Event>
    struct Subscription {
        int priority;  // Higher = processed first
        std::function<bool(const Event&)> filter;  // Optional
        std::function<void(const Event&)> handler;
    };

    template<typename Event>
    void subscribe(std::function<void(const Event&)> handler,
                   int priority = 0,
                   std::function<bool(const Event&)> filter = nullptr) {
        auto& subs = subscriptions_[typeid(Event)];
        subs.push_back({
            priority,
            [filter, handler](const std::any& e) -> bool {
                const auto& event = std::any_cast<const Event&>(e);
                if (filter && !filter(event)) return false;
                handler(event);
                return true;
            }
        });
        // Sort by priority (descending)
        std::sort(subs.begin(), subs.end(),
            [](const auto& a, const auto& b) {
                return a.priority > b.priority;
            });
    }

    template<typename Event>
    void publish(const Event& event) {
        auto it = subscriptions_.find(typeid(Event));
        if (it == subscriptions_.end()) return;

        for (auto& sub : it->second) {
            sub.handler(event);
        }
    }

private:
    struct Sub {
        int priority;
        std::function<bool(const std::any&)> handler;
    };
    std::unordered_map<std::type_index, std::vector<Sub>> subscriptions_;
};

// Usage with priority and filter:
// High-priority security check processes first
mediator.subscribe<LoginEvent>(
    [](const LoginEvent& e) { audit_log(e); },
    /*priority=*/100  // Processed before normal handlers
);

// Filtered: only handle events for VIP customers
mediator.subscribe<OrderEvent>(
    [](const OrderEvent& e) { notify_vip_team(e); },
    /*priority=*/0,
    [](const OrderEvent& e) { return e.customer_tier == "VIP"; }
);
```

The priority sort happens at subscription time, so dispatch is just a sequential loop over already-sorted handlers - no overhead per publish call.

### Q3: Request/response mediator for synchronous operations

**Answer:**

The event bus shown above is fire-and-forget. Sometimes you need a result back - a query, not a notification. The request/response mediator handles exactly that pattern. It is the C++ equivalent of MediatR in the .NET world:

```cpp
// === Request/Response mediator (like MediatR in .NET) ===
template<typename TResponse>
struct IRequest {
    using ResponseType = TResponse;
};

class RequestMediator {
public:
    template<typename TRequest>
    void register_handler(
        std::function<typename TRequest::ResponseType(const TRequest&)> handler) {
        handlers_[typeid(TRequest)] = [handler](const std::any& req) -> std::any {
            return handler(std::any_cast<const TRequest&>(req));
        };
    }

    template<typename TRequest>
    typename TRequest::ResponseType send(const TRequest& request) {
        auto it = handlers_.find(typeid(TRequest));
        if (it == handlers_.end())
            throw std::runtime_error("No handler registered");

        auto result = it->second(request);
        return std::any_cast<typename TRequest::ResponseType>(result);
    }

private:
    std::unordered_map<std::type_index,
        std::function<std::any(const std::any&)>> handlers_;
};

// === Request/response types ===
struct GetUserRequest : IRequest<UserDTO> {
    int user_id;
};

struct CreateOrderRequest : IRequest<std::string> {  // returns order_id
    int customer_id;
    std::vector<LineItem> items;
};

// === Usage ===
void setup(RequestMediator& mediator) {
    mediator.register_handler<GetUserRequest>(
        [&](const GetUserRequest& req) -> UserDTO {
            return user_repo.find(req.user_id).to_dto();
        });

    mediator.register_handler<CreateOrderRequest>(
        [&](const CreateOrderRequest& req) -> std::string {
            auto order = Order::create(req.customer_id, req.items);
            order_repo.save(order);
            return order.id();
        });
}

void controller(RequestMediator& mediator) {
    auto user = mediator.send(GetUserRequest{42});
    auto order_id = mediator.send(CreateOrderRequest{42, items});
}
```

Each request type carries its response type as an associated typedef. The mediator uses that to cast the return value correctly. The controller never directly references a repository - it only talks to the mediator, keeping its dependencies minimal and testable.

---

## Notes

- Mediator eliminates N×N coupling; each subsystem depends only on the mediator and the event type definitions.
- The event types (DTOs) act as the **contract** between subsystems - changing them is the only way to break things.
- Watch for the mediator becoming a "god object" - if it grows unwieldy, split it into domain-specific mediators.
- For async systems, the mediator can queue events and dispatch on dedicated threads.
- Testing is straightforward: subscribe a test handler to the mediator and verify that the expected events are published.
- An in-process mediator is synchronous by default; wrap handlers in `std::async` for async dispatch if you need it.
