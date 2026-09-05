# Safety & Security

Memory safety, input validation, secure coding practices, and vulnerability prevention.

**Topics:** 26

## Contents

- [Avoid use of rand() and use cryptographically secure randomness](Avoid_use_of_rand_and_use_cryptographically_secure_randomness.md)
- [Avoid using rand() and seed cryptographic-quality randomness correctly](Avoid_using_rand_and_seed_cryptographic-quality_randomness_correctly.md)
- [Detect integer overflow at compile time and runtime safely](Detect_integer_overflow_at_compile_time_and_runtime_safely.md)
- [Detect integer overflow before it occurs using built-in overflow checks](Detect_integer_overflow_before_it_occurs_using_built-in_overflow_checks.md)
- [Enable Control Flow Integrity (CFI) to prevent vtable hijacking](Enable_Control_Flow_Integrity_CFI_to_prevent_vtable_hijacking.md)
- [Enable hardened standard library modes for bounds checking](Enable_hardened_standard_library_modes_for_bounds_checking.md)
- [Enable hardened STL mode for bounds-checked containers in debug builds](Enable_hardened_STL_mode_for_bounds-checked_containers_in_debug_builds.md)
- [Harden the standard library with debug mode and hardening modes](Harden_the_standard_library_with_debug_mode_and_hardening_modes.md)
- [Implement secure coding practices: input validation and output sanitization](Implement_secure_coding_practices_input_validation_and_output_sanitization.md)
- [Prevent format string vulnerabilities and audit for unsafe printf-family calls](Prevent_format_string_vulnerabilities_and_audit_for_unsafe_printf-family_calls.md)
- [Prevent format string vulnerabilities with type-safe formatting](Prevent_format_string_vulnerabilities_with_type-safe_formatting.md)
- [Prevent use-after-free with ownership discipline and lifetime analysis](Prevent_use-after-free_with_ownership_discipline_and_lifetime_analysis.md)
- [Understand and prevent buffer overflow vulnerabilities in C++ code](Understand_and_prevent_buffer_overflow_vulnerabilities_in_C++_code.md)
- [Understand and prevent format string vulnerabilities in C++ logging](Understand_and_prevent_format_string_vulnerabilities_in_C++_logging.md)
- [Understand and prevent race conditions in security-critical code](Understand_and_prevent_race_conditions_in_security-critical_code.md)
- [Understand integer promotion pitfalls in security-critical code](Understand_integer_promotion_pitfalls_in_security-critical_code.md)
- [Understand stack canaries, ASLR, and NX bits as defence-in-depth](Understand_stack_canaries_ASLR_and_NX_bits_as_defence-in-depth.md)
- [Use __builtin_add_overflow and std::add_overflow (C++26) for safe arithmetic](Use___builtin_add_overflow_and_stdadd_overflow_C++26_for_safe_arithmetic.md)
- [Use CFI (Control Flow Integrity) to prevent vtable hijacking attacks](Use_CFI_Control_Flow_Integrity_to_prevent_vtable_hijacking_attacks.md)
- [Use Control Flow Integrity (CFI) to prevent vtable hijacking](Use_Control_Flow_Integrity_CFI_to_prevent_vtable_hijacking.md)
- [Use OS entropy sources correctly: std::random_device and /dev/urandom](Use_OS_entropy_sources_correctly_stdrandom_device_and_devurandom.md)
- [Use sanitizer-driven fuzzing for security vulnerability discovery](Use_sanitizer-driven_fuzzing_for_security_vulnerability_discovery.md)
- [Use stack canaries, ASLR, and NX bits for exploit mitigation](Use_stack_canaries_ASLR_and_NX_bits_for_exploit_mitigation.md)
- [Use std::cmp_less and std::in_range (C++20) for safe integer comparisons](Use_stdcmp_less_and_stdin_range_C++20_for_safe_integer_comparisons.md)
- [Use std::cmp_less and std::in_range (C++20) for safe numeric comparisons](Use_stdcmp_less_and_stdin_range_C++20_for_safe_numeric_comparisons.md)
- [Use std::cmp_less, std::in_range, and safe integer comparison (C++20)](Use_stdcmp_less_stdin_range_and_safe_integer_comparison_C++20.md)

## Notes

- Buffer overflows are the most exploited C++ vulnerability - use `std::span`, `std::array`, and bounds checks instead of raw C arrays.
- Use `std::string` / `std::string_view` instead of raw `char*` for string handling - automatic memory management eliminates whole categories of bugs.
- Integer overflow on signed types is undefined behavior - use unsigned arithmetic or check before performing the operation.
- Always initialize variables - uninitialized reads are UB and a common source of vulnerabilities that attackers can exploit.
- Use RAII to prevent resource leaks that can lead to denial-of-service or exploitable states.
- Static analysis tools (Clang-Tidy, Coverity, PVS-Studio) catch security issues before deployment - integrate them into CI.
- The C++ Core Guidelines define profiles (bounds, type, lifetime) for safety enforcement across a codebase.
- `std::format` eliminates format string vulnerabilities that are present in `printf` - the format string is validated at compile time.
- Avoid `reinterpret_cast` and `void*` - they bypass the type system and enable type confusion vulnerabilities.
- Fuzz testing (libFuzzer, AFL) discovers edge cases that manual testing misses, especially in parsing and input-handling code.
