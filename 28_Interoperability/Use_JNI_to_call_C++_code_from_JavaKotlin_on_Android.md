# Use JNI to call C++ code from Java/Kotlin on Android

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://developer.android.com/training/articles/perf-jni>  

---

## Topic Overview

This topic covers **practical JNI patterns** for Android: writing native functions that accept Java types, understanding reference lifecycle risks, and building cross-ABI native libraries with CMake. The focus is on the Android-specific workflow and toolchain.

If the previous JNI topic was about the conceptual model, this one is about the day-to-day mechanics: processing Java arrays in C++, keeping the local reference table from overflowing, and setting up the CMake build for multiple ABIs. These are the patterns you'll actually repeat in a real Android project.

### Android JNI Build Pipeline

When Gradle builds your Android project, it invokes CMake with the NDK toolchain. CMake then compiles your C++ code for each ABI you have listed in your `abiFilters`. The result is a separate `.so` for each architecture, all bundled into the APK.

```cpp
CMakeLists.txt
    │
    ▼ (gradle calls cmake with NDK toolchain)
┌─────────────────────────────────────────┐
│ NDK Toolchain (android.toolchain.cmake) │
│  ├─ armeabi-v7a  -> arm-linux-androideabi│
│  ├─ arm64-v8a    -> aarch64-linux-android│
│  ├─ x86          -> i686-linux-android   │
│  └─ x86_64       -> x86_64-linux-android │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│ APK/AAB contains:                       │
│  lib/armeabi-v7a/libmyapp.so           │
│  lib/arm64-v8a/libmyapp.so             │
│  lib/x86_64/libmyapp.so                │
└─────────────────────────────────────────┘
```

### JNI Signature Encoding

JNI uses a compact string encoding to describe method signatures when you look up methods by name (via `GetMethodID`). You'll also see these signatures in error messages and decompiled Android apps. The encoding is dense but mechanical - once you see the pattern, it's easy to construct.

| Java Type | JNI Signature | Example |
| --- | :---: | --- |
| `void` | `V` | `()V` = void method, no args |
| `int` | `I` | `(I)I` = int -> int |
| `long` | `J` | `(J)V` = takes long |
| `String` | `Ljava/lang/String;` | `(Ljava/lang/String;)V` |
| `int[]` | `[I` | `([I)V` = takes int array |
| `float[][]` | `[[F` | 2D float array |

---

## Self-Assessment

### Q1: Write a JNI function that receives a Java String, processes it in C++, and returns a jstring

**Answer:**

The Java side declares native methods with the `native` keyword. The static block calls `System.loadLibrary` which triggers loading of the compiled `.so` at class initialization time.

```java
// ═══════════ Java side: ImageProcessor.java ═══════════
package com.example.imageapp;

public class ImageProcessor {
    static { System.loadLibrary("imageproc"); }

    // Native methods
    public native String detectFormat(String filePath);
    public native int[] processPixels(int[] pixels, int width, int height, String filter);
    public native float computeHistogram(int[] pixels, float[] output);
}
```

On the C++ side, notice the two different array patterns. For strings, you call `GetStringUTFChars` and must release it when done. For integer arrays, you call `GetIntArrayElements` and release it with a flag that controls whether changes are copied back to the Java side - `JNI_ABORT` means "don't copy back, I only read it."

```cpp
// ═══════════ C++ side: image_proc.cpp ═══════════
#include <jni.h>
#include <string>
#include <vector>
#include <algorithm>
#include <cmath>

extern "C" {

// ─── String processing ───
JNIEXPORT jstring JNICALL
Java_com_example_imageapp_ImageProcessor_detectFormat(
    JNIEnv* env, jobject thiz, jstring filePath)
{
    // Java String -> C++ std::string
    const char* path = env->GetStringUTFChars(filePath, nullptr);
    if (!path) return nullptr;
    std::string pathStr(path);
    env->ReleaseStringUTFChars(filePath, path);

    // Detect format from extension
    std::string format = "unknown";
    auto dot = pathStr.rfind('.');
    if (dot != std::string::npos) {
        std::string ext = pathStr.substr(dot + 1);
        std::transform(ext.begin(), ext.end(), ext.begin(), ::tolower);
        if (ext == "png") format = "PNG";
        else if (ext == "jpg" || ext == "jpeg") format = "JPEG";
        else if (ext == "webp") format = "WebP";
        else if (ext == "bmp") format = "BMP";
    }

    // C++ string -> Java String
    return env->NewStringUTF(format.c_str());
}

// ─── Array processing ───
JNIEXPORT jintArray JNICALL
Java_com_example_imageapp_ImageProcessor_processPixels(
    JNIEnv* env, jobject thiz,
    jintArray pixels, jint width, jint height, jstring filter)
{
    // Get filter name
    const char* filterName = env->GetStringUTFChars(filter, nullptr);
    std::string filterStr(filterName);
    env->ReleaseStringUTFChars(filter, filterName);

    // Get pixel array
    jint* pixelData = env->GetIntArrayElements(pixels, nullptr);
    jint length = env->GetArrayLength(pixels);

    // Process pixels (grayscale filter example)
    std::vector<jint> result(length);
    if (filterStr == "grayscale") {
        for (jint i = 0; i < length; ++i) {
            int a = (pixelData[i] >> 24) & 0xFF;
            int r = (pixelData[i] >> 16) & 0xFF;
            int g = (pixelData[i] >> 8)  & 0xFF;
            int b =  pixelData[i]        & 0xFF;
            int gray = (r * 299 + g * 587 + b * 114) / 1000;
            result[i] = (a << 24) | (gray << 16) | (gray << 8) | gray;
        }
    }

    // Release input array (JNI_ABORT = don't copy back changes)
    env->ReleaseIntArrayElements(pixels, pixelData, JNI_ABORT);

    // Create and return new array
    jintArray output = env->NewIntArray(length);
    env->SetIntArrayRegion(output, 0, length, result.data());
    return output;
}

}  // extern "C"
```

The return pattern for arrays is: allocate a new Java array with `NewIntArray`, fill it with `SetIntArrayRegion`, and return it. The JVM takes ownership of the returned array - you don't release it.

### Q2: Explain JNI local vs global references and the risk of local reference table overflow

**Answer:**

The reason this trips people up is that JNI references are not the same as C++ pointers. A JNI reference is a handle into the JVM's object table. There are three kinds, and confusing them causes either crashes or memory leaks.

The comparison table below is worth studying:

```cpp
JNI Reference Types:
                                                     GC can
Type       Lifetime                    Table limit    collect?
─────────  ──────────────────────────  ────────────  ─────────
Local      Until method returns (or    512            No
           DeleteLocalRef)             (default)
Global     Until DeleteGlobalRef       No limit       No
Weak       Until DeleteWeakGlobalRef   No limit       YES
           or GC collects the object
```

Here is the overflow problem in action, and two ways to fix it. The first (explicit `DeleteLocalRef`) is simpler; the second (`PushLocalFrame`/`PopLocalFrame`) is cleaner when you create many local refs per iteration and don't want to track them individually.

```cpp
// ═══════════ PROBLEM: Local ref overflow in loops ═══════════
JNIEXPORT void JNICALL
Java_com_example_App_processList(JNIEnv* env, jobject thiz, jobject list)
{
    jclass listClass = env->GetObjectClass(list);
    jmethodID sizeMethod = env->GetMethodID(listClass, "size", "()I");
    jmethodID getMethod = env->GetMethodID(listClass, "get", "(I)Ljava/lang/Object;");

    jint size = env->CallIntMethod(list, sizeMethod);

    for (jint i = 0; i < size; ++i) {
        // EACH call creates a LOCAL reference
        jobject item = env->CallObjectMethod(list, getMethod, i);
        jstring str = (jstring)item;

        const char* chars = env->GetStringUTFChars(str, nullptr);
        // ... process ...
        env->ReleaseStringUTFChars(str, chars);

        // ═══ WITHOUT THIS: 10000 items = 10000 refs -> CRASH ═══
        env->DeleteLocalRef(item);
    }

    env->DeleteLocalRef(listClass);  // Good practice
}

// ═══════════ ALTERNATIVE: PushLocalFrame / PopLocalFrame ═══════════
JNIEXPORT void JNICALL
Java_com_example_App_batchProcess(JNIEnv* env, jobject thiz, jobjectArray items)
{
    jint len = env->GetArrayLength(items);

    for (jint i = 0; i < len; ++i) {
        // Reserve space for 10 local refs in this frame
        if (env->PushLocalFrame(10) < 0) return;  // OutOfMemoryError

        jobject item = env->GetObjectArrayElement(items, i);
        jclass cls = env->GetObjectClass(item);
        // ... create more local refs ...

        // PopLocalFrame releases ALL local refs created in this frame
        env->PopLocalFrame(nullptr);
        // Pass a jobject to PopLocalFrame to keep one ref as return value
    }
}
```

The `PushLocalFrame`/`PopLocalFrame` pattern is particularly clean because it guarantees cleanup even if you hit an error path in the middle of a loop iteration - no individual `DeleteLocalRef` calls to forget.

### Q3: Use CMake's Android NDK toolchain file to build a .so for multiple ABIs

**Answer:**

The CMakeLists.txt below is a complete, production-ready setup for an Android native library. Pay attention to the NDK-specific libraries: `android` gives you access to native window and asset management, `log` gives you Logcat, and `jnigraphics` lets you work directly with Android `Bitmap` objects from C++.

```cmake
# ═══════════ CMakeLists.txt (in app/src/main/cpp/) ═══════════
cmake_minimum_required(VERSION 3.18)
project(imageproc LANGUAGES CXX)

# Create shared library
add_library(imageproc SHARED
    image_proc.cpp
    utils.cpp
)

# C++ standard
target_compile_features(imageproc PRIVATE cxx_std_17)

# Compiler warnings
target_compile_options(imageproc PRIVATE
    -Wall -Wextra -Wpedantic
)

# Link NDK libraries
target_link_libraries(imageproc
    android          # native activity, window, etc.
    log              # __android_log_print
    jnigraphics      # AndroidBitmap functions
)

# Strip debug symbols in release (smaller APK)
set_target_properties(imageproc PROPERTIES
    INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE
)
```

The Gradle configuration is where you specify which ABIs to build and passes extra arguments to CMake. The `-DANDROID_STL=c++_shared` flag tells the NDK to use the shared version of the C++ standard library rather than statically linking it into every `.so` - this is important when you have multiple native libraries in one app.

```groovy
// ═══════════ app/build.gradle.kts ═══════════
android {
    ndkVersion = "25.2.9519653"

    defaultConfig {
        ndk {
            // Specify which ABIs to build
            abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86_64")
        }

        externalNativeBuild {
            cmake {
                cppFlags += "-std=c++17"
                arguments += listOf(
                    "-DANDROID_STL=c++_shared",  // shared libc++
                    "-DANDROID_ARM_NEON=TRUE"     // NEON on ARM
                )
            }
        }
    }

    externalNativeBuild {
        cmake {
            path = file("src/main/cpp/CMakeLists.txt")
            version = "3.22.1"
        }
    }
}
```

You can also drive the build from the command line without Android Studio. This is useful for CI pipelines or when you want to quickly test a single ABI.

```bash
# Build from command line (without Android Studio):
cmake -B build \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_NATIVE_API_LEVEL=24 \
  -DANDROID_STL=c++_shared

cmake --build build

# Build for multiple ABIs:
for ABI in armeabi-v7a arm64-v8a x86_64; do
  cmake -B build-$ABI \
    -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=$ABI \
    -DANDROID_NATIVE_API_LEVEL=24
  cmake --build build-$ABI
done
```

---

## Notes

- `JNI_ABORT` in `ReleaseArrayElements` means "don't copy changes back" - use it when you only read the array; use `0` (or `JNI_COMMIT` + `0`) when you've modified the data and want it written back.
- `GetStringUTFChars` returns Modified UTF-8 rather than standard UTF-8 - this matters for supplementary Unicode characters (code points above U+FFFF), which are encoded differently.
- The `env` pointer is **thread-local** - never cache it across threads. If you need JNI access from a background thread, use `JavaVM::AttachCurrentThread` to get a fresh `env`.
- Method IDs and field IDs are stable across calls and do not count as JNI references - cache them in `JNI_OnLoad` for performance, not correctness.
- Use `ExceptionCheck`/`ExceptionClear` after JNI calls that may throw Java exceptions; a pending exception in the JNI frame can cause undefined behavior if ignored.
- For Android 12+, consider using the newer `JNI_CreateJavaVM` API for standalone native apps.
