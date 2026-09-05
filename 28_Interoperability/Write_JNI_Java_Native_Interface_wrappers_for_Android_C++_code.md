# Write JNI (Java Native Interface) wrappers for Android C++ code

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://docs.oracle.com/en/java/javase/17/docs/specs/jni/>  

---

## Topic Overview

This topic covers **advanced JNI patterns**: implementing native methods with correct naming conventions, converting between `jstring` and `std::string` with proper UTF-8 handling, and using `RegisterNatives` for dynamic method registration (avoiding fragile name-mangled dispatch).

JNI is Java's mechanism for calling native code - which in Android almost always means C++. The API is notoriously verbose, but once you understand the patterns they repeat consistently: get a pointer into JVM memory, use it, release it. Skipping the release is a common source of subtle memory leaks.

### Static vs Dynamic JNI Registration

There are two ways to connect a `native` Java method to its C++ implementation. The static approach relies on a naming convention that the JVM uses to find the function by name at runtime. The dynamic approach registers a table of function pointers explicitly in `JNI_OnLoad`, which the JVM calls when your library is loaded.

```cpp
Static Registration (naming convention):
  Java: native void process()  in com.example.Foo
  C++:  Java_com_example_Foo_process(JNIEnv*, jobject)
  // JVM finds by name at runtime (slow first call, fragile names)

Dynamic Registration (RegisterNatives):
  C++:  any_name(JNIEnv*, jobject)  // any function name
  Register: JNINativeMethod methods[] = {
    {"process", "()V", (void*)any_name}
  };
  // Registered in JNI_OnLoad (fast, refactorable, less error-prone)
```

### String Conversion Pitfalls

JNI has multiple string access functions with important differences. The most common mistake is forgetting to release the pointer returned by `GetStringUTFChars`, which leaks memory. Use the table below to choose the right function for your situation.

| JNI Function | Returns | Encoding | Must Release? |
| --- | --- | --- | :---: |
| `GetStringUTFChars` | `const char*` | Modified UTF-8 | Yes (`ReleaseStringUTFChars`) |
| `GetStringChars` | `const jchar*` | UTF-16 | Yes (`ReleaseStringChars`) |
| `GetStringRegion` | copies into buffer | UTF-16 | No (no alloc) |
| `GetStringUTFRegion` | copies into buffer | Modified UTF-8 | No (no alloc) |
| `NewStringUTF` | `jstring` | Modified UTF-8 | No (new object) |

---

## Self-Assessment

### Q1: Implement a native method with the JNI signature Java_com_example_Foo_bar and register it

**Answer:**

Here is a complete example with both the Java declaration and the C++ implementation. Static registration requires the function name to exactly match the Java package, class, and method name - dots become underscores. One mistake in the name and the JVM throws `UnsatisfiedLinkError` at runtime with no helpful message about what went wrong.

```java
// Java side: com/example/crypto/CryptoLib.java
package com.example.crypto;

public class CryptoLib {
    static { System.loadLibrary("crypto_jni"); }

    // Native methods
    public native byte[] encrypt(byte[] data, String key);
    public native byte[] decrypt(byte[] data, String key);
    public native String hash(String input, String algorithm);
}
```

The C++ side must use `extern "C"` to prevent name mangling, which would otherwise change the function name and break the JVM's lookup.

```cpp
// C++ side: crypto_jni.cpp
#include <jni.h>
#include <string>
#include <vector>
#include <numeric>

// Static registration: name MUST match Java_<package>_<class>_<method>
extern "C" {

JNIEXPORT jbyteArray JNICALL
Java_com_example_crypto_CryptoLib_encrypt(
    JNIEnv* env,
    jobject thiz,       // 'this' in Java
    jbyteArray data,
    jstring key)
{
    // Get key
    const char* keyStr = env->GetStringUTFChars(key, nullptr);
    if (!keyStr) return nullptr;
    std::string keyStd(keyStr);
    env->ReleaseStringUTFChars(key, keyStr);

    // Get data array
    jbyte* dataPtr = env->GetByteArrayElements(data, nullptr);
    jint dataLen = env->GetArrayLength(data);

    // Simple XOR "encryption" (for demonstration - NOT secure!)
    std::vector<jbyte> encrypted(dataLen);
    for (jint i = 0; i < dataLen; ++i) {
        encrypted[i] = dataPtr[i] ^ keyStd[i % keyStd.size()];
    }

    env->ReleaseByteArrayElements(data, dataPtr, JNI_ABORT);

    // Create result array
    jbyteArray result = env->NewByteArray(dataLen);
    env->SetByteArrayRegion(result, 0, dataLen, encrypted.data());
    return result;
}

JNIEXPORT jstring JNICALL
Java_com_example_crypto_CryptoLib_hash(
    JNIEnv* env, jobject thiz,
    jstring input, jstring algorithm)
{
    const char* inputStr = env->GetStringUTFChars(input, nullptr);
    if (!inputStr) return nullptr;

    // Simple hash for demo
    uint64_t hash = 0;
    for (const char* p = inputStr; *p; ++p) {
        hash = hash * 31 + *p;
    }
    env->ReleaseStringUTFChars(input, inputStr);

    char hexBuf[17];
    snprintf(hexBuf, sizeof(hexBuf), "%016llx", (unsigned long long)hash);
    return env->NewStringUTF(hexBuf);
}

}  // extern "C"
```

### Q2: Convert between JNI jstring and std::string correctly including UTF-8 handling

**Answer:**

String handling in JNI is where most bugs hide. The reason is that JNI's "UTF-8" is actually Modified UTF-8 - a variant of CESU-8 that differs from standard UTF-8 in two ways: the null character (U+0000) is encoded as the two-byte sequence `0xC0 0x80` instead of a single `0x00`, and Unicode supplementary characters (anything above U+FFFF, like most emoji) are encoded using surrogate pairs rather than the standard four-byte sequence. For ASCII-only text this does not matter. For emoji and certain CJK characters, you need to be aware of it.

The RAII wrapper below makes the pattern safe by tying `ReleaseStringUTFChars` to the destructor, so you cannot forget the release.

```cpp
#include <jni.h>
#include <string>

// RAII wrapper for JNI string access
class JniString {
    JNIEnv* env_;
    jstring jstr_;
    const char* chars_;
public:
    JniString(JNIEnv* env, jstring jstr)
        : env_(env), jstr_(jstr), chars_(nullptr)
    {
        if (jstr_) {
            chars_ = env_->GetStringUTFChars(jstr_, nullptr);
        }
    }

    ~JniString() {
        if (chars_) {
            env_->ReleaseStringUTFChars(jstr_, chars_);
        }
    }

    // Convert to std::string (copies - safe after release)
    std::string str() const {
        return chars_ ? std::string(chars_) : std::string();
    }

    const char* c_str() const { return chars_; }
    bool valid() const { return chars_ != nullptr; }

    // Non-copyable
    JniString(const JniString&) = delete;
    JniString& operator=(const JniString&) = delete;
};

// UTF-8 handling details
extern "C"
JNIEXPORT jstring JNICALL
Java_com_example_App_processText(JNIEnv* env, jobject thiz, jstring input)
{
    // Method 1: GetStringUTFChars (Modified UTF-8)
    // WARNING: JNI "Modified UTF-8" differs from standard UTF-8:
    //   - U+0000 encoded as 0xC0 0x80 (not 0x00)
    //   - Supplementary chars (U+10000+) use surrogate pairs in CESU-8
    {
        JniString str(env, input);
        if (!str.valid()) return nullptr;
        std::string processed = str.str();
        // processed is Modified UTF-8 - fine for ASCII, careful with emoji/CJK
    }

    // Method 2: GetStringChars (UTF-16) - correct for all Unicode
    {
        const jchar* utf16 = env->GetStringChars(input, nullptr);
        jint len = env->GetStringLength(input);

        // Process UTF-16 data...
        // Convert to std::u16string or std::wstring
        std::u16string u16str(reinterpret_cast<const char16_t*>(utf16), len);

        env->ReleaseStringChars(input, utf16);

        // For proper UTF-8: use ICU or manual conversion from UTF-16
    }

    // Method 3: GetStringUTFRegion (no allocation, no release needed)
    {
        jint len = env->GetStringUTFLength(input);
        std::string buf(len, '\0');
        env->GetStringUTFRegion(input, 0, env->GetStringLength(input), buf.data());
        // buf contains Modified UTF-8 - efficient, no release needed
    }

    // Return a new Java String from C++ string
    std::string result = "processed text";
    return env->NewStringUTF(result.c_str());
    // NewStringUTF expects Modified UTF-8 input
}
```

### Q3: Use RegisterNatives for dynamic JNI registration instead of name-mangled static dispatch

**Answer:**

`RegisterNatives` is the production-quality approach. Instead of the JVM hunting for a function named `Java_com_example_myapp_MathLib_greet` at runtime, you build a table that maps Java method names to C++ function pointers, and you register the whole table in `JNI_OnLoad`. This fails fast (at library load time rather than first use), works correctly with Proguard/R8 name obfuscation, and lets you rename your C++ functions without touching Java.

```cpp
#include <jni.h>
#include <string>
#include <android/log.h>

#define TAG "NativeReg"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)

// Native implementations (any function name!)
static jstring native_greet(JNIEnv* env, jobject thiz, jstring name) {
    const char* nameStr = env->GetStringUTFChars(name, nullptr);
    std::string greeting = "Hello, " + std::string(nameStr) + "!";
    env->ReleaseStringUTFChars(name, nameStr);
    return env->NewStringUTF(greeting.c_str());
}

static jint native_add(JNIEnv* env, jobject thiz, jint a, jint b) {
    return a + b;
}

static jdoubleArray native_linspace(
    JNIEnv* env, jobject thiz, jdouble start, jdouble end, jint count)
{
    jdoubleArray result = env->NewDoubleArray(count);
    if (count <= 1) {
        env->SetDoubleArrayRegion(result, 0, 1, &start);
        return result;
    }
    std::vector<jdouble> values(count);
    for (jint i = 0; i < count; ++i) {
        values[i] = start + (end - start) * i / (count - 1);
    }
    env->SetDoubleArrayRegion(result, 0, count, values.data());
    return result;
}

// Registration table
// Format: { "javaMethodName", "(JNI signature)return", functionPointer }
static JNINativeMethod methods[] = {
    {"greet",    "(Ljava/lang/String;)Ljava/lang/String;", (void*)native_greet},
    {"add",      "(II)I",                                   (void*)native_add},
    {"linspace", "(DDI)[D",                                 (void*)native_linspace},
};

// JNI_OnLoad: register all native methods
extern "C" JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    // Find the Java class
    jclass clazz = env->FindClass("com/example/myapp/MathLib");
    if (!clazz) {
        LOGI("ERROR: class not found");
        return JNI_ERR;
    }

    // Register ALL methods at once
    int count = sizeof(methods) / sizeof(methods[0]);
    if (env->RegisterNatives(clazz, methods, count) < 0) {
        LOGI("ERROR: RegisterNatives failed");
        return JNI_ERR;
    }

    LOGI("Registered %d native methods", count);
    env->DeleteLocalRef(clazz);
    return JNI_VERSION_1_6;
}

// Advantages over static registration:
//
// Static:  Java_com_example_myapp_MathLib_greet(...)
//   BAD: Rename package/class -> must rename C++ function
//   BAD: Long ugly names
//   BAD: Symbol lookup at first call (slower)
//   BAD: Proguard can break it (name obfuscation)
//
// Dynamic: RegisterNatives in JNI_OnLoad
//   GOOD: Any C++ function name
//   GOOD: Rename via table update, not function rename
//   GOOD: All methods registered upfront (fail-fast)
//   GOOD: Works with Proguard (use @Keep on native methods)
//   GOOD: Smaller symbol table (no exported Java_* names)
```

The Java side is completely independent of what your C++ functions are named - it only cares about the names in the registration table.

```java
// Java side: MathLib.java
package com.example.myapp;

public class MathLib {
    static { System.loadLibrary("mathlib"); }

    // These names match the RegisterNatives table - NOT the C++ function names
    public native String greet(String name);
    public native int add(int a, int b);
    public native double[] linspace(double start, double end, int count);
}
```

---

## Notes

- `RegisterNatives` is the preferred approach for production Android apps - it is faster, more refactorable, and Proguard-safe.
- JNI type signatures use `L<fully/qualified/name>;` for object types and a `[` prefix for arrays. Use `javap -s com.example.MyClass` to generate the correct signature strings for any class.
- Modified UTF-8 is not the same as standard UTF-8 - you will only notice the difference with emoji and CJK supplementary characters, but when you do notice it, it will be confusing.
- `GetStringUTFRegion` is the best choice for short strings you are going to copy immediately, since it avoids any allocation and requires no release.
- Always check for `nullptr` after `GetStringUTFChars` - a null return signals that an `OutOfMemoryError` has been thrown in the JVM.
- Keep native methods annotated with `@Keep` when using Proguard or R8 to prevent them from being renamed or removed.
