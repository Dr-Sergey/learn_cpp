# Move Semantics & Value Categories

This folder covers one of the most practically impactful corners of modern C++: how objects move instead of copy, what value categories actually mean, and how the language uses all of this to squeeze out performance. If you've ever wondered why `std::move` doesn't actually move anything, or why forgetting `noexcept` on a move constructor silently slows down your `std::vector`, this is the folder for you.

**Topics:** 22

## Contents

- [Know Ref-Qualified Member Functions and Overloading on `*this` Value Category](Know_ref-qualified_member_functions_and_overloading_on_this_value_category.md)
- [Know That `std::initializer_list` Always Copies and Never Moves Elements](Know_that_stdinitializer_list_always_copies_and_never_moves_elements.md)
- [Know the std::relocate proposals and trivial relocatability (C++26 direction)](Know_the_stdstdrelocate_proposals_and_trivial_relocatability_Cpp26_direction.md)
- [Know When `return std::move(x)` Is Harmful and Prevents NRVO](Know_when_return_stdmovex_is_harmful_and_prevents_NRVO.md)
- [Know When to Mark Functions `noexcept` and Its Performance Implications](Know_when_to_mark_functions_noexcept_and_understand_its_performance_implications.md)
- [Master Perfect Forwarding with std::forward and Universal References](Master_perfect_forwarding_with_stdforward_and_universal_references.md)
- [Understand Conditional `noexcept` and Its Impact on `std::vector` Reallocation](Understand_conditional_noexcept_and_its_impact_on_stdvector_reallocation.md)
- [Understand deducing this interaction with move semantics (C++23)](Understand_deducing-this_interaction_with_move_semantics_Cpp23.md)
- [Understand Forwarding References vs Rvalue References Syntactically](Understand_forwarding_references_vs_rvalue_references_syntactically.md)
- [Understand Guaranteed Copy Elision (prvalue materialization, C++17)](Understand_guaranteed_copy_elision_prvalue_materialization_C++17.md)
- [Understand How Move Semantics Interact with Virtual Functions](Understand_how_move_semantics_interact_with_virtual_functions.md)
- [Understand Implicit Move from Return Statements (C++23 Clarification)](Understand_implicit_move_from_return_statements_C++23_clarification.md)
- [Understand Moved-From State Guarantees in the Standard Library](Understand_moved-from_state_guarantees_in_the_standard_library.md)
- [Understand Sink Parameters and By-Value vs Const-Reference Trade-offs](Understand_sink_parameters_and_by-value_vs_const-reference_trade-offs.md)
- [Understand Sink Parameters: When to Take by Value and When by Rvalue Reference](Understand_sink_parameters_when_to_take_by_value_and_when_by_rvalue_reference.md)
- [Understand std::move and Why It Does Not Actually Move Anything](Understand_stdmove_and_why_it_does_not_actually_move_anything.md)
- [Understand std::swap vs memberwise swap performance trade-offs](Understand_stdstdswap_vs_memberwise_swap_performance_trade-offs.md)
- [Understand the Five Value Categories: lvalue, rvalue, xvalue, prvalue, glvalue](Understand_the_five_value_categories_lvalue_rvalue_xvalue_prvalue_glvalue.md)
- [Understand Trivially Relocatable Types and memcpy-Based Relocation](Understand_trivially_relocatable_types_and_memcpy-based_relocation.md)
- [Use `std::forward_like` (C++23) for Forwarding with Ownership Propagation](Use_stdforward_like_C++23_for_forwarding_with_ownership_propagation.md)
- [Use std::move_only_function (C++23) for Non-Copyable Callable Wrappers](Use_stdmove_only_function_C++23_for_non-copyable_callable_wrappers.md)
- [Write Correct Move Constructors and Move Assignment Operators](Write_correct_move_constructors_and_move_assignment_operators.md)

## Notes

- `std::move` does not move - it casts to an rvalue reference, enabling move constructors.
- Moved-from objects must be in a valid but unspecified state - never assume their value.
- `std::forward` preserves value category in perfect forwarding - use only with forwarding references.
- Value categories (lvalue, xvalue, prvalue) determine which overload gets called.
- The copy-and-swap idiom provides a strong exception guarantee for assignment operators.
- Guaranteed copy elision (C++17) means prvalues are never materialized unless needed.
- Return by value - compilers apply NRVO/RVO, and move semantics handles the rest.
- `noexcept` on move constructors is critical - containers like `std::vector` require it for strong exception safety.
- Don't `std::move` in return statements with local variables - it prevents NRVO.
- Rvalue reference members in classes are almost always a design mistake.
