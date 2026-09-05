# Templates & Generic Programming

Templates are one of C++'s most powerful - and most intimidating - features. Once it clicks, though, you'll find yourself reaching for them constantly. This section covers everything from basic function templates to SFINAE, concepts, variadic packs, and template metaprogramming.

**Topics:** 38

## Contents

- [Know the Curiously Recurring Template Pattern (CRTP) and Its Uses](Know_the_Curiously_Recurring_Template_Pattern_CRTP_and_its_uses.md)
- [Understand `std::initializer_list` Interaction with Template Deduction](Understand_stdinitializer_list_interaction_with_template_deduction.md)
- [Understand Class Template Specialization for Traits (Detection Idiom)](Understand_class_template_specialization_for_traits_detection_idiom.md)
- [Understand Concept Subsumption and Overload Ordering](Understand_concept_subsumption_and_overload_ordering.md)
- [Understand Explicit Instantiation and Prevent Implicit Instantiation](Understand_explicit_instantiation_and_prevent_implicit_instantiation.md)
- [Understand How to Detect Whether a Type Is a Specialization of a Template](Understand_how_to_detect_whether_a_type_is_a_specialization_of_a_template.md)
- [Understand Partial and Full Template Specialization](Understand_partial_and_full_template_specialization.md)
- [Understand SFINAE and Know When to Replace It with Concepts](Understand_SFINAE_and_know_when_to_replace_it_with_Concepts.md)
- [Understand Substitution Failure vs Hard Error in SFINAE](Understand_substitution_failure_vs_hard_error_in_SFINAE.md)
- [Understand Template Argument Substitution Order and Its Effect on Errors](Understand_template_argument_substitution_order_and_its_effect_on_errors.md)
- [Understand Template Instantiation and Its Compile-Time Cost](Understand_template_instantiation_and_its_compile-time_cost.md)
- [Understand Template Specialization Ordering and Partial Ordering Rules](Understand_template_specialization_ordering_and_partial_ordering_rules.md)
- [Understand Template Template Parameters](Understand_template_template_parameters.md)
- [Understand the Interaction Between Templates and Exceptions in Stack Unwinding](Understand_the_interaction_between_templates_and_exceptions_in_stack_unwinding.md)
- [Use `std::apply` and `std::make_from_tuple` for Tuple-Based Generic Invocation](Use_stdapply_and_stdmake_from_tuple_for_tuple-based_generic_invocation.md)
- [Use `std::common_type` to Deduce the Common Type of a Set of Types](Use_stdcommon_type_to_deduce_the_common_type_of_a_set_of_types.md)
- [Use `std::conjunction`, `std::disjunction`, and `std::negation` for Compound Type Predicates](Use_stdconjunction_stddisjunction_and_stdnegation_for_compound_type_predicates.md)
- [Use `std::declval` to Refer to Type Expressions Without Constructing Objects](Use_stddeclval_to_refer_to_type_expressions_without_constructing_objects.md)
- [Use `std::function_ref` (C++26) as a Lightweight Non-Owning Callable](Use_stdfunction_ref_C++26_as_a_lightweight_non-owning_callable.md)
- [Use `std::tuple` as a Type-Level Cons Cell for Compile-Time Type Lists](Use_stdtuple_as_a_type-level_cons_cell_for_compile-time_type_lists.md)
- [Use `std::type_traits` to Build a Generic JSON Serializer](Use_stdtype_traits_to_build_a_generic_JSON_serializer.md)
- [Use Abbreviated Function Templates with `auto` Parameters (C++20)](Use_abbreviated_function_templates_with_auto_parameters_C++20.md)
- [Use Concepts to Constrain Return Types, Not Just Parameters](Use_concepts_to_constrain_return_types_and_not_just_parameters.md)
- [Use Fold Expressions (C++17) to Operate on Parameter Packs](Use_fold_expressions_C++17_to_operate_on_parameter_packs.md)
- [Use Non-Type Template Parameters (NTTPs) Including Class Types (C++20)](Use_non-type_template_parameters_NTTPs_including_class_types_C++20.md)
- [Use Tag Dispatch for Selecting Overloads Based on Type Properties](Use_tag_dispatch_for_selecting_overloads_based_on_type_properties.md)
- [Use Template Specialization to Customize Standard Library Traits](Use_template_specialization_to_customize_standard_library_traits.md)
- [Use the Hidden-Friend Idiom to Improve Overload Resolution](Use_the_hidden-friend_idiom_to_improve_overload_resolution.md)
- [Use Variadic Templates and Parameter Packs Correctly](Use_variadic_templates_and_parameter_packs_correctly.md)
- [Write a constexpr string_view Command Dispatcher Using Template Metaprogramming](Write_a_constexpr_string_view_command_dispatcher_using_template_metaprogramming.md)
- [Write a Generic Event Bus Using Templates and Type Erasure](Write_a_generic_event_bus_using_templates_and_type_erasure.md)
- [Write a Generic scope_exit RAII Guard Using Templates and Lambdas](Write_a_generic_scope_exit_RAII_guard_using_templates_and_lambdas.md)
- [Write a Type List and Implement Compile-Time Operations on It](Write_a_type_list_and_implement_compile-time_operations_on_it.md)
- [Write Expression Templates to Defer Computation and Eliminate Temporaries](Write_expression_templates_to_defer_computation_and_eliminate_temporaries.md)
- [Write Expression Templates to Eliminate Temporaries in Arithmetic Chains](Write_expression_templates_to_eliminate_temporaries_in_arithmetic_chains.md)
- [Write Function Templates with Multiple Type Parameters](Write_function_templates_with_multiple_type_parameters.md)
- [Write Recursive Type Lists and Compile-Time Type Manipulation](Write_recursive_type_lists_and_compile-time_type_manipulation.md)
- [Write Type-Erased Wrappers Using Templates](Write_type-erased_wrappers_using_templates.md)

## Notes

- SFINAE (Substitution Failure Is Not An Error) enables compile-time overload selection
- Prefer C++20 concepts over SFINAE - they give clearer error messages and intent
- Template instantiation happens at use site - put definitions in headers (or use explicit instantiation)
- Variadic templates use parameter packs and fold expressions (C++17) for clean recursive patterns
- CRTP (Curiously Recurring Template Pattern) provides static polymorphism without virtual dispatch
- Type erasure combines templates and runtime polymorphism - see std::function, std::any
- Non-type template parameters (NTTP) can be floating-point and class types since C++20
- Template specialization (full and partial) customizes behavior for specific types
- Two-phase lookup: non-dependent names resolve at definition, dependent names at instantiation
- if constexpr (C++17) enables compile-time branching without SFINAE tricks
