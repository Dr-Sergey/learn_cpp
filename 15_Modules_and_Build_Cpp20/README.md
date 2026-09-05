# Modules & Build (C++20)

C++20 modules, module partitions, and their interaction with build systems.

**Topics:** 14

## Contents

- [Handle Modules and Traditional Headers in Mixed Codebases](Handle_modules_and_traditional_headers_in_mixed_codebases.md)
- [Know Build System Challenges with Module Dependency Scanning](Know_build_system_challenges_with_module_dependency_scanning.md)
- [Know how CMake 3.28+ supports C++20 modules](Know_how_CMake_328+_supports_C++20_modules.md)
- [Know Named Modules vs Header Units and When to Use Each](Know_named_modules_vs_header_units_and_when_to_use_each.md)
- [Plan Incremental Migration from Headers to Modules](Plan_incremental_migration_from_headers_to_modules.md)
- [Understand C++20 modules and how they replace header files](Understand_C++20_modules_and_how_they_replace_header_files.md)
- [Understand global module fragment for macros in module translation units](Understand_global_module_fragment_for_macros_in_module_translation_units.md)
- [Understand how global module fragments work for macro dependencies](Understand_how_global_module_fragments_work_for_macro_dependencies.md)
- [Understand how to mix modules with header-based code using header units](Understand_how_to_mix_modules_with_header-based_code_using_header_units.md)
- [Understand Module Linkage vs External Linkage in Module Context](Understand_module_linkage_vs_external_linkage_in_module_context.md)
- [Understand Module Ownership: Interface and Implementation Units](Understand_module_ownership_interface_and_implementation_units.md)
- [Understand module partitions and private module fragments](Understand_module_partitions_and_private_module_fragments.md)
- [Understand Module Reachability and Visibility Rules](Understand_module_reachability_and_visibility_rules.md)
- [Use precompiled headers (PCH) strategically before migrating to modules](Use_precompiled_headers_PCH_strategically_before_migrating_to_modules.md)

## Notes

- Modules (C++20) replace headers - they provide better encapsulation and faster compilation.
- `export module` declares a module interface; `module` (without `export`) declares an implementation unit.
- Module partitions split large modules into manageable parts with internal linkage control.
- Unlike headers, modules do not leak macros or pollute the global namespace.
- Build system support for modules is still evolving - CMake 3.28+ has stable support via `FILE_SET CXX_MODULES`.
- `import std;` (C++23) imports the entire standard library as a single module.
- Header units (`import <header>;`) bridge legacy headers into the module world without requiring source changes.
- Module initialization order is well-defined - unlike the static initialization order fiasco with headers.
- Private module fragments hide implementation details within the interface file itself.
- Modules require a DAG (directed acyclic graph) build order - circular dependencies are forbidden by design.
