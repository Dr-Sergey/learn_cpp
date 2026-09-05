# Use std::tie for lexicographic comparison of multiple struct members

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/tuple/tie>  

---

## Topic Overview

`std::tie` (header `<tuple>`) creates a `std::tuple` of **lvalue references** to its arguments. Its most iconic use is writing concise, correct lexicographic comparisons for structs with multiple members - without manually chaining `if` statements.

The reason this idiom is worth knowing is that the manual approach is surprisingly easy to get wrong. You have to chain comparisons in the right order, handle the "if equal, move to next field" logic, and get the direction right for each field. `std::tie` hands all of that off to `std::tuple`'s built-in comparison, which does it correctly by definition.

### How std::tie Enables Lexicographic Comparison

```cpp
Manual approach (error-prone):          std::tie approach (one line):
                                        
if (a.last < b.last) return true;       return std::tie(a.last, a.first, a.age)
if (a.last > b.last) return false;            < std::tie(b.last, b.first, b.age);
if (a.first < b.first) return true;
if (a.first > b.first) return false;    // tuple's operator< does the exact same
return a.age < b.age;                   // multi-field comparison automatically
```

### How It Works

`std::tie(x, y, z)` returns `std::tuple<T1&, T2&, T3&>` - a tuple of references to the original variables. `std::tuple`'s relational operators perform **element-wise lexicographic comparison**, which is exactly what you need for struct ordering.

### Core Example

Here the `Employee` struct's ordering is: department first, then name within the same department, then salary. `std::tie` expresses that priority order in one readable line.

```cpp
#include <tuple>
#include <string>
#include <iostream>
#include <vector>
#include <algorithm>

struct Employee {
    std::string department;
    std::string name;
    int salary;

    // Lexicographic: department first, then name, then salary
    bool operator<(const Employee& other) const {
        return std::tie(department, name, salary)
             < std::tie(other.department, other.name, other.salary);
    }

    bool operator==(const Employee& other) const {
        return std::tie(department, name, salary)
            == std::tie(other.department, other.name, other.salary);
    }
};

int main() {
    std::vector<Employee> team{
        {"Engineering", "Zara", 90000},
        {"Engineering", "Alice", 85000},
        {"Design", "Bob", 75000},
    };

    std::sort(team.begin(), team.end());

    for (const auto& e : team)
        std::cout << e.department << " | " << e.name << " | " << e.salary << "\n";
    // Output:
    // Design | Bob | 75000
    // Engineering | Alice | 85000
    // Engineering | Zara | 90000
}
```

Design sorts before Engineering alphabetically, and within Engineering the names sort by first name. The `std::tie` call handles all of that automatically.

### std::tie for Unpacking (Multi-Assignment)

`std::tie` has a second use: because it returns a tuple of references, you can assign a tuple to it and the values flow through to the original variables.

```cpp
#include <tuple>
#include <iostream>

std::tuple<int, double, std::string> get_record() {
    return {42, 3.14, "hello"};
}

int main() {
    int id;
    double value;
    std::string label;

    // Unpack tuple into existing variables:
    std::tie(id, value, label) = get_record();

    std::cout << id << ", " << value << ", " << label << "\n";
    // Output: 42, 3.14, hello

    // Use std::ignore to skip unwanted fields:
    int id2;
    std::tie(id2, std::ignore, std::ignore) = get_record();
    std::cout << "Just the id: " << id2 << "\n";
    // Output: Just the id: 42
}
```

> **C++17 note:** Structured bindings (`auto [id, val, label] = get_record();`) largely replace `std::tie` for unpacking, but `std::tie` is still essential for comparison operators and rebinding existing variables.

---

## Self-Assessment

### Q1: Implement operator< for a struct with three members using std::tie(a,b,c) < std::tie(o.a,o.b,o.c)

**Answer:**

Watch how the sort output demonstrates lexicographic ordering: `Adams` before `Smith`, and among `Smith` entries the first name breaks ties, then grade.

```cpp
#include <tuple>
#include <string>
#include <iostream>
#include <vector>
#include <algorithm>
#include <set>

struct Student {
    std::string last_name;
    std::string first_name;
    int grade;

    bool operator<(const Student& o) const {
        return std::tie(last_name, first_name, grade)
             < std::tie(o.last_name, o.first_name, o.grade);
    }

    bool operator==(const Student& o) const {
        return std::tie(last_name, first_name, grade)
            == std::tie(o.last_name, o.first_name, o.grade);
    }
};

int main() {
    std::vector<Student> students{
        {"Smith", "John", 90},
        {"Adams", "Jane", 95},
        {"Smith", "Alice", 88},
        {"Smith", "John", 85},
    };

    std::sort(students.begin(), students.end());

    for (const auto& s : students) {
        std::cout << s.last_name << ", " << s.first_name
                  << " (grade: " << s.grade << ")\n";
    }
    // Output:
    // Adams, Jane (grade: 95)
    // Smith, Alice (grade: 88)
    // Smith, John (grade: 85)     <- same name, lower grade first
    // Smith, John (grade: 90)

    // Works with ordered containers too:
    std::set<Student> roster(students.begin(), students.end());
    std::cout << "Set size: " << roster.size() << "\n"; // 4 (all distinct)

    // WHY this is better than manual comparison:
    // 1. Impossible to forget a tiebreaker field
    // 2. Impossible to get comparison direction wrong on one field
    // 3. Adding a new field to the struct: just add it to the tie() calls
    // 4. Compiler optimizes away the tuple construction entirely
}
```

`std::tie` leverages `std::tuple`'s built-in lexicographic comparison. It compares `last_name` first; if equal, compares `first_name`; if still equal, compares `grade`. The tuple is never actually constructed on the heap - the compiler optimizes it into the same comparisons you'd write manually.

### Q2: Explain how std::tie creates a tuple of lvalue references

**Answer:**

The reason `std::tie` is zero-cost for comparisons - and why assigning to it works at all - is that it binds references, not copies. This example proves it by modifying through the tuple and observing the originals change.

```cpp
#include <tuple>
#include <iostream>
#include <string>
#include <type_traits>

int main() {
    int x = 10;
    double y = 3.14;
    std::string z = "hello";

    // std::tie creates a tuple of REFERENCES, not copies
    auto t = std::tie(x, y, z);

    // Prove it's references:
    static_assert(std::is_same_v<decltype(t), std::tuple<int&, double&, std::string&>>);

    // Modifying through the tuple modifies the original:
    std::get<0>(t) = 42;
    std::get<2>(t) = "world";
    std::cout << "x = " << x << "\n";    // 42 (modified!)
    std::cout << "z = " << z << "\n";    // "world" (modified!)

    // This is WHY tie works for comparison:
    // std::tie(a, b) < std::tie(c, d)
    // creates two tuples of references and compares them.
    // NO COPIES of a, b, c, d are made - just references.

    // Contrast with std::make_tuple which COPIES:
    auto t2 = std::make_tuple(x, y, z);
    static_assert(std::is_same_v<decltype(t2), std::tuple<int, double, std::string>>);
    // t2 holds copies - modifying t2 does NOT modify x, y, z

    std::get<0>(t2) = 999;
    std::cout << "x still = " << x << "\n"; // 42 (unchanged)

    // Under the hood, std::tie is essentially:
    // template<typename... Args>
    // std::tuple<Args&...> tie(Args&... args) {
    //     return std::tuple<Args&...>(args...);
    // }
}
```

`std::tie(a, b, c)` returns `std::tuple<A&, B&, C&>` - a tuple whose elements are lvalue references to the original variables. No copies are made. This is why it's zero-overhead for comparisons and why assigning to a `std::tie(...)` expression writes through to the original variables.

### Q3: Show that std::tie can be used on the left side of an assignment for multi-assignment

**Answer:**

This is the unpacking pattern. Because `std::tie` creates references, the assignment `std::tie(a, b, c) = some_tuple` is equivalent to writing `a = get<0>(...); b = get<1>(...); c = get<2>(...);`.

```cpp
#include <tuple>
#include <iostream>
#include <string>
#include <map>

// Function returning multiple values as a tuple
std::tuple<bool, int, std::string> parse_config(const std::string& line) {
    auto pos = line.find('=');
    if (pos == std::string::npos)
        return {false, 0, "parse error"};

    std::string key = line.substr(0, pos);
    std::string val = line.substr(pos + 1);
    return {true, static_cast<int>(val.size()), key};
}

int main() {
    // === Multi-assignment from a tuple ===
    bool success;
    int length;
    std::string key;

    // std::tie on the LEFT side of = unpacks into existing variables:
    std::tie(success, length, key) = parse_config("timeout=30");

    std::cout << std::boolalpha;
    std::cout << "success: " << success << "\n";  // true
    std::cout << "length:  " << length << "\n";   // 2
    std::cout << "key:     " << key << "\n";       // "timeout"

    // === Skip fields with std::ignore ===
    bool ok;
    std::tie(ok, std::ignore, std::ignore) = parse_config("bad line");
    std::cout << "ok: " << ok << "\n"; // false

    // === Swap two structs' members elegantly ===
    int a = 1, b = 2, c = 3;
    int x = 10, y = 20, z = 30;

    // Swap (a,b,c) with (x,y,z) in one statement:
    std::tie(a, b, c) = std::make_tuple(x, y, z);
    std::cout << a << " " << b << " " << c << "\n"; // 10 20 30

    // === Unpacking std::map::insert result (pre-C++17) ===
    std::map<std::string, int> m;
    std::map<std::string, int>::iterator it;
    bool inserted;
    std::tie(it, inserted) = m.insert({"key", 42});
    std::cout << "Inserted: " << inserted << ", value: " << it->second << "\n";
    // Output: Inserted: true, value: 42

    // C++17 equivalent with structured bindings:
    // auto [it2, inserted2] = m.insert({"key2", 99});
}
```

Because `std::tie` produces a tuple of references, assigning a tuple to it writes each element into the corresponding original variable. `std::ignore` acts as a sink for unwanted tuple elements. Before C++17 structured bindings, this was the standard way to unpack multi-return functions and `std::pair` results from map operations.

---

## Notes

- **C++20 alternative:** For comparison operators, prefer `auto operator<=>(const T&) const = default;` which auto-generates all six operators with member-wise comparison - no `std::tie` needed.
- **Performance:** `std::tie` comparisons are zero-overhead - compilers eliminate the tuple entirely and emit the same code as hand-written comparisons.
- **Pitfall:** Ensure the member order in both `tie()` calls matches exactly. A mismatch silently produces wrong comparisons.
- **`std::ignore`:** Only valid in `std::tie` context. It cannot be used standalone.
- **Prefer structured bindings (C++17)** for unpacking: `auto [a, b, c] = func();` is cleaner than `std::tie(a, b, c) = func();`, but structured bindings declare new variables while `std::tie` assigns to existing ones.
- Compile with `-std=c++17 -Wall -Wextra`.
