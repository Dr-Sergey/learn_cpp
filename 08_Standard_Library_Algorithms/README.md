# Standard Library - Algorithms

STL algorithms, parallel algorithms, ranges algorithms, and their correct application.

**Topics:** 39

## Contents

- [Know set and heap algorithms: set_union, merge, push_heap, pop_heap](Know_set_and_heap_algorithms_set_union_merge_push_heap_pop_heap.md)
- [Prefer standard algorithms over hand-rolled loops](Prefer_standard_algorithms_over_hand-rolled_loops.md)
- [Understand and use std::sort, partial_sort, nth_element, and stable_sort](Understand_and_use_stdsort_partial_sort_nth_element_and_stable_sort.md)
- [Use binary search algorithms on sorted ranges](Use_binary_search_algorithms_on_sorted_ranges.md)
- [Use erase-remove idiom and the erase_if free functions (C++20)](Use_erase-remove_idiom_and_the_erase_if_free_functions_C++20.md)
- [Use parallel execution policies (C++17) for algorithm parallelism](Use_parallel_execution_policies_C++17_for_algorithm_parallelism.md)
- [Use std::accumulate and std::reduce with custom binary operations](Use_stdaccumulate_and_stdreduce_with_custom_binary_operations.md)
- [Use std::adjacent_find and std::unique for duplicate detection](Use_stdadjacent_find_and_stdunique_for_duplicate_detection.md)
- [Use std::copy_if, std::remove_if, and std::count_if with predicates](Use_stdcopy_if_stdremove_if_and_stdcount_if_with_predicates.md)
- [Use std::gcd and std::lcm (C++17) for number theory in templates](Use_stdgcd_and_stdlcm_C++17_for_number_theory_in_templates.md)
- [Use std::inclusive_scan and exclusive_scan for prefix sum computation](Use_stdinclusive_scan_and_exclusive_scan_for_prefix_sum_computation.md)
- [Use std::iota for filling ranges with sequentially increasing values](Use_stdiota_for_filling_ranges_with_sequentially_increasing_values.md)
- [Use std::iota to fill a range with incrementing values](Use_stdiota_to_fill_a_range_with_incrementing_values.md)
- [Use std::is_sorted and is_sorted_until for pre-condition checking](Use_stdis_sorted_and_is_sorted_until_for_pre-condition_checking.md)
- [Use std::is_sorted and std::is_sorted_until for invariant checking](Use_stdis_sorted_and_stdis_sorted_until_for_invariant_checking.md)
- [Use std::merge and std::inplace_merge for sorted sequence merging](Use_stdmerge_and_stdinplace_merge_for_sorted_sequence_merging.md)
- [Use std::minmax_element to find both extremes in a single pass](Use_stdminmax_element_to_find_both_extremes_in_a_single_pass.md)
- [Use std::next_permutation and prev_permutation for combinatorial enumeration](Use_stdnext_permutation_and_prev_permutation_for_combinatorial_enumeration.md)
- [Use std::next_permutation to iterate over all permutations](Use_stdnext_permutation_to_iterate_over_all_permutations.md)
- [Use std::partial_sum to compute prefix sums for range queries](Use_stdpartial_sum_to_compute_prefix_sums_for_range_queries.md)
- [Use std::partition and stable_partition for in-place filtering](Use_stdpartition_and_stable_partition_for_in-place_filtering.md)
- [Use std::partition and std::stable_partition for in-place filtering](Use_stdpartition_and_stdstable_partition_for_in-place_filtering.md)
- [Use std::ranges::chunk_by (C++23) to group consecutive elements by predicate](Use_stdrangeschunk_by_C++23_to_group_consecutive_elements_by_predicate.md)
- [Use std::ranges::contains (C++23) for readable element search](Use_stdrangescontains_C++23_for_readable_element_search.md)
- [Use std::ranges::find_last and find_last_if (C++23)](Use_stdrangesfind_last_and_find_last_if_C++23.md)
- [Use std::ranges::fold_left and fold_right (C++23)](Use_stdrangesfold_left_and_fold_right_C++23.md)
- [Use std::ranges::starts_with and ends_with (C++23) for range prefix/suffix checks](Use_stdrangesstarts_with_and_ends_with_C++23_for_range_prefixsuffix_checks.md)
- [Use std::ranges::to for materializing views into containers (C++23)](Use_stdrangesto_for_materializing_views_into_containers_C++23.md)
- [Use std::ranges::views::zip and views::enumerate (C++23)](Use_stdrangesviewszip_and_viewsenumerate_C++23.md)
- [Use std::ranges::zip_transform (C++23) for element-wise operations on multiple ranges](Use_stdrangeszip_transform_C++23_for_element-wise_operations_on_multiple_ranges.md)
- [Use std::ranges::zip_transform (C++23) to transform multiple ranges](Use_stdrangeszip_transform_C++23_to_transform_multiple_ranges.md)
- [Use std::rotate and std::rotate_copy for in-place cyclic shifting](Use_stdrotate_and_stdrotate_copy_for_in-place_cyclic_shifting.md)
- [Use std::rotate to implement circular buffer semantics](Use_stdrotate_to_implement_circular_buffer_semantics.md)
- [Use std::sample (C++17) for random reservoir sampling](Use_stdsample_C++17_for_random_reservoir_sampling.md)
- [Use std::search with Boyer-Moore searcher for fast string pattern matching](Use_stdsearch_with_Boyer-Moore_searcher_for_fast_string_pattern_matching.md)
- [Use std::shift_left and std::shift_right (C++20)](Use_stdshift_left_and_stdshift_right_C++20.md)
- [Use std::transform with output iterator adaptors](Use_stdtransform_with_output_iterator_adaptors.md)
- [Use std::transform, std::for_each, and std::generate correctly](Use_stdtransform_stdfor_each_and_stdgenerate_correctly.md)
- [Use std::transform_reduce for parallel map-reduce operations](Use_stdtransform_reduce_for_parallel_map-reduce_operations.md)

## Notes

- Prefer standard algorithms over raw loops - they express intent and are easier to parallelize.
- `std::ranges` (C++20) algorithms accept containers directly and support projections for cleaner comparators.
- Parallel execution policies (`std::execution::par`) can parallelize algorithms with one argument change.
- `std::sort` is not stable - use `std::stable_sort` if equal elements must maintain relative order.
- `std::transform` is the functional map - combine with output iterators for flexibility.
- `std::accumulate` uses a strict left fold; `std::reduce` allows parallel execution but requires an associative and commutative operation.
- `std::ranges::views` are lazy - they compose without allocating intermediate containers.
- Always check iterator validity - algorithms that logically remove elements (`std::remove`) need the erase-remove idiom to actually shrink the container.
- `std::find_if` with a lambda is the modern replacement for hand-written search loops.
- `std::nth_element` is O(n) average for partial ordering - much faster than a full sort when you only need the kth element.
