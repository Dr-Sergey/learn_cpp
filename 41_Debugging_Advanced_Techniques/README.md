# Advanced Debugging Techniques

Systematic debugging methodology for C++ systems: debugging optimized code, time-travel debugging, custom visualizers, coroutine debugging, post-mortem analysis, and remote debugging.

**Topics:** 12

## Contents

- [Analyze core dumps and minidumps post-mortem](Analyze_core_dumps_and_minidumps_post-mortem.md)
- [Debug across library boundaries with mixed debug/release and symbol servers](Debug_across_library_boundaries_with_mixed_debug-release_and_symbol_servers.md)
- [Debug coroutines and inspect suspended coroutine frames](Debug_coroutines_and_inspect_suspended_coroutine_frames.md)
- [Debug memory corruption with patterns, guard pages, and red zones](Debug_memory_corruption_with_patterns_guard_pages_and_red_zones.md)
- [Debug multi-threaded deadlocks with lock-order graphs and Helgrind](Debug_multi-threaded_deadlocks_with_lock-order_graphs_and_Helgrind.md)
- [Debug optimized code and understand inlining, tail calls, and missing frames](Debug_optimized_code_and_understand_inlining_tail_calls_and_missing_frames.md)
- [Debug template instantiation errors systematically with concept diagnostics](Debug_template_instantiation_errors_systematically_with_concept_diagnostics.md)
- [Profile and debug build times with -ftime-trace and ClangBuildAnalyzer](Profile_and_debug_build_times_with_ftime-trace_and_ClangBuildAnalyzer.md)
- [Remote debug embedded targets with GDB server](Remote_debug_embedded_targets_with_GDB_server.md)
- [Use std::stacktrace (C++23) for programmatic diagnostics](Use_stdstdstacktrace_Cpp23_for_programmatic_diagnostics.md)
- [Use time-travel debugging with rr record and replay](Use_time-travel_debugging_with_rr_record_and_replay.md)
- [Write custom GDB and LLDB pretty-printers for project-specific types](Write_custom_GDB_and_LLDB_pretty-printers_for_project-specific_types.md)

## Notes

- GDB/LLDB pretty-printers format std::vector, std::map, etc. — install them for readable debugging
- Conditional breakpoints (reak if x > 100) reduce noise when debugging loops
- Watchpoints halt execution when a memory address changes — essential for tracking mysterious mutations
- 
r (record and replay) allows reverse debugging — step backwards through execution
- ASan + UBSan in debug builds catch issues at the exact point of occurrence
- Core dumps + symbols enable post-mortem debugging — configure ulimit -c unlimited on Linux
- Distributed tracing (OpenTelemetry) helps debug issues across microservice boundaries
- std::stacktrace (C++23) provides programmatic stack traces without platform-specific code
- Debugging optimized code requires understanding that variables may be optimized out — use olatile hints
- Memory debugging tools (Valgrind, Dr. Memory) find leaks and corruption the sanitizers may miss
