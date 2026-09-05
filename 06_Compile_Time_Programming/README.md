# Compile-Time Programming

constexpr, consteval, static_assert, template metaprogramming, and compile-time computation techniques.

**Topics:** 29

## Contents

- [Build a Compile-Time Finite State Machine Using Template Specialization](Build_a_compile-time_finite_state_machine_using_template_specialization.md)
- [Generate Dispatch Tables at Compile Time Using Parameter Pack Expansion](Generate_dispatch_tables_at_compile_time_using_parameter_pack_expansion.md)
- [Know constexpr exception support status and workarounds](Know_constexpr_exception_support_status_and_workarounds.md)
- [Understand `requires` Expressions and the `requires` Clause (C++20)](Understand_requires_expressions_and_the_requires_clause_C++20.md)
- [Understand `static_assert` and Its Role in Template Constraints](Understand_static_assert_and_its_role_in_template_constraints.md)
- [Understand expansion statements for C++26](Understand_expansion_statements_for_Cpp26.md)
- [Understand How to Use `if consteval` (C++23) to Branch on Constant Evaluation Context](Understand_how_to_use_if_consteval_C++23_to_branch_on_constant_evaluation_contex.md)
- [Understand Pack Indexing (C++26) for Accessing Parameter Pack Elements by Index](Understand_pack_indexing_C++26_for_accessing_parameter_pack_elements_by_index.md)
- [Understand SFINAE with `enable_if` vs Concepts - Choose Concepts](Understand_SFINAE_with_enable_if_vs_concepts_choose_concepts.md)
- [Use `__cpp_*` Feature Test Macros to Detect Standard Library Features](Use___cpp__feature_test_macros_to_detect_standard_library_features.md)
- [Use `consteval` for Functions That Must Only Run at Compile Time (C++20)](Use_consteval_for_functions_that_must_only_run_at_compile_time_C++20.md)
- [Use `constexpr` Containers and Algorithms (C++20)](Use_constexpr_containers_and_algorithms_C++20.md)
- [Use `constexpr` Lambdas (C++17) in Compile-Time Algorithms](Use_constexpr_lambdas_C++17_in_compile-time_algorithms.md)
- [Use `constexpr` Lambdas as Compile-Time Predicates](Use_constexpr_lambdas_as_compile-time_predicates.md)
- [Use `if constexpr` to Select Code Branches at Compile Time](Use_if_constexpr_to_select_code_branches_at_compile_time.md)
- [Use `std::array` as a Compile-Time Lookup Table](Use_stdarray_as_a_compile-time_lookup_table.md)
- [Use `std::array` with `constexpr` to Generate Lookup Tables at Compile Time](Use_stdarray_with_constexpr_to_generate_lookup_tables_at_compile_time.md)
- [Use `std::bit_width`, `std::countl_zero`, and Other Bit Utilities (C++20)](Use_stdbit_width_stdcountl_zero_and_other_bit_utilities_C++20.md)
- [Use `std::integral_constant` and Compile-Time Value Wrappers](Use_stdintegral_constant_and_compile-time_value_wrappers.md)
- [Use `std::is_constant_evaluated` (C++20) for Dual Compile/Runtime Paths](Use_stdis_constant_evaluated_C++20_for_dual_compileruntime_paths.md)
- [Use `std::make_integer_sequence` and `std::index_sequence` for Compile-Time Iteration](Use_stdmake_integer_sequence_and_stdindex_sequence_for_compile-time_iteration.md)
- [Use `std::to_array` (C++20) to Deduce Array Size from Initializer](Use_stdto_array_C++20_to_deduce_array_size_from_initializer.md)
- [Use constexpr dynamic memory allocation with transient allocation (C++20)](Use_constexpr_dynamic_memory_allocation_with_transient_allocation_Cpp20.md)
- [Use static_assert with concept-based diagnostics for better error messages](Use_static_assert_with_concept-based_diagnostics_for_better_error_messages.md)
- [Write `constexpr` Functions That Work Both at Compile Time and Runtime](Write_constexpr_functions_that_work_both_at_compile_time_and_runtime.md)
- [Write a Compile-Time Prime Sieve as a `constexpr std::array`](Write_a_compile-time_prime_sieve_as_a_constexpr_stdarray.md)
- [Write a Compile-Time String Parser Using `consteval`](Write_a_compile-time_string_parser_using_consteval.md)
- [Write Compile-Time Hash Maps Using `constexpr` Arrays (Perfect Hashing)](Write_compile-time_hash_maps_using_constexpr_arrays_perfect_hashing.md)
- [Write Compile-Time String Parsing Using `consteval` and `std::string_view`](Write_compile-time_string_parsing_using_consteval_and_stdstring_view.md)

## Notes

- `constexpr` functions can run at both compile time and runtime - the compiler decides based on context.
- `consteval` (C++20) forces compile-time evaluation - use it for immediate functions that must never run at runtime.
- `constexpr` containers (`std::vector`, `std::string`) arrived in C++20 for transient allocations.
- Template metaprogramming is being replaced by `constexpr` and `consteval` for most use cases.
- `static_assert` validates compile-time conditions with clear error messages.
- `std::is_constant_evaluated()` lets a function detect whether it's running at compile time.
- Compile-time string processing is possible with `constexpr std::string` in C++20.
- `constexpr` virtual functions work since C++20 for compile-time polymorphism.
- Fold expressions (C++17) replace recursive template patterns for variadic operations.
- `std::source_location` provides compile-time file/line info without macros.
