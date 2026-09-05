# Memory & Ownership

Smart pointers (unique_ptr, shared_ptr), RAII, allocators, PMR, memory layout, and ownership semantics.

**Topics:** 36

## Contents

- [Handle Exceptions Safely in Constructors That Manage Multiple Resources](Handle_exceptions_safely_in_constructors_that_manage_multiple_resources.md)
- [Implement a Memory Pool with Free-List Recycling](Implement_a_memory_pool_with_free-list_recycling.md)
- [Know How to Use `std::allocator_traits` to Write Allocator-Aware Code](Know_how_to_use_stdallocator_traits_to_write_allocator-aware_code.md)
- [Know out-of-line allocation strategies for coroutine frames](Know_out-of-line_allocation_strategies_for_coroutine_frames.md)
- [Know the Difference Between `operator new` and `::operator new`](Know_the_difference_between_operator_new_and_operator_new.md)
- [Know the Rules of the Rule of Zero, Rule of Three, and Rule of Five](Know_the_rules_of_the_Rule_of_Zero_Rule_of_Three_and_Rule_of_Five.md)
- [Know When `shared_ptr` Is Appropriate and Understand Its Overhead](Know_when_shared_ptr_is_appropriate_and_understand_its_overhead.md)
- [Know When to Use Intrusive Reference Counting vs `std::shared_ptr`](Know_when_to_use_intrusive_reference_counting_vs_stdshared_ptr.md)
- [Understand `aligned_alloc` and Over-Aligned Types](Understand_aligned_alloc_and_over-aligned_types.md)
- [Understand `aligned_storage`, `aligned_union`, and Their C++23 Deprecation](Understand_aligned_storage_aligned_union_and_their_C++23_deprecation.md)
- [Understand `memory_resource` Interface and How to Chain Fallback Allocators](Understand_memory_resource_interface_and_how_to_chain_fallback_allocators.md)
- [Understand Dangling Pointers vs Dangling References — UB in Both Cases](Understand_dangling_pointers_vs_dangling_references_UB_in_both_cases.md)
- [Understand How Deleter Types Affect `unique_ptr` Size](Understand_how_deleter_types_affect_unique_ptr_size.md)
- [Understand implicit object creation and std::start_lifetime_as (C++20/C++23)](Understand_implicit_object_creation_and_stdstdstart_lifetime_as_Cpp20_and_Cpp23.md)
- [Understand Placement `new` and Its Valid Use Cases](Understand_placement_new_and_its_valid_use_cases.md)
- [Understand RAII and Apply It to All Resource Types](Understand_RAII_and_apply_it_to_all_resource_types.md)
- [Understand Stack vs Heap Allocation Tradeoffs and Prefer Stack Allocation](Understand_stack_vs_heap_allocation_tradeoffs_and_prefer_stack_allocation.md)
- [Understand std::destroy_at vs Explicit Destructor Call and Their Guarantees](Understand_stddestroy_at_vs_explicit_destructor_call_and_their_guarantees.md)
- [Understand std::owner_less for comparing weak_ptr in ordered containers](Understand_stdstdowner_less_for_comparing_weak_ptr_in_ordered_containers.md)
- [Understand std::pmr::monotonic_buffer_resource and Arena Allocation](Understand_stdpmrmonotonic_buffer_resource_and_arena_allocation.md)
- [Understand Strict Aliasing Rules and Their Impact on Optimization](Understand_strict_aliasing_rules_and_their_impact_on_optimization.md)
- [Understand the Dangers of Dangling Pointers and How Ownership Semantics Eliminate Them](Understand_the_dangers_of_dangling_pointers_and_how_ownership_semantics_eliminat.md)
- [Understand the operator new/delete Overloading and Class-Specific Allocators](Understand_the_operator_newdelete_overloading_and_class-specific_allocators.md)
- [Understand the Small String Optimization (SSO) in std::string](Understand_the_small_string_optimization_SSO_in_stdstring.md)
- [Use make_unique and make_shared Instead of Raw new](Use_make_unique_and_make_shared_instead_of_raw_new.md)
- [Use std::assume_aligned (C++20) to Inform the Optimizer of Pointer Alignment](Use_stdassume_aligned_C++20_to_inform_the_optimizer_of_pointer_alignment.md)
- [Use std::destroy, std::construct_at, and Related Low-Level Memory Utilities (C++17/20)](Use_stddestroy_stdconstruct_at_and_related_low-level_memory_utilities_C++1720.md)
- [Use std::make_shared_for_overwrite and std::make_unique_for_overwrite (C++20)](Use_stdstdmake_shared_for_overwrite_and_stdstdmake_unique_for_overwrite_Cpp20.md)
- [Use std::out_ptr and std::inout_ptr (C++23) for Smart Pointer Interop with C APIs](Use_stdout_ptr_and_stdinout_ptr_C++23_for_smart_pointer_interop_with_C_APIs.md)
- [Use std::pmr::string and PMR Containers to Avoid Allocator Template Proliferation](Use_stdpmrstring_and_PMR_containers_to_avoid_allocator_template_proliferation.md)
- [Use std::pmr::unsynchronized_pool_resource for Fast Thread-Local Pool Allocation](Use_stdpmrunsynchronized_pool_resource_for_fast_thread-local_pool_allocation.md)
- [Use std::span (C++20) to Pass Non-Owning Views to Contiguous Memory](Use_stdspan_C++20_to_pass_non-owning_views_to_contiguous_memory.md)
- [Use std::start_lifetime_as (C++23) for Safe Reinterpretation of Byte Arrays](Use_stdstart_lifetime_as_C++23_for_safe_reinterpretation_of_byte_arrays.md)
- [Use std::unique_ptr as the Default Ownership Smart Pointer](Use_stdunique_ptr_as_the_default_ownership_smart_pointer.md)
- [Use std::unique_ptr with Custom Deleters for Non-Heap Resources](Use_stdunique_ptr_with_custom_deleters_for_non-heap_resources.md)
- [Use std::weak_ptr to Break Cyclic Ownership](Use_stdweak_ptr_to_break_cyclic_ownership.md)

## Notes

- std::unique_ptr is zero-overhead — prefer it as the default ownership tool
- std::shared_ptr has atomic reference counting overhead — don't use it unless sharing is truly needed
- std::weak_ptr breaks cyclic references and enables safe observer patterns
- Custom deleters on unique_ptr change its type; on shared_ptr they don't (type erasure)
- Always use std::make_unique / std::make_shared — they are exception-safe and efficient
- Placement new constructs objects in pre-allocated memory — pair it with explicit destructor calls
- The allocator model in C++ separates allocation from construction — understand std::pmr for arena patterns
- Stack vs heap: stack is faster but limited; prefer stack allocation unless lifetime exceeds scope
- RAII (Resource Acquisition Is Initialization) is the core C++ idiom for resource management
- std::span (C++20) provides safe, non-owning views over contiguous memory
