# Use JNI for Android C++ development with the NDK

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://developer.android.com/ndk/guides/jni>  

---

## Topic Overview

**JNI (Java Native Interface)** allows Java/Kotlin code on Android to call C++ functions compiled via the **NDK (Native Development Kit)**. This is essential for performance-critical code (audio, video, ML, game engines) and reusing existing C/C++ libraries.

The reason JNI exists is that Android apps run on the Java Virtual Machine (or ART, Android Runtime), but sometimes you need to drop into native code - either because you have an existing C/C++ library you want to reuse, or because the JVM's overhead is too high for tight inner loops. JNI is the bridge between those two worlds.

### JNI Call Flow on Android

Here is the big picture of how a Kotlin call ends up executing C++ code. The key points are: the function naming convention (`Java_package_Class_method`) is how the JVM finds the right native function without any registration step, and `System.loadLibrary` triggers the `dlopen` of your compiled `.so` file.

```cpp
Kotlin/Java Layer                       Native Layer (C++)
┌──────────────────────┐               ┌───────────────────────────────┐
│ class NativeLib {    │               │ #include <jni.h>              │
│   external fun       │               │                               │
│     process(s: String│──────────────►│ JNIEXPORT jstring JNICALL     │
│     ): String        │   JNI call    │ Java_com_app_NativeLib_process│
│ }                    │◄──────────────│ (JNIEnv* env, jobject thiz,   │
│                      │   return      │  jstring input) { ... }       │
│ companion object {   │               │                               │
│   init {             │               │ // Loaded from .so:           │
│     System.loadLib   │──────────────►│ libapp.so                     │
│     ("app")          │   dlopen      │                               │
│   }                  │               └───────────────────────────────┘
│ }                    │
└──────────────────────┘
```

### JNI Type Mapping

Every Java/Kotlin primitive type has a corresponding JNI type on the C++ side. The JNI types are just typedefs - `jint` is really just `int32_t`, `jstring` is really just a pointer. The table below is your quick reference when translating method signatures.

| Java/Kotlin Type | JNI C++ Type | Size |
| --- | :---: | :---: |
| `boolean` | `jboolean` | 8 bit |
| `int` | `jint` | 32 bit |
| `long` | `jlong` | 64 bit |
| `float` | `jfloat` | 32 bit |
| `double` | `jdouble` | 64 bit |
| `String` | `jstring` | pointer |
| `int[]` | `jintArray` | pointer |
| `Object` | `jobject` | pointer |

---

## Self-Assessment

### Q1: Write a JNI method that takes a Java String, processes it in C++, and returns a Java String

**Answer:**

The Kotlin side is straightforward: mark the function `external` and call `System.loadLibrary` in the companion object's `init` block. That `external` keyword tells the compiler "this function will be provided at runtime from native code."

```kotlin
// ═══════════ Kotlin side: NativeLib.kt ═══════════
package com.example.myapp

class NativeLib {
    // Declare native methods
    external fun processString(input: String): String
    external fun computeHash(data: String): Int

    companion object {
        init {
            System.loadLibrary("myapp")  // loads libmyapp.so
        }
    }
}

// Usage:
// val lib = NativeLib()
// val result = lib.processString("Hello Android!")
// println(result)  // "HELLO ANDROID!" (uppercased by C++)
```

On the C++ side, the function naming convention is the critical piece. Every dot in the Java package name becomes an underscore, and the class name and method name follow. The comments below explain the two most important steps: the string conversion (with its mandatory Release call) and the return conversion.

```cpp
// ═══════════ C++ side: native-lib.cpp ═══════════
#include <jni.h>
#include <string>
#include <algorithm>
#include <numeric>

extern "C" {

// Function name MUST follow: Java_<package>_<class>_<method>
// com.example.myapp.NativeLib.processString -> underscores replace dots
JNIEXPORT jstring JNICALL
Java_com_example_myapp_NativeLib_processString(
    JNIEnv* env,       // JNI environment — all JNI calls go through this
    jobject thiz,      // the NativeLib instance (use jclass for static methods)
    jstring input)     // Java String parameter
{
    // ─── Convert Java String -> C++ string ───
    const char* utf8 = env->GetStringUTFChars(input, nullptr);
    if (!utf8) return nullptr;  // OutOfMemoryError thrown

    std::string cpp_str(utf8);

    // MUST release after copying — it's a JNI resource
    env->ReleaseStringUTFChars(input, utf8);

    // ─── Process in C++ ───
    std::transform(cpp_str.begin(), cpp_str.end(), cpp_str.begin(),
                   [](unsigned char c) { return std::toupper(c); });

    // ─── Convert C++ string -> Java String and return ───
    return env->NewStringUTF(cpp_str.c_str());
}

JNIEXPORT jint JNICALL
Java_com_example_myapp_NativeLib_computeHash(
    JNIEnv* env, jobject thiz, jstring data)
{
    const char* str = env->GetStringUTFChars(data, nullptr);
    if (!str) return 0;

    // Simple hash computation
    std::string s(str);
    env->ReleaseStringUTFChars(data, str);

    uint32_t hash = 0;
    for (char c : s) {
        hash = hash * 31 + static_cast<uint32_t>(c);
    }
    return static_cast<jint>(hash);
}

}  // extern "C"
```

The `GetStringUTFChars` / `ReleaseStringUTFChars` pair is a resource management pattern that you'll repeat constantly in JNI code. Think of it like `new`/`delete` - every `Get` must have a matching `Release`.

### Q2: Manage JNI local and global references to avoid reference table overflow

**Answer:**

This is one of the most common JNI bugs and one of the hardest to diagnose in the field. The JNI runtime gives each native call a local reference table - by default it holds 512 entries. Every time you call a JNI function that returns a Java object (like `GetObjectArrayElement`), it adds an entry to that table. In a loop processing thousands of items, you can easily overflow the table and crash the app with a cryptic error.

The fix is simple: delete each local reference as soon as you're done with it inside the loop.

```cpp
#include <jni.h>
#include <string>

// ═══════════ Local references: auto-freed when native method returns ═══════════
// Each JNI method gets a local reference table (default: 512 entries)
// Creating too many locals in a loop -> TABLE OVERFLOW -> crash

JNIEXPORT void JNICALL
Java_com_example_myapp_NativeLib_processArray(
    JNIEnv* env, jobject thiz, jobjectArray items)
{
    jint length = env->GetArrayLength(items);

    for (jint i = 0; i < length; ++i) {
        // Each GetObjectArrayElement creates a LOCAL reference
        jstring item = (jstring)env->GetObjectArrayElement(items, i);

        const char* str = env->GetStringUTFChars(item, nullptr);
        // ... process str ...
        env->ReleaseStringUTFChars(item, str);

        // ═══════════ CRITICAL: Delete local ref in loops! ═══════════
        env->DeleteLocalRef(item);
        // Without this, 10000 iterations = 10000 local refs -> OVERFLOW
    }
}

// ═══════════ Global references: survive across JNI calls ═══════════
// Use when caching Java objects in C++ static/global variables

static jclass     g_class_ref = nullptr;  // Global ref to Java class
static jmethodID  g_method_id = nullptr;  // Method IDs don't need ref

JNIEXPORT jint JNICALL
JNI_OnLoad(JavaVM* vm, void* reserved)
{
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    // FindClass returns LOCAL ref — convert to GLOBAL for caching
    jclass local_class = env->FindClass("com/example/myapp/Logger");
    if (!local_class) return JNI_ERR;

    g_class_ref = (jclass)env->NewGlobalRef(local_class);
    env->DeleteLocalRef(local_class);  // Don't need local anymore

    // Method IDs are NOT references — no NewGlobalRef needed
    g_method_id = env->GetStaticMethodID(
        g_class_ref, "log", "(Ljava/lang/String;)V");

    return JNI_VERSION_1_6;
}

JNIEXPORT void JNICALL
JNI_OnUnload(JavaVM* vm, void* reserved)
{
    JNIEnv* env;
    vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6);

    // ═══════════ MUST delete global refs to avoid memory leak ═══════════
    if (g_class_ref) {
        env->DeleteGlobalRef(g_class_ref);
        g_class_ref = nullptr;
    }
}

// ═══════════ Reference type comparison ═══════════
//
// Type        | Lifetime              | Freed by          | Use case
// ------------|----------------------|-------------------|------------------
// Local       | Current native call  | Auto / DeleteLocal| Temporary objects
// Global      | Until DeleteGlobalRef| Manual            | Cached classes
// Weak Global | Until GC'd           | DeleteWeakGlobal  | Caches that allow GC
```

The `JNI_OnLoad` pattern is worth memorising. It runs when `System.loadLibrary` loads your `.so`, which makes it a natural place to cache class references and method IDs that you'll need repeatedly. The key detail is that `FindClass` returns a local reference - you must promote it to a global reference before storing it in a static variable, otherwise the local ref will be invalidated as soon as `JNI_OnLoad` returns.

### Q3: Use the Android NDK's android_log_print for C++ logging visible in Logcat

**Answer:**

Standard C++ output (`printf`, `std::cout`) goes nowhere on Android - there is no console. Instead, the NDK provides `__android_log_print` which sends messages to Android's Logcat system. The macros below wrap it into a comfortable API that mirrors the Android log levels you're used to from Java.

```cpp
// ═══════════ Logging from C++ via Android's Logcat ═══════════
#include <jni.h>
#include <android/log.h>  // NDK logging header
#include <string>
#include <chrono>

// Define a log tag for filtering in Logcat
#define LOG_TAG "MyAppNative"

// Convenience macros (matching Android log levels)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,  LOG_TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,  LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

// ═══════════ Usage in native methods ═══════════
JNIEXPORT jint JNICALL
Java_com_example_myapp_NativeLib_heavyComputation(
    JNIEnv* env, jobject thiz, jint n)
{
    LOGI("heavyComputation called with n=%d", n);

    auto start = std::chrono::steady_clock::now();

    // Simulate heavy work
    jint result = 0;
    for (jint i = 0; i < n; ++i) {
        result += i * i;
    }

    auto end = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

    LOGD("Computation result: %d (took %lld ms)", result, (long long)ms);

    if (ms > 1000) {
        LOGW("heavyComputation took over 1 second!");
    }

    return result;
}

// ═══════════ Error logging pattern ═══════════
JNIEXPORT jstring JNICALL
Java_com_example_myapp_NativeLib_loadFile(
    JNIEnv* env, jobject thiz, jstring path)
{
    const char* c_path = env->GetStringUTFChars(path, nullptr);
    if (!c_path) {
        LOGE("Failed to get UTF chars from path string");
        return nullptr;
    }

    LOGI("Loading file: %s", c_path);

    FILE* f = fopen(c_path, "r");
    if (!f) {
        LOGE("Failed to open file: %s (errno=%d)", c_path, errno);
        env->ReleaseStringUTFChars(path, c_path);
        // Throw Java exception
        env->ThrowNew(
            env->FindClass("java/io/FileNotFoundException"),
            c_path);
        return nullptr;
    }

    // Read file...
    fclose(f);
    env->ReleaseStringUTFChars(path, c_path);
    return env->NewStringUTF("file contents here");
}

// In Logcat, filter with: adb logcat -s MyAppNative
// Output:
// D/MyAppNative: Computation result: 328350 (took 2 ms)
// I/MyAppNative: Loading file: /data/data/com.example.myapp/files/config.json
```

You need to link the `log` library in your CMakeLists.txt for `__android_log_print` to be available. Without this, you'll get a linker error at build time.

```cmake
# CMakeLists.txt — link against Android log library
cmake_minimum_required(VERSION 3.18)
project(myapp)

add_library(myapp SHARED native-lib.cpp)

# Link against the NDK's log library
target_link_libraries(myapp
    android      # for native app glue (optional)
    log          # for __android_log_print
)

target_compile_features(myapp PRIVATE cxx_std_17)
```

---

## Notes

- JNI function names follow strict naming: `Java_<package>_<class>_<method>` with `_` for `.` and `_1` for actual underscores in method names.
- Always call `ReleaseStringUTFChars` after `GetStringUTFChars` - the memory is a JNI-managed resource and will leak otherwise.
- `DeleteLocalRef` is mandatory inside loops; Android's local ref table is limited (512 entries by default) and overflow causes a hard crash.
- Use `JNI_OnLoad` with `RegisterNatives` if you want to avoid the fragile naming convention and register functions explicitly instead.
- Link the `log` library in CMakeLists.txt to use `__android_log_print`.
- NDK supports C++17 by default; C++20 is available with NDK r25 and later.
