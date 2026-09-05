# Testing in Practice

Practical testing strategies for real C++ projects: unit testing, integration testing, embedded testing, fuzzing, sanitizers, CI pipelines, and test architecture for production systems.

**Topics:** 32

## Contents

- [Apply sanitizers (ASan, UBSan, TSan, MSan) in your test pipeline](Apply_sanitizers_ASan_UBSan_TSan_MSan_in_your_test_pipeline.md)
- [Benchmark-driven testing with Google Benchmark integration](Benchmark-driven_testing_with_Google_Benchmark_integration.md)
- [Design contract tests for API boundaries between modules](Design_contract_tests_for_API_boundaries_between_modules.md)
- [Design testable C++ code with dependency injection and interfaces](Design_testable_C++_code_with_dependency_injection_and_interfaces.md)
- [Implement fuzz testing with libFuzzer or AFL for C++ code](Implement_fuzz_testing_with_libFuzzer_or_AFL_for_C++_code.md)
- [Implement mutation testing to evaluate test suite quality](Implement_mutation_testing_to_evaluate_test_suite_quality.md)
- [Implement parameterized tests for data-driven testing](Implement_parameterized_tests_for_data-driven_testing.md)
- [Master Google Mock - matchers, actions, and expectations](Master_Google_Mock_-_matchers_actions_and_expectations.md)
- [Measure and enforce code coverage with gcov, lcov, and llvm-cov](Measure_and_enforce_code_coverage_with_gcov_lcov_and_llvm-cov.md)
- [Monitor stack usage and memory leaks in embedded test environments](Monitor_stack_usage_and_memory_leaks_in_embedded_test_environments.md)
- [Set up continuous integration testing with GitHub Actions for C++](Set_up_continuous_integration_testing_with_GitHub_Actions_for_C++.md)
- [Set up cross-compilation testing with QEMU for ARM targets](Set_up_cross-compilation_testing_with_QEMU_for_ARM_targets.md)
- [Set up Google Test and Google Mock in a CMake project](Set_up_Google_Test_and_Google_Mock_in_a_CMake_project.md)
- [Test communication protocols (SPI, I2C, UART) with mock drivers](Test_communication_protocols_SPI_I2C_UART_with_mock_drivers.md)
- [Test concurrent code with ThreadSanitizer and stress testing](Test_concurrent_code_with_ThreadSanitizer_and_stress_testing.md)
- [Test embedded bootloaders and firmware update paths](Test_embedded_bootloaders_and_firmware_update_paths.md)
- [Test embedded C++ code on host with hardware abstraction layers](Test_embedded_C++_code_on_host_with_hardware_abstraction_layers.md)
- [Test error handling paths - exceptions, error codes, and edge cases](Test_error_handling_paths_-_exceptions_error_codes_and_edge_cases.md)
- [Test interrupt service routines (ISR) and real-time constraints](Test_interrupt_service_routines_ISR_and_real-time_constraints.md)
- [Test memory-constrained embedded code with custom allocators](Test_memory-constrained_embedded_code_with_custom_allocators.md)
- [Test private implementation details vs testing through public interfaces](Test_private_implementation_details_vs_testing_through_public_interfaces.md)
- [Test safety-critical C++ code for MISRA and AUTOSAR compliance](Test_safety-critical_C++_code_for_MISRA_and_AUTOSAR_compliance.md)
- [Test template-heavy code with explicit instantiation and type lists](Test_template-heavy_code_with_explicit_instantiation_and_type_lists.md)
- [Use Catch2 as a lightweight alternative to Google Test](Use_Catch2_as_a_lightweight_alternative_to_Google_Test.md)
- [Use CTest and CMake testing infrastructure effectively](Use_CTest_and_CMake_testing_infrastructure_effectively.md)
- [Use Hardware-in-the-Loop (HIL) testing for embedded C++ systems](Use_Hardware-in-the-Loop_HIL_testing_for_embedded_C++_systems.md)
- [Use mock objects to simulate hardware peripherals in embedded testing](Use_mock_objects_to_simulate_hardware_peripherals_in_embedded_testing.md)
- [Use static analysis tools (clang-tidy, cppcheck) as part of testing](Use_static_analysis_tools_clang-tidy_cppcheck_as_part_of_testing.md)
- [Use test fixtures for shared setup and teardown in Google Test](Use_test_fixtures_for_shared_setup_and_teardown_in_Google_Test.md)
- [Write effective unit tests with Arrange-Act-Assert pattern](Write_effective_unit_tests_with_Arrange-Act-Assert_pattern.md)
- [Write integration tests for multi-component systems in C++](Write_integration_tests_for_multi-component_systems_in_C++.md)
- [Write property-based tests using RapidCheck for C++](Write_property-based_tests_using_RapidCheck_for_C++.md)

## Notes

- Write tests before or alongside code - testing after the fact often misses edge cases.
- Test behavior, not implementation - tests should survive refactoring.
- Use fixtures (SetUp/TearDown) to share expensive setup between related tests.
- Parameterized tests (TEST_P) reduce duplication when testing the same logic with different inputs.
- Test names should describe the scenario and expected outcome - ThrowsOnNullInput, not Test1.
- Arrange-Act-Assert (AAA) pattern structures each test clearly.
- Test error paths as thoroughly as success paths - most bugs hide in edge cases.
- Keep unit tests fast (under 1ms each) - slow tests discourage running them frequently.
- Death tests (EXPECT_DEATH) verify that code correctly terminates on invalid input.
- Use CI to run tests on every commit - broken tests should block merging.
