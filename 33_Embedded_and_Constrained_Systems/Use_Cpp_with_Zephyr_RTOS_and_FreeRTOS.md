# Use C++ with Zephyr RTOS and FreeRTOS

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** <https://docs.zephyrproject.org/latest/> · <https://www.freertos.org/>  

---

## Topic Overview

### C++ on RTOS - Why and How

Both Zephyr and FreeRTOS are written in C, but real-world embedded applications often use C++ for type safety, RAII, templates, and zero-cost abstractions. The key challenges are:

1. **Linking C++ code with a C kernel** - requires `extern "C"` at the boundaries.
2. **C++ runtime support** - constructors for global objects, exceptions, RTTI.
3. **Thread/task creation** - wrapping C-style task APIs in type-safe C++ wrappers.
4. **Memory constraints** - controlling what C++ features cost in flash/RAM.

The good news is that all of these have clean solutions. The techniques below show you the patterns that experienced embedded C++ developers use day to day.

### FreeRTOS with C++

FreeRTOS tasks are created with C function pointers. C++ member functions and lambdas cannot be passed directly - a member function has a hidden `this` parameter, and a capturing lambda has no way to decay to a raw function pointer. The solution is the **trampoline pattern**: pass a plain `static` function as the entry point, and smuggle `this` through the `pvParameters` argument.

<!-- compile: needs POSIX header `FreeRTOS.h` -->
```cpp
#include "FreeRTOS.h"
#include "task.h"
#include <cstdint>

// Type-safe task wrapper
class Task {
    TaskHandle_t handle_ = nullptr;
    const char*  name_;
    uint16_t     stack_size_;
    UBaseType_t  priority_;

public:
    Task(const char* name, uint16_t stack_size, UBaseType_t priority)
        : name_(name), stack_size_(stack_size), priority_(priority) {}

    virtual ~Task() {
        if (handle_) vTaskDelete(handle_);
    }

    Task(const Task&) = delete;
    Task& operator=(const Task&) = delete;

    bool start() {
        BaseType_t result = xTaskCreate(
            &Task::task_trampoline,  // C-compatible static function
            name_,
            stack_size_,
            this,                    // pvParameters = this pointer
            priority_,
            &handle_
        );
        return result == pdPASS;
    }

    void suspend() { vTaskSuspend(handle_); }
    void resume()  { vTaskResume(handle_); }

protected:
    // Override this in derived classes
    virtual void run() = 0;

private:
    static void task_trampoline(void* param) {
        auto* self = static_cast<Task*>(param);
        self->run();
        // If run() returns, delete the task
        vTaskDelete(nullptr);
    }
};

// Usage
class SensorTask : public Task {
    uint32_t interval_ms_;
public:
    SensorTask(uint32_t interval_ms)
        : Task("sensor", 256, 2), interval_ms_(interval_ms) {}

protected:
    void run() override {
        while (true) {
            read_sensors();
            process_data();
            vTaskDelay(pdMS_TO_TICKS(interval_ms_));
        }
    }
};

int main() {
    SensorTask sensor_task(100);
    sensor_task.start();
    vTaskStartScheduler();
    // Should never reach here
}
```

The `task_trampoline` static function is the bridge between the C world (FreeRTOS) and the C++ world (your `Task` subclass). Once inside `run()`, you are in fully idiomatic C++.

### RAII Wrappers for FreeRTOS Primitives

Wrapping FreeRTOS primitives in RAII classes gives you automatic cleanup, prevents forgetting to unlock a mutex, and makes ownership explicit. Notice that `LockGuard` works exactly like `std::lock_guard` - the mental model transfers directly.

```cpp
#include "FreeRTOS.h"
#include "semphr.h"

class Mutex {
    SemaphoreHandle_t handle_;
public:
    Mutex() : handle_(xSemaphoreCreateMutex()) {
        configASSERT(handle_ != nullptr);
    }

    ~Mutex() { vSemaphoreDelete(handle_); }

    Mutex(const Mutex&) = delete;
    Mutex& operator=(const Mutex&) = delete;

    bool lock(TickType_t timeout = portMAX_DELAY) {
        return xSemaphoreTake(handle_, timeout) == pdTRUE;
    }

    void unlock() {
        xSemaphoreGive(handle_);
    }
};

// RAII lock guard - exactly like std::lock_guard
class LockGuard {
    Mutex& mtx_;
public:
    explicit LockGuard(Mutex& m) : mtx_(m) { mtx_.lock(); }
    ~LockGuard() { mtx_.unlock(); }
    LockGuard(const LockGuard&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;
};

// Type-safe queue wrapper
template<typename T, size_t Capacity>
class Queue {
    QueueHandle_t handle_;
public:
    Queue() : handle_(xQueueCreate(Capacity, sizeof(T))) {
        configASSERT(handle_ != nullptr);
    }
    ~Queue() { vQueueDelete(handle_); }

    bool send(const T& item, TickType_t timeout = portMAX_DELAY) {
        return xQueueSend(handle_, &item, timeout) == pdTRUE;
    }

    bool receive(T& item, TickType_t timeout = portMAX_DELAY) {
        return xQueueReceive(handle_, &item, timeout) == pdTRUE;
    }

    [[nodiscard]] size_t size() const {
        return uxQueueMessagesWaiting(handle_);
    }
};
```

The `Queue<T, Capacity>` template is worth calling out specifically - `sizeof(T)` is computed at compile time (no manual size argument), and `send`/`receive` take typed references instead of `void*`, so type mismatches are compile errors rather than runtime crashes.

### Zephyr RTOS with C++

Zephyr has native C++ support. You enable it in `prj.conf`:

```ini
CONFIG_CPLUSPLUS=y
CONFIG_STD_CPP17=y
CONFIG_LIB_CPLUSPLUS=y
# Optional: if you need exceptions and RTTI
# CONFIG_EXCEPTIONS=y
# CONFIG_RTTI=y
```

Once those options are set, Zephyr compiles `.cpp` files with the C++ compiler automatically - no extra CMake plumbing required. Here is a thread wrapper that parallels the FreeRTOS one:

```cpp
#include <zephyr/kernel.h>
#include <array>

template<size_t StackSize = 1024>
class ZephyrThread {
    k_thread       thread_;
    k_thread_entry_t entry_;

    // Stack must be defined with K_THREAD_STACK_DEFINE or similar
    K_THREAD_STACK_MEMBER(stack_, StackSize);

public:
    ZephyrThread() = default;
    ~ZephyrThread() { k_thread_abort(&thread_); }

    ZephyrThread(const ZephyrThread&) = delete;
    ZephyrThread& operator=(const ZephyrThread&) = delete;

    void start(k_thread_entry_t entry, void* p1 = nullptr,
               void* p2 = nullptr, void* p3 = nullptr,
               int priority = 5) {
        k_thread_create(
            &thread_,
            stack_,
            K_THREAD_STACK_SIZEOF(stack_),
            entry,
            p1, p2, p3,
            priority,
            0,       // options
            K_NO_WAIT
        );
    }

    void join() { k_thread_join(&thread_, K_FOREVER); }
};

// Zephyr mutex RAII
class ZephyrMutex {
    k_mutex mtx_;
public:
    ZephyrMutex()  { k_mutex_init(&mtx_); }

    void lock()    { k_mutex_lock(&mtx_, K_FOREVER); }
    void unlock()  { k_mutex_unlock(&mtx_); }

    // Compatible with std::lock_guard pattern
};
```

### Zephyr Device Driver in C++

Zephyr's device model is C-based, but you can wrap it in a clean C++ class. The `[[nodiscard]]` on `is_ready()` encourages callers to check initialization before using the sensor.

```cpp
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

class TemperatureSensor {
    const device* dev_;

public:
    explicit TemperatureSensor(const char* label)
        : dev_(device_get_binding(label))
    {
        if (!dev_ || !device_is_ready(dev_)) {
            // Handle error - in no-exceptions builds, set error flag
            dev_ = nullptr;
        }
    }

    [[nodiscard]] bool is_ready() const { return dev_ != nullptr; }

    [[nodiscard]] float read_celsius() const {
        sensor_value val;
        sensor_sample_fetch(dev_);
        sensor_channel_get(dev_, SENSOR_CHAN_AMBIENT_TEMP, &val);
        return static_cast<float>(val.val1) +
               static_cast<float>(val.val2) / 1'000'000.0f;
    }
};

// Usage in Zephyr main
void main_thread(void*, void*, void*) {
    TemperatureSensor temp("BME280");
    if (!temp.is_ready()) {
        printk("Sensor not found!\n");
        return;
    }

    while (true) {
        float c = temp.read_celsius();
        printk("Temperature: %.1f C\n", static_cast<double>(c));
        k_sleep(K_SECONDS(1));
    }
}
```

### Build System Integration

**FreeRTOS with CMake (typical)**:

```cmake
add_executable(firmware
    main.cpp
    tasks/sensor_task.cpp
)
target_compile_features(firmware PRIVATE cxx_std_17)
target_link_libraries(firmware PRIVATE freertos_kernel)
# Disable exceptions and RTTI to save flash
target_compile_options(firmware PRIVATE -fno-exceptions -fno-rtti)
```

**Zephyr** - just add `.cpp` files normally; the build system handles C++ compilation when `CONFIG_CPLUSPLUS=y`.

---

## Self-Assessment

### Q1: Why can't you pass a C++ lambda directly to `xTaskCreate`

`xTaskCreate` expects a function pointer of type `void (*)(void*)`. A C++ lambda with captures has a unique closure type and **cannot decay to a function pointer** - only captureless lambdas can. The reason is that a capturing lambda stores data (the captured variables) alongside its code, so it cannot be represented as a plain function address.

The solution is the **trampoline pattern**: pass a `static` member function (or captureless lambda) as the entry, and pass `this` (or a context pointer) as `pvParameters`. The trampoline casts `pvParameters` back and calls the actual method.

```cpp
// This works - captureless lambda decays to function pointer
xTaskCreate([](void* p) {
    static_cast<MyTask*>(p)->run();
}, "task", 256, this, 1, &handle);
```

### Q2: Show a type-safe FreeRTOS queue that prevents type mismatches

The `Queue<T, Capacity>` template above does this. Benefits:

- `sizeof(T)` is computed at compile time - no manual size parameter.
- `send()` and `receive()` take `T&` - no `void*` casts.
- `Capacity` is a template parameter - part of the type, visible in error messages.
- RAII: queue is deleted in destructor.

### Q3: What must you configure to use C++ with Zephyr

Add to `prj.conf`:

```ini
CONFIG_CPLUSPLUS=y       # Enable C++ compiler
CONFIG_STD_CPP17=y       # C++17 standard (or CONFIG_STD_CPP20)
CONFIG_LIB_CPLUSPLUS=y   # Link the C++ standard library
```

Optionally:

- `CONFIG_EXCEPTIONS=y` - enable exception support (adds ~10-20 KB flash)
- `CONFIG_RTTI=y` - enable RTTI (needed for `dynamic_cast`, `typeid`)
- For minimal builds, keep both off and use `-fno-exceptions -fno-rtti`

---

## Notes

- FreeRTOS stack sizes are in **words** (4 bytes on ARM), not bytes - size accordingly to avoid silent stack overflows.
- Zephyr thread stacks must be declared with `K_THREAD_STACK_DEFINE` for MPU alignment - using a plain array will not work correctly.
- Global C++ objects are constructed before `main()` - ensure the RTOS kernel is initialized first if constructors use kernel APIs.
- Prefer `constexpr` and templates over virtual functions in ISR-adjacent code where predictable, bounded execution time matters.
- Both Zephyr and FreeRTOS support static allocation - avoid `new`/`delete` in constrained systems.
