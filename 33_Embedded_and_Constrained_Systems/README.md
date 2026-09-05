# Embedded & Constrained Systems

C++ development for bare-metal, RTOS, and resource-constrained environments: freestanding implementations, no-heap strategies, ISR patterns, MISRA/AUTOSAR compliance, and hardware abstractions.

**Topics:** 18

## Contents

- [Achieve Deterministic Execution and WCET Analysis](Achieve_deterministic_execution_and_WCET_analysis.md)
- [Analyze Stack Usage and Enforce Static Memory Budgets](Analyze_stack_usage_and_enforce_static_memory_budgets.md)
- [Design bootloader and firmware update architecture in C++](Design_bootloader_and_firmware_update_architecture_in_Cpp.md)
- [Design Hardware Register Abstractions with volatile and MMIO](Design_hardware_register_abstractions_with_volatile_and_MMIO.md)
- [Design Interrupt-Safe C++ Patterns for ISR Contexts](Design_interrupt-safe_C++_patterns_for_ISR_contexts.md)
- [Develop with -fno-exceptions and -fno-rtti Constraints](Develop_with_fno-exceptions_and_fno-rtti_constraints.md)
- [Handle Real-Time Scheduling and Priority Inversion in C++](Handle_real-time_scheduling_and_priority_inversion_in_C++.md)
- [Implement Fixed-Point Arithmetic Without Floating Point](Implement_fixed-point_arithmetic_without_floating_point.md)
- [Implement power management and low-power modes from C++](Implement_power_management_and_low-power_modes_from_Cpp.md)
- [Know ELF and hex binary layout and linker script customization](Know_ELF_and_hex_binary_layout_and_linker_script_customization.md)
- [Know MISRA C++ and AUTOSAR C++ Coding Standards](Know_MISRA_C++_and_AUTOSAR_C++_coding_standards.md)
- [Understand C++ exception table overhead and when to use -fno-exceptions](Understand_Cpp_exception_table_overhead_and_when_to_use_fno-exceptions.md)
- [Understand Freestanding vs Hosted C++ Implementations](Understand_freestanding_vs_hosted_C++_implementations.md)
- [Use C++ on ARM Cortex-M and RISC-V Microcontrollers](Use_C++_on_ARM_Cortex-M_and_RISC-V_microcontrollers.md)
- [Use C++ with Zephyr RTOS and FreeRTOS](Use_Cpp_with_Zephyr_RTOS_and_FreeRTOS.md)
- [Use Static Allocation Strategies Without Heap](Use_static_allocation_strategies_without_heap.md)
- [Use std::inplace_vector (C++26) for heap-free containers in embedded](Use_stdstdinplace_vector_Cpp26_for_heap-free_containers_in_embedded.md)
- [Write Bare-Metal Startup Code and Linker Scripts for C++](Write_bare-metal_startup_code_and_linker_scripts_for_C++.md)

## Notes

- Embedded C++ often prohibits dynamic allocation - use `std::array`, stack buffers, and placement new instead of `new`/`delete`.
- `-fno-exceptions` and `-fno-rtti` reduce binary size and overhead significantly in constrained environments.
- `volatile`-qualified peripheral registers must be accessed through `volatile` pointers - but `volatile` is not atomic and does not replace synchronization.
- Real-time systems require deterministic timing - avoid allocations, exceptions, and unbounded loops in time-critical paths.
- Interrupt-safe programming needs `volatile` and atomic operations - regular variables may be optimized away by the compiler across an ISR boundary.
- Static analysis and MISRA C++ compliance are common requirements in safety-critical embedded development.
- `constexpr` computing moves work to compile time - ideal for lookup tables, CRC generation, and configuration constants.
- Bit manipulation and register maps benefit from strongly-typed wrapper classes that prevent accidental register misuse.
- Memory-mapped I/O requires specific alignment and access size - use `reinterpret_cast` carefully and only where justified by a documented deviation.
- Cross-compilation toolchains (ARM GCC, IAR, Keil) have different standard library support - always verify which headers are available in your freestanding environment.
