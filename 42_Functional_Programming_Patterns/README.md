# Functional Programming Patterns in C++

Applying functional programming concepts in modern C++: monadic error chaining, algebraic data types, persistent data structures, expression templates, continuation-passing style, and pattern matching.

**Topics:** 9

## Contents

- [Apply the expression template pattern for lazy evaluation](Apply_the_expression_template_pattern_for_lazy_evaluation.md)
- [Design pure functions and referential transparency for testability](Design_pure_functions_and_referential_transparency_for_testability.md)
- [Implement monadic error chaining with std::expected and std::optional](Implement_monadic_error_chaining_with_stdstdexpected_and_stdstdoptional.md)
- [Implement pattern matching manually with std::visit + overload sets](Implement_pattern_matching_manually_with_stdstdvisit_overload_sets.md)
- [Implement persistent and immutable data structures in C++](Implement_persistent_and_immutable_data_structures_in_Cpp.md)
- [Implement railway-oriented programming for error handling](Implement_railway-oriented_programming_for_error_handling.md)
- [Use algebraic data types with std::variant as sum type and std::tuple as product type](Use_algebraic_data_types_with_stdstdvariant_as_sum_type_and_stdstdtuple_as_produ.md)
- [Use continuation-passing style (CPS) with coroutines and senders](Use_continuation-passing_style_CPS_with_coroutines_and_senders.md)
- [Use std::views pipeline as a functional transformation chain](Use_stdstdviews_pipeline_as_a_functional_transformation_chain.md)

## Notes

- Monadic chaining with `std::optional::and_then` / `std::expected::and_then` (C++23) enables composable error handling without manual error-check boilerplate.
- Pure functions (no side effects, same input -> same output) are easier to test and parallelize - C++ doesn't enforce purity but you can follow the discipline.
- `std::variant` + `std::visit` implements algebraic data types (sum types) in C++, with exhaustive compile-time case checking.
- Expression templates defer computation to a single pass - used in linear algebra libraries like Eigen and Blaze to eliminate temporary objects.
- Persistent (immutable) data structures share unchanged parts between versions, making them efficient for undo/version history and safe for concurrent access.
- Railway-oriented programming chains operations that may fail, short-circuiting on the first error and carrying it to the end of the pipeline.
- `std::views` pipelines (C++20) provide lazy functional transformations over ranges with no intermediate allocations.
- Continuation-passing style (CPS) maps naturally to coroutines and the sender/receiver pattern - coroutines are essentially syntactic sugar for CPS.
- Pattern matching proposals (C++26) aim to add `inspect` expressions for functional-style dispatch, replacing the `overloaded` helper trick.
- Avoid mutable shared state - this is the core principle of functional programming and applies directly to concurrent C++ to avoid data races.
