# Know MSVC ABI and Class Layout with /d1reportSingleClassLayout

**Category:** ABI and Binary Compatibility  
**Standard:** C++17, C++20 (MSVC-specific ABI)  
**Reference:** https://learn.microsoft.com/en-us/cpp/build/reference/compiler-options-listed-by-category  

---

## Topic Overview

The MSVC ABI differs significantly from the Itanium ABI in vtable layout, name mangling, exception handling, and class memory layout. Unlike Itanium, which uses a single vtable group with thunks, MSVC uses **separate vfptr members** for each base class that introduces virtual functions. Understanding these differences is essential when maintaining cross-platform libraries, debugging memory corruption, or interfacing with COM objects.

The reason this trips people up is that MSVC and Itanium/GCC can produce binaries from identical C++ source that have different `sizeof`, different member offsets, and different vtable structures. If you are writing a library that must work with both compilers - or debugging a crash that only happens on one - you need to be able to reason about each ABI separately.

The undocumented `/d1reportSingleClassLayout<ClassName>` compiler switch is an invaluable diagnostic tool. It prints the complete memory layout of a class including padding, vtable pointer placement, virtual base table pointers (vbptr), and member offsets. Use it via: `cl /d1reportSingleClassLayoutMyClass /c myfile.cpp`. Visual Studio also supports this through the "Class Layout" window in newer versions.

Here is a side-by-side comparison of the two ABIs on the points that matter most for cross-platform work:

| Feature                  | Itanium ABI             | MSVC ABI                          |
| --- | --- | --- |
| Vtable structure         | Single vtable group     | Separate vfptr per base           |
| Virtual base handling    | VTT (Virtual Table Table)| vbptr (virtual base pointer)     |
| offset-to-top           | In vtable at [-2]       | Not used; vbptr table instead     |
| Name mangling            | `_Z` prefix scheme      | `?` prefix scheme                 |
| RTTI location            | vtable[-1]              | First slot of vftable (COL)       |
| Exception handling       | DWARF/.eh_frame         | SEH-based with FuncInfo           |
| Thunks                   | this-adjusting thunks   | adjustor thunks + separate tables |

MSVC places the **Complete Object Locator (COL)** as the first hidden entry (index -1) in each vftable. The COL contains offsets needed for `dynamic_cast`: the offset of the vftable within the complete object, the offset of the complete object from the vftable pointer, and a pointer to the type descriptor and class hierarchy descriptor. This structure is fundamentally different from Itanium's typeinfo pointer approach.

Member layout in MSVC follows specific alignment rules with `#pragma pack` support. Empty base class optimization (EBCO) behaves differently: MSVC historically did not apply EBCO when the empty base is also the first member's type, though `__declspec(empty_bases)` was added later to opt in. These layout differences mean that `sizeof(T)` can differ between compilers for the same class definition.

---

## Self-Assessment

### Q1: Use `/d1reportSingleClassLayout` to analyze a class with virtual and multiple inheritance, and predict member offsets

Let's look at a class with three virtual base classes and see exactly how MSVC lays it out in memory. The important thing to watch is that MSVC inserts a separate vfptr for each base class that brings in its own virtual functions - this is the opposite of Itanium's approach of putting everything in one vtable group:

```cpp
// Compile: cl /d1reportSingleClassLayoutWidget /c widget.cpp
// This prints the full layout to the build output

#include <cstdio>
#include <cstddef>

struct Drawable {
    virtual void draw() = 0;
    virtual ~Drawable() = default;
    int color = 0xFF0000;
};

struct Resizable {
    virtual void resize(int w, int h) = 0;
    virtual ~Resizable() = default;
    int width = 100, height = 50;
};

struct EventHandler {
    virtual void on_click() {}
    virtual void on_hover() {}
    int handler_id = 0;
};

struct Widget : Drawable, Resizable, EventHandler {
    void draw() override { std::printf("draw\n"); }
    void resize(int w, int h) override { width = w; height = h; }
    void on_click() override { std::printf("click\n"); }
    int widget_id = 42;
};

// Expected MSVC layout (64-bit):
//
// class Widget     size(72):
//     +---
//  0  | +--- (base class Drawable)
//  0  | | {vfptr}            <-- Drawable's vftable
//  8  | | color
//     | | <padding>(4)
//     | +---
// 16  | +--- (base class Resizable)
// 16  | | {vfptr}            <-- Resizable's vftable
// 24  | | width
// 28  | | height
//     | +---
// 32  | +--- (base class EventHandler)
// 32  | | {vfptr}            <-- EventHandler's vftable
// 40  | | handler_id
//     | | <padding>(4)
//     | +---
// 48  | widget_id
//     | <padding>(4)
//     +---
//
// Widget::$vftable@Drawable@:
//   | &Widget::draw
//   | &Widget::{dtor}
//
// Widget::$vftable@Resizable@:
//   | &Widget::resize
//   | &thunk Widget::{dtor}    (adjustor thunk, this-=16)
//
// Widget::$vftable@EventHandler@:
//   | &Widget::on_click
//   | &EventHandler::on_hover
//   | &thunk Widget::{dtor}    (adjustor thunk, this-=32)

void verify_layout() {
    Widget w;
    std::printf("sizeof(Widget) = %zu\n", sizeof(Widget));

    // Verify base subobject offsets using static_cast
    auto widget_addr = reinterpret_cast<char*>(&w);
    auto drawable_addr = reinterpret_cast<char*>(static_cast<Drawable*>(&w));
    auto resizable_addr = reinterpret_cast<char*>(static_cast<Resizable*>(&w));
    auto handler_addr = reinterpret_cast<char*>(static_cast<EventHandler*>(&w));

    std::printf("Drawable   offset: %td\n", drawable_addr - widget_addr);
    std::printf("Resizable  offset: %td\n", resizable_addr - widget_addr);
    std::printf("EventHandler offset: %td\n", handler_addr - widget_addr);

    // Demonstrate adjustor thunk behavior
    Resizable* rp = &w;
    rp->resize(200, 300);     // this pointer is adjusted before entering Widget::resize
    std::printf("After resize: %d x %d\n", w.width, w.height);
}

int main() {
    verify_layout();
    return 0;
}
```

Notice the three separate vfptr entries at offsets 0, 16, and 32. Each base class that introduces virtuals gets its own vtable pointer. The "adjustor thunks" in the Resizable and EventHandler vtables are small generated stubs that subtract the base-subobject offset from `this` before jumping to the real `Widget` method, because the callee expects a `Widget*` at offset 0 but the vfptr for Resizable points to offset 16.

### Q2: Analyze vbptr layout differences between MSVC and Itanium for virtual inheritance

Virtual inheritance (the diamond pattern) is where MSVC and Itanium diverge most visibly. Itanium stores virtual-base offsets inside the vtable itself. MSVC uses a completely separate mechanism - the **vbptr** (virtual base pointer), which points to a small table of offsets called the vbtable. This costs an extra pointer per virtualizing base class:

```cpp
#include <cstdio>
#include <cstddef>

struct Base {
    virtual void identify() { std::printf("Base\n"); }
    int base_val = 10;
};

struct Left : virtual Base {
    void identify() override { std::printf("Left\n"); }
    int left_val = 20;
};

struct Right : virtual Base {
    void identify() override { std::printf("Right\n"); }
    int right_val = 30;
};

struct Diamond : Left, Right {
    void identify() override { std::printf("Diamond\n"); }
    int diamond_val = 40;
};

// MSVC layout with /d1reportSingleClassLayoutDiamond (64-bit):
//
// class Diamond    size(56):
//     +---
//  0  | +--- (base class Left)
//  0  | | {vfptr}         <-- Left's vftable (with Diamond overrides)
//  8  | | {vbptr}         <-- Virtual base table pointer
// 16  | | left_val
//     | +---
// 24  | +--- (base class Right)
// 24  | | {vfptr}         <-- Right's vftable
// 32  | | {vbptr}         <-- Virtual base table pointer
// 40  | | right_val
//     | +---
// 44  | diamond_val
//     +---
//     +--- (virtual base Base)
// 48  | base_val
//     +---
//
// vbtable for Left:
//   [0] = -8   (offset from vbptr to Left subobject start)
//   [1] = 40   (offset from vbptr to virtual Base)
//
// vbtable for Right:
//   [0] = -8   (offset from vbptr to Right subobject start)
//   [1] = 16   (offset from vbptr to virtual Base)

void demonstrate_vbptr() {
    Diamond d;
    std::printf("sizeof(Diamond) = %zu\n", sizeof(Diamond));

    // On MSVC, the vbptr is a separate pointer (not part of vtable)
    // On Itanium, virtual base offsets are stored IN the vtable

    Base* bp = &d;
    bp->identify();  // "Diamond" - resolved through vftable/vbptr chain

    // Verify single Base subobject
    Left* lp = &d;
    Right* rp = &d;
    Base* from_left = lp;
    Base* from_right = rp;
    std::printf("Single Base instance: %s\n",
                from_left == from_right ? "yes" : "no");

    // Offsets
    auto base = reinterpret_cast<char*>(&d);
    std::printf("Left  at +%td\n", reinterpret_cast<char*>(lp) - base);
    std::printf("Right at +%td\n", reinterpret_cast<char*>(rp) - base);
    std::printf("Base  at +%td\n", reinterpret_cast<char*>(from_left) - base);
}

int main() {
    demonstrate_vbptr();
    return 0;
}
```

Each virtualizing base (Left and Right) gets a vbptr at a fixed offset inside its subobject. At runtime, to find the shared `Base`, the code loads the vbptr, then indexes into the vbtable to find the byte offset to the virtual base. This two-level indirection is more expensive than Itanium's single vtable lookup, but it means the vbtable can be shared across all instances of the same class.

### Q3: Understand MSVC name decoration and use `undname` to debug linker errors from ABI mismatches

MSVC's name mangling scheme (sometimes called "decoration") encodes far more information than Itanium's `_Z` scheme - calling convention, cv-qualifiers, return type, and the full reversed namespace path all appear in the decorated name. This makes LNK2019 errors more informative once you know how to read them, and the `undname` tool turns decorated names back into readable C++:

```cpp
// MSVC mangles names differently from Itanium
// Use: undname <decorated_name> to demangle
// Or:  dumpbin /SYMBOLS myobj.obj | findstr "myFunc"

#include <cstdio>

namespace network::v2 {

struct Config {
    int timeout_ms;
    bool use_tls;
};

class [[nodiscard]] Connection {
public:
    // MSVC decoration: ?connect@Connection@v2@network@@QEAA_NAEBUConfig@23@@Z
    // Breakdown:
    //   ? = decorated name prefix
    //   connect = function name
    //   @Connection@v2@network@@ = qualified name (reversed)
    //   Q = public member, E = __ptr64, A = __cdecl, A = non-const
    //   _N = return type bool
    //   AEBUConfig@23@ = const ref to Config in namespace v2::network
    //   @Z = end of parameter list
    bool connect(const Config& cfg) {
        std::printf("Connecting with timeout=%d tls=%d\n",
                    cfg.timeout_ms, cfg.use_tls);
        return true;
    }

    // MSVC decoration: ?send@Connection@v2@network@@QEAA_NPEBDH@Z
    // P = pointer, E = __ptr64, B = const, D = char, H = int
    bool send(const char* data, int length) {
        std::printf("Sending %d bytes\n", length);
        return true;
    }

    // Template methods get encoded type parameters
    template <typename T>
    void configure(T value);

    virtual ~Connection() = default;
};

}  // namespace network::v2

// Common linker error scenario:
// LNK2019: unresolved external symbol
//   "?connect@Connection@v2@network@@QEAA_NAEBUConfig@23@@Z"
//
// Debugging steps:
// 1. cl /Zs /FAcs /c file.cpp  -> produces assembly listing with decorations
// 2. dumpbin /SYMBOLS file.obj  -> shows all exported symbols
// 3. undname "?connect@Connection@v2@network@@QEAA_NAEBUConfig@23@@Z"
//    -> public: bool __cdecl network::v2::Connection::connect(
//                   struct network::v2::Config const &)
//
// Common ABI mismatch causes:
// - Different calling conventions (__cdecl vs __stdcall)
// - Different packing (#pragma pack differences)
// - x86 vs x64 decoration differences
// - /GR- (disable RTTI) changes vtable layout
// - /EHsc vs /EHa changes exception handling tables

int main() {
    network::v2::Config cfg{5000, true};
    network::v2::Connection conn;
    conn.connect(cfg);
    conn.send("hello", 5);
    return 0;
}
```

The decoration scheme encodes calling convention and cv-qualifiers because MSVC supports multiple calling conventions (`__cdecl`, `__stdcall`, `__fastcall`) and a function compiled with one convention is genuinely incompatible at the binary level with a call site expecting another. A mismatch in `/GR-` (disabling RTTI) is particularly sneaky because it silently removes the type descriptor from the vtable, so a TU compiled with RTTI tries to read a pointer that does not exist in the other TU's vtable.

---

## Notes

- MSVC uses separate vfptr members for each polymorphic base class, unlike Itanium's unified vtable group - this means `sizeof` is larger with multiple inheritance on MSVC.
- The vbptr (virtual base pointer) is an MSVC-specific mechanism; Itanium stores virtual base offsets in the vtable itself via the VTT.
- Use `/d1reportSingleClassLayout<Name>` (no space before the class name) as a build flag - it's undocumented but stable across MSVC versions.
- MSVC's Complete Object Locator replaces Itanium's typeinfo-in-vtable scheme and is required for `dynamic_cast` to work across DLL boundaries.
- `__declspec(empty_bases)` is needed on MSVC to get standard-conforming EBCO when an empty base conflicts with the first data member.
- MSVC name mangling encodes calling convention, cv-qualifiers, return type, and full namespace path - making linker errors more informative but harder to read without `undname`.
- The `/d1reportAllClassLayout` flag dumps layouts for all classes in a translation unit - extremely verbose but useful for automated analysis.
