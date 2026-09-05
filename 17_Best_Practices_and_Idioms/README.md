# Best Practices & Idioms

Core Guidelines, naming conventions, const-correctness, API design, and proven C++ idioms.

**Topics:** 55

## Contents

- [Apply const-propagation through pointer and reference members](Apply_const-propagation_through_pointer_and_reference_members.md)
- [Apply data-oriented design for cache-friendly data structures](Apply_data-oriented_design_for_cache-friendly_data_structures.md)
- [Apply data-oriented design: prefer struct-of-arrays over array-of-structs](Apply_data-oriented_design_prefer_struct-of-arrays_over_array-of-structs.md)
- [Apply the copy-and-swap idiom for exception-safe assignment](Apply_the_copy-and-swap_idiom_for_exception-safe_assignment.md)
- [Apply the Law of Demeter to reduce coupling between classes](Apply_the_Law_of_Demeter_to_reduce_coupling_between_classes.md)
- [Apply the Principle of Minimal Interface to reduce coupling](Apply_the_Principle_of_Minimal_Interface_to_reduce_coupling.md)
- [Apply the Single Responsibility Principle at the function level](Apply_the_Single_Responsibility_Principle_at_the_function_level.md)
- [Apply the type-state pattern to prevent operations on invalid states](Apply_the_type-state_pattern_to_prevent_operations_on_invalid_states.md)
- [Follow the Core Guidelines: prefer interfaces that are easy to use correctly](Follow_the_Core_Guidelines_prefer_interfaces_that_are_easy_to_use_correctly.md)
- [Know the C++ Core Guidelines' resource management checklist](Know_the_C++_Core_Guidelines_resource_management_checklist.md)
- [Know when NOT to use templates: prefer concrete types for non-generic code](Know_when_NOT_to_use_templates_prefer_concrete_types_for_non-generic_code.md)
- [Prefer algorithms with projections over manual key extraction lambdas](Prefer_algorithms_with_projections_over_manual_key_extraction_lambdas.md)
- [Prefer composition over inheritance for code reuse](Prefer_composition_over_inheritance_for_code_reuse.md)
- [Prefer free functions over member functions for generic operations](Prefer_free_functions_over_member_functions_for_generic_operations.md)
- [Profile before optimizing and use data-driven optimization](Profile_before_optimizing_and_use_data-driven_optimization.md)
- [Understand ABI stability and binary compatibility concerns](Understand_ABI_stability_and_binary_compatibility_concerns.md)
- [Understand and apply the Dependency Inversion Principle in C++](Understand_and_apply_the_Dependency_Inversion_Principle_in_C++.md)
- [Understand and prevent undefined behavior (UB)](Understand_and_prevent_undefined_behavior_UB.md)
- [Understand object lifetime extension pitfalls with range-based for](Understand_object_lifetime_extension_pitfalls_with_range-based_for.md)
- [Understand the as-if rule and what the compiler is allowed to optimize](Understand_the_as-if_rule_and_what_the_compiler_is_allowed_to_optimize.md)
- [Understand the costs of header inclusion and how to minimize them](Understand_the_costs_of_header_inclusion_and_how_to_minimize_them.md)
- [Understand the fragile base class problem and how to mitigate it](Understand_the_fragile_base_class_problem_and_how_to_mitigate_it.md)
- [Understand the Lakos Rule: noexcept for narrow contracts](Understand_the_Lakos_Rule_noexcept_for_narrow_contracts.md)
- [Understand the Passkey idiom for selective friend access](Understand_the_Passkey_idiom_for_selective_friend_access.md)
- [Understand the Poison Pill Idiom to disable specific base class functionality](Understand_the_Poison_Pill_Idiom_to_disable_specific_base_class_functionality.md)
- [Use [[assume(expr)]] (C++23) to communicate assumptions to the optimizer](Use_assumeexpr_C++23_to_communicate_assumptions_to_the_optimizer.md)
- [Use [[nodiscard]] on types (not just functions) to enforce checked error returns](Use_nodiscard_on_types_not_just_functions_to_enforce_checked_error_returns.md)
- [Use designated initializers (C++20) for self-documenting struct initialization](Use_designated_initializers_C++20_for_self-documenting_struct_initialization.md)
- [Use early return to reduce nesting and improve readability (guard clauses)](Use_early_return_to_reduce_nesting_and_improve_readability_guard_clauses.md)
- [Use enum class bitmasks safely with a library helper](Use_enum_class_bitmasks_safely_with_a_library_helper.md)
- [Use named constants instead of magic numbers throughout the codebase](Use_named_constants_instead_of_magic_numbers_throughout_the_codebase.md)
- [Use Named Constructor Idiom for expressive object construction](Use_Named_Constructor_Idiom_for_expressive_object_construction.md)
- [Use Named Return Value Optimization (NRVO) friendly code patterns](Use_Named_Return_Value_Optimization_NRVO_friendly_code_patterns.md)
- [Use poison pills to prevent accidental conversions in generic code](Use_poison_pills_to_prevent_accidental_conversions_in_generic_code.md)
- [Use policy tags and tag types for zero-cost interface parameterization](Use_policy_tags_and_tag_types_for_zero-cost_interface_parameterization.md)
- [Use precondition documentation with Expects (GSL) or [[pre]] (C++26)](Use_precondition_documentation_with_Expects_GSL_or_pre_C++26.md)
- [Use semantic versioning and ABI versioning for shared libraries](Use_semantic_versioning_and_ABI_versioning_for_shared_libraries.md)
- [Use static analysis tools: clang-tidy, cppcheck, and sanitizers](Use_static_analysis_tools_clang-tidy_cppcheck_and_sanitizers.md)
- [Use static polymorphism via templates to avoid vtable overhead in hot paths](Use_static_polymorphism_via_templates_to_avoid_vtable_overhead_in_hot_paths.md)
- [Use std::as_const and const_cast safely](Use_stdas_const_and_const_cast_safely.md)
- [Use std::exchange for implementing move semantics cleanly](Use_stdexchange_for_implementing_move_semantics_cleanly.md)
- [Use std::launder and understand its necessity with placement new](Use_stdlaunder_and_understand_its_necessity_with_placement_new.md)
- [Use std::numeric_limits and std::clamp for robust numeric code](Use_stdnumeric_limits_and_stdclamp_for_robust_numeric_code.md)
- [Use std::unreachable() (C++23) to mark impossible code paths](Use_stdunreachable_C++23_to_mark_impossible_code_paths.md)
- [Use string_view for compile-time string operations with constexpr](Use_string_view_for_compile-time_string_operations_with_constexpr.md)
- [Use strong typedefs (phantom types) to prevent unit confusion](Use_strong_typedefs_phantom_types_to_prevent_unit_confusion.md)
- [Use structured bindings with custom types via tuple protocol](Use_structured_bindings_with_custom_types_via_tuple_protocol.md)
- [Use structured logging with key-value pairs instead of printf-style strings](Use_structured_logging_with_key-value_pairs_instead_of_printf-style_strings.md)
- [Use the Empty Base Class Optimization (EBO) with [[no_unique_address]]](Use_the_Empty_Base_Class_Optimization_EBO_with_no_unique_address.md)
- [Use the Pimpl idiom to reduce compilation dependencies](Use_the_Pimpl_idiom_to_reduce_compilation_dependencies.md)
- [Use the type-safe builder pattern with designated initializers](Use_the_type-safe_builder_pattern_with_designated_initializers.md)
- [Write const-correct code throughout](Write_const-correct_code_throughout.md)
- [Write self-documenting code: expressive names and minimal comments](Write_self-documenting_code_expressive_names_and_minimal_comments.md)
- [Write self-testing code using constexpr and static_assert as documentation](Write_self-testing_code_using_constexpr_and_static_assert_as_documentation.md)
- [Write test fixtures that use RAII for resource management](Write_test_fixtures_that_use_RAII_for_resource_management.md)

## Notes

- Follow the Rule of Zero: let compiler-generated special members handle resource management wherever possible - the less destructor/copy/move boilerplate you write, the less there is to get wrong.
- Use RAII for all resources - files, locks, sockets, memory. If you're writing manual cleanup code, there's almost certainly a wrapper type that should be doing that for you.
- Prefer `const` by default - const references, const member functions, const local variables. Add mutation only where you actually need it.
- Apply the Single Responsibility Principle - each class and function should do one thing well. If you're struggling to name something, it's often because it's doing too many things.
- Use `[[nodiscard]]` on functions where ignoring the return value is likely a bug - error codes and resource handles are the classic cases.
- Prefer `std::string_view` parameters over `const std::string&` when you don't need ownership. It works with string literals, `std::string`, and any other contiguous character sequence without copying.
- Use `enum class` (scoped enums) to prevent implicit conversions and namespace pollution. Unscoped enums leak their enumerators into the enclosing scope and convert silently to `int`.
- Name types and functions to express intent - code is read far more than it is written, and a well-chosen name is the cheapest form of documentation you have.
- Avoid raw `new`/`delete` - use smart pointers and containers exclusively. Raw owning pointers are exception-unsafe and easy to leak.
- Write `noexcept` on move constructors and move assignment operators - containers like `std::vector` use this information to decide whether they can move elements during reallocation, and getting it wrong means copies where you expect moves.
