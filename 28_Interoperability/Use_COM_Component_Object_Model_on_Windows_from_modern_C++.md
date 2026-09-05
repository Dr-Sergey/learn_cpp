# Use COM (Component Object Model) on Windows from modern C++

**Category:** Interoperability  
**Standard:** C++11  
**Reference:** <https://docs.microsoft.com/en-us/windows/win32/com/>  

---

## Topic Overview

This topic focuses on **practical modern COM usage** in C++ - using smart pointers (`WRL::ComPtr`, `wil::com_ptr`), RAII initialization, and safe interface querying. COM is still essential for DirectX, Shell APIs, Media Foundation, and WinRT. If you have ever worked with DirectX or the Windows Shell APIs, you have already been using COM whether you knew it or not.

The core idea behind COM is that every object implements at least the `IUnknown` interface, which gives you `AddRef`, `Release`, and `QueryInterface`. The first two manage the object's lifetime via reference counting. The third lets you ask whether the object also implements some other interface. The problem with raw COM is that forgetting to call `Release` leaks the object permanently - RAII wrappers like `ComPtr` exist exactly to prevent that.

### COM Smart Pointer Comparison

The table below shows your three options. Raw pointers are included for completeness, but you should almost never use them in new code.

| Feature | Raw `IUnknown*` | `WRL::ComPtr` | `wil::com_ptr` |
| --- | :---: | :---: | :---: |
| Auto Release | No | Yes | Yes |
| Header | N/A | `<wrl/client.h>` | `<wil/com.h>` |
| QueryInterface | Manual | `.As<T>()` | `.query<T>()` |
| Error handling | Manual HRESULT | Manual HRESULT | Throws on failure |
| Null check | Manual | `.Get() != nullptr` | `.get() != nullptr` |
| Part of SDK | - | Yes (Windows SDK) | No (NuGet: `wil`) |

### COM Initialization Pattern

Every thread that uses COM must call `CoInitializeEx` before creating any COM objects, and `CoUninitialize` when it's done. Forgetting the `CoUninitialize` call won't crash you immediately, but it can cause resource leaks and subtle instability. The standard fix is an RAII wrapper that ties the lifetime of COM initialization to a scope.

```cpp
Program Start
    │
    ▼
CoInitializeEx(NULL, COINIT_APARTMENTTHREADED)
    │
    ▼
┌──────────────────────┐
│  Use COM objects     │ <- CoCreateInstance / QueryInterface
│  ComPtr handles      │
│  AddRef/Release      │
└──────────────────────┘
    │
    ▼
CoUninitialize()  <- MUST match every CoInitializeEx
    │
    ▼
Program End
```

---

## Self-Assessment

### Q1: Initialize COM with CoInitializeEx and wrap IUnknown in a wil::com_ptr for RAII

**Answer:**

The `ComInitializer` struct below is the standard RAII approach. When it goes out of scope, the destructor automatically calls `CoUninitialize` if initialization succeeded - so you can never forget it. After that, notice how `ComPtr<IShellItem>` handles the `Release` call automatically when the function returns.

```cpp
#include <windows.h>
#include <wrl/client.h>
#include <shobjidl.h>
#include <cstdio>

using Microsoft::WRL::ComPtr;

// ═══════════ RAII COM initializer ═══════════
struct ComInitializer {
    HRESULT hr;
    ComInitializer(DWORD model = COINIT_APARTMENTTHREADED) {
        hr = CoInitializeEx(nullptr, model);
    }
    ~ComInitializer() {
        if (SUCCEEDED(hr)) CoUninitialize();
    }
    bool ok() const { return SUCCEEDED(hr); }

    // Non-copyable
    ComInitializer(const ComInitializer&) = delete;
    ComInitializer& operator=(const ComInitializer&) = delete;
};

int main() {
    // RAII: COM initialized/uninitialized automatically
    ComInitializer com;
    if (!com.ok()) {
        printf("COM init failed: 0x%lx\n", com.hr);
        return 1;
    }

    // Create a Shell item from a path
    ComPtr<IShellItem> item;
    HRESULT hr = SHCreateItemFromParsingName(
        L"C:\\Windows\\System32\\notepad.exe",
        nullptr,
        IID_PPV_ARGS(&item)   // IID_PPV_ARGS extracts IID + **ptr
    );

    if (SUCCEEDED(hr)) {
        PWSTR name = nullptr;
        hr = item->GetDisplayName(SIGDN_NORMALDISPLAY, &name);
        if (SUCCEEDED(hr)) {
            printf("Item: %ls\n", name);
            CoTaskMemFree(name);  // COM strings allocated with CoTaskMemAlloc
        }
    }

    // item automatically Release'd when ComPtr destroyed
    // COM automatically uninitialized when ComInitializer destroyed
    return 0;
}
// Output: Item: notepad.exe
```

Notice `IID_PPV_ARGS(&item)` - this macro extracts the interface ID from the ComPtr's template type parameter and passes both the IID and the address to receive the pointer. It prevents a common bug where you pass the wrong IID for the pointer type.

### Q2: Query an interface with QueryInterface and handle E_NOINTERFACE gracefully

**Answer:**

`QueryInterface` is how COM lets a single object implement multiple interfaces. You ask "does this object also implement `IFileDialog2`?" and it either hands back a pointer or returns `E_NOINTERFACE`. The `ComPtr::As` method is the clean way to do this - it calls `QueryInterface` under the hood and stores the result.

```cpp
#include <windows.h>
#include <wrl/client.h>
#include <shobjidl.h>
#include <shlobj.h>
#include <cstdio>

using Microsoft::WRL::ComPtr;

void query_interface_demo() {
    // Create a FileOpenDialog COM object
    ComPtr<IFileOpenDialog> dialog;
    HRESULT hr = CoCreateInstance(
        CLSID_FileOpenDialog,
        nullptr,
        CLSCTX_INPROC_SERVER,
        IID_PPV_ARGS(&dialog)
    );
    if (FAILED(hr)) return;

    // ═══════════ Method 1: ComPtr::As (recommended) ═══════════
    ComPtr<IFileDialog2> dialog2;
    hr = dialog.As(&dialog2);  // QueryInterface wrapper

    if (hr == E_NOINTERFACE) {
        printf("IFileDialog2 not supported on this OS\n");
    } else if (SUCCEEDED(hr)) {
        printf("IFileDialog2 available\n");
        // dialog2 can be used — auto Release on scope exit
    }

    // ═══════════ Method 2: Raw QueryInterface ═══════════
    ComPtr<IUnknown> unknown;
    hr = dialog->QueryInterface(IID_PPV_ARGS(&unknown));
    // This always succeeds — every COM object supports IUnknown

    // ═══════════ Method 3: Query for unrelated interface (will fail) ═══════════
    ComPtr<IShellItem> shell_item;
    hr = dialog.As(&shell_item);
    if (hr == E_NOINTERFACE) {
        printf("FileOpenDialog doesn't implement IShellItem (expected)\n");
    }

    // ═══════════ Safe pattern: attempt and use if available ═══════════
    ComPtr<IFileDialogCustomize> customizer;
    if (SUCCEEDED(dialog.As(&customizer))) {
        // Add a checkbox to the dialog
        customizer->AddCheckButton(100, L"Open as read-only", FALSE);
        customizer->AddComboBox(101);
        customizer->AddControlItem(101, 0, L"Option A");
        customizer->AddControlItem(101, 1, L"Option B");
    }
    // If QI failed, customizer is nullptr — RAII, no leak
}

// ═══════════ HRESULT error handling helper ═══════════
void check_hr(HRESULT hr, const char* context) {
    if (SUCCEEDED(hr)) return;

    switch (hr) {
        case E_NOINTERFACE:
            printf("%s: interface not supported\n", context);
            break;
        case E_OUTOFMEMORY:
            printf("%s: out of memory\n", context);
            break;
        case E_INVALIDARG:
            printf("%s: invalid argument\n", context);
            break;
        case REGDB_E_CLASSNOTREG:
            printf("%s: COM class not registered\n", context);
            break;
        default:
            printf("%s: failed with 0x%08lx\n", context, hr);
            break;
    }
}
```

The "safe pattern" at the bottom is the one you'll use most often: try the `QueryInterface`, and if it succeeds, use the richer interface - if it fails, just skip the feature. The `ComPtr` handles cleanup in both branches so there's no way to leak.

### Q3: Use WRL::ComPtr or wil::com_ptr instead of raw IUnknown pointers for automatic Release

**Answer:**

This example deliberately shows both the raw-pointer version and the `ComPtr` version side by side for the same Direct2D code. The raw version looks manageable at first, but notice how every early-return error path requires a manual `Release` call - miss one and you have a permanent leak.

```cpp
#include <windows.h>
#include <wrl/client.h>
#include <d2d1.h>
#include <cstdio>
#pragma comment(lib, "d2d1.lib")

using Microsoft::WRL::ComPtr;

// BAD: Raw pointers — easy to leak
void raw_pointer_danger() {
    ID2D1Factory* factory = nullptr;
    HRESULT hr = D2D1CreateFactory(
        D2D1_FACTORY_TYPE_SINGLE_THREADED,
        &factory
    );
    if (FAILED(hr)) return;

    ID2D1HwndRenderTarget* target = nullptr;
    D2D1_RENDER_TARGET_PROPERTIES rtProps = D2D1::RenderTargetProperties();
    D2D1_HWND_RENDER_TARGET_PROPERTIES hwndProps = {};

    hr = factory->CreateHwndRenderTarget(rtProps, hwndProps, &target);
    if (FAILED(hr)) {
        factory->Release();  // Must remember to Release!
        return;              // If you forget -> LEAK
    }

    // Use target...
    target->Release();      // Must Release in correct order
    factory->Release();     // Miss one -> LEAK
}

// GOOD: ComPtr — automatic, exception-safe
void comptr_safe() {
    ComPtr<ID2D1Factory> factory;
    HRESULT hr = D2D1CreateFactory(
        D2D1_FACTORY_TYPE_SINGLE_THREADED,
        IID_PPV_ARGS(&factory)
    );
    if (FAILED(hr)) return;

    ComPtr<ID2D1HwndRenderTarget> target;
    D2D1_RENDER_TARGET_PROPERTIES rtProps = D2D1::RenderTargetProperties();
    D2D1_HWND_RENDER_TARGET_PROPERTIES hwndProps = {};

    hr = factory->CreateHwndRenderTarget(
        rtProps, hwndProps, target.GetAddressOf()
    );
    if (FAILED(hr)) return;  // factory auto-Released, no leak

    // Use target...
    // Both auto-Released when function exits — even if exception thrown
}

// ═══════════ ComPtr operations cheat sheet ═══════════
void comptr_operations() {
    ComPtr<ID2D1Factory> ptr;

    // Create
    D2D1CreateFactory(D2D1_FACTORY_TYPE_SINGLE_THREADED, IID_PPV_ARGS(&ptr));

    // Copy (AddRef)
    ComPtr<ID2D1Factory> copy = ptr;

    // Move (no AddRef/Release)
    ComPtr<ID2D1Factory> moved = std::move(ptr);
    // ptr is now nullptr

    // Query interface
    ComPtr<IUnknown> unk;
    moved.As(&unk);  // QI + AddRef on new interface

    // Get raw pointer (no AddRef — don't Release it!)
    ID2D1Factory* raw = moved.Get();

    // Get address for output params (resets first!)
    ComPtr<ID2D1Factory> out;
    // SomeFunc(out.GetAddressOf());  // Releases old, stores new

    // Detach (takes ownership — YOU must Release)
    ID2D1Factory* detached = moved.Detach();
    detached->Release();  // Your responsibility

    // Reset (Release + set to nullptr)
    copy.Reset();

    // Swap
    ComPtr<ID2D1Factory> a, b;
    a.Swap(b);
}
```

The `comptr_operations` function is a handy reference for all the `ComPtr` operations you'll use regularly. The one that trips people up is `Detach` - after calling it, the ComPtr no longer manages the object, so you own the `Release` call yourself.

---

## Notes

- `IID_PPV_ARGS(&ptr)` extracts the IID from the ComPtr's template type, preventing IID/type mismatches that are otherwise a silent source of bugs.
- `wil::com_ptr` (Windows Implementation Libraries) throws `wil::ResultException` on failure instead of returning HRESULT - more C++-idiomatic for code that uses exceptions.
- Never call `Release()` on a pointer obtained via `ComPtr::Get()` - the ComPtr still owns it and will double-release.
- `ComPtr::ReleaseAndGetAddressOf()` resets and returns `**` - use this for output parameters, not `GetAddressOf()`, when you're replacing an existing object.
- COM objects can live in DLLs or EXEs (out-of-process servers) - ComPtr handles both transparently.
- For WinRT, prefer `winrt::com_ptr` from C++/WinRT over WRL.
