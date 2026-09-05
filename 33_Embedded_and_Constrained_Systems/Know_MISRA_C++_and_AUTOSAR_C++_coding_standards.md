# Know MISRA C++ and AUTOSAR C++ Coding Standards

**Category:** Embedded and Constrained Systems  
**Standard:** C++20, C++23, C++14 (R22-11)  
**Reference:** <https://misra.org.uk/misra-c-plus-plus-2023/>  

---

## Topic Overview

**MISRA C++ 2023** and **AUTOSAR C++ 14** are the two dominant coding standards for safety-critical C++ in automotive, aerospace, medical devices, and industrial control. They constrain C++ usage to a deterministic, analyzable, and safe subset - banning features that introduce undefined behavior, implementation-defined behavior, non-deterministic execution, or code that is difficult for static analysis tools to reason about.

MISRA C++ 2023 is the latest revision, fully targeting **C++17** (with guidance on C++20/23 features). It replaces the outdated MISRA C++ 2008 which only covered C++03. AUTOSAR C++ 14 (Adaptive Platform, Release 22-11) targets **C++14** and is widely used in automotive ECU development. There is significant overlap between the two standards, but key differences exist - particularly in rule count, classification, and how strictly dynamic memory is treated.

Here is a quick comparison of the two standards side by side:

| Aspect | MISRA C++ 2023 | AUTOSAR C++ 14 |
| --- | --- | --- |
| C++ standard targeted | C++17 | C++14 |
| Number of rules | ~180 rules | ~400 rules |
| Rule classification | Mandatory, Required, Advisory | Required, Advisory |
| Exceptions (C++) | Banned (no `throw`/`try`/`catch`) | Banned for ASIL-B+ |
| RTTI | Banned | Banned |
| Dynamic memory | Banned (`new`/`delete`/`malloc`) | Restricted (deterministic allocators OK) |
| `goto` | Banned | Banned |
| Unions | Restricted (discriminated only) | Restricted |
| `reinterpret_cast` | Restricted (HW register access OK) | Restricted |
| `#define` macros | Severely restricted | Restricted |
| Lambda captures | Allowed with restrictions | Allowed with restrictions |
| Templates | Allowed (instantiation audited) | Allowed |
| `auto` type deduction | Allowed with restrictions | Allowed with restrictions |
| Smart pointers | `unique_ptr` recommended | `unique_ptr` recommended |
| Raw pointer arithmetic | Banned (use `span`, `array`) | Restricted |
| Compliance tool examples | Polyspace, PC-lint, LDRA, Parasoft | Same + AUTOSAR-specific checks |

The safety integrity level (SIL or ASIL) of your system determines how strictly you must apply these standards. A quality management (QM) component has no hard requirement; an ASIL-D component requires full compliance, formal verification, and tool qualification. This table shows how that maps across three major industry standards:

```cpp
Safety Integrity Levels and Standard Applicability:

ISO 26262 (Automotive):
┌───────────┬──────────────────────────────────────────┐
│ ASIL Level│ Coding Standard Requirement              │
├───────────┼──────────────────────────────────────────┤
│ QM        │ Best practices (optional)                │
│ ASIL-A    │ MISRA/AUTOSAR recommended                │
│ ASIL-B    │ MISRA/AUTOSAR required, -fno-exceptions  │
│ ASIL-C    │ Full MISRA compliance + static analysis  │
│ ASIL-D    │ Full compliance + formal verification    │
└───────────┴──────────────────────────────────────────┘

IEC 61508 (Industrial):
  SIL 1-2: Recommended  │  SIL 3-4: Mandatory + tool qualification
DO-178C (Aerospace):
  DAL-E: Advisory  │  DAL-A: Full compliance + MC/DC coverage
```

Both standards require **static analysis tool qualification** - the tools must themselves be validated to the appropriate safety standard. A "clean" PC-lint or Polyspace run against MISRA C++ 2023 rules is a common certification requirement. Deviations from rules must be formally documented with a rationale, approved by the safety manager, and traceable. You cannot just suppress a warning and move on - the process exists to create an audit trail.

---

## Self-Assessment

### Q1: Demonstrate the top 10 most impactful MISRA C++ 2023 rules with code examples showing violations and compliant alternatives

These are the rules you will encounter most often in real safety-critical codebases. Each one is shown with a non-compliant pattern and the compliant replacement.

```cpp
// misra_top10.cpp - Key MISRA C++ 2023 rules with examples
#include <cstdint>
#include <array>
#include <span>
#include <optional>
#include <type_traits>

// ================================================================
// Rule 0.1.2: A variable must be used after being assigned
// ================================================================
void rule_0_1_2() {
    // NON-COMPLIANT:
    // int x = compute();   // x assigned but never read -> dead store

    // COMPLIANT:
    [[maybe_unused]] int debug_val = 42;  // Explicitly mark if intentional
}

// ================================================================
// Rule 6.0.1: Block scope variables shall not hide outer scope
// ================================================================
int value = 10;  // file scope

void rule_6_0_1() {
    // NON-COMPLIANT:
    // int value = 20;  // shadows file-scope 'value'

    // COMPLIANT:
    int local_value = 20;  // distinct name
    (void)local_value;
}

// ================================================================
// Rule 6.7.2: Global objects shall not be used (prefer encapsulation)
// ================================================================
// NON-COMPLIANT:
// int g_sensor_count = 0;  // mutable global state

// COMPLIANT: Encapsulate in a function or class
namespace sensors {
    inline std::uint32_t& count() {
        static std::uint32_t c = 0;
        return c;
    }
}

// ================================================================
// Rule 6.8.1: An object shall not be accessed beyond its lifetime
// ================================================================
// NON-COMPLIANT:
// int* dangling() {
//     int local = 42;
//     return &local;  // dangling pointer!
// }

// COMPLIANT: Return by value or use optional
std::optional<int> safe_lookup(int key) {
    if (key == 42) return 100;
    return std::nullopt;
}

// ================================================================
// Rule 8.2.3: A function should have a single point of exit
// (Advisory in MISRA 2023, Required in some profiles)
// ================================================================
// NON-COMPLIANT (multiple returns):
// int classify(int x) {
//     if (x < 0) return -1;
//     if (x == 0) return 0;
//     return 1;
// }

// COMPLIANT:
int classify(int x) {
    int result;
    if (x < 0) {
        result = -1;
    } else if (x == 0) {
        result = 0;
    } else {
        result = 1;
    }
    return result;  // single exit
}

// ================================================================
// Rule 8.18.2: Pointer arithmetic shall not be used
// ================================================================
void rule_8_18_2(const std::uint8_t* data, std::size_t len) {
    // NON-COMPLIANT:
    // for (int i = 0; i < len; ++i) {
    //     process(*(data + i));  // pointer arithmetic
    // }

    // COMPLIANT: Use std::span
    auto view = std::span(data, len);
    for (auto byte : view) {
        (void)byte;  // process
    }
}

// ================================================================
// Rule 11.3.1: reinterpret_cast shall not be used
// (Exception: memory-mapped I/O with documented deviation)
// ================================================================
// NON-COMPLIANT (general use):
// auto* p = reinterpret_cast<int*>(some_void_ptr);

// COMPLIANT deviation for HW registers (requires formal deviation record):
// /*cppcheck-suppress misra-cpp2023-11.3.1 ; HW register access */
// auto* uart_dr = reinterpret_cast<volatile std::uint32_t*>(0x4001'1004);

// ================================================================
// Rule 15.0.2: goto shall not be used
// ================================================================
// NON-COMPLIANT:
// goto cleanup;

// COMPLIANT: Use RAII, structured control flow, or lambda
void process() {
    struct Cleanup {
        ~Cleanup() { /* release resources */ }
    } guard;
    // ... if error, just return - destructor handles cleanup
}

// ================================================================
// Rule 18.3.1: Dynamic memory shall not be used
// ================================================================
// NON-COMPLIANT:
// auto* p = new Sensor();
// std::vector<int> v;  // heap-allocating

// COMPLIANT:
struct Sensor { std::uint16_t id; float value; };
std::array<Sensor, 32> sensor_array{};  // static allocation

// ================================================================
// Rule 21.6.1: The Standard Library input/output functions
// shall not be used (no <iostream>, no printf in production)
// ================================================================
// NON-COMPLIANT:
// std::cout << "Hello";    // pulls in massive I/O subsystem
// printf("value: %d", x);  // type-unsafe, locale-dependent

// COMPLIANT: Use hardware-specific logging (UART, SWO, etc.)
```

### Q2: Write a complete AUTOSAR C++ 14 compliant driver class that demonstrates rule-conforming patterns for resource management, error handling, and type safety

The key insight here is that AUTOSAR bans exceptions, so every function that can fail must communicate that failure through its return type. Notice how `[[nodiscard]]` enforces that callers check the returned `Status` - ignoring errors is not allowed.

```cpp
// autosar_driver.h - AUTOSAR C++ 14 compliant peripheral driver
// Conformance: AUTOSAR C++ 14 Rules checked via clang-tidy + Parasoft
#pragma once

#include <cstdint>
#include <array>
#include <type_traits>
#include <limits>

namespace driver {

// A18-1-1: C-style arrays shall not be used -> use std::array
// A0-4-2: Type std::uint*_t aliases shall be used instead of basic types

// Error type: A15-0-1 exceptions banned -> use error codes
enum class Status : std::uint8_t {
    kOk              = 0U,
    kNotInitialized  = 1U,
    kInvalidParam    = 2U,
    kTimeout         = 3U,
    kBusy            = 4U,
    kHardwareFault   = 5U,
};

// A7-1-5: auto shall not be used for fundamental types
// A7-2-1: enum underlying type shall be specified

// Configuration struct - trivially copyable for safety
struct AdcConfig {
    std::uint8_t  channel;       // 0-15
    std::uint16_t sample_cycles; // ADC clock cycles per sample
    std::uint8_t  resolution;    // 8, 10, 12 bits
};

// A12-1-1: Constructors shall initialize all non-static members
// A12-8-1: Move/copy not declared -> Rule of Zero
static_assert(std::is_trivially_copyable_v<AdcConfig>,
              "AdcConfig must be trivially copyable");

// A5-1-1: Literal values shall not be used (use named constants)
namespace adc_constants {
    constexpr std::uint8_t  kMaxChannel      = 15U;
    constexpr std::uint16_t kMaxSampleCycles = 480U;
    constexpr std::uint16_t kTimeoutCycles   = 10000U;
    constexpr std::uintptr_t kAdcBaseAddr    = 0x4001'2000U;
}

// Driver class - AUTOSAR compliant
class AdcDriver final {  // A12-1-6: Class shall be final or have virtual dtor
public:
    // A12-1-1: All members initialized
    AdcDriver() = default;

    // A15-5-1: Class shall not propagate exceptions -> noexcept on all
    // A13-2-1: Assignment operator shall return reference
    AdcDriver(const AdcDriver&)            = delete;  // A12-8-4: No copy for HW driver
    AdcDriver& operator=(const AdcDriver&) = delete;
    AdcDriver(AdcDriver&&)                 = delete;  // Non-movable singleton HW resource
    AdcDriver& operator=(AdcDriver&&)      = delete;

    // A8-4-2: [in] params by const ref, [out] by pointer
    [[nodiscard]]
    Status init(const AdcConfig& config) noexcept {
        // A5-1-1: No magic numbers - use named constants
        if (config.channel > adc_constants::kMaxChannel) {
            return Status::kInvalidParam;
        }
        if (config.sample_cycles > adc_constants::kMaxSampleCycles) {
            return Status::kInvalidParam;
        }
        if ((config.resolution != 8U) &&
            (config.resolution != 10U) &&
            (config.resolution != 12U)) {
            return Status::kInvalidParam;
        }

        config_ = config;

        // Hardware register access (deviation A11-3-1 for MMIO)
        volatile auto* cr1 = reinterpret_cast<volatile std::uint32_t*>(
            adc_constants::kAdcBaseAddr + 0x04U);
        volatile auto* cr2 = reinterpret_cast<volatile std::uint32_t*>(
            adc_constants::kAdcBaseAddr + 0x08U);

        *cr1 = static_cast<std::uint32_t>(config.resolution) << 24U;
        *cr2 = 1U;  // Enable ADC (ADON bit)

        initialized_ = true;
        return Status::kOk;
    }

    // A8-4-7: Return values shall be used (enforced via [[nodiscard]])
    [[nodiscard]]
    Status read(std::uint16_t* const out_value) noexcept {
        // A0-1-2: Check preconditions
        if (!initialized_) {
            return Status::kNotInitialized;
        }
        // A5-3-1: Null pointer dereference prohibited
        if (out_value == nullptr) {
            return Status::kInvalidParam;
        }

        volatile auto* sr = reinterpret_cast<volatile std::uint32_t*>(
            adc_constants::kAdcBaseAddr);
        volatile auto* dr = reinterpret_cast<volatile std::uint32_t*>(
            adc_constants::kAdcBaseAddr + 0x4CU);

        // Start conversion
        volatile auto* cr2 = reinterpret_cast<volatile std::uint32_t*>(
            adc_constants::kAdcBaseAddr + 0x08U);
        *cr2 |= (1U << 30U);  // SWSTART

        // A6-5-2: Loop shall have a bounded iteration count
        std::uint16_t timeout = adc_constants::kTimeoutCycles;
        bool conversion_done = false;

        while (timeout > 0U) {
            --timeout;
            if ((*sr & (1U << 1U)) != 0U) {  // EOC flag
                conversion_done = true;
                break;  // A6-6-1: break allowed in loops (not goto)
            }
        }

        if (!conversion_done) {
            return Status::kTimeout;
        }

        *out_value = static_cast<std::uint16_t>(*dr & 0x0FFFU);
        return Status::kOk;
    }

    [[nodiscard]]
    Status deinit() noexcept {
        if (!initialized_) {
            return Status::kNotInitialized;
        }
        volatile auto* cr2 = reinterpret_cast<volatile std::uint32_t*>(
            adc_constants::kAdcBaseAddr + 0x08U);
        *cr2 = 0U;
        initialized_ = false;
        return Status::kOk;
    }

private:
    AdcConfig config_{};
    bool initialized_{false};
};

// A3-1-6: Trivial definitions in headers; non-trivial in .cpp files
// A7-1-1: constexpr where possible

} // namespace driver
```

### Q3: Set up a static analysis pipeline that checks both MISRA C++ 2023 and AUTOSAR C++ 14 rules using open-source and commercial tools

Static analysis is not optional for safety-critical code - it is a certification requirement. Here is a CI pipeline that runs three tools: `cppcheck` for MISRA rules, `clang-tidy` for AUTOSAR-aligned checks, and PC-lint Plus for full commercial coverage.

```yaml
# .github/workflows/misra-compliance.yml
# CI pipeline for MISRA C++ 2023 + AUTOSAR C++ 14 static analysis

name: Safety Compliance Check
on: [push, pull_request]

jobs:
  cppcheck-misra:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install cppcheck (MISRA addon)

        run: |
          sudo apt-get install -y cppcheck
          # MISRA C++ 2023 rules (via cppcheck addon)
          # Requires misra_cpp_2023.json rules file (from MISRA license)

      - name: Run cppcheck MISRA

        run: |
          cppcheck --enable=all \
                   --std=c++17 \
                   --suppress=missingIncludeSystem \
                   --addon=misra \
                   --addon-python=python3 \
                   --error-exitcode=1 \
                   --xml --xml-version=2 \
                   src/ 2> cppcheck-misra-report.xml

      - name: Upload MISRA report

        uses: actions/upload-artifact@v4
        with:
          name: misra-report
          path: cppcheck-misra-report.xml

  clang-tidy-autosar:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install clang-tidy 17+

        run: |
          wget https://apt.llvm.org/llvm.sh
          chmod +x llvm.sh
          sudo ./llvm.sh 17

      - name: Run clang-tidy (AUTOSAR checks)

        run: |
          clang-tidy-17 \
            -checks='-*,
              cert-*,
              cppcoreguidelines-*,
              bugprone-*,
              performance-*,
              readability-braces-around-statements,
              readability-identifier-naming,
              readability-magic-numbers,
              misc-no-recursion,
              misc-non-private-member-variables-in-classes,
              hicpp-*' \
            -warnings-as-errors='*' \
            -header-filter='src/.*' \
            src/*.cpp -- \
            -std=c++17 -fno-exceptions -fno-rtti

  pc-lint-plus:
    runs-on: ubuntu-latest
    container:
      image: pclintplus/pclp:latest  # Commercial license required
    steps:

      - uses: actions/checkout@v4

      - name: Run PC-lint Plus (MISRA C++ 2023 + AUTOSAR)

        run: |
          pclp64 \
            -b \
            au-misra-cpp-2023.lnt \
            au-autosar.lnt \
            std_c++17.lnt \
            -i src/ \
            src/*.cpp
```

The `.clang-tidy` configuration file controls which checks run and what naming conventions are enforced. This file captures the AUTOSAR A2-10-1 naming convention (classes in CamelCase, members with `_` suffix, constants prefixed with `k`):

```ini
# .clang-tidy - AUTOSAR/MISRA profile
---
Checks: >
  -*,
  bugprone-*,
  cert-*,
  cppcoreguidelines-*,
  hicpp-*,
  misc-no-recursion,
  misc-non-private-member-variables-in-classes,
  performance-*,
  readability-braces-around-statements,
  readability-identifier-naming,
  readability-magic-numbers,
  -cppcoreguidelines-pro-type-reinterpret-cast,

WarningsAsErrors: '*'

CheckOptions:
  # AUTOSAR A2-10-1: Identifier naming convention

  - key: readability-identifier-naming.ClassCase

    value: CamelCase

  - key: readability-identifier-naming.FunctionCase

    value: CamelCase

  - key: readability-identifier-naming.VariableCase

    value: lower_case

  - key: readability-identifier-naming.ConstantCase

    value: CamelCase

  - key: readability-identifier-naming.ConstantPrefix

    value: 'k'

  - key: readability-identifier-naming.MemberPrefix

    value: ''

  - key: readability-identifier-naming.MemberSuffix

    value: '_'

  - key: readability-identifier-naming.NamespaceCase

    value: lower_case

  # AUTOSAR M0-1-3: Unused variable threshold

  - key: misc-unused-parameters.StrictMode

    value: true

  # MISRA Rule 5.1.1: No magic numbers

  - key: readability-magic-numbers.IgnoredIntegerValues

    value: '0;1;2;4;8;16;32;64'

  - key: readability-magic-numbers.IgnoredFloatingPointValues

    value: '0.0;1.0;0.5'

FormatStyle: file
HeaderFilterRegex: 'src/.*\.h(pp)?$'
```

You can also add compile-time compliance assertions that catch common violations at build time, before the static analysis tool even runs:

```cpp
// compliance_report.h - Generate rule compliance summary at build time
#pragma once

// Compile-time compliance assertions
// These static_asserts catch common MISRA/AUTOSAR violations at build time

#include <type_traits>
#include <cstdint>
#include <climits>

namespace compliance {

// MISRA Rule 3.9.2: Underlying type of bit fields
static_assert(CHAR_BIT == 8, "MISRA: CHAR_BIT must be 8");

// AUTOSAR A0-4-2: Fixed-width integer types used
static_assert(sizeof(std::uint32_t) == 4, "uint32_t must be 4 bytes");
static_assert(sizeof(std::int64_t)  == 8, "int64_t must be 8 bytes");

// MISRA Rule 6.2.4: Floating-point not used in safety-critical paths
// (compile-time flag to detect accidental float usage)
template <typename T>
constexpr bool is_safety_type_v =
    std::is_integral_v<T> || std::is_enum_v<T>;

// AUTOSAR A12-1-1: All members initialized
template <typename T>
constexpr bool is_safe_aggregate_v =
    std::is_trivially_copyable_v<T> &&
    std::is_standard_layout_v<T>;

// Helper: static_assert that a type meets safety requirements
#define ASSERT_SAFE_TYPE(T) \
    static_assert(compliance::is_safe_aggregate_v<T>, \
                  "MISRA/AUTOSAR: " #T " must be trivially copyable and standard layout")

} // namespace compliance
```

---

## Notes

- **MISRA C++ 2023** is a complete rewrite - it does **not** just add rules to MISRA C++ 2008. Code compliant with 2008 may violate 2023 rules. A full re-audit is required when upgrading.
- AUTOSAR C++ 14 is maintained by the AUTOSAR consortium and freely downloadable (unlike MISRA which requires a license purchase).
- Both standards require a **deviation process**: if you must violate a rule (e.g., `reinterpret_cast` for MMIO), you document the deviation with: rule number, rationale, scope, reviewer approval, and traceability ID.
- `clang-tidy` covers ~60% of AUTOSAR/MISRA rules via `cert-*`, `cppcoreguidelines-*`, `bugprone-*`, and `hicpp-*` checks. For full coverage, commercial tools (Polyspace, PC-lint Plus, LDRA, Parasoft, QA-C++) are needed.
- MISRA 2023 **permits** `constexpr`, `consteval`, `if constexpr`, `auto`, lambdas, templates, `std::optional`, `std::variant`, `std::array`, `std::span`, and structured bindings - it is far more modern-C++-friendly than MISRA 2008 was.
- MISRA 2023 **bans**: exceptions, RTTI, `goto`, `union` (except tagged), C-style casts, `reinterpret_cast` (with deviation), `#define` function-like macros, `malloc`/`free`, `new`/`delete`, `setjmp`/`longjmp`, `signal`, and recursion.
- Every project should generate a **Guideline Compliance Summary (GCS)** - a document listing each rule, its compliance status (compliant / deviation / not applicable), and evidence. This is a formal audit artifact for ISO 26262 / IEC 61508 certification.
