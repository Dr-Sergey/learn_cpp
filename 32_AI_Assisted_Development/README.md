# AI-Assisted C++ Development

Best practices for leveraging LLMs and AI assistants in C++ development: prompt engineering, code generation, review, refactoring, testing, and architecture design with AI tools.

**Topics:** 22

## Contents

- [Apply AI to optimize C++ code for performance - SIMD, cache, algorithms](Apply_AI_to_optimize_C++_code_for_performance_-_SIMD_cache_algorithms.md)
- [Apply AI tools to migrate C++ codebases between standards (C++14 to C++20)](Apply_AI_tools_to_migrate_C++_codebases_between_standards_C++14_to_C++20.md)
- [Apply AI-assisted code review to catch subtle C++ bugs](Apply_AI-assisted_code_review_to_catch_subtle_C++_bugs.md)
- [Apply AI-assisted debugging - explaining crash dumps and undefined behavior](Apply_AI-assisted_debugging_-_explaining_crash_dumps_and_undefined_behavior.md)
- [Best practices for prompt engineering when working with C++ and LLMs](Best_practices_for_prompt_engineering_when_working_with_C++_and_LLMs.md)
- [Design effective prompts for C++ code - context, constraints, and examples](Design_effective_prompts_for_C++_code_-_context_constraints_and_examples.md)
- [Generate API documentation and usage examples with AI tools](Generate_API_documentation_and_usage_examples_with_AI_tools.md)
- [Generate boilerplate code - serialization, operators, constructors with AI](Generate_boilerplate_code_-_serialization_operators_constructors_with_AI.md)
- [Leverage AI for embedded C++ - register maps, HAL code, and drivers](Leverage_AI_for_embedded_C++_-_register_maps_HAL_code_and_drivers.md)
- [Leverage AI for refactoring legacy C++ code to modern standards](Leverage_AI_for_refactoring_legacy_C++_code_to_modern_standards.md)
- [Leverage AI to generate property-based tests and fuzzing harnesses](Leverage_AI_to_generate_property-based_tests_and_fuzzing_harnesses.md)
- [Leverage LLMs for design pattern suggestions in C++ projects](Leverage_LLMs_for_design_pattern_suggestions_in_C++_projects.md)
- [Use AI assistants for CMake and build system configuration](Use_AI_assistants_for_CMake_and_build_system_configuration.md)
- [Use AI assistants for learning new C++ features with interactive examples](Use_AI_assistants_for_learning_new_C++_features_with_interactive_examples.md)
- [Use AI for architecture design discussions and trade-off analysis](Use_AI_for_architecture_design_discussions_and_trade-off_analysis.md)
- [Use AI to analyze and fix compiler errors and warnings](Use_AI_to_analyze_and_fix_compiler_errors_and_warnings.md)
- [Use AI to explain and document complex C++ template metaprogramming](Use_AI_to_explain_and_document_complex_C++_template_metaprogramming.md)
- [Use AI to generate cross-platform compatibility layers in C++](Use_AI_to_generate_cross-platform_compatibility_layers_in_C++.md)
- [Use AI to generate unit tests and test fixtures for C++ code](Use_AI_to_generate_unit_tests_and_test_fixtures_for_C++_code.md)
- [Use GitHub Copilot effectively for C++ development workflows](Use_GitHub_Copilot_effectively_for_C++_development_workflows.md)
- [Use LLM assistants effectively for C++ code generation and review](Use_LLM_assistants_effectively_for_C++_code_generation_and_review.md)
- [Validate AI-generated C++ code - common pitfalls and verification strategies](Validate_AI-generated_C++_code_-_common_pitfalls_and_verification_strategies.md)

## Notes

- AI code generation works best with clear context: C++ version, compiler, error strategy, and naming conventions
- Always review AI-generated C++ code for UB, lifetime issues, and missing error handling
- Few-shot examples from your own codebase teach AI your project's style better than descriptions
- Use AI to generate boilerplate (serialization, operators, constructors) — it excels at repetitive patterns
- AI-assisted code review catches subtle bugs — but don't skip human review entirely
- Iterative prompting (scaffold → implement → verify) beats single-shot generation for complex code
- AI is excellent at explaining complex template metaprogramming and compiler errors
- Test AI-generated code with sanitizers (ASan, UBSan) — AI often produces code with subtle UB
- Use AI for CMake configuration help — build systems are well-suited for AI assistance
- Prompt anti-requirements ("no Boost", "no exceptions") prevent unwanted dependencies in generated code
