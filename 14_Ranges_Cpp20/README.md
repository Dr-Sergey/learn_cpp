# Ranges (C++20)

Range-based views, adaptors, projections, and the ranges library for composable data processing.

**Topics:** 24

## Contents

- [Understand range algorithms' structured return types](Understand_range_algorithms_structured_return_types.md)
- [Understand sentinels and the split between begin/end types in ranges](Understand_sentinels_and_the_split_between_beginend_types_in_ranges.md)
- [Understand the difference between range categories: contiguous, random-access, bidirectional, forward, input](Understand_the_difference_between_range_categories_contiguous_random-access_bidi.md)
- [Understand the ranges concept hierarchy: range, view, viewable_range](Understand_the_ranges_concept_hierarchy_range_view_viewable_range.md)
- [Use range adaptors: filter, transform, take, drop, and their composition](Use_range_adaptors_filter_transform_take_drop_and_their_composition.md)
- [Use range algorithms instead of iterator-pair algorithms](Use_range_algorithms_instead_of_iterator-pair_algorithms.md)
- [Use std::ranges::subrange for explicit iterator-sentinel pairs](Use_stdrangessubrange_for_explicit_iterator-sentinel_pairs.md)
- [Use views::as_rvalue (C++23) to move elements from a range](Use_viewsas_rvalue_C++23_to_move_elements_from_a_range.md)
- [Use views::cartesian_product (C++23) for Cartesian product of ranges](Use_viewscartesian_product_C++23_for_Cartesian_product_of_ranges.md)
- [Use views::chunk and views::slide for windowed processing (C++23)](Use_viewschunk_and_viewsslide_for_windowed_processing_C++23.md)
- [Use views::concat (C++26) to lazily concatenate multiple ranges](Use_viewsconcat_C++26_to_lazily_concatenate_multiple_ranges.md)
- [Use views::elements to project a range of tuples to a specific index](Use_viewselements_to_project_a_range_of_tuples_to_a_specific_index.md)
- [Use views::flat_map (transform + join) to flatmap ranges without intermediates](Use_viewsflat_map_transform_+_join_to_flatmap_ranges_without_intermediates.md)
- [Use views::iota, views::repeat, and other generating views](Use_viewsiota_viewsrepeat_and_other_generating_views.md)
- [Use views::join and views::join_with for flattening nested ranges](Use_viewsjoin_and_viewsjoin_with_for_flattening_nested_ranges.md)
- [Use views::repeat and views::cycle (C++26) for infinite repeating ranges](Use_viewsrepeat_and_viewscycle_C++26_for_infinite_repeating_ranges.md)
- [Use views::reverse for lazy bidirectional reversal](Use_viewsreverse_for_lazy_bidirectional_reversal.md)
- [Use views::split for tokenizing ranges lazily](Use_viewssplit_for_tokenizing_ranges_lazily.md)
- [Use views::split to tokenize a range by a delimiter without allocation](Use_viewssplit_to_tokenize_a_range_by_a_delimiter_without_allocation.md)
- [Use views::stride (C++23) to sample every Nth element of a range](Use_viewsstride_C++23_to_sample_every_Nth_element_of_a_range.md)
- [Use views::take_while and drop_while for predicate-based range truncation](Use_viewstake_while_and_drop_while_for_predicate-based_range_truncation.md)
- [Use views::take_while and views::drop_while for predicate-based trimming](Use_viewstake_while_and_viewsdrop_while_for_predicate-based_trimming.md)
- [Use views::transform with projection to avoid boilerplate lambdas](Use_viewstransform_with_projection_to_avoid_boilerplate_lambdas.md)
- [Write custom views using view_interface and range adaptors](Write_custom_views_using_view_interface_and_range_adaptors.md)

## Notes

- Ranges replace iterator pairs with single range objects - cleaner and less error-prone.
- Views are lazy - `std::views::filter | std::views::transform` composes without allocation.
- `std::ranges::to<Container>()` (C++23) materializes a lazy view into a container.
- Range adaptors use the pipe (`|`) syntax for readable left-to-right composition.
- Sentinel-based ranges allow different types for begin and end - enables null-terminated strings.
- `std::views::zip` (C++23) combines multiple ranges element-wise into tuples.
- Borrowed ranges ensure iterators remain valid even after the range is destroyed.
- Projections in range algorithms eliminate the need for custom comparators in most cases.
- `std::views::chunk` and `std::views::slide` (C++23) enable windowed processing patterns.
- Custom views require implementing `begin()`, `end()`, and satisfying range concepts.
