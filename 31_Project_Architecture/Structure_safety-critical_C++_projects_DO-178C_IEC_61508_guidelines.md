# Structure safety-critical C++ projects (DO-178C, IEC 61508 guidelines)

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Safety-critical software** comes with regulatory requirements that reach far beyond "does it compile and pass tests." You have to show that your development process itself was rigorous, that every requirement is traceable to code and tests, that your language subset is restricted to safe patterns, and - at the highest integrity levels - that you can provide formal verification evidence.

**DO-178C** is the standard for airborne systems (avionics), **IEC 61508** covers industrial functional safety (think process control and machinery), and **ISO 26262** targets automotive software. All three share the same underlying philosophy: the more dangerous a failure is, the more disciplined your process must be.

### Safety Integrity Levels

If the table feels like a lot at first glance, the pattern is simple - each standard has a tiered scale where the top level demands the strictest controls. They all converge on MISRA C++ as the language subset and on very high (often 100%) coverage metrics.

| Standard | Levels | Highest Level | C++ Subset | Key Requirement |
| --- | --- | --- | --- | --- |
| **DO-178C** | DAL A-E | DAL A | MISRA C++ | MC/DC coverage |
| **IEC 61508** | SIL 1-4 | SIL 4 | MISRA C++ | Formal verification |
| **ISO 26262** | ASIL A-D | ASIL D | AUTOSAR C++ | FMEA, fault injection |
| **IEC 62304** | Class A-C | Class C | MISRA C++ | Medical device software |

---

## Self-Assessment

### Q1: Structure a safety-critical project with traceability

**Answer:**

The most important thing to understand here is that the folder structure is not optional decoration - it is required evidence. The `traceability_matrix.csv` is what an auditor or certifying authority will ask to see first: they want to follow a thread from a requirement all the way to a test result without gaps.

```cpp
safety_critical_project/
├── docs/
│   ├── SRS/                     # Software Requirements Specification
│   │   ├── SRS-001_motor_control.md
│   │   └── SRS-002_fault_detection.md
│   ├── SDD/                     # Software Design Description
│   │   ├── SDD-001_architecture.md
│   │   └── SDD-002_safety_mechanisms.md
│   ├── traceability_matrix.csv  # Requirements -> Design -> Code -> Tests
│   └── FMEA/                    # Failure Mode and Effects Analysis
│       └── FMEA_motor_subsystem.xlsx
├── src/
│   ├── safety/                  # Safety-critical code (highest rigor)
│   │   ├── watchdog.cpp
│   │   ├── fault_handler.cpp
│   │   └── safety_monitor.cpp
│   ├── app/                     # Application code
│   │   ├── motor_control.cpp    # SRS-001 implementation
│   │   └── sensor_fusion.cpp
│   └── platform/                # Platform abstraction
│       ├── hal_stm32.cpp
│       └── safe_io.cpp
├── tests/
│   ├── unit/                    # Unit tests (statement + branch coverage)
│   ├── integration/             # Integration tests
│   ├── safety/                  # Safety-specific tests
│   │   ├── test_fault_injection.cpp
│   │   ├── test_watchdog_timeout.cpp
│   │   └── test_stack_overflow.cpp
│   └── coverage/                # Coverage reports
├── tools/
│   ├── static_analysis/         # MISRA checker config
│   │   ├── misra_cpp_2023.cfg
│   │   └── cppcheck_safety.cfg
│   └── formal_verification/     # Model checking configs
├── ci/
│   ├── safety_pipeline.yml      # CI with mandatory quality gates
│   └── traceability_check.py    # Verifies all requirements traced
└── CMakeLists.txt
```

Notice how the traceability tags flow directly into the source code. Each Doxygen comment records which requirement this function satisfies, which design section describes it, what its safety integrity level is, and which test verifies it. Without those tags, the code and the documents are independent islands - with them, automated tools can check the matrix for you.

```cpp
// === Code with traceability tags ===

/**
 * @brief Motor control PID loop
 * @requirement SRS-001: Motor shall maintain speed within +-2% of setpoint
 * @design SDD-001-3: PID controller with anti-windup
 * @safety ASIL-B: Failure could cause uncontrolled motor speed
 * @tested test_motor_control.cpp::test_pid_steady_state
 */
class MotorController {
public:
    /**
     * @requirement SRS-001.1: Update rate shall be 1kHz +-1%
     * @param setpoint_rpm Target speed in RPM (0-10000)
     * @param actual_rpm Measured speed from encoder
     * @return PWM duty cycle (0.0 - 1.0)
     */
    float update(float setpoint_rpm, float actual_rpm) {
        // Defensive programming: validate inputs
        if (setpoint_rpm < 0.0f || setpoint_rpm > MAX_RPM) {
            fault_handler_.report(FaultCode::InvalidSetpoint);
            return 0.0f;  // Safe state
        }
        if (actual_rpm < 0.0f || actual_rpm > MAX_RPM * 1.1f) {
            fault_handler_.report(FaultCode::SensorOutOfRange);
            return 0.0f;  // Safe state
        }

        float error = setpoint_rpm - actual_rpm;
        integral_ += error * dt_;

        // Anti-windup: clamp integral
        integral_ = std::clamp(integral_, -MAX_INTEGRAL, MAX_INTEGRAL);

        float derivative = (error - prev_error_) / dt_;
        prev_error_ = error;

        float output = kp_ * error + ki_ * integral_ + kd_ * derivative;
        return std::clamp(output, 0.0f, 1.0f);
    }

private:
    static constexpr float MAX_RPM = 10000.0f;
    static constexpr float MAX_INTEGRAL = 1000.0f;
    float kp_ = 0.1f, ki_ = 0.01f, kd_ = 0.001f;
    float dt_ = 0.001f;  // 1kHz
    float integral_ = 0.0f;
    float prev_error_ = 0.0f;
    FaultHandler& fault_handler_;
};
```

Every `return 0.0f` above is a deliberate "safe state" return - not an error path to handle later. In safety-critical code you always define what the system does when something goes wrong, and you handle it right there.

### Q2: Implement safety mechanisms and defensive programming

**Answer:**

The key idea here is defense in depth. You don't rely on a single check; you layer multiple independent monitoring mechanisms so that one failure cannot silently propagate. The `SafetyMonitor` class below shows that pattern: it runs a software watchdog, checks the stack, cross-validates redundant sensors, and monitors supply voltage - all in a single cycle function. If any check fails, a fault is classified by severity and the system moves to an appropriate safe state.

```cpp
// === Safety monitor: independent watchdog + plausibility checks ===

/**
 * @requirement SRS-SAFETY-001: System shall detect and handle faults
 *              within 10ms
 * @safety SIL-2: Required for all safety functions
 */
class SafetyMonitor {
public:
    enum class SafeState {
        Normal,
        Degraded,     // Reduced functionality
        SafeShutdown  // Controlled stop
    };

    void check_cycle() {
        // 1. Software watchdog: detect stuck tasks
        for (auto& [name, task] : monitored_tasks_) {
            if (task.cycles_since_heartbeat++ > task.max_allowed) {
                report_fault(FaultCode::TaskTimeout, name);
            }
        }

        // 2. Stack overflow detection
        if (get_stack_remaining() < MIN_STACK_BYTES) {
            report_fault(FaultCode::StackOverflow, "main");
        }

        // 3. Plausibility check: cross-validate redundant sensors
        float sensor_a = read_temperature_sensor_a();
        float sensor_b = read_temperature_sensor_b();
        if (std::abs(sensor_a - sensor_b) > SENSOR_TOLERANCE) {
            report_fault(FaultCode::SensorDisagreement, "temperature");
        }

        // 4. Voltage monitoring
        if (read_supply_voltage() < MIN_VOLTAGE ||
            read_supply_voltage() > MAX_VOLTAGE) {
            report_fault(FaultCode::PowerOutOfRange, "supply");
        }

        // 5. Kick hardware watchdog (must be called periodically)
        hardware_watchdog_kick();
    }

    void report_fault(FaultCode code, const char* source) {
        // Log fault (non-volatile memory for post-mortem)
        fault_log_.record(code, source, get_timestamp());

        // Determine safe state based on severity
        switch (classify_severity(code)) {
            case Severity::Warning:
                // Continue with degraded mode
                safe_state_ = SafeState::Degraded;
                break;
            case Severity::Critical:
                // Controlled shutdown
                safe_state_ = SafeState::SafeShutdown;
                enter_safe_state();
                break;
        }
    }

    void heartbeat(const char* task_name) {
        monitored_tasks_[task_name].cycles_since_heartbeat = 0;
    }

private:
    void enter_safe_state() {
        // Disable actuators in safe order
        motor_controller_.emergency_stop();
        valve_controller_.close_all();
        // Enable safe indication (LED, buzzer)
        status_led_.set(LED_RED, BLINK_FAST);
    }

    SafeState safe_state_ = SafeState::Normal;
    FaultLog fault_log_;  // Persistent fault log
    struct TaskMonitor { uint32_t cycles_since_heartbeat; uint32_t max_allowed; };
    std::unordered_map<std::string, TaskMonitor> monitored_tasks_;
};

// === N-version programming: redundant computation ===
template<typename Func, typename... Args>
auto vote_2_of_3(Func f1, Func f2, Func f3, Args&&... args) {
    auto r1 = f1(std::forward<Args>(args)...);
    auto r2 = f2(std::forward<Args>(args)...);
    auto r3 = f3(std::forward<Args>(args)...);

    if (r1 == r2) return r1;
    if (r1 == r3) return r1;
    if (r2 == r3) return r2;

    // All disagree: fault
    throw SafetyFault("Voter disagreement");
}
```

The `vote_2_of_3` function at the end is a classic N-version programming technique: run the same computation three independent ways and pick the majority result. If all three disagree, something is seriously wrong and you throw a safety fault. This is commonly used for computations that control actuators.

### Q3: CI pipeline with safety quality gates

**Answer:**

In a safety project, the CI pipeline is not just a convenience - it is part of the evidence package. Every stage below is a formal quality gate. Notice that `sign_off` is `when: manual`, meaning a human must explicitly approve the pipeline before the output is considered ready for certification review.

```yaml
# === safety_pipeline.yml ===
stages:
  - build
  - static_analysis
  - test
  - coverage
  - traceability
  - sign_off

static_analysis:
  stage: static_analysis
  script:
    # MISRA C++ compliance (mandatory for DO-178C / IEC 61508)
    - cppcheck --enable=all --addon=misra --suppress=missingInclude
              --error-exitcode=1 src/
    # Polyspace or Parasoft for formal MISRA checking
    - parasoft-cpptest -config misra_cpp_2023 -report misra_report.html src/

    # Clang-tidy with safety-relevant checks
    - clang-tidy src/**/*.cpp
              -checks='-*,bugprone-*,cert-*,misc-*,performance-*'
  artifacts:
    paths: [misra_report.html]

unit_tests:
  stage: test
  script:
    - cmake --preset host-test
    - cmake --build build/host-test
    - ctest --test-dir build/host-test --output-on-failure

coverage:
  stage: coverage
  script:
    - cmake --preset host-test -DCOVERAGE=ON
    - cmake --build build/host-test
    - ctest --test-dir build/host-test
    - gcovr --branches --fail-under-branch 100  # MC/DC for DAL A
    - gcovr --fail-under-line 100               # Statement coverage
  artifacts:
    paths: [coverage_report/]

traceability:
  stage: traceability
  script:
    # Verify every requirement has:
    # - Design reference
    # - Implementation (code tag)
    # - Test case
    - python tools/traceability_check.py
            --requirements docs/SRS/
            --design docs/SDD/
            --code src/
            --tests tests/
            --report traceability_report.html
  artifacts:
    paths: [traceability_report.html]

sign_off:
  stage: sign_off
  when: manual  # Requires human approval
  script:
    - echo "All quality gates passed. Ready for certification review."
```

The compiler flags below are the ones you set for MISRA builds. Exceptions and RTTI are disabled because safety standards require you to know the exact control flow - dynamic exception handling and runtime type information introduce paths that are hard to analyze formally.

```cpp
// === MISRA C++ compliant patterns ===

// MISRA Rule 0-1-1: No unreachable code
// MISRA Rule 5-0-1: No implicit integral conversions
// MISRA Rule 6-4-2: All switch cases shall end with break
// MISRA Rule 15-3-1: Exceptions only for error handling (or disabled)

// Compile with strict MISRA settings:
// -fno-exceptions -fno-rtti -Wall -Werror -Wextra
// -Wconversion -Wsign-conversion -Wcast-align
```

---

## Notes

- **Traceability** is the number-one requirement: every requirement -> design -> code -> test must be traceable, with no gaps.
- **MISRA C++ 2023** is the current language subset standard for safety-critical C++.
- **MC/DC coverage** (Modified Condition/Decision Coverage) is required at DO-178C DAL A - it is stricter than branch coverage because every condition in a Boolean expression must be shown to independently affect the outcome.
- Avoid dynamic memory allocation (`new`/`delete`) in safety-critical code; use static allocation with known worst-case memory footprints.
- Disable exceptions and RTTI in safety-critical builds (`-fno-exceptions -fno-rtti`) so the control flow remains fully analyzable.
- **Defensive programming**: validate all inputs, check return codes, assert invariants - every function that receives external data should treat that data as suspect.
- Safety-critical code reviews require independence: the reviewer must not be the same person as the author.
- Static analysis tools commonly used for safety certification include Polyspace, Parasoft, PC-lint, and LDRA.
- The hardware watchdog must be kicked periodically; if software hangs or gets stuck in a loop, the hardware timer expires and forces a reset - this is your last line of defense.
