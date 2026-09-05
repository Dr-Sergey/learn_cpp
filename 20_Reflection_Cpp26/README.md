# Reflection (C++26)

Compile-time reflection is one of the most transformative additions in C++26. It lets you ask the compiler about types, structs, and enums at compile time - and then act on the answers to generate code automatically. No more macros, no more external code generators, no more keeping hand-written schema definitions in sync with your data structures.

**Topics:** 22

## Contents

- [Enumerate enum values and names at compile time using reflection](Enumerate_enum_values_and_names_at_compile_time_using_reflection.md)
- [Enumerate struct members at compile time using reflection](Enumerate_struct_members_at_compile_time_using_reflection.md)
- [Enumerate struct members at compile time using std::meta::members_of](Enumerate_struct_members_at_compile_time_using_stdmetamembers_of.md)
- [Generate code from reflections using template splicing [:...]:](Generate_code_from_reflections_using_template_splicing_.md)
- [Generate switch dispatch tables from enum reflections](Generate_switch_dispatch_tables_from_enum_reflections.md)
- [Implement compile-time ORM and query generation via reflection](Implement_compile-time_ORM_and_query_generation_via_reflection.md)
- [Implement compile-time struct serialization using reflection](Implement_compile-time_struct_serialization_using_reflection.md)
- [Understand static reflection with the ^ operator and std::meta](Understand_static_reflection_with_the_operator_and_stdmeta.md)
- [Understand the C++26 static reflection model with the ^ operator](Understand_the_C++26_static_reflection_model_with_the_operator.md)
- [Understand the C++26 static reflection model with the ^ splice operator](Understand_the_C++26_static_reflection_model_with_the_splice_operator.md)
- [Understand the difference between compile-time and runtime reflection in C++](Understand_the_difference_between_compile-time_and_runtime_reflection_in_C++.md)
- [Understand the limits of C++26 reflection: what is and is not reflectable](Understand_the_limits_of_C++26_reflection_what_is_and_is_not_reflectable.md)
- [Understand the splice operator [: :] for injecting reflections back into code](Understand_the_splice_operator_for_injecting_reflections_back_into_code.md)
- [Understand token injection and code injection proposals](Understand_token_injection_and_code_injection_proposals.md)
- [Use reflection for automatic debug printing and logging](Use_reflection_for_automatic_debug_printing_and_logging.md)
- [Use reflection for compile-time interface checking beyond concepts](Use_reflection_for_compile-time_interface_checking_beyond_concepts.md)
- [Use reflection for enum-to-string conversion without macros](Use_reflection_for_enum-to-string_conversion_without_macros.md)
- [Use reflection to generate comparison and hash operators automatically](Use_reflection_to_generate_comparison_and_hash_operators_automatically.md)
- [Use reflection to implement automatic enum-to-string conversion](Use_reflection_to_implement_automatic_enum-to-string_conversion.md)
- [Use reflection to implement automatic JSON serialization](Use_reflection_to_implement_automatic_JSON_serialization.md)
- [Use reflection to implement automatic struct serialization](Use_reflection_to_implement_automatic_struct_serialization.md)
- [Use std::meta::define_class to synthesize new types at compile time](Use_stdmetadefine_class_to_synthesize_new_types_at_compile_time.md)

## Notes

- C++26 static reflection (P2996) provides compile-time introspection of types and declarations - the compiler becomes queryable at build time.
- `^^T` (the reflection operator) produces a `meta::info` object representing type `T` - think of it as a compile-time handle to a type.
- Splicing (`[:refl:]`) converts a `meta::info` handle back into code - it is the inverse of reflection, bridging the meta world and the code world.
- Reflection enables automatic serialization, ORM, and enum-to-string without any macros or manual registration.
- `std::meta::members_of` enumerates class members at compile time, giving you the struct's own field list to iterate.
- `template for` loops iterate over reflected members, replacing old-style recursive template metaprogramming with straightforward iteration.
- All reflection happens within `consteval` functions - introspection is a compile-time-only activity with zero runtime overhead.
- Annotations (proposed alongside P2996) may allow attaching metadata to declarations for reflection consumption in a future standard.
- Reflection makes macros, X-macros, and external code generation tools largely obsolete for structural introspection tasks.
- The `std::meta` namespace provides the building blocks - `identifier_of`, `type_of`, `offset_of`, `size_of`, `enumerators_of`, and more.
