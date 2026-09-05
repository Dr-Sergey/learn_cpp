# Build Systems & CI

CMake, package management, CI/CD pipelines, and build configuration best practices.

**Topics:** 24

## Contents

- [Debug build non-determinism to achieve reproducible C++ builds](Understand_reproducible_builds_and_how_to_achieve_them_in_C++_2.md)
- [Design platform-specific CMakePresets.json configurations](Use_CMakePresetsjson_for_reproducible_shareable_build_configurations_3.md)
- [Drive day-to-day builds with CMakePresets.json workflow presets](Use_CMakePresetsjson_for_reproducible_shareable_build_configurations_2.md)
- [Invoke a CMake toolchain file and control find_package in cross builds](Set_up_cross-compilation_with_CMake_toolchain_files_2.md)
- [Model Conan 2.0 dependency types: requires, tool_requires, and test_requires](Use_Conan_20_for_C++_package_management_2.md)
- [Set up a complete GitHub Actions CI pipeline for a C++ CMake project](Set_up_a_complete_GitHub_Actions_CI_pipeline_for_a_C++_CMake_project.md)
- [Set up a cross-compilation workflow with CMAKE_FIND_ROOT_PATH modes](Set_up_cross-compilation_with_CMake_toolchain_files_3.md)
- [Set up a multi-stage Docker build for hermetic C++ CI](Set_up_a_multi-stage_Docker_build_for_hermetic_C++_CI.md)
- [Set up cross-compilation with CMake toolchain files](Set_up_cross-compilation_with_CMake_toolchain_files.md)
- [Set up remote build caching with CMake and Bazel](Set_up_remote_build_caching_with_CMake_and_Bazel.md)
- [Set up static analysis in CI with clang-tidy and cppcheck](Set_up_static_analysis_in_CI_with_clang-tidy_and_cppcheck.md)
- [Understand and use CMake's install() and export() for library distribution](Understand_and_use_CMakes_install_and_export_for_library_distribution.md)
- [Understand reproducible builds and eliminate non-determinism](Understand_reproducible_builds_and_eliminate_non-determinism.md)
- [Understand reproducible builds and how to achieve them in C++](Understand_reproducible_builds_and_how_to_achieve_them_in_C++.md)
- [Use Bazel or Buck2 as an alternative to CMake for large-scale builds](Use_Bazel_or_Buck2_as_an_alternative_to_CMake_for_large-scale_builds.md)
- [Use Bazel or Buck2 as CMake alternatives for large monorepos](Use_Bazel_or_Buck2_as_CMake_alternatives_for_large_monorepos.md)
- [Use CMake module dependency scanning for C++20 modules](Use_CMake_module_dependency_scanning_for_C++20_modules.md)
- [Use CMake's FetchContent for dependency management without a package manager](Use_CMakes_FetchContent_for_dependency_management_without_a_package_manager.md)
- [Use CMakePresets.json for reproducible, shareable build configurations](Use_CMakePresetsjson_for_reproducible_shareable_build_configurations.md)
- [Use compile_commands.json and Bear for IDE integration](Use_compile_commandsjson_and_Bear_for_IDE_integration.md)
- [Use Conan 2.0 for C++ package management](Use_Conan_20_for_C++_package_management.md)
- [Use Conan 2.0 for dependency management with CMake integration](Use_Conan_20_for_dependency_management_with_CMake_integration.md)
- [Use dependency scanning for C++20 modules in CMake 3.28+](Use_dependency_scanning_for_C++20_modules_in_CMake_328+.md)
- [Use Ninja Multi-Config for single-configure multi-config builds](Use_Ninja_Multi-Config_for_single-configure_multi-config_builds.md)

## Notes

- CMake is the dominant C++ build system - learn `target_link_libraries`, FetchContent, and presets.
- CMake presets (`CMakePresets.json`) standardize build configurations across developers so everyone uses the same flags without having to remember them.
- Vcpkg and Conan are the two main C++ package managers - vcpkg integrates especially tightly with CMake via its toolchain file.
- CI pipelines should build with multiple compilers (GCC, Clang, MSVC) to catch portability issues early, before they become surprises.
- Enable sanitizers (ASan, UBSan, TSan) in CI - they catch bugs at runtime that unit tests alone will never see.
- Unity builds (jumbo builds) combine translation units to reduce build times by cutting down on repeated header parsing.
- ccache caches compilation results so unchanged files aren't recompiled - essential for fast iterative development on large projects.
- Use `EXPORT` and `install()` in CMake to create reusable packages that downstream consumers can pick up with a simple `find_package`.
- Reproducible builds ensure binary output is identical given the same source - this aids security auditing and debugging.
- Keep build types separate: Debug (assertions + symbols), Release (optimizations), RelWithDebInfo (both) - mixing them up leads to confusing behavior.
