# Standard Library — New in C++23/26

C++23 and C++26 added a lot of genuinely useful library features - not just language tweaks, but new containers, new formatting power, new concurrency tools, and new ways to write safer abstractions. This folder covers the ones worth knowing.

**Topics:** 25

## Contents

- [Understand Compile-Time Format Checking with `std::basic_format_string`](Understand_compile-time_format_checking_with_stdbasic_format_string.md)
- [Understand std::hazard_pointer for lock-free memory reclamation](Understand_stdhazard_pointer_for_lock-free_memory_reclamation.md)
- [Understand std::text_encoding for portable character encoding](Understand_stdtext_encoding_for_portable_character_encoding.md)
- [Use `chunk_by`, `stride`, and `cartesian_product` Views (C++23)](Use_stdchunk_by_stride_and_cartesian_product_views_C++23.md)
- [Use `std::copyable_function` (C++26) as a `std::function` Replacement](Use_stdcopyable_function_C++26_as_a_std_function_replacement.md)
- [Use `std::forward_like` (C++23) to Propagate Owner Value Category](Use_stdforward_like_C++23_to_propagate_owner_value_category.md)
- [Use `std::function_ref` (C++26) as a Lightweight Non-Owning Callable Reference](Use_stdfunction_ref_C++26_as_a_lightweight_non-owning_callable_reference.md)
- [Use `std::osyncstream` (C++20) for Synchronized Concurrent Output](Use_stdosyncstream_C++20_for_synchronized_concurrent_output.md)
- [Use `std::ranges::to` (C++23) for Materializing Views into Containers](Use_stdranges_to_C++23_for_materializing_views_into_containers.md)
- [Use `std::to_underlying` (C++23) for Safe Enum-to-Integer Conversion](Use_stdto_underlying_C++23_for_safe_enum_to_integer_conversion.md)
- [Use `std::unreachable()` (C++23) to Mark Impossible Code Paths](Use_stdunreachable_C++23_to_mark_impossible_code_paths.md)
- [Use `std::views::enumerate` (C++23) for Indexed Range Iteration](Use_stdviewsenumerate_C++23_for_indexed_range_iteration.md)
- [Use Formatted Ranges Output with `std::format` (C++23)](Use_formatted_ranges_output_with_stdformat_C++23.md)
- [Use std::debugging utilities (C++26): breakpoint and is_debugger_present](Use_stddebugging_utilities_C++26_breakpoint_and_is_debugger_present.md)
- [Use std::debugging utilities: std::breakpoint and std::is_debugger_present](Use_stddebugging_utilities_stdbreakpoint_and_stdis_debugger_present.md)
- [Use std::flat_map and flat_set in depth: construction, merge, and extract](Use_stdflat_map_and_flat_set_in_depth_construction_merge_and_extract.md)
- [Use std::hive (C++26) in depth: stable addresses, iteration, and erasure](Use_stdhive_C++26_in_depth_stable_addresses_iteration_and_erasure.md)
- [Use std::indirect and std::polymorphic (C++26) for value-semantic polymorphism](Use_stdindirect_and_stdpolymorphic_C++26_for_value-semantic_polymorphism.md)
- [Use std::indirect and std::polymorphic for value-semantic polymorphism](Use_stdindirect_and_stdpolymorphic_for_value-semantic_polymorphism.md)
- [Use std::inplace_vector (C++26) for fixed-capacity stack-allocated sequences](Use_stdinplace_vector_C++26_for_fixed-capacity_stack-allocated_sequences.md)
- [Use std::inplace_vector for fixed-capacity stack allocation](Use_stdinplace_vector_for_fixed-capacity_stack_allocation.md)
- [Use std::linalg (C++26) for portable linear algebra on mdspan](Use_stdlinalg_C++26_for_portable_linear_algebra_on_mdspan.md)
- [Use std::linalg for linear algebra without external dependencies](Use_stdlinalg_for_linear_algebra_without_external_dependencies.md)
- [Use std::rcu (Read-Copy-Update) for scalable read-heavy data structures](Use_stdrcu_Read-Copy-Update_for_scalable_read-heavy_data_structures.md)
- [Use std::text_encoding (C++26) for portable encoding detection](Use_stdtext_encoding_C++26_for_portable_encoding_detection.md)

## Notes

- `std::expected` (C++23) is the standard way to return a value or an error without exceptions.
- `std::flat_map` / `std::flat_set` (C++23) store elements in contiguous memory for better cache performance.
- `std::generator` (C++23) is the standard coroutine-based lazy sequence generator.
- `std::print` / `std::println` (C++23) provide Python-like formatted output.
- `std::ranges::to` (C++23) materializes lazy ranges into concrete containers.
- `std::mdspan` (C++23) provides multi-dimensional non-owning views over contiguous data.
- `std::stacktrace` (C++23) captures call stacks programmatically for diagnostics.
- `import std;` (C++23) imports the entire standard library as a named module.
- `std::views::zip` and `std::views::enumerate` (C++23) add missing range adaptors.
- Static reflection and `std::execution` are the headline features coming in C++26.
