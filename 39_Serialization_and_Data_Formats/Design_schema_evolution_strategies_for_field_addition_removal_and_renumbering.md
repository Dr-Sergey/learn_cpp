# Design schema evolution strategies for field addition, removal, and renumbering

**Category:** Serialization and Data Formats  
**Standard:** C++17  
**Reference:** <https://protobuf.dev/programming-guides/proto3/#updating>  

---

## Topic Overview

As software evolves, data schemas change. The real challenge is that old binary data keeps showing up - maybe it is sitting in a database, maybe an old client is still running, maybe you are replaying an event log from two years ago. Schema evolution must allow new code to read old data and, ideally, old code to read new data gracefully.

### Evolution Operations and Their Safety

The table below summarises which operations are safe, which are risky, and which will corrupt your data. If the table feels like a lot, the short version is: adding optional fields is always safe, and you should never change a field's type or reuse a field number.

| Operation | Backward Compatible? | Forward Compatible? | Safe? |
| --- | --- | --- | --- |
| Add optional field | Yes | Yes (old readers skip) | Safe |
| Add required field | No (old data lacks it) | Yes | Dangerous |
| Remove optional field | Yes (new readers use default) | Yes | Safe |
| Remove required field | Yes | No (old readers expect it) | Dangerous |
| Rename field | Depends on wire format | - | Use field IDs |
| Change field type | No (incompatible) | No | Never do this |
| Reuse field number | No (data corruption) | No | Never do this |

### Strategy 1: Always-Optional Fields with Defaults

The safest and most common approach is to make every new field optional with a sensible default. When old data is read, the missing field just uses its default value - no error, no corruption. The key insight is that field names are irrelevant at the wire level; what matters are the numeric field IDs, so a renamed field in code must keep its original ID on the wire.

```cpp
#include <optional>
#include <string>
#include <iostream>

// Version 1
struct UserV1 {
    uint32_t id;
    std::string name;
    // field_id=1: id, field_id=2: name
};

// Version 2: added email (optional, field_id=3)
struct UserV2 {
    uint32_t id;
    std::string name;
    std::optional<std::string> email;  // New in v2 - absent in old data
};

// Version 3: deprecated age, added phone (field_id=4)
struct UserV3 {
    uint32_t id;
    std::string name;
    std::optional<std::string> email;
    // field_id=3 is RETIRED - never reused
    std::optional<std::string> phone;  // field_id=4
};

void process_user(const UserV3& user) {
    std::cout << "ID: " << user.id << "\n";
    std::cout << "Name: " << user.name << "\n";
    std::cout << "Email: " << user.email.value_or("(not provided)") << "\n";
    std::cout << "Phone: " << user.phone.value_or("(not provided)") << "\n";
}
```

Notice how `UserV3` uses `std::optional` for fields that might be absent in older records. The `value_or()` call gives you a graceful fallback without any conditional logic at the call site.

### Strategy 2: Field ID Registry

When a field is removed, its ID must be permanently retired - not recycled. The reason this trips people up is that it feels wasteful to leave a number "unused forever", but recycling an ID is far worse: old readers will interpret the new field's bytes using the old type, producing silent garbage. A field ID registry makes those reservations explicit and auditable.

```cpp
#include <cstdint>
#include <set>
#include <iostream>

// Maintain a registry of all field IDs ever used
// Mark deprecated fields as RESERVED - never reassign their IDs

namespace schema {
    // Active fields
    constexpr uint16_t FIELD_ID       = 1;
    constexpr uint16_t FIELD_NAME     = 2;
    constexpr uint16_t FIELD_EMAIL    = 3;
    constexpr uint16_t FIELD_PHONE    = 5;

    // RESERVED - formerly used, never reuse!
    constexpr uint16_t RESERVED_AGE   = 4;  // Removed in v3

    const std::set<uint16_t> reserved_ids = {4};

    bool is_reserved(uint16_t id) {
        return reserved_ids.count(id) > 0;
    }
}
```

You can hook `is_reserved()` into your schema validator to catch accidental ID reuse at startup or in tests, long before it reaches production.

### Strategy 3: Union-Based Schema Versioning

Sometimes the versions diverge enough that you want to keep distinct struct types per version and provide explicit upgrade functions. A `std::variant` lets you carry any version in one handle, and `std::visit` with `if constexpr` lets you probe which fields a particular version has at compile time.

```cpp
#include <variant>
#include <string>

struct SchemaV1 { uint32_t id; std::string name; };
struct SchemaV2 { uint32_t id; std::string name; std::string email; };
struct SchemaV3 { uint32_t id; std::string name; std::string email; std::string phone; };

using AnySchema = std::variant<SchemaV1, SchemaV2, SchemaV3>;

// Convert old schemas to latest
SchemaV3 upgrade(const AnySchema& data) {
    return std::visit([](const auto& v) -> SchemaV3 {
        SchemaV3 result;
        result.id = v.id;
        result.name = v.name;
        if constexpr (requires { v.email; })
            result.email = v.email;
        if constexpr (requires { v.phone; })
            result.phone = v.phone;
        return result;
    }, data);
}
```

The `if constexpr (requires { v.email; })` idiom asks at compile time whether the active variant type has an `email` member. If it does not (e.g. for `SchemaV1`), that branch is simply omitted. It is a clean, zero-overhead way to write a single upgrade function that handles all historical versions.

---

## Self-Assessment

### Q1: Why is changing a field's type more dangerous than removing it

Changing a field's type (e.g., `int32` to `string`) means old readers interpret new data with the wrong type - silent data corruption with no error. Removing a field with a tagged format simply means old readers see a missing field and use a default. Type changes are never backward or forward compatible.

### Q2: Design a protocol buffer-style schema with reserved fields

Here is how protobuf expresses the same "never reuse" guarantee at the schema definition level. The `reserved` keyword enforces this rule at compile time - the protobuf compiler will reject any `.proto` file that tries to reassign a reserved field number or name.

```protobuf
// Protobuf .proto file demonstrating reserved fields
message User {
    uint32 id = 1;
    string name = 2;
    // Field 3 was "age" - removed in v3, RESERVED forever
    reserved 3;
    reserved "age";
    string email = 4;
    string phone = 5;
}
```

### Q3: Implement a migration function between schema versions

A migration function converts a record from one version to the next. For multi-version jumps you chain these together. The pattern below shows V1 to V2; for V1 to V3 you would call `migrate_v1_to_v2` and then a corresponding `migrate_v2_to_v3`.

```cpp
#include <string>
#include <optional>
#include <iostream>
#include <functional>
#include <vector>

struct UserV1 { uint32_t id; std::string name; };
struct UserV2 { uint32_t id; std::string name; std::optional<std::string> email; };

UserV2 migrate_v1_to_v2(const UserV1& v1) {
    return {v1.id, v1.name, std::nullopt}; // email defaults to absent
}

// Chain migrations for multi-version jumps
template<typename From, typename To>
using Migration = std::function<To(const From&)>;

int main() {
    UserV1 old_user{1, "Alice"};
    UserV2 new_user = migrate_v1_to_v2(old_user);
    std::cout << "Migrated: " << new_user.name
              << " email=" << new_user.email.value_or("(none)") << "\n";
}
```

Notice that the email field is explicitly set to `std::nullopt` - the new reader knows the field was not present in the original record, which is different from "the user has no email address". Keeping that distinction is important for data quality.

---

## Notes

- **Golden rule**: never reuse field IDs and never change field types.
- Protobuf, FlatBuffers, and Avro all implement these evolution principles differently.
- `std::optional` is the natural C++ representation for "field may be absent."
- Keep a schema changelog document tracking every field addition/removal with version numbers.
