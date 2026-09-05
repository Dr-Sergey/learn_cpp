# Implement JSON serialization with nlohmann-json, simdjson, and glaze

**Category:** Serialization and Data Formats  
**Standard:** C++17, C++20  
**Reference:** <https://github.com/nlohmann/json> · <https://simdjson.org> · <https://github.com/stephenberry/glaze>  

---

## Topic Overview

C++ has no built-in JSON support. Three libraries cover the spectrum from ease-of-use to maximum performance, and picking the right one for your situation matters more than you might think.

### Comparison

If the table feels like a lot, the short version is: use nlohmann/json when you want readable code, simdjson when you are processing large files, and glaze when you want compile-time type safety with close-to-simdjson speed.

| Library | Style | Parse Speed | Ease of Use | Reflection | Allocation |
| --- | --- | --- | --- | --- | --- |
| nlohmann/json | DOM, SAX | ~400 MB/s | Excellent | No (macros) | Yes |
| simdjson | On-demand | ~4 GB/s | Good | No | Minimal |
| glaze | Compile-time | ~2 GB/s | Good | Yes (C++20) | Minimal |

### nlohmann/json - The Ergonomic Choice

nlohmann/json is the go-to library for most C++ JSON work. The `NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE` macro generates the serialization and deserialization code for you without touching the struct definition. The example below also shows dynamic field access with `.value()`, which gives you a default when a field is missing - a pattern you will use constantly in real code.

<!-- compile: needs third-party library header `nlohmann/json.hpp` -->
```cpp
#include <nlohmann/json.hpp>
#include <iostream>
#include <string>
#include <vector>

using json = nlohmann::json;

struct Person {
    std::string name;
    int age;
    std::vector<std::string> hobbies;
};

// Macro-based serialization (non-intrusive)
NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(Person, name, age, hobbies)

int main() {
    // Serialize
    Person alice{"Alice", 30, {"coding", "hiking"}};
    json j = alice;
    std::cout << j.dump(2) << "\n";
    // {
    //   "age": 30,
    //   "hobbies": ["coding", "hiking"],
    //   "name": "Alice"
    // }

    // Deserialize
    auto text = R"({"name":"Bob","age":25,"hobbies":["reading"]})";
    auto bob = json::parse(text).get<Person>();
    std::cout << bob.name << " is " << bob.age << "\n";

    // Dynamic access
    json config = json::parse(R"({"debug": true, "port": 8080})");
    bool debug = config.value("debug", false);
    int port = config.value("port", 3000);
    std::cout << "Port: " << port << ", debug: " << debug << "\n";
}
```

Notice that `j.dump(2)` produces pretty-printed JSON with 2-space indentation - very handy for debugging. `config.value("port", 3000)` returns `3000` if the `"port"` key is absent, which is cleaner than a `contains()` check followed by a get.

### simdjson - The Performance Choice

simdjson parses JSON using SIMD instructions (AVX2 on x86, NEON on ARM), which lets it reach throughput around 4 GB/s on modern hardware. The "on-demand" API is the key innovation: values are only materialised when you actually access them. If you have a 10 MB file with 50 fields per record but only need one field, roughly 98% of the parsing work never happens. Note the `_padded` string literal - simdjson requires input to have extra bytes past the end so its SIMD reads do not go out of bounds.

```cpp
#include <simdjson.h>
#include <iostream>
#include <string>

void parse_with_simdjson() {
    simdjson::ondemand::parser parser;

    auto json_str = R"([
        {"name": "Alice", "score": 95.5},
        {"name": "Bob", "score": 87.3}
    ])"_padded;  // simdjson requires padded input

    auto doc = parser.iterate(json_str);
    for (auto item : doc.get_array()) {
        std::string_view name = item["name"].get_string();
        double score = item["score"].get_double();
        std::cout << name << ": " << score << "\n";
    }
    // Output:
    //   Alice: 95.5
    //   Bob: 87.3
}

// On-demand parsing: fields are parsed only when accessed
// ~10x faster than nlohmann for large documents
```

### glaze - Compile-Time Reflection (C++20)

glaze sits between nlohmann and simdjson: nearly as fast as simdjson, but with a type-safe struct mapping defined at compile time. You describe the mapping once in a `glz::meta` specialisation, and then `glz::write_json` / `glz::read_json` handle everything without macros or runtime type dispatch.

```cpp
#include <glaze/glaze.hpp>
#include <iostream>

struct Config {
    std::string host = "localhost";
    int port = 8080;
    bool debug = false;
};

// glaze uses compile-time reflection - no macros needed (C++20)
template<>
struct glz::meta<Config> {
    using T = Config;
    static constexpr auto value = object(
        "host", &T::host,
        "port", &T::port,
        "debug", &T::debug
    );
};

void use_glaze() {
    // Serialize
    Config cfg{"example.com", 443, true};
    std::string json = glz::write_json(cfg);
    std::cout << json << "\n";
    // {"host":"example.com","port":443,"debug":true}

    // Deserialize
    Config parsed;
    glz::read_json(parsed, json);
    std::cout << parsed.host << ":" << parsed.port << "\n";
}
```

The `glz::meta` specialisation is the entire "schema" - it lives at compile time, so glaze can generate optimal serialization code without any runtime reflection overhead.

---

## Self-Assessment

### Q1: When to choose each library

- **nlohmann/json**: Rapid prototyping, small data, schema-less exploration, or anywhere you value a clean and readable API over raw speed.
- **simdjson**: Parsing large JSON files (logs, API responses), read-only scenarios, maximum throughput. The on-demand API particularly shines when you only need a small subset of fields.
- **glaze**: Struct-based serialization where you want high performance with compile-time type safety in a C++20 project.

### Q2: Show error handling with nlohmann/json

nlohmann/json throws typed exceptions for parse errors and type mismatches. The `.value()` method is your first line of defence for optional fields - it avoids exceptions entirely by returning a default when the key is missing.

```cpp
#include <nlohmann/json.hpp>
#include <iostream>

void safe_parse() {
    try {
        auto j = nlohmann::json::parse("{invalid json}");
    } catch (const nlohmann::json::parse_error& e) {
        std::cerr << "Parse error: " << e.what() << "\n";
    }

    // Safe field access with .value() and defaults
    auto j = nlohmann::json::parse(R"({"name": "Alice"})");

    std::string name = j.value("name", "unknown");
    int age = j.value("age", -1);  // Missing field - default

    // Check field existence
    if (j.contains("email")) {
        std::cout << "Has email\n";
    }

    // Type-safe access with .get<T>()
    try {
        int wrong = j["name"].get<int>(); // Throws type_error
    } catch (const nlohmann::json::type_error& e) {
        std::cerr << "Type error: " << e.what() << "\n";
    }
}
```

### Q3: Parse a 10MB JSON file efficiently with simdjson on-demand

The on-demand parser is the right tool for large files where you only need certain fields. It never builds a full DOM tree, so memory usage is proportional to the fields you actually read, not the total file size.

```cpp
#include <simdjson.h>

// On-demand parsing: only materializes values you access.
// For a 10MB file with 100K records, if you only need 1 field,
// ~99% of the data is never allocated.

void extract_field(const char* filename) {
    simdjson::ondemand::parser parser;
    auto json = simdjson::padded_string::load(filename);

    auto doc = parser.iterate(json);
    double total = 0;
    int count = 0;

    for (auto record : doc.get_array()) {
        // Only "score" is materialized - other fields are skipped
        double score = record["score"].get_double();
        total += score;
        ++count;
    }
    // Processes ~4 GB/s on modern hardware
}
```

The comment says it all: if you access only `"score"`, every other field in every record is skipped at the parser level - no allocation, no string copying. For log aggregation workloads this is the difference between a 10-second job and a 1-second job.

---

## Notes

- nlohmann/json is the most widely used C++ JSON library - excellent for most use cases.
- simdjson exploits SIMD instructions (AVX2/NEON) for parsing - truly zero-allocation on-demand mode.
- glaze combines compile-time type safety with performance close to simdjson.
- For **writing** JSON, all three are roughly comparable; the speed gap is primarily in parsing.
- Always validate and sanitize JSON from untrusted sources.
