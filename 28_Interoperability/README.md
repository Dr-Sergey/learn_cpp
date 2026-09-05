# Interoperability

C/C++ interop, FFI, ABI stability, and interfacing with other languages and systems.

**Topics:** 25

## Contents

- [Compile C++ to WebAssembly with Emscripten](Compile_C++_to_WebAssembly_with_Emscripten.md)
- [Compile C++ to WebAssembly with Emscripten (Part 2: Bindings and Memory)](Compile_C++_to_WebAssembly_with_Emscripten_2.md)
- [Compile C++ to WebAssembly with Emscripten (Part 3: I/O, Debugging, and Optimization)](Compile_C++_to_WebAssembly_with_Emscripten_3.md)
- [Handle calling convention differences between C++ and external languages](Handle_calling_convention_differences_between_C++_and_external_languages.md)
- [Interoperate with Rust via a C ABI boundary using the cxx crate](Interoperate_with_Rust_via_a_C_ABI_boundary_using_the_cxx_crate.md)
- [Interoperate with Rust via a C ABI using the cxx crate (Ownership and Error Handling)](Interoperate_with_Rust_via_a_C_ABI_using_the_cxx_crate.md)
- [Link C and C++ objects and match calling conventions on Windows](Understand_C_interop_extern_C_name_mangling_and_calling_conventions_2.md)
- [Set up a pybind11 build with CMake and scikit-build](Use_pybind11_to_expose_C++_classes_and_functions_to_Python_2.md)
- [Understand C interop: extern C, name mangling, and calling conventions](Understand_C_interop_extern_C_name_mangling_and_calling_conventions.md)
- [Understand COM and DCOM interop on Windows](Understand_COM_and_DCOM_interop_on_Windows.md)
- [Use COM (Component Object Model) on Windows from modern C++](Use_COM_Component_Object_Model_on_Windows_from_modern_C++.md)
- [Use ctypes / cffi from Python to call C++ shared libraries](Use_ctypes_cffi_from_Python_to_call_C++_shared_libraries.md)
- [Use extern C for C interoperability and name mangling control](Use_extern_C_for_C_interoperability_and_name_mangling_control.md)
- [Use JNI for Android C++ development with the NDK](Use_JNI_for_Android_C++_development_with_the_NDK.md)
- [Use JNI to call C++ code from Java/Kotlin on Android](Use_JNI_to_call_C++_code_from_JavaKotlin_on_Android.md)
- [Use nanobind as a faster, smaller alternative to pybind11](Use_nanobind_as_a_faster_smaller_alternative_to_pybind11.md)
- [Use nanobind as a lighter-weight pybind11 alternative](Use_nanobind_as_a_lighter-weight_pybind11_alternative.md)
- [Use nanobind for lightweight, fast Python bindings](Use_nanobind_for_lightweight_fast_Python_bindings.md)
- [Use pybind11 for C++/Python interoperability](Use_pybind11_for_C++Python_interoperability.md)
- [Use pybind11 to expose C++ classes and functions to Python](Use_pybind11_to_expose_C++_classes_and_functions_to_Python.md)
- [Use SWIG to generate bindings for multiple languages from one interface file](Use_SWIG_to_generate_bindings_for_multiple_languages_from_one_interface_file.md)
- [Use the cxx crate for safe Rust/C++ interoperability](Use_the_cxx_crate_for_safe_RustC++_interoperability.md)
- [Wrap a C library with a modern C++ RAII interface](Wrap_a_C_library_with_a_modern_C++_RAII_interface.md)
- [Write JNI (Java Native Interface) wrappers for Android C++ code](Write_JNI_Java_Native_Interface_wrappers_for_Android_C++_code.md)
- [Write portable C++ targeting both MSVC and GCC/Clang simultaneously](Write_portable_C++_targeting_both_MSVC_and_GCCClang_simultaneously.md)

## Notes

- `extern "C"` disables name mangling - required for C++ functions callable from C or other languages.
- C++ can call C libraries directly via `extern "C"` headers - no wrapper needed.
- Python integration (pybind11, nanobind) wraps C++ classes and functions for Python consumption.
- FFI (Foreign Function Interface) requires C-compatible types - no classes, templates, or exceptions across boundaries.
- SWIG generates bindings for multiple languages from annotated C++ headers.
- JNI (Java Native Interface) connects C++ with Java - manage JNI references carefully to avoid leaks.
- `std::span` and raw pointers are the typical boundary types for interop.
- ABI compatibility matters - mixing compiler versions or standard libraries can cause subtle bugs.
- gRPC and Protocol Buffers provide language-agnostic RPC - good for microservice boundaries.
- WebAssembly (Emscripten) compiles C++ to run in browsers - a growing interop target.
