# Lambda & Functional

Lambda expressions, captures, generic lambdas, std::function, and functional programming patterns in C++.

**Topics:** 21

## Contents

- [Know static lambdas (C++23) and when they enable optimizations](Know_static_lambdas_Cpp23_and_when_they_enable_optimizations.md)
- [Understand lambda capture modes: by value, by reference, and init-captures](Understand_lambda_capture_modes_by_value_by_reference_and_init-captures.md)
- [Understand lambda captures in coroutines and lifetime pitfalls](Understand_lambda_captures_in_coroutines_and_lifetime_pitfalls.md)
- [Understand template lambdas (C++20) with explicit template parameter lists](Understand_template_lambdas_C++20_with_explicit_template_parameter_lists.md)
- [Understand the closure type's special member functions](Understand_the_closure_types_special_member_functions.md)
- [Use immediately invoked lambdas for complex variable initialization](Use_immediately_invoked_lambdas_for_complex_variable_initialization.md)
- [Use lambda-based overload sets (the overload pattern)](Use_lambda-based_overload_sets_the_overload_pattern.md)
- [Use lambdas as custom deleters for smart pointers](Use_lambdas_as_custom_deleters_for_smart_pointers.md)
- [Use pack expansion in lambda init-captures (C++20)](Use_pack_expansion_in_lambda_init-captures_Cpp20.md)
- [Use std::bind_front (C++20) for partial application](Use_stdbind_front_C++20_for_partial_application.md)
- [Use std::invoke for generic callable invocation](Use_stdinvoke_for_generic_callable_invocation.md)
- [Use std::not_fn (C++17) to negate predicates](Use_stdnot_fn_C++17_to_negate_predicates.md)
- [Use std::ranges::views::adjacent and adjacent_transform (C++23)](Use_stdrangesviewsadjacent_and_adjacent_transform_C++23.md)
- [Use std::ranges::views::transform with a projection to avoid intermediate lambdas](Use_stdrangesviewstransform_with_a_projection_to_avoid_intermediate_lambdas.md)
- [Write a memoization wrapper using templates and std::unordered_map](Write_a_memoization_wrapper_using_templates_and_stdunordered_map.md)
- [Write a trampoline for mutual recursion between lambdas without std::function](Write_a_trampoline_for_mutual_recursion_between_lambdas_without_stdfunction.md)
- [Write generic lambdas and understand their template mechanics (C++14)](Write_generic_lambdas_and_understand_their_template_mechanics_C++14.md)
- [Write higher-order functions that compose predicates](Write_higher-order_functions_that_compose_predicates.md)
- [Write point-free style function composition using higher-order functions](Write_point-free_style_function_composition_using_higher-order_functions.md)
- [Write recursive lambdas using deducing-this (C++23)](Write_recursive_lambdas_using_deducing-this_C++23.md)
- [Write stateful lambdas and understand their mutable semantics](Write_stateful_lambdas_and_understand_their_mutable_semantics.md)

## Notes

- Lambdas are syntactic sugar for unnamed function objects — the compiler generates a closure class
- Capture by value [=] copies; capture by reference [&] aliases — beware dangling references
- Generic lambdas (uto parameters) are template function objects since C++14
- mutable lambdas can modify by-value captures — without it, operator() is const
- std::function type-erases callables but has overhead — prefer templates or uto when possible
- Immediately-invoked lambda expressions (IILE) are useful for complex initialization of const variables
- C++20 lambdas can be used in unevaluated contexts and have template parameter lists
- std::invoke calls any callable uniformly — works with functions, lambdas, and member pointers
- Recursive lambdas need std::function or a Y-combinator pattern — they cannot capture themselves
- Capture [*this] (C++17) copies the whole object — needed when the lambda outlives 	his
"@

"14_Ranges_Cpp20" = @"

## Notes

- Ranges replace iterator pairs with single range objects — cleaner and less error-prone
- Views are lazy — std::views::filter | std::views::transform composes without allocation
- std::ranges::to<Container>() (C++23) materializes a lazy view into a container
- Range adaptors use the pipe (|) syntax for readable left-to-right composition
- Sentinel-based ranges allow different types for begin and end — enables null-terminated strings
- std::views::zip (C++23) combines multiple ranges element-wise into tuples
- Borrowed ranges ensure iterators remain valid even after the range is destroyed
- Projections in range algorithms eliminate the need for custom comparators in most cases
- std::views::chunk and std::views::slide (C++23) enable windowed processing patterns
- Custom views require implementing egin(), end(), and satisfying range concepts
