# Standard Library — Utilities

Utility types and functions: tuple, pair, chrono, formatting, string operations, and more.

**Topics:** 37

## Contents

- [Know std::function and its performance characteristics](Know_stdfunction_and_its_performance_characteristics.md)
- [Understand std::integer_sequence patterns for index tricks](Understand_stdstdinteger_sequence_patterns_for_index_tricks.md)
- [Use std::addressof to safely get an object's address despite operator& overloads](Use_stdaddressof_to_safely_get_an_objects_address_despite_operator_overloads.md)
- [Use std::bit_cast (C++20) for safe type punning](Use_stdbit_cast_C++20_for_safe_type_punning.md)
- [Use std::bit_floor and bit_ceil (C++20) for power-of-two alignment](Use_stdbit_floor_and_bit_ceil_C++20_for_power-of-two_alignment.md)
- [Use std::byte for type-safe raw memory manipulation](Use_stdbyte_for_type-safe_raw_memory_manipulation.md)
- [Use std::byteswap (C++23) and std::endian for portable endianness handling](Use_stdbyteswap_C++23_and_stdendian_for_portable_endianness_handling.md)
- [Use std::chrono calendar and timezone features (C++20)](Use_stdstdchrono_calendar_and_timezone_features_Cpp20.md)
- [Use std::chrono for time measurement and duration arithmetic](Use_stdchrono_for_time_measurement_and_duration_arithmetic.md)
- [Use std::copyable_function (C++26) vs std::move_only_function (C++23) vs std::function](Use_stdstdcopyable_function_Cpp26_vs_stdstdmove_only_function_Cpp23_vs_stdstdfun.md)
- [Use std::expected vs std::optional decision matrix](Use_stdstdexpected_vs_stdstdoptional_decision_matrix.md)
- [Use std::filesystem for portable file system operations (C++17)](Use_stdfilesystem_for_portable_file_system_operations_C++17.md)
- [Use std::format (C++20) for type-safe string formatting](Use_stdformat_C++20_for_type-safe_string_formatting.md)
- [Use std::generator (C++23) as a standard coroutine range](Use_stdgenerator_C++23_as_a_standard_coroutine_range.md)
- [Use std::hash and define custom hash specializations](Use_stdhash_and_define_custom_hash_specializations.md)
- [Use std::initializer_list correctly in constructors and assignment](Use_stdinitializer_list_correctly_in_constructors_and_assignment.md)
- [Use std::is_within_lifetime (C++26) for constexpr lifetime queries](Use_stdstdis_within_lifetime_Cpp26_for_constexpr_lifetime_queries.md)
- [Use std::linalg (C++26) for linear algebra operations](Use_stdlinalg_C++26_for_linear_algebra_operations.md)
- [Use std::mdspan (C++23) for multi-dimensional array views](Use_stdmdspan_C++23_for_multi-dimensional_array_views.md)
- [Use std::midpoint and std::lerp for safe numeric interpolation](Use_stdmidpoint_and_stdlerp_for_safe_numeric_interpolation.md)
- [Use std::numeric_limits to write portable numeric code](Use_stdnumeric_limits_to_write_portable_numeric_code.md)
- [Use std::optional monadic operations: transform, and_then, or_else (C++23)](Use_stdoptional_monadic_operations_transform_and_then_or_else_C++23.md)
- [Use std::print and std::println (C++23) for formatted output](Use_stdprint_and_stdprintln_C++23_for_formatted_output.md)
- [Use std::random_device and mt19937 for high-quality random number generation](Use_stdrandom_device_and_mt19937_for_high-quality_random_number_generation.md)
- [Use std::ratio for compile-time rational arithmetic](Use_stdratio_for_compile-time_rational_arithmetic.md)
- [Use std::reference_wrapper for storing references in containers](Use_stdreference_wrapper_for_storing_references_in_containers.md)
- [Use std::source_location (C++20) for zero-overhead logging metadata](Use_stdsource_location_C++20_for_zero-overhead_logging_metadata.md)
- [Use std::spanstream (C++23) for in-memory I/O on raw buffers](Use_stdspanstream_C++23_for_in-memory_IO_on_raw_buffers.md)
- [Use std::ssize (C++20) to get a signed size from a container](Use_stdssize_C++20_to_get_a_signed_size_from_a_container.md)
- [Use std::stacktrace (C++23) for diagnostic stack capture](Use_stdstacktrace_C++23_for_diagnostic_stack_capture.md)
- [Use std::string's from_chars and to_chars for locale-independent conversion](Use_stdstrings_from_chars_and_to_chars_for_locale-independent_conversion.md)
- [Use std::strong_ordering, weak_ordering, partial_ordering as <=> return types](Use_stdstrong_ordering_weak_ordering_partial_ordering_as_return_types.md)
- [Use std::tie for lexicographic comparison of multiple struct members](Use_stdtie_for_lexicographic_comparison_of_multiple_struct_members.md)
- [Use std::to_chars and std::from_chars for allocation-free conversions](Use_stdto_chars_and_stdfrom_chars_for_allocation-free_conversions.md)
- [Use std::tuple and std::get for heterogeneous value grouping](Use_stdtuple_and_stdget_for_heterogeneous_value_grouping.md)
- [Use std::views::as_const (C++23) to create const views of mutable ranges](Use_stdviewsas_const_C++23_to_create_const_views_of_mutable_ranges.md)
- [Use std::views::keys and views::values to iterate map keys or values](Use_stdviewskeys_and_viewsvalues_to_iterate_map_keys_or_values.md)

## Notes

- std::optional represents a value that may not exist — prefer over sentinel values and raw pointers
- std::variant is a type-safe union — use std::visit with overload sets for pattern matching
- std::any provides type-erased storage — use sparingly, prefer std::variant when types are known
- std::expected (C++23) combines error handling with return values — better than std::optional for errors
- std::format (C++20) replaces printf and << streams with type-safe formatting
- std::chrono provides type-safe time handling — avoid raw integers for durations
- std::filesystem (C++17) handles paths, directory traversal, and file operations portably
- std::string_view avoids copies but beware dangling — it does not own the string
- Monadic operations on std::optional (.and_then, .transform) enable functional chaining (C++23)
- std::to_underlying (C++23) safely converts scoped enums to their underlying type
