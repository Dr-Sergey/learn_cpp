# Understand COM ABI Implications on Windows

**Category:** ABI and Binary Compatibility  
**Standard:** COM  - Windows platform (Component Object Model)  
**Reference:** https://learn.microsoft.com/en-us/windows/win32/com/the-component-object-model  

---

## Topic Overview

COM (Component Object Model) is Microsoft's binary interface standard that achieves what C++ cannot: **stable ABI across compilers, languages, and versions**. Every DirectX call, every Shell extension, every OLE automation object uses COM. The genius of COM is that it reduces the ABI contract to the simplest possible element: a **vtable pointer** - a C array of function pointers laid out in a guaranteed order. Any compiler that can produce a compatible vtable layout can create and consume COM objects.

The reason this works where ordinary C++ does not is that COM strips away every source of ABI instability. There is no name mangling (C entry points, GUID-based identity), no exception propagation across boundaries (everything returns `HRESULT`), no data members in interfaces (so `sizeof` is always `sizeof(void*)`), and no template expansion. What remains is a plain array of function pointers - something that compilers from completely different vendors, or even hand-written assembly, can agree on.

The foundation of COM is `IUnknown`, an interface with exactly three methods in a fixed vtable layout:

```cpp
// IUnknown vtable layout (immutable since 1993):
//   Slot 0: QueryInterface(REFIID riid, void** ppv)
//   Slot 1: AddRef()
//   Slot 2: Release()
//
// Every COM interface inherits from IUnknown and extends the vtable:
//   IMyInterface vtable:
//     Slot 0: QueryInterface  (inherited)
//     Slot 1: AddRef          (inherited)
//     Slot 2: Release         (inherited)
//     Slot 3: MyMethod1       (new)
//     Slot 4: MyMethod2       (new)
```

Every design decision in COM serves the goal of ABI stability. The table below shows why each choice was made:

| COM Design Choice        | Why It Ensures ABI Stability                        |
| --- | --- |
| Vtable-based dispatch    | Binary layout identical across all compilers         |
| No data members in interfaces | sizeof(interface) is always sizeof(void*)       |
| HRESULT return type      | Uniform error handling, no exceptions across boundary|
| GUID-based identity      | No name mangling dependency                          |
| Reference counting       | Language-neutral lifetime management                 |
| `__stdcall` convention   | Calling convention locked down                       |
| No inheritance of implementation | Only interface (vtable) inheritance          |

COM's apartment threading model is another critical ABI aspect. Objects declare their threading requirements (STA: Single-Threaded Apartment, MTA: Multi-Threaded Apartment), and COM automatically marshals calls across apartment boundaries. This means an STA object can be safely used from any thread - the COM runtime handles synchronization transparently via hidden message loops and proxy/stub pairs.

---

## Self-Assessment

### Q1: Implement a minimal COM object from scratch without ATL to understand the exact binary layout requirements

Building a COM object without ATL forces you to see every piece of the machinery. The vtable is just the C++ virtual dispatch table, the reference count is an atomic integer, and `QueryInterface` is a manual type-safe dispatch based on GUIDs. Here is the full minimal implementation:

```cpp
#ifdef _WIN32
#include <windows.h>
#include <unknwn.h>
#include <cstdio>
#include <atomic>

// {E3A1B2C4-5D6F-7890-ABCD-EF0123456789}
static const GUID IID_ICalculator =
    {0xE3A1B2C4, 0x5D6F, 0x7890,
     {0xAB, 0xCD, 0xEF, 0x01, 0x23, 0x45, 0x67, 0x89}};

// COM interface declaration - pure abstract class with virtual methods
// Uses __stdcall calling convention (STDMETHODCALLTYPE)
class __declspec(novtable) ICalculator : public IUnknown {
public:
    virtual HRESULT STDMETHODCALLTYPE Add(
        double a, double b, double* result) = 0;
    virtual HRESULT STDMETHODCALLTYPE Multiply(
        double a, double b, double* result) = 0;
    virtual HRESULT STDMETHODCALLTYPE GetLastResult(
        double* result) = 0;
};

// Vtable layout of ICalculator:
//   [0] QueryInterface   (from IUnknown)
//   [1] AddRef           (from IUnknown)
//   [2] Release          (from IUnknown)
//   [3] Add              (from ICalculator)
//   [4] Multiply         (from ICalculator)
//   [5] GetLastResult    (from ICalculator)
//
// This layout is IDENTICAL regardless of which compiler builds it.
// GCC, Clang, MSVC, Delphi, even hand-written assembly - all produce
// the same vtable structure.


// === Implementation ===
class Calculator final : public ICalculator {
public:
    Calculator() : ref_count_(1), last_result_(0.0) {
        std::printf("Calculator created\n");
    }

    ~Calculator() {
        std::printf("Calculator destroyed\n");
    }

    // IUnknown implementation
    HRESULT STDMETHODCALLTYPE QueryInterface(
            REFIID riid, void** ppv) override {
        if (!ppv) return E_POINTER;

        if (riid == IID_IUnknown) {
            *ppv = static_cast<IUnknown*>(this);
        } else if (riid == IID_ICalculator) {
            *ppv = static_cast<ICalculator*>(this);
        } else {
            *ppv = nullptr;
            return E_NOINTERFACE;
        }

        AddRef();
        return S_OK;
    }

    ULONG STDMETHODCALLTYPE AddRef() override {
        return ref_count_.fetch_add(1, std::memory_order_relaxed) + 1;
    }

    ULONG STDMETHODCALLTYPE Release() override {
        ULONG count = ref_count_.fetch_sub(1, std::memory_order_acq_rel) - 1;
        if (count == 0) {
            delete this;
        }
        return count;
    }

    // ICalculator implementation
    HRESULT STDMETHODCALLTYPE Add(
            double a, double b, double* result) override {
        if (!result) return E_POINTER;
        last_result_ = a + b;
        *result = last_result_;
        return S_OK;
    }

    HRESULT STDMETHODCALLTYPE Multiply(
            double a, double b, double* result) override {
        if (!result) return E_POINTER;
        last_result_ = a * b;
        *result = last_result_;
        return S_OK;
    }

    HRESULT STDMETHODCALLTYPE GetLastResult(double* result) override {
        if (!result) return E_POINTER;
        *result = last_result_;
        return S_OK;
    }

private:
    std::atomic<ULONG> ref_count_;
    double last_result_;
};

// Factory function - exported from DLL
extern "C" HRESULT CreateCalculator(ICalculator** ppCalc) {
    if (!ppCalc) return E_POINTER;
    *ppCalc = new (std::nothrow) Calculator();
    return *ppCalc ? S_OK : E_OUTOFMEMORY;
}

int main() {
    ICalculator* calc = nullptr;
    HRESULT hr = CreateCalculator(&calc);
    if (FAILED(hr)) return 1;

    double result = 0;
    calc->Add(3.0, 4.0, &result);
    std::printf("3 + 4 = %g\n", result);

    calc->Multiply(5.0, 6.0, &result);
    std::printf("5 * 6 = %g\n", result);

    calc->Release();
    return 0;
}
#endif
```

Notice that `QueryInterface` always calls `AddRef` before returning a pointer. Every out-parameter that receives a COM pointer arrives with a reference count of 1 already on it. Forgetting this is the single most common source of COM memory leaks in hand-written code.

### Q2: Implement multiple COM interfaces on a single object with correct QueryInterface navigation and reference counting

When a single object implements multiple COM interfaces, the COM identity rule becomes critical: `QueryInterface(IID_IUnknown)` called through any interface pointer must always return the exact same pointer value. This is how COM determines whether two interface pointers refer to the same object. If the returned `IUnknown*` differs depending on which interface you called through, you have violated the identity rule and COM infrastructure - including containers, marshaling, and aggregation - will break.

```cpp
#ifdef _WIN32
#include <windows.h>
#include <cstdio>
#include <atomic>
#include <cstring>
#include <cwchar>

// Interface GUIDs
// {A1B2C3D4-E5F6-7890-1234-567890ABCDEF}
static const GUID IID_IPersistable =
    {0xA1B2C3D4, 0xE5F6, 0x7890,
     {0x12, 0x34, 0x56, 0x78, 0x90, 0xAB, 0xCD, 0xEF}};

// {B2C3D4E5-F6A7-8901-2345-6789ABCDEF01}
static const GUID IID_ISerializable =
    {0xB2C3D4E5, 0xF6A7, 0x8901,
     {0x23, 0x45, 0x67, 0x89, 0xAB, 0xCD, 0xEF, 0x01}};

class IPersistable : public IUnknown {
public:
    virtual HRESULT STDMETHODCALLTYPE Save(const wchar_t* path) = 0;
    virtual HRESULT STDMETHODCALLTYPE Load(const wchar_t* path) = 0;
};

class ISerializable : public IUnknown {
public:
    virtual HRESULT STDMETHODCALLTYPE Serialize(
        unsigned char* buf, int capacity, int* written) = 0;
    virtual HRESULT STDMETHODCALLTYPE Deserialize(
        const unsigned char* buf, int length) = 0;
};

// Document implements BOTH interfaces
// CRITICAL: all paths through QueryInterface must return
// the SAME IUnknown pointer - the COM identity rule
class Document final : public IPersistable, public ISerializable {
public:
    Document() : ref_count_(1) {}

    // IUnknown - handles multiple interface identity correctly
    HRESULT STDMETHODCALLTYPE QueryInterface(
            REFIID riid, void** ppv) override {
        if (!ppv) return E_POINTER;

        if (riid == IID_IUnknown) {
            // CRITICAL: Always return the SAME IUnknown*
            // Convention: use the first listed base class
            *ppv = static_cast<IPersistable*>(this);
        } else if (riid == IID_IPersistable) {
            *ppv = static_cast<IPersistable*>(this);
        } else if (riid == IID_ISerializable) {
            *ppv = static_cast<ISerializable*>(this);
        } else {
            *ppv = nullptr;
            return E_NOINTERFACE;
        }

        AddRef();
        return S_OK;
    }

    ULONG STDMETHODCALLTYPE AddRef() override {
        return ref_count_.fetch_add(1, std::memory_order_relaxed) + 1;
    }

    ULONG STDMETHODCALLTYPE Release() override {
        ULONG count = ref_count_.fetch_sub(1, std::memory_order_acq_rel) - 1;
        if (count == 0) delete this;
        return count;
    }

    // IPersistable
    HRESULT STDMETHODCALLTYPE Save(const wchar_t* path) override {
        std::wprintf(L"Saving to: %s\n", path);
        return S_OK;
    }

    HRESULT STDMETHODCALLTYPE Load(const wchar_t* path) override {
        std::wprintf(L"Loading from: %s\n", path);
        return S_OK;
    }

    // ISerializable
    HRESULT STDMETHODCALLTYPE Serialize(
            unsigned char* buf, int capacity, int* written) override {
        if (!buf || !written) return E_POINTER;
        const char* data = "DOC_DATA";
        int len = 8;
        if (capacity < len) return HRESULT_FROM_WIN32(ERROR_INSUFFICIENT_BUFFER);
        std::memcpy(buf, data, len);
        *written = len;
        return S_OK;
    }

    HRESULT STDMETHODCALLTYPE Deserialize(
            const unsigned char* buf, int length) override {
        std::printf("Deserialized %d bytes\n", length);
        return S_OK;
    }

private:
    std::atomic<ULONG> ref_count_;
};


// === COM Identity Rule Verification ===
void verify_identity() {
    Document* doc = new Document();

    IUnknown* unk1 = nullptr;
    IUnknown* unk2 = nullptr;

    // Get IUnknown through different paths
    IPersistable* persist = nullptr;
    doc->QueryInterface(IID_IPersistable, reinterpret_cast<void**>(&persist));
    persist->QueryInterface(IID_IUnknown, reinterpret_cast<void**>(&unk1));

    ISerializable* serial = nullptr;
    doc->QueryInterface(IID_ISerializable, reinterpret_cast<void**>(&serial));
    serial->QueryInterface(IID_IUnknown, reinterpret_cast<void**>(&unk2));

    // COM IDENTITY RULE: both must be the same pointer
    std::printf("Identity check: %s\n",
                (unk1 == unk2) ? "PASS" : "FAIL");

    unk1->Release();
    unk2->Release();
    serial->Release();
    persist->Release();
    doc->Release();
}

int main() {
    verify_identity();
    return 0;
}
#endif
```

The reason the identity check uses `static_cast<IPersistable*>(this)` for `IID_IUnknown` rather than `static_cast<IUnknown*>(this)` is subtle. With multiple inheritance, `static_cast<IUnknown*>(this)` would be ambiguous - both `IPersistable` and `ISerializable` inherit from `IUnknown`, so the compiler cannot resolve which `IUnknown` subobject you mean. By always casting to the first base class (`IPersistable`) and letting that implicitly convert to `IUnknown`, you guarantee a single, consistent address.

### Q3: Understand COM apartment threading models and their implications for cross-apartment calls and marshaling

The apartment model is one of the more counterintuitive parts of COM. The key insight is that COM objects are not just code - they carry a threading contract. An STA object promises "I will only be called on the thread that created me." An MTA object promises "I am fully thread-safe." COM enforces these contracts automatically by inserting proxy/stub pairs between apartments that marshal calls through a hidden message loop.

```cpp
#ifdef _WIN32
#include <windows.h>
#include <objbase.h>
#include <cstdio>

// COM Threading Models:
//
// | Model      | Annotation           | Thread Access        | Marshaling          |
// |------------|----------------------|----------------------|---------------------|
// | STA        | ThreadingModel=Apartment | Single thread only | Auto via proxy/stub |
// | MTA        | ThreadingModel=Free     | Any thread in MTA  | No marshal in MTA   |
// | Both       | ThreadingModel=Both     | Any apartment       | Optimized           |
// | Neutral    | ThreadingModel=Neutral  | Any thread, no switch| Minimal overhead  |
//
// Key insights:
// - STA has a message loop; COM marshals calls through messages
// - MTA objects must be fully thread-safe
// - Cross-apartment calls go through proxy/stub -> significant overhead
// - "Both" means the object adapts to the caller's apartment

// === Demonstrating apartment initialization ===

void demonstrate_sta() {
    // Initialize COM for Single-Threaded Apartment
    HRESULT hr = CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED);
    if (FAILED(hr)) {
        std::printf("STA init failed: 0x%08lx\n", hr);
        return;
    }

    std::printf("STA initialized on thread %lu\n", GetCurrentThreadId());

    // Objects created here live in this STA
    // Only this thread can directly call their methods
    // Other threads get a proxy that marshals calls back to this thread

    // STA requires a message loop for marshaling to work!
    // In a real app: while(GetMessage(&msg,...)) { ... DispatchMessage(&msg); }

    CoUninitialize();
}

void demonstrate_mta() {
    // Initialize COM for Multi-Threaded Apartment
    HRESULT hr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
    if (FAILED(hr)) {
        std::printf("MTA init failed: 0x%08lx\n", hr);
        return;
    }

    std::printf("MTA initialized on thread %lu\n", GetCurrentThreadId());

    // Objects created here live in the MTA
    // Any MTA thread can call them directly (must be thread-safe!)
    // STA threads get proxies

    CoUninitialize();
}


// === Smart pointer pattern for COM (similar to CComPtr/ComPtr) ===
template <typename T>
class ComPtr {
public:
    ComPtr() : ptr_(nullptr) {}
    ~ComPtr() { release(); }

    // No copy - prevent double-Release
    ComPtr(const ComPtr&) = delete;
    ComPtr& operator=(const ComPtr&) = delete;

    // Move support
    ComPtr(ComPtr&& other) noexcept : ptr_(other.ptr_) {
        other.ptr_ = nullptr;
    }
    ComPtr& operator=(ComPtr&& other) noexcept {
        if (this != &other) {
            release();
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    T* get() const { return ptr_; }
    T** put() { release(); return &ptr_; }  // For out-params
    T* operator->() const { return ptr_; }
    explicit operator bool() const { return ptr_ != nullptr; }

    // QueryInterface helper
    template <typename U>
    HRESULT as(ComPtr<U>& other) const {
        return ptr_->QueryInterface(__uuidof(U),
                                    reinterpret_cast<void**>(other.put()));
    }

    void release() {
        if (ptr_) {
            ptr_->Release();
            ptr_ = nullptr;
        }
    }

    // Prevent leaks: detach without Release
    T* detach() {
        T* tmp = ptr_;
        ptr_ = nullptr;
        return tmp;
    }

private:
    T* ptr_;
};


// === Cross-apartment penalty demonstration ===
//
// Scenario: STA object called from MTA thread
//
// Direct call cost:    ~10 nanoseconds (vtable dispatch)
// Cross-apt call cost: ~10,000 nanoseconds (marshal -> message -> unmarshal)
//
// This 1000x overhead is why threading model choice matters!
//
// Call flow for cross-apartment:
//
// MTA Thread                    STA Thread
//    |                              |
//    |-- proxy->Method() ------->   |
//    |   (serialized params)        |
//    |                     PostMessage to STA window
//    |                              |
//    |                     <---- dispatch Method()
//    |                              |
//    |                     ---- return HRESULT ---->
//    |<-- unmarshal result ------   |
//    |                              |
//
// The proxy/stub pair handles parameter marshaling
// using NDR (Network Data Representation) - the same
// format used for DCOM across machines

int main() {
    demonstrate_sta();

    // Re-init as MTA for demonstration
    demonstrate_mta();

    std::printf("COM apartment demo complete\n");
    return 0;
}
#endif
```

The 1000x overhead number is worth sitting with. A direct virtual call takes about 10 nanoseconds. A cross-apartment call - where COM has to serialize parameters, post a message to the STA thread's hidden window, wait for that thread to pump its message loop, execute the method, and marshal the return value back - takes around 10 microseconds. If you are calling an STA object in a tight loop from a background thread, you will see it. The fix is either to design interfaces that batch work (minimize round-trips), or to ensure the object lives in the right apartment for your threading model.

---

## Notes

- COM achieves cross-compiler binary compatibility by reducing the ABI contract to a C-compatible vtable layout with `__stdcall` calling convention - no name mangling, no C++ exceptions, no templates.
- The **COM Identity Rule** states that `QueryInterface(IID_IUnknown)` must always return the same pointer - this is how COM determines object identity, and violating it breaks containers and COM infrastructure.
- Reference counting in COM is per-interface in the general case, though most implementations use a single atomic counter for the entire object.
- STA objects do not need to be thread-safe - COM serializes all calls through a hidden window message loop. But this means you must pump messages or calls from other threads will deadlock.
- Cross-apartment calls have roughly 1000x overhead compared to direct calls due to marshaling - design interfaces to minimize round-trips.
- Use `ComPtr<T>` (from `<wrl/client.h>` or your own) for RAII management of COM pointers - manual `AddRef`/`Release` is the number one source of COM memory leaks.
- COM is essentially the first successful implementation of the Interface/Factory/Query pattern that later appeared in CORBA, Java RMI, D-Bus, and even modern Rust trait objects.
