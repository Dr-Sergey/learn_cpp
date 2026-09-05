# ABI & Binary Compatibility

ABI (Application Binary Interface) is the contract that governs how compiled code communicates at the machine level - calling conventions, vtable layout, name mangling, struct padding, exception unwinding. When that contract is broken, you get link errors, crashes, or silent wrong-behavior at runtime. This section covers everything you need to reason about ABI stability, control it deliberately, and avoid the classic pitfalls.

**Topics:** 10

## Contents

- [Control Symbol Visibility for Shared Libraries](Control_symbol_visibility_for_shared_libraries.md)
- [Design ABI-Stable C++ Library Interfaces](Design_ABI-stable_C++_library_interfaces.md)
- [Implement SO and DLL Versioning Strategies](Implement_SO_and_DLL_versioning_strategies.md)
- [Know MSVC ABI and Class Layout with /d1reportSingleClassLayout](Know_MSVC_ABI_and_class_layout_with_reportSingleClassLayout.md)
- [Understand abi_tag Attribute for Symbol Versioning](Understand_abi_tag_attribute_for_symbol_versioning.md)
- [Understand COM ABI Implications on Windows](Understand_COM_ABI_implications_on_Windows.md)
- [Understand Itanium ABI Layout Rules for Vtables, RTTI, and Exceptions](Understand_Itanium_ABI_layout_rules_for_vtables_RTTI_and_exceptions.md)
- [Understand What Changes Break ABI in C++ Libraries](Understand_what_changes_break_ABI_in_C++_libraries.md)
- [Use extern "C" Wrappers for Stable FFI Boundaries](Use_extern_C_wrappers_for_stable_FFI_boundaries.md)
- [Use Inline Namespace Versioning for ABI Evolution](Use_inline_namespace_versioning_for_ABI_evolution.md)

## Notes

- ABI defines how code interacts at the binary level: calling conventions, vtable layout, and name mangling are all part of the contract.
- The Itanium ABI is used by GCC and Clang on most platforms; MSVC has its own distinct ABI with different vtable and virtual-inheritance layouts.
- Changing virtual function order, adding virtual functions, or modifying class layout breaks ABI and will cause silent mismatches or crashes in already-compiled code.
- Inline namespaces allow ABI versioning - `std::string` changed its ABI between libstdc++ versions using exactly this mechanism.
- The Pimpl idiom preserves ABI because adding private members only changes the hidden `Impl` struct, not the class size visible to users.
- `extern "C"` functions have a stable, name-mangling-free ABI - use them for plugin interfaces and FFI boundaries.
- `std::function` and `std::any` provide ABI-stable type erasure that is useful at library boundaries.
- Symbol visibility (`__attribute__((visibility("default")))`) controls what gets exported from a shared library and directly affects load time and ODR safety.
- Mixing different C++ standard library versions (libstdc++ vs libc++) in one process is dangerous and almost always leads to subtle corruption.
- Use `-fvisibility=hidden` and explicit export macros to minimize your ABI surface area - smaller is safer.
