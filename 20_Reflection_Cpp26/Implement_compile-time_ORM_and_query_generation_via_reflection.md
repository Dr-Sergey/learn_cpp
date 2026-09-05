# Implement compile-time ORM and query generation via reflection

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996>  

---

## Topic Overview

### The Vision

Traditional C++ ORM libraries force you to maintain two parallel definitions: the C++ struct and the database schema. Any time you rename a field, change a type, or add a column, you have to update both sides and hope you did not miss a spot. With C++26 reflection, you can map C++ structs to database tables **automatically** at compile time - no macros, no external code generators, no runtime schema registration. The struct definition itself becomes the schema.

### Mapping Structs to SQL Tables

Here is how you use reflection to generate a `CREATE TABLE` SQL statement directly from a struct's member list. The function walks `nonstatic_data_members_of` to get field names and types, maps each C++ type to its SQL equivalent, and assembles the statement at compile time.

```cpp
#include <meta>
#include <string>
#include <format>

template<typename T>
consteval std::string generate_create_table() {
    std::string sql = "CREATE TABLE ";
    sql += std::meta::identifier_of(^T);
    sql += " (";

    bool first = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        if (!first) sql += ", ";
        sql += std::meta::identifier_of(m);
        sql += " ";
        sql += sql_type_name<typename [:std::meta::type_of(m):]>();
        first = false;
    }

    sql += ");";
    return sql;
}

template<typename T>
consteval auto sql_type_name() -> std::string {
    if constexpr (std::same_as<T, int>) return "INTEGER";
    else if constexpr (std::same_as<T, double>) return "REAL";
    else if constexpr (std::same_as<T, std::string>) return "TEXT";
    else if constexpr (std::same_as<T, bool>) return "BOOLEAN";
    else return "BLOB";
}

struct User {
    int id;
    std::string name;
    std::string email;
    int age;
};

// At compile time:
constexpr auto sql = generate_create_table<User>();
// sql == "CREATE TABLE User (id INTEGER, name TEXT, email TEXT, age INTEGER);"
```

The `consteval` keyword ensures this all runs at compile time. If you rename `User::email` to `User::email_address`, the next compile automatically produces `email_address TEXT` in the SQL - no manual schema update needed.

### Generating INSERT Statements

For runtime values you need a runtime function, but the field enumeration still happens at compile time. The `template for` loop is expanded by the compiler; only the value-formatting runs at runtime.

```cpp
template<typename T>
std::string generate_insert(const T& obj) {
    std::string sql = "INSERT INTO ";
    sql += std::meta::identifier_of(^T);
    sql += " (";

    // Column names
    bool first = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        if (!first) sql += ", ";
        sql += std::meta::identifier_of(m);
        first = false;
    }

    sql += ") VALUES (";

    // Values
    first = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        if (!first) sql += ", ";
        sql += to_sql_value(obj.[:m:]);
        first = false;
    }

    sql += ");";
    return sql;
}

std::string to_sql_value(int v) { return std::to_string(v); }
std::string to_sql_value(const std::string& v) {
    return "'" + v + "'";  // NOTE: use parameterized queries in production!
}

// Usage:
User u{1, "Alice", "alice@example.com", 30};
auto insert = generate_insert(u);
// "INSERT INTO User (id, name, email, age) VALUES (1, 'Alice', 'alice@example.com', 30);"
```

Note the comment: string concatenation into SQL is shown here only for illustration. In real code you must use parameterized queries - see Q3 below for why and how.

### Type-Safe Query Builder

You can also build a fluent query builder where the `where_eq` method is typed to the actual field type. Because `MemberPtr` is a reflection of a specific member, the value argument is automatically typed to match that field - the compiler catches type mismatches at the call site.

```cpp
template<typename T>
struct query_builder {
    std::string select_clause_ = "*";
    std::string where_clause_;

    template<auto MemberPtr>
    query_builder& where_eq(const typename [:std::meta::type_of(MemberPtr):]& value) {
        where_clause_ = std::format("{} = {}",
            std::meta::identifier_of(MemberPtr),
            to_sql_value(value));
        return *this;
    }

    std::string build() const {
        std::string sql = std::format("SELECT {} FROM {}",
            select_clause_,
            std::meta::identifier_of(^T));
        if (!where_clause_.empty())
            sql += " WHERE " + where_clause_;
        return sql + ";";
    }
};

// Usage:
auto q = query_builder<User>{}
    .where_eq<^User::age>(30)
    .build();
// "SELECT * FROM User WHERE age = 30;"
```

The `typename [:std::meta::type_of(MemberPtr):]` in the parameter type is doing a lot of work: it reflects the member pointer to get the member's type, then splices that type back as the parameter type. So `.where_eq<^User::age>(30)` works, but `.where_eq<^User::name>(30)` would be a compile error because `name` is a `std::string`, not an `int`.

### Mapping Query Results Back to Structs

The reverse direction - reading a row from a database result and populating a struct - is equally clean. Reflection gives you the members in declaration order, and you iterate them in order, reading each column by index.

```cpp
template<typename T>
T row_to_struct(const sql::Row& row) {
    T obj;
    int col = 0;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        using MemberType = typename [:std::meta::type_of(m):];
        obj.[:m:] = row.get<MemberType>(col++);
    }
    return obj;
}

// Usage:
auto user = row_to_struct<User>(result_row);
```

Add a field to `User`, and `row_to_struct` automatically reads it from the next column - no manual update required.

---

## Self-Assessment

### Q1: Generate a CREATE TABLE statement for a given struct

See the `generate_create_table<T>()` function above. For `struct Order { int id; std::string product; double price; };`, it produces:

```sql
CREATE TABLE Order (id INTEGER, product TEXT, price REAL);
```

### Q2: Why is reflection-based ORM safer than macro-based approaches

The key difference is that macros operate on text and have no knowledge of the type system, while reflection operates on the actual type graph. If the table feels like a lot, the core point is: with macros you have two separate things to keep in sync; with reflection there is only one thing - the struct.

| Property | Macros | Reflection |
| --- | --- | --- |
| Type safety | No | Yes - member types are checked at compile time |
| Refactoring | Fragile - renamed members silently break | Safe - renamed members update automatically |
| Compile errors | Cryptic | Clear - reflection errors are at the member level |
| Maintenance | Must manually sync macro definitions | Zero - struct definition IS the schema |

### Q3: What security concern exists in the SQL generation examples

**SQL injection.** The `to_sql_value(string)` function shown above uses string concatenation to build the SQL, which means user-supplied strings go directly into the query. If someone passes `"'; DROP TABLE User; --"` as a name, the concatenated SQL becomes a destructive multi-statement query.

In production, always use **parameterized queries** - generate the SQL structure with placeholders, then bind the actual values separately through the database driver's prepared statement API:

```cpp
template<typename T>
auto generate_parameterized_insert() {
    std::string sql = "INSERT INTO " + std::string(std::meta::identifier_of(^T)) + " (";
    // ... column names ...
    sql += ") VALUES (";
    int n = std::meta::nonstatic_data_members_of(^T).size();
    for (int i = 0; i < n; ++i) {
        if (i > 0) sql += ", ";
        sql += "?";  // parameterized placeholder
    }
    sql += ");";
    return sql;
}
// Then bind values through prepared statements, never string concatenation.
```

The `?` placeholders are never treated as SQL by the database driver - they are replaced with properly escaped values at the binding step. This is the only safe way to include user input in queries.

---

## Notes

- Reflection-based ORM eliminates the struct-to-schema synchronization problem entirely - your struct definition is your schema.
- Always use parameterized queries in production - never concatenate user input into SQL strings.
- The compile-time nature of `consteval` generation means zero runtime overhead for schema validation and query structure.
- This pattern extends naturally to any structured serialization format: JSON, MessagePack, Protocol Buffers, or any other format where your struct fields map to named slots.
