# Type System & Deduction

Type deduction rules (auto, decltype), type traits, concepts, CTAD, and type-safe vocabulary types (optional, variant, expected).

**Topics:** 31

## Contents

- [Know all the ways a type can be incomplete and the rules around using incomplete types](Know_all_the_ways_a_type_can_be_incomplete_and_the_rules_around_using_incomplete.md)
- [Know how to use std::common_reference_t in range iterator requirements](Know_how_to_use_stdcommon_reference_t_in_range_iterator_requirements.md)
- [Know std::in_place_t and std::in_place_type_t for in-place construction](Know_stdin_place_t_and_stdin_place_type_t_for_in-place_construction.md)
- [Know the standard type traits and how to use them in template constraints](Know_the_standard_type_traits_and_how_to_use_them_in_template_constraints.md)
- [Know when and how to use std::expected (C++23)](Know_when_and_how_to_use_stdexpected_C++23.md)
- [Master auto type deduction rules and when they differ from template deduction](Master_auto_type_deduction_rules_and_when_they_differ_from_template_deduction.md)
- [Understand `decltype` and `decltype(auto)`](Understand_decltype_and_decltypeauto.md)
- [Understand `std::any` and When to Prefer It Over `void*` or `variant`](Understand_stdany_and_when_to_prefer_it_over_void_or_variant.md)
- [Understand `std::common_reference` and Its Role in Ranges](Understand_stdcommon_reference_and_its_role_in_ranges.md)
- [Understand `std::conditional_t` for Selecting Types at Compile Time](Understand_stdconditional_t_for_selecting_types_at_compile_time.md)
- [Understand `std::is_nothrow_move_constructible` and Its Effect on `std::vector`](Understand_stdis_nothrow_move_constructible_and_its_effect_on_vector.md)
- [Understand class template argument deduction (CTAD, C++17)](Understand_class_template_argument_deduction_CTAD_C++17.md)
- [Understand Concepts (C++20) as named type constraints](Understand_Concepts_C++20_as_named_type_constraints.md)
- [Understand CTAD Deduction Guides for User-Defined Class Templates](Understand_CTAD_deduction_guides_for_user-defined_class_templates.md)
- [Understand Overload Resolution: Ranking of Conversion Sequences](Understand_overload_resolution_ranking_of_conversion_sequences.md)
- [Understand the Concepts Standard Library: `std::regular`, `std::semiregular`, `std::copyable`](Understand_the_Concepts_standard_library_stdregular_stdsemiregular_stdcopyable.md)
- [Understand the Difference Between `std::decay` and `std::remove_reference`](Understand_the_difference_between_stddecay_and_stdremove_reference.md)
- [Understand the Interaction Between `auto` and `initializer_list` Deduction](Understand_the_interaction_between_auto_and_initializer_list_deduction.md)
- [Use `std::convertible_to` and `std::constructible_from` Concepts](Use_stdconvertible_to_and_stdconstructible_from_concepts.md)
- [Use `std::is_invocable` and `std::invoke_result` for Callable Trait Introspection](Use_stdis_invocable_and_stdinvoke_result_for_callable_trait_introspection.md)
- [Use `std::is_nothrow_move_constructible` to Guide Safe Generic Code](Use_stdis_nothrow_move_constructible_to_guide_safe_generic_code.md)
- [Use `std::is_trivially_copyable` to Validate memcpy-Safe Types](Use_stdis_trivially_copyable_to_validate_memcpy-safe_types.md)
- [Use `std::optional` Correctly as a Nullable Value Type](Use_stdoptional_correctly_as_a_nullable_value_type.md)
- [Use `std::remove_pointer` and `std::add_pointer` in Pointer Type Transformations](Use_stdremove_pointer_and_stdadd_pointer_in_pointer_type_transformations.md)
- [Use `std::span` with Static Extent for Compile-Time Size Checking](Use_stdspan_with_static_extent_for_compile-time_size_checking.md)
- [Use `std::type_identity` to Prevent Deduction in Template Arguments](Use_stdtype_identity_to_prevent_deduction_in_template_arguments.md)
- [Use `std::type_index` for Runtime Type Keys in Maps](Use_stdtype_index_for_runtime_type_keys_in_maps.md)
- [Use `std::underlying_type` to Work with Enum Underlying Types](Use_stdunderlying_type_to_work_with_enum_underlying_types.md)
- [Use `std::variant` and `std::visit` for Type-Safe Unions](Use_stdvariant_and_stdvisit_for_type-safe_unions.md)
- [Use Type Aliases with `using` Instead of `typedef`](Use_type_aliases_with_using_instead_of_typedef.md)
- [Use Variable Templates as Type Trait Shorthands](Use_variable_templates_as_type_trait_shorthands.md)

## Notes

- uto deduction strips references and cv-qualifiers — use decltype(auto) to preserve them
- CTAD (Class Template Argument Deduction) reduces verbosity but can produce surprising results
- decltype(expr) vs decltype((expr)) — the extra parentheses change the result to a reference
- Template argument deduction and uto follow almost identical rules (with minor exceptions)
- Type traits (std::is_same_v, std::decay_t) are essential for writing correct generic code
- Forwarding references (T&& in template context) are not rvalue references — know the difference
- std::common_type is used by the standard library to resolve mixed-type expressions
- Deduction guides help CTAD work correctly for user-defined types
- uto in return types can silently change API when implementation changes — be cautious
- Use static_assert with type traits to enforce type constraints at compile time
