# Testing & Verification

Unit testing, property-based testing, fuzzing, code coverage, and verification techniques.

**Topics:** 26

## Contents

- [Apply test-driven development (TDD) workflow in C++](Apply_test-driven_development_TDD_workflow_in_C++.md)
- [Apply the TDD workflow in C++ with GoogleTest](Apply_Test-Driven_Development_TDD_workflow_in_C++_2.md)
- [Avoid misleading micro-benchmarks when measuring optimizations](Use_benchmark-driven_development_measure_before_and_after_every_optimization_2.md)
- [Integrate C++ testing with Python using pybind11 + pytest](Integrate_C++_testing_with_Python_using_pybind11_+_pytest.md)
- [Set up integration testing of C++ code with Python using pybind11](Set_up_integration_testing_of_C++_code_with_Python_using_pybind11.md)
- [Test C++ code from Python using pybind11 and pytest](Test_C++_code_from_Python_using_pybind11_and_pytest.md)
- [Understand test doubles: fakes, stubs, spies, and mocks and when to use each](Understand_test_doubles_fakes_stubs_spies_and_mocks_and_when_to_use_each.md)
- [Understand test isolation and avoid shared mutable state in test suites](Understand_test_isolation_and_avoid_shared_mutable_state_in_test_suites.md)
- [Use Approval Tests for characterization testing of legacy code](Use_Approval_Tests_for_characterization_testing_of_legacy_code.md)
- [Use benchmark-driven development: measure before and after every optimization](Use_benchmark-driven_development_measure_before_and_after_every_optimization.md)
- [Use benchmark-driven development: set and enforce performance budgets](Use_benchmark-driven_development_set_and_enforce_performance_budgets.md)
- [Use Google Mock (gMock) for dependency injection and behavior verification](Use_Google_Mock_gMock_for_dependency_injection_and_behavior_verification.md)
- [Use Google Mock / trompeloeil for mock objects in unit tests](Use_Google_Mock_trompeloeil_for_mock_objects_in_unit_tests.md)
- [Use Google Mock for expressive test doubles in C++ unit tests](Use_Google_Mock_for_expressive_test_doubles_in_C++_unit_tests.md)
- [Use mutation testing to measure test suite effectiveness](Use_mutation_testing_to_measure_test_suite_effectiveness.md)
- [Use mutation testing with Mull to measure test suite quality](Use_mutation_testing_with_Mull_to_measure_test_suite_quality.md)
- [Use mutation testing with mutmut or LLVM's mutation testing to validate test quality](Use_mutation_testing_with_mutmut_or_LLVMs_mutation_testing_to_validate_test_qual.md)
- [Use property-based testing with RapidCheck](Use_property-based_testing_with_RapidCheck.md)
- [Use property-based testing with RapidCheck to find edge cases](Use_property-based_testing_with_RapidCheck_to_find_edge_cases.md)
- [Use sanitizer-driven testing as part of the CI pipeline](Use_sanitizer-driven_testing_as_part_of_the_CI_pipeline.md)
- [Use Sanitizers in CI to catch bugs that tests alone miss](Use_Sanitizers_in_CI_to_catch_bugs_that_tests_alone_miss.md)
- [Use trompeloeil for header-only mocking with Catch2 or doctest](Use_trompeloeil_for_header-only_mocking_with_Catch2_or_doctest.md)
- [Use trompeloeil for header-only mocking with expression templates](Use_trompeloeil_for_header-only_mocking_with_expression_templates.md)
- [Write fuzz-driven unit tests using structure-aware fuzzing](Write_fuzz-driven_unit_tests_using_structure-aware_fuzzing.md)
- [Write parameterized tests for table-driven test cases](Write_parameterized_tests_for_table-driven_test_cases.md)
- [Write testable C++ code by designing for dependency injection](Write_testable_C++_code_by_designing_for_dependency_injection.md)

## Notes

- GoogleTest (gtest) and Catch2 are the two most popular C++ testing frameworks
- Property-based testing (RapidCheck) generates random inputs to find edge cases
- Fuzz testing (libFuzzer, AFL++) discovers crashes and undefined behavior through random mutation
- Mock objects (gmock, trompeloeil) isolate units under test from dependencies
- Code coverage (gcov, llvm-cov) measures which lines are exercised - aim for meaningful coverage, not 100%
- Sanitizers (ASan, TSan, UBSan) should run alongside tests in CI for maximum bug detection
- Static analysis complements testing - analyzers catch bugs that may never trigger at runtime
- Contract testing (pre/post-conditions) validates assumptions at API boundaries
- Benchmark tests (Google Benchmark) prevent performance regressions between releases
- Integration tests verify component interactions - don't rely solely on unit tests
