# Understand Module Linkage vs External Linkage in Module Context

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** [cppreference - Modules](https://en.cppreference.com/w/cpp/language/modules)  

---

## Topic Overview

C++20 introduces **module linkage** - a new linkage category that sits between external linkage and internal linkage. Before C++20, entities had three linkage kinds: **external** (visible across all TUs), **internal** (`static` or anonymous namespace, visible only within one TU), and **no linkage** (block-scope variables). Module linkage adds a fourth: entities visible across all translation units **of the same module** but invisible to importers.

Module linkage applies to non-exported declarations with external linkage that appear in the purview of a named module. The key rule: if a declaration in a module unit would normally have external linkage but is **not exported**, it has **module linkage** instead. This applies to functions, variables, classes, enums, and templates.

| Linkage | Visibility | Typical Use |
| --- | --- | --- |
| External | All TUs everywhere | Exported module API, global symbols |
| Module | All TUs of the owning module | Module-internal helpers shared across module units |
| Internal (`static` / anon NS) | Single TU only | TU-private implementation details |
| No linkage | Block scope | Local variables |

Here's a diagram showing how the three tiers look within a single module:

```cpp
+--------------------------------------------------------------+
|  Module "engine" (3 translation units)                       |
|                                                              |
|  +----------------+  +--------------+  +------------------+ |
|  | engine.cppm    |  | engine_a.cpp |  | engine_b.cpp     | |
|  | (interface)    |  | (impl unit)  |  | (impl unit)      | |
|  |                |  |              |  |                   | |
|  | export f()  ---+--+--------------+--+--> external link  | |
|  | helper()    ---+--+--------------+--+--> module link    | |
|  | static g() ---+--+-X           |  |   internal link    | |
|  +----------------+  +--------------+  +------------------+ |
|                                                              |
|  external: visible everywhere (importers can see f())        |
|  module:   visible within engine's TUs only (helper())       |
|  internal: visible in declaring TU only (g())                |
+--------------------------------------------------------------+
```

The practical value of module linkage is significant. Before modules, sharing a helper function across multiple `.cpp` files of a library required either giving it external linkage (and polluting the global symbol table), or duplicating it with internal linkage in each TU. Module linkage provides the **Goldilocks option** - shared within the module, hidden from everyone else. This reduces symbol table bloat and prevents accidental ODR violations with identically named symbols in other libraries.

---

## Self-Assessment

### Q1: Identify the linkage of each declaration in this module

Let's walk through a complete example. Pay attention to which declarations have `export`, which don't, and which use `static` or anonymous namespaces - those are your cues:

```cpp
// ---------- network.cppm (primary module interface unit) ----------
export module network;

export class Connection {        // external linkage (exported)
public:
    void send(const char* data);
    void close();
};

class ConnectionPool {           // MODULE linkage (in module purview, not exported)
public:
    Connection& acquire();
    void release(Connection& c);
};

namespace {
    int internal_counter = 0;    // internal linkage (anonymous namespace)
}

static void log_internal(const char* msg) {  // internal linkage (static)
    // ...
}

void validate_connection(Connection& c) {     // MODULE linkage (not exported)
    // ...
}

export ConnectionPool& get_pool();            // external linkage (exported)
// Note: ConnectionPool is reachable (in interface) but not visible to importers

// ---------- network_impl.cpp (module implementation unit) ----------
module network;

// Can use ConnectionPool - it has module linkage, visible here
ConnectionPool global_pool;

ConnectionPool& get_pool() { return global_pool; }

void Connection::send(const char* data) {
    validate_connection(*this);  // OK: module linkage, visible within module
    // log_internal("send");     // ERROR: internal linkage, only in network.cppm
    // ...
}
```

Here is the full linkage breakdown for every declaration:

| Declaration | Linkage | Reason |
| --- | --- | --- |
| `Connection` | External | Exported (`export class`) |
| `ConnectionPool` | Module | In module purview, not exported |
| `internal_counter` | Internal | Anonymous namespace |
| `log_internal` | Internal | `static` function |
| `validate_connection` | Module | In module purview, not exported |
| `get_pool` | External | Exported |
| `global_pool` | Module | In implementation unit, not exported |

Notice that `validate_connection` is usable from `network_impl.cpp` but `log_internal` is not - that's exactly the module-vs-internal distinction at work.

---

### Q2: How does module linkage interact with the linker, and how can you diagnose linkage issues

The thing that makes module linkage truly safe - and not just a convention like "put internal stuff in a `detail` namespace" - is that the linker enforces it through name mangling. Module-linkage symbols get the module name baked into their mangled name:

```cpp
// ---------- Module "audio" ----------
// audio.cppm
export module audio;

export void play_sound(int id);

void mix_channels(float* buf, int n);  // module linkage

// audio_impl.cpp
module audio;

void mix_channels(float* buf, int n) {
    // implementation
}

void play_sound(int id) {
    float buffer[1024];
    mix_channels(buffer, 1024);    // OK: same module
}

// ---------- Module "video" ----------
// video.cppm
export module video;

void mix_channels(float* buf, int n);  // DIFFERENT symbol - module linkage under "video"

void mix_channels(float* buf, int n) {
    // completely separate implementation
}

// ---------- main.cpp ----------
import audio;
import video;

int main() {
    play_sound(42);
    // No conflict: audio::mix_channels and video::mix_channels
    // are different symbols at the linker level despite identical names
}
```

Both modules have a function called `mix_channels` with the same signature, and there is no conflict. The linker sees something like `_ZW5audio12mix_channelsPfi` vs `_ZW5video12mix_channelsPfi` (GCC Itanium mangling), which are completely different symbols:

```cpp
Linker symbol table (conceptual):
  _Z10play_soundi             <- external linkage (exported from audio)
  _ZW5audio12mix_channelsPfi  <- module linkage (audio::mix_channels)
  _ZW5video12mix_channelsPfi  <- module linkage (video::mix_channels)
```

To diagnose linkage issues, use `nm` (Linux/macOS) or `dumpbin /symbols` (MSVC) to inspect object files. Module-linkage symbols will have module-name-qualified mangling.

---

### Q3: When should you use module linkage vs internal linkage vs external linkage

Here's a realistic database module that uses all three linkage kinds deliberately. The comments show the reasoning at each decision point:

```cpp
// ---------- database.cppm (module interface) ----------
export module database;

// External linkage - public API
export class Database {
public:
    bool connect(const char* connstr);
    void disconnect();
    int execute(const char* sql);
};

// Module linkage - shared across module TUs, hidden from importers
struct PreparedStatement {
    int handle;
    const char* sql;
    bool valid;
};

PreparedStatement prepare(Database& db, const char* sql);
void finalize(PreparedStatement& stmt);

// ---------- database_cache.cpp (implementation unit) ----------
module database;

#include <unordered_map>

// Module linkage - accessible from other module TUs
std::unordered_map<std::string, PreparedStatement> stmt_cache;

PreparedStatement prepare(Database& db, const char* sql) {
    auto it = stmt_cache.find(sql);
    if (it != stmt_cache.end()) return it->second;
    // ... prepare new statement
    PreparedStatement stmt{/*...*/};
    stmt_cache[sql] = stmt;
    return stmt;
}

// Internal linkage - only this TU
static bool is_cache_full() {
    return stmt_cache.size() > 1000;
}

// ---------- database_exec.cpp (implementation unit) ----------
module database;

int Database::execute(const char* sql) {
    auto stmt = prepare(*this, sql);  // OK: module linkage
    // is_cache_full();               // ERROR: internal to database_cache.cpp
    int result = /* ... */;
    finalize(stmt);                   // OK: module linkage
    return result;
}
```

If the decision feels fuzzy, this table makes it mechanical:

| Question | If Yes - Linkage |
| --- | --- |
| Does external code need to call it? | **External** (use `export`) |
| Do multiple TUs within the module share it? | **Module** (don't export) |
| Is it only used in one TU? | **Internal** (`static` or `namespace {}`) |
| Is it a block-scope variable? | **No linkage** |

---

## Notes

- Module linkage is **automatic**: any non-exported entity with would-be external linkage in a module's purview gets module linkage. No keyword required.
- Module linkage entities are **name-mangled** with the module name, preventing cross-module symbol collisions at the linker level.
- `static` and anonymous namespaces still produce **internal linkage** inside modules - this is unchanged from pre-module C++.
- Module linkage replaces the common pattern of "put helpers in an anonymous namespace" when those helpers need to be shared across multiple implementation units of the same library.
- **Inline functions** with module linkage: the definition must be in the module interface unit (or an interface partition) for the compiler to inline across TUs within the module.
- Class members of a module-linkage class inherit the class's linkage - you don't need to separately mark member functions.
- Module linkage entities **cannot** be accessed via `extern` declarations from outside the module - the mangled name is module-qualified.
- Watch for accidental module linkage: if you forget `export` on a function meant to be public, it silently gets module linkage. Use compiler warnings (e.g., `-Wunused-function` can sometimes catch unexported functions).
- Module linkage has zero runtime cost - it's purely a compile-time/link-time visibility mechanism.
