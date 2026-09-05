# Core Language Fundamentals

This is the bedrock. Everything else in the project leans on the rules collected here: how types behave, how objects get initialized, how control flow and attributes work, and the quiet language rules that decide whether your code does what you think it does. None of it is glamorous, but it's the stuff that separates "it compiled" from "I know why it compiled." Work through these first.

**Topics:** 67

## Contents

- [Know argument-dependent lookup (ADL) and its implications](Know_argument-dependent_lookup_ADL_and_its_implications.md)
- [Know how integer promotion and usual arithmetic conversions work](Know_how_integer_promotion_and_usual_arithmetic_conversions_work.md)
- [Know how operator overloading rules constrain which operators can be overloaded](Know_how_operator_overloading_rules_constrain_which_operators_can_be_overloaded.md)
- [Know how reinterpret_cast interacts with strict aliasing](Know_how_reinterpret_cast_interacts_with_strict_aliasing.md)
- [Know how user-defined literals work and define your own](Know_how_user-defined_literals_work_and_define_your_own.md)
- [Know how volatile differs from atomic and when each is appropriate](Know_how_volatile_differs_from_atomic_and_when_each_is_appropriate.md)
- [Know implicit vs explicit default, copy, and move member generation rules](Know_implicit_vs_explicit_default_copy_and_move_member_generation_rules.md)
- [Know the [[indeterminate]] attribute (C++26) for uninitialized variables](Know_the_indeterminate_attribute_Cpp26_for_uninitialized_variables.md)
- [Know the difference between standard-layout types and their serialization guarantees](Know_the_difference_between_standard-layout_types_and_their_serialization_guaran.md)
- [Know the difference between std::string::substr returning a copy vs string_view::substr returning a view](Know_the_difference_between_stdstringsubstr_returning_a_copy_vs_string_viewsubst.md)
- [Know the exact rules for when a destructor is implicitly deleted](Know_the_exact_rules_for_when_a_destructor_is_implicitly_deleted.md)
- [Know the initialization order of global and static local variables](Know_the_initialization_order_of_global_and_static_local_variables.md)
- [Know the initialization rules for static local variables and their thread-safety](Know_the_initialization_rules_for_static_local_variables_and_their_thread-safety.md)
- [Know the lifetime extension rules for temporaries bound to const references](Know_the_lifetime_extension_rules_for_temporaries_bound_to_const_references.md)
- [Know the rules for [[no_unique_address]] and its effect on class layout](Know_the_rules_for_no_unique_address_and_its_effect_on_class_layout.md)
- [Know the rules for delegating constructors and their exception safety implications](Know_the_rules_for_delegating_constructors_and_their_exception_safety_implicatio.md)
- [Know the rules for initializing references to base class subobjects](Know_the_rules_for_initializing_references_to_base_class_subobjects.md)
- [Know the rules for when copy/move constructors are implicitly deleted or defaulted](Know_the_rules_for_when_copymove_constructors_are_implicitly_deleted_or_defaulte.md)
- [Know the rules of structured bindings (C++17)](Know_the_rules_of_structured_bindings_C++17.md)
- [Know when and how to use __has_include for conditional compilation](Know_when_and_how_to_use___has_include_for_conditional_compilation.md)
- [Master brace-initialization and the uniform initialization syntax](Master_brace-initialization_and_the_uniform_initialization_syntax.md)
- [Master the rules of aggregate initialization](Master_the_rules_of_aggregate_initialization.md)
- [Understand [[fallthrough]] in switch statements and why it matters](Understand_fallthrough_in_switch_statements_and_why_it_matters.md)
- [Understand aggregate CTAD and its interaction with designated initializers](Understand_aggregate_CTAD_and_its_interaction_with_designated_initializers.md)
- [Understand bit fields and their portability constraints](Understand_bit_fields_and_their_portability_constraints.md)
- [Understand co_return in coroutines and the final_suspend protocol](Understand_co_return_in_coroutines_and_the_final_suspend_protocol.md)
- [Understand constexpr variables and when they are evaluated at compile time](Understand_constexpr_variables_and_when_they_are_evaluated_at_compile_time.md)
- [Understand constinit (C++20) to guarantee static initialization](Understand_constinit_C++20_to_guarantee_static_initialization.md)
- [Understand conversion sequences and overload resolution ranking](Understand_conversion_sequences_and_overload_resolution_ranking.md)
- [Understand explicit constructor and conversion operators](Understand_explicit_constructor_and_conversion_operators.md)
- [Understand function-try-blocks for catching exceptions from member initializers](Understand_function-try-blocks_for_catching_exceptions_from_member_initializers.md)
- [Understand how inheritance affects object layout and pointer adjustment](Understand_how_inheritance_affects_object_layout_and_pointer_adjustment.md)
- [Understand how to read complex C++ type declarations](Understand_how_to_read_complex_C++_type_declarations.md)
- [Understand how virtual function tables (vtables) are laid out in memory](Understand_how_virtual_function_tables_vtables_are_laid_out_in_memory.md)
- [Understand inheriting constructors and their limitations](Understand_inheriting_constructors_and_their_limitations.md)
- [Understand inline variables (C++17) and their use in headers](Understand_inline_variables_C++17_and_their_use_in_headers.md)
- [Understand integer literal suffixes and their effect on type deduction](Understand_integer_literal_suffixes_and_their_effect_on_type_deduction.md)
- [Understand lambda capture of *this vs this (C++17)](Understand_lambda_capture_of_this_vs_this_C++17.md)
- [Understand linkage: internal, external, no linkage, and module linkage](Understand_linkage_internal_external_no_linkage_and_module_linkage.md)
- [Understand member initializer lists and their order of initialization](Understand_member_initializer_lists_and_their_order_of_initialization.md)
- [Understand multidimensional subscript operator[] (C++23)](Understand_multidimensional_subscript_operator_C++23.md)
- [Understand multidimensional subscript operator[] (C++23) - Part 2](Understand_multidimensional_subscript_operator_C++23_2.md)
- [Understand name hiding and why using declarations are sometimes needed](Understand_name_hiding_and_why_using_declarations_are_sometimes_needed.md)
- [Understand pack expansion in base class lists and using declarations](Understand_pack_expansion_in_base_class_lists_and_using_declarations.md)
- [Understand pack expansion in base class lists and using declarations - Part 2](Understand_pack_expansion_in_base_class_lists_and_using_declarations_2.md)
- [Understand placeholder variables with _ and auto (C++26)](Understand_placeholder_variables_with_auto_and_underscore_Cpp26.md)
- [Understand scoped enums (enum class) and their advantages](Understand_scoped_enums_enum_class_and_their_advantages.md)
- [Understand static operator() and static operator[] (C++23)](Understand_static_operator_and_static_operator_Cpp23.md)
- [Understand std::string::substr vs std::string_view::substr lifetime differences](Understand_stdstringsubstr_vs_stdstring_viewsubstr_lifetime_differences.md)
- [Understand template deduction for inherited constructors](Understand_template_deduction_for_inherited_constructors.md)
- [Understand the [[nodiscard]], [[maybe_unused]], [[likely]], [[unlikely]] attributes](Understand_the_nodiscard_maybe_unused_likely_unlikely_attributes.md)
- [Understand the difference between static_cast, reinterpret_cast, const_cast, and dynamic_cast](Understand_the_difference_between_static_cast_reinterpret_cast_const_cast_and_dy.md)
- [Understand the difference between value types and reference types](Understand_the_difference_between_value_types_and_reference_types.md)
- [Understand the interaction between const member functions and mutable members](Understand_the_interaction_between_const_member_functions_and_mutable_members.md)
- [Understand the Itanium ABI name mangling and extern C](Understand_the_Itanium_ABI_name_mangling_and_extern_C.md)
- [Understand the lookup rules for operator overloads and ADL interaction](Understand_the_lookup_rules_for_operator_overloads_and_ADL_interaction.md)
- [Understand the most vexing parse and brace initialization as a fix](Understand_the_most_vexing_parse_and_brace_initialization_as_a_fix.md)
- [Understand the one definition rule (ODR) precisely](Understand_the_one_definition_rule_ODR_precisely.md)
- [Understand trailing return types and when they are required](Understand_trailing_return_types_and_when_they_are_required.md)
- [Understand two-phase name lookup in templates](Understand_two-phase_name_lookup_in_templates.md)
- [Understand unsequenced evaluation and sequence points](Understand_unsequenced_evaluation_and_sequence_points.md)
- [Understand what makes a type trivially copyable and why it matters for serialization](Understand_what_makes_a_type_trivially_copyable_and_why_it_matters_for_serializa.md)
- [Understand zero-initialization, default-initialization, and value-initialization](Understand_zero-initialization_default-initialization_and_value-initialization.md)
- [Use if-init and switch-init statements (C++17)](Use_if-init_and_switch-init_statements_C++17.md)
- [Use nullptr instead of NULL or 0 for null pointers](Use_nullptr_instead_of_NULL_or_0_for_null_pointers.md)
- [Use range-based for loops correctly and know their caveats](Use_range-based_for_loops_correctly_and_know_their_caveats.md)
- [Use string literals correctly: raw strings, char8_t, UTF-8, and string_view](Use_string_literals_correctly_raw_strings_char8_t_UTF-8_and_string_view.md)

## Notes

A few things worth carrying with you as you read this section:

- Initialization is one of the most error-prone corners of C++ - get the brace-init vs parentheses differences straight early, before bad habits set in.
- ADL (argument-dependent lookup) is the key to understanding how operators and free functions actually get found - it shows up everywhere once you notice it.
- The rule of 0/3/5 governs which special members the compiler writes for you - review it before writing any class that owns a resource.
- `volatile` is NOT a synchronization mechanism - reach for `std::atomic` whenever threads are involved.
- Structured bindings (C++17) make working with tuples, pairs, and aggregates far more pleasant.
- `constinit` (C++20) heads off the static initialization order fiasco for globals that aren't `constexpr`.
- `reinterpret_cast` almost always violates strict aliasing - prefer `std::bit_cast` (C++20) when you just need to reinterpret bits.
- Scoped enums (`enum class`) shut down implicit conversions and stop names leaking into the surrounding scope.
- Remember that member initializer list order follows *declaration* order, not the order you wrote in the list.
- Static local variable initialization has been thread-safe since C++11 - the so-called "magic statics."
