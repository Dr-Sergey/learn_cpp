# Serialize and deserialize std::variant and polymorphic types

**Category:** Serialization and Data Formats  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

Serializing `std::variant` and polymorphic class hierarchies requires encoding which type is active. `std::variant` uses a type index; polymorphic types typically use a string type discriminator.

The reason this is harder than serializing a plain struct is that the reader does not know ahead of time which type it will encounter. You have to embed that information in the serialized bytes and then use it during deserialization to reconstruct the right type. The two main scenarios - `std::variant` and virtual class hierarchies - solve this the same way conceptually but with different machinery.

### Serializing std::variant

With `std::variant`, the active type is identified by its index within the variant's type list. We write that index as a one-byte type tag before the value data, and then during deserialization we switch on the tag to know how many bytes to read and how to interpret them. The `std::visit` call dispatches to the right lambda branch depending on which alternative is currently active:

```cpp
#include <variant>
#include <string>
#include <iostream>
#include <cstdint>
#include <vector>
#include <cstring>

using Value = std::variant<int, double, std::string, bool>;

// Binary serialization with type tag
void serialize_value(const Value& v, std::vector<uint8_t>& buf) {
    uint8_t type_tag = static_cast<uint8_t>(v.index());
    buf.push_back(type_tag);

    std::visit([&buf](const auto& val) {
        using T = std::decay_t<decltype(val)>;
        if constexpr (std::is_same_v<T, std::string>) {
            uint32_t len = static_cast<uint32_t>(val.size());
            auto p = reinterpret_cast<const uint8_t*>(&len);
            buf.insert(buf.end(), p, p + 4);
            buf.insert(buf.end(), val.begin(), val.end());
        } else {
            auto p = reinterpret_cast<const uint8_t*>(&val);
            buf.insert(buf.end(), p, p + sizeof(T));
        }
    }, v);
}

Value deserialize_value(const uint8_t*& pos) {
    uint8_t tag = *pos++;
    switch (tag) {
        case 0: { int v; std::memcpy(&v, pos, 4); pos += 4; return v; }
        case 1: { double v; std::memcpy(&v, pos, 8); pos += 8; return v; }
        case 2: {
            uint32_t len;
            std::memcpy(&len, pos, 4); pos += 4;
            std::string s(reinterpret_cast<const char*>(pos), len);
            pos += len;
            return s;
        }
        case 3: { bool v = (*pos++ != 0); return v; }
        default: throw std::runtime_error("Unknown type tag");
    }
}

int main() {
    std::vector<Value> values = {42, 3.14, std::string("hello"), true};
    std::vector<uint8_t> buf;

    for (auto& v : values) serialize_value(v, buf);
    std::cout << "Serialized: " << buf.size() << " bytes\n";

    const uint8_t* pos = buf.data();
    for (size_t i = 0; i < values.size(); ++i) {
        auto v = deserialize_value(pos);
        std::visit([](const auto& val) {
            std::cout << "  " << val << "\n";
        }, v);
    }
}
```

One thing worth keeping in mind: the tag values are positional. Tag `0` means "the first type in the variant list" - in this case `int`. If you ever reorder the type list (e.g. moving `bool` before `int`), old serialized data becomes unreadable because the tag numbers no longer match. The safe habit is to assign explicit tag constants rather than relying on `v.index()` directly.

### Serializing Polymorphic Types with a Type Registry

For a virtual class hierarchy the problem is similar but the mechanism is different. During deserialization there is no existing object - you need to create one of the correct derived type from scratch. Virtual dispatch cannot help you here because virtual functions operate on objects that already exist.

The standard solution is a factory registry: a map from type name strings to factory functions. Each derived class registers itself at startup. During serialization you write the type name as a string; during deserialization you look up that name in the registry and call the matching factory. Here is a complete example:

```cpp
#include <memory>
#include <string>
#include <unordered_map>
#include <functional>
#include <iostream>

struct Shape {
    virtual ~Shape() = default;
    virtual std::string type_name() const = 0;
    virtual void serialize(std::ostream& os) const = 0;
    virtual void print() const = 0;
};

struct Circle : Shape {
    double radius;
    Circle(double r = 0) : radius(r) {}
    std::string type_name() const override { return "Circle"; }
    void serialize(std::ostream& os) const override { os << radius; }
    void print() const override { std::cout << "Circle(r=" << radius << ")"; }
};

struct Rect : Shape {
    double w, h;
    Rect(double w = 0, double h = 0) : w(w), h(h) {}
    std::string type_name() const override { return "Rect"; }
    void serialize(std::ostream& os) const override { os << w << " " << h; }
    void print() const override { std::cout << "Rect(" << w << "x" << h << ")"; }
};

// Factory registry for deserialization
using Factory = std::function<std::unique_ptr<Shape>(std::istream&)>;
std::unordered_map<std::string, Factory>& registry() {
    static std::unordered_map<std::string, Factory> r;
    return r;
}

template<typename T>
bool register_type(const std::string& name, Factory f) {
    registry()[name] = std::move(f);
    return true;
}

// Register types
static auto _c = register_type<Circle>("Circle", [](std::istream& is) {
    double r; is >> r; return std::make_unique<Circle>(r);
});
static auto _r = register_type<Rect>("Rect", [](std::istream& is) {
    double w, h; is >> w >> h; return std::make_unique<Rect>(w, h);
});

// Serialize: write type name + data
void save(std::ostream& os, const Shape& s) {
    os << s.type_name() << " ";
    s.serialize(os);
    os << "\n";
}

// Deserialize: read type name, look up factory
std::unique_ptr<Shape> load(std::istream& is) {
    std::string type;
    if (!(is >> type)) return nullptr;
    auto it = registry().find(type);
    if (it == registry().end()) throw std::runtime_error("Unknown type: " + type);
    return it->second(is);
}
```

The `static auto _c = register_type<Circle>(...)` lines run before `main` and populate the registry. This is the self-registration idiom - each type knows how to register itself, so you never forget to add a new type to a central list.

---

## Self-Assessment

### Q1: Serialize a variant to JSON with type discrimination

Instead of a compact binary tag, you can use a JSON envelope with explicit `"type"` and `"value"` keys. This makes the serialized form human-readable and easy to consume from other languages:

```cpp
#include <variant>
#include <string>
#include <iostream>

using Value = std::variant<int, double, std::string>;

std::string variant_to_json(const Value& v) {
    return std::visit([](const auto& val) -> std::string {
        using T = std::decay_t<decltype(val)>;
        if constexpr (std::is_same_v<T, std::string>)
            return "{\"type\":\"string\",\"value\":\"" + val + "\"}";
        else if constexpr (std::is_same_v<T, int>)
            return "{\"type\":\"int\",\"value\":" + std::to_string(val) + "}";
        else
            return "{\"type\":\"double\",\"value\":" + std::to_string(val) + "}";
    }, v);
}
```

### Q2: Explain why virtual functions alone are insufficient for deserialization

Virtual functions dispatch on an existing object's type. During deserialization, no object exists yet - you need to **create** an object of the correct type from a type tag. This requires a factory/registry pattern external to the class hierarchy.

The reason this trips people up is that virtual dispatch feels like a solution to the "which type is this?" question. And it is - but only once you already have an object. The chicken-and-egg problem is that to get an object, you need to know the type first, and that knowledge has to come from the serialized data via something that is not virtual dispatch.

### Q3: Handle forward compatibility when new types are added to a variant

When a new alternative is added to the variant, old deserializers encounter an unknown type tag. Use a fallback strategy: skip the unknown data (if length-prefixed) or return a default/error value.

The practical advice here is to always length-prefix variable-size alternatives, even if they currently have a known fixed size. That way an old reader can skip past an unknown tag by reading the length and advancing the cursor, rather than crashing or producing corrupt output.

---

## Notes

- `std::variant::index()` is stable only if you never reorder alternatives - use explicit type tags instead.
- For polymorphic types, a type registry (factory map) is the standard pattern.
- nlohmann/json can serialize variants automatically with `NLOHMANN_JSON_SERIALIZE_ENUM`.
- Consider `std::any` if the set of types is truly unbounded (but you lose type safety).
