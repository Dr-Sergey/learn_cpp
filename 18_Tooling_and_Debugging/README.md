# Tooling & Debugging

Compilers, sanitizers, profilers, debuggers, static analysis, and development tools.

**Topics:** 48

## Contents

- [Fuzz C++ code with structured inputs and multiple sanitizers](Use_sanitizer_coverage_and_libFuzzer_for_fuzz_testing_C++_code_2.md)
- [Set up continuous integration with GitHub Actions for CMake C++ projects](Set_up_continuous_integration_with_GitHub_Actions_for_CMake_C++_projects.md)
- [Set up sanitizer-aware CI to catch regressions early](Set_up_sanitizer-aware_CI_to_catch_regressions_early.md)
- [Understand and use address-space sanitizer for security auditing](Understand_and_use_address-space_sanitizer_for_security_auditing.md)
- [Use -fsanitize=memory (MemorySanitizer) for uninitialized read detection](Use_-fsanitizememory_MemorySanitizer_for_uninitialized_read_detection.md)
- [Use -fsanitize=thread (ThreadSanitizer) to detect data races systematically](Use_-fsanitizethread_ThreadSanitizer_to_detect_data_races_systematically.md)
- [Use -ftime-trace (Clang) to identify compilation bottlenecks](Use_-ftime-trace_Clang_to_identify_compilation_bottlenecks.md)
- [Use -O2 vs -O3 vs -Oz: understand what each optimization level enables](Use_-O2_vs_-O3_vs_-Oz_understand_what_each_optimization_level_enables.md)
- [Use -Wall, -Wextra, -Wpedantic and treat warnings as errors in CI](Use_-Wall_-Wextra_-Wpedantic_and_treat_warnings_as_errors_in_CI.md)
- [Use AddressSanitizer, MemorySanitizer, and ThreadSanitizer effectively](Use_AddressSanitizer_MemorySanitizer_and_ThreadSanitizer_effectively.md)
- [Use Bloaty McBloatface to analyze and reduce binary size](Use_Bloaty_McBloatface_to_analyze_and_reduce_binary_size.md)
- [Use Bloaty McBloatface to analyze binary size contributors](Use_Bloaty_McBloatface_to_analyze_binary_size_contributors.md)
- [Use C++ Insights (cppinsights.io) to understand compiler transformations](Use_C++_Insights_cppinsightsio_to_understand_compiler_transformations.md)
- [Use Catch2, Google Test, or doctest for unit testing C++ code](Use_Catch2_Google_Test_or_doctest_for_unit_testing_C++_code.md)
- [Use ccache to speed up repeated compilation](Use_ccache_to_speed_up_repeated_compilation.md)
- [Use ccache to speed up repeated compilation cycles](Use_ccache_to_speed_up_repeated_compilation_cycles.md)
- [Use clang-format for consistent automatic code formatting](Use_clang-format_for_consistent_automatic_code_formatting.md)
- [Use clang-format to enforce consistent code style in a team](Use_clang-format_to_enforce_consistent_code_style_in_a_team.md)
- [Use clangd and compile_commands.json for IDE code intelligence](Use_clangd_and_compile_commandsjson_for_IDE_code_intelligence.md)
- [Use CMake modern targets and avoid directory-level commands](Use_CMake_modern_targets_and_avoid_directory-level_commands.md)
- [Use compile-time benchmarking to track build performance regressions](Use_compile-time_benchmarking_to_track_build_performance_regressions.md)
- [Use compiler diagnostics: -ftemplate-backtrace-limit and concept diagnostics](Use_compiler_diagnostics_-ftemplate-backtrace-limit_and_concept_diagnostics.md)
- [Use compiler explorer (Godbolt) to inspect generated assembly](Use_compiler_explorer_Godbolt_to_inspect_generated_assembly.md)
- [Use Compiler Explorer with multiple compilers to check portability](Use_Compiler_Explorer_with_multiple_compilers_to_check_portability.md)
- [Use ctest with labels and parallel execution for large test suites](Use_ctest_with_labels_and_parallel_execution_for_large_test_suites.md)
- [Use debuggers: breakpoints, watchpoints, and post-mortem core dumps](Use_debuggers_breakpoints_watchpoints_and_post-mortem_core_dumps.md)
- [Use Dr. Memory or WinDbg for Windows-specific memory debugging](Use_Dr_Memory_or_WinDbg_for_Windows-specific_memory_debugging.md)
- [Use gdb pretty-printers for STL types](Use_gdb_pretty-printers_for_STL_types.md)
- [Use GDB Python scripts for visualizing complex data structures](Use_GDB_Python_scripts_for_visualizing_complex_data_structures.md)
- [Use hardware performance counters with perf stat for cache and branch analysis](Use_hardware_performance_counters_with_perf_stat_for_cache_and_branch_analysis.md)
- [Use include-what-you-use (IWYU) to enforce minimal includes](Use_include-what-you-use_IWYU_to_enforce_minimal_includes.md)
- [Use link-time optimization (LTO) to enable cross-TU inlining and devirtualization](Use_link-time_optimization_LTO_to_enable_cross-TU_inlining_and_devirtualization.md)
- [Use LTO (Link-Time Optimization) to enable cross-TU optimization](Use_LTO_Link-Time_Optimization_to_enable_cross-TU_optimization.md)
- [Use Ninja as a fast build backend for CMake](Use_Ninja_as_a_fast_build_backend_for_CMake.md)
- [Use objdump or llvm-objdump to inspect binary layout and symbol sizes](Use_objdump_or_llvm-objdump_to_inspect_binary_layout_and_symbol_sizes.md)
- [Use objdump or llvm-objdump to inspect binary structure and symbol sizes](Use_objdump_or_llvm-objdump_to_inspect_binary_structure_and_symbol_sizes.md)
- [Use perf flamegraphs for visual CPU profiling](Use_perf_flamegraphs_for_visual_CPU_profiling.md)
- [Use profile-guided optimization (PGO) for data-driven compiler optimization](Use_profile-guided_optimization_PGO_for_data-driven_compiler_optimization.md)
- [Use Quick Bench or Google Benchmark for micro-benchmarking](Use_Quick_Bench_or_Google_Benchmark_for_micro-benchmarking.md)
- [Use Sanitizer coverage and libFuzzer for fuzz testing C++ code](Use_Sanitizer_coverage_and_libFuzzer_for_fuzz_testing_C++_code.md)
- [Use source-based code coverage (LLVM) to find untested code paths](Use_source-based_code_coverage_LLVM_to_find_untested_code_paths.md)
- [Use static analysis with clang-tidy for modernization and readability checks](Use_static_analysis_with_clang-tidy_for_modernization_and_readability_checks.md)
- [Use symbol visibility control to reduce shared library symbol table size](Use_symbol_visibility_control_to_reduce_shared_library_symbol_table_size.md)
- [Use the Clang AST dump to understand how the compiler sees your code](Use_the_Clang_AST_dump_to_understand_how_the_compiler_sees_your_code.md)
- [Use Tracy, Orbit, or Superluminal for real-time profiling](Use_Tracy_Orbit_or_Superluminal_for_real-time_profiling.md)
- [Use Valgrind/Helgrind for memory and thread error detection](Use_ValgrindHelgrind_for_memory_and_thread_error_detection.md)
- [Use vcpkg or Conan for C++ dependency management](Use_vcpkg_or_Conan_for_C++_dependency_management.md)
- [Use vcpkg or Conan for C++ dependency management in CMake projects](Use_vcpkg_or_Conan_for_C++_dependency_management_in_CMake_projects.md)

## Notes

- AddressSanitizer (ASan) catches buffer overflows, use-after-free, and leaks at runtime.
- UndefinedBehaviorSanitizer (UBSan) detects signed overflow, null dereference, and misaligned access.
- ThreadSanitizer (TSan) finds data races - essential for concurrent code development.
- Clang-Tidy applies hundreds of static analysis checks and auto-fixes - integrate it in CI.
- clang-format enforces consistent code style - define a .clang-format file per project.
- Compile with -Wall -Wextra -Wpedantic -Werror (GCC/Clang) to catch issues early.
- Use Compiler Explorer (godbolt.org) to inspect assembly output and verify optimizations.
- Valgrind/Memcheck detects memory errors but runs 10-50x slower than sanitizers.
- Core dumps + GDB/LLDB are essential for post-mortem debugging - learn bt, frame, print.
- ccache and sccache dramatically reduce rebuild times by caching compilation results.
