# API & Library Design

Principles and patterns for designing reusable, maintainable C++ libraries and APIs: minimal interfaces, customization points, ABI stability, documentation, and distribution strategies.

**Topics:** 15

## Contents

- [Apply Hyrum's Law and understand implicit API contracts](Apply_Hyrums_Law_and_understand_implicit_API_contracts.md)
- [Choose between header-only, compiled, and module-based library distribution](Choose_between_header-only_compiled_and_module-based_library_distribution.md)
- [Control symbol visibility with attributes and declspec for shared libraries](Control_symbol_visibility_with_attributes_and_declspec_for_shared_libraries.md)
- [Design allocator-aware containers and types](Design_allocator-aware_containers_and_types.md)
- [Design C-compatible wrapper APIs for FFI consumers](Design_C-compatible_wrapper_APIs_for_FFI_consumers.md)
- [Design callback and extension interfaces](Design_callback_and_extension_interfaces_with_stdstdfunction_templates_and_type-.md)
- [Design exception-safe vs noexcept library boundaries](Design_exception-safe_vs_noexcept_library_boundaries.md)
- [Design minimal, hard-to-misuse C++ APIs following Scott Meyers' principles](Design_minimal_hard-to-misuse_Cpp_APIs_following_Scott_Meyers_principles.md)
- [Generate documentation with Doxygen, Standardese, or hdoc](Generate_documentation_with_Doxygen_Standardese_or_hdoc.md)
- [Provide customization points via ADL, tag_invoke, or CPO patterns](Provide_customization_points_via_ADL_tag_invoke_or_CPO_patterns.md)
- [Provide forward declaration headers to reduce compile times](Provide_forward_declaration_headers_to_reduce_compile_times.md)
- [Provide noexcept guarantees on move operations for container compatibility](Provide_noexcept_guarantees_on_move_operations_for_container_compatibility.md)
- [Test API usability with compile-fail tests using static_assert and SFINAE](Test_API_usability_with_compile-fail_tests_using_static_assert_and_SFINAE.md)
- [Use inline namespaces for ABI versioning without breaking client code](Use_inline_namespaces_for_ABI_versioning_without_breaking_client_code.md)
- [Write Concepts-constrained library interfaces with good diagnostics](Write_Concepts-constrained_library_interfaces_with_good_diagnostics.md)

## Notes

- Design APIs for the caller - minimize boilerplate, prevent misuse, and make correct usage obvious.
- Use strong types (wrapper classes) instead of `bool` or `int` parameters to avoid argument confusion.
- `[[nodiscard]]` on return values forces callers to handle results - especially important for error returns.
- Prefer value semantics in APIs - accept by value, return by value, and let move semantics optimize.
- Hide implementation details (Pimpl, modules, opaque types) to allow internal changes without breaking users.
- Header-only libraries maximize ease of use; compiled libraries maximize build speed.
- Provide both throwing and non-throwing overloads where appropriate (`at()` vs `operator[]`).
- Use concepts to constrain template parameters - they provide clear error messages and serve as documentation.
- Namespace everything - avoid polluting the global namespace with library symbols.
- Semantic versioning (semver) communicates API/ABI stability expectations to users.
