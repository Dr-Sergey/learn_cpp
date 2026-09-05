# Use Approval Tests for characterization testing of legacy code

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://approvaltests.com/>  

---

## Topic Overview

**Approval Tests** (aka snapshot tests) capture the actual output of code and compare it against a previously "approved" baseline. They're ideal for **characterization testing**: documenting what legacy code *actually does* before refactoring, without needing to understand the logic first.

The reason this workflow is so useful for legacy code is that it inverts the usual challenge. Normally you need to know the expected output to write a test. With approval tests, you let the code tell you what the expected output is - you run it, inspect the result, and if it looks correct you approve it. From that point on, any change to the output becomes a visible diff, which is exactly what you want when refactoring.

### Workflow

The key insight is that the first run is always a failure - there's nothing to compare against yet. That's intentional. You inspect the received output, decide it's correct, and approve it. All subsequent runs then guard against any deviation.

```cpp
1. Run test -> capture output to "received" file
2. First run: no "approved" file -> test FAILS
3. Inspect "received" -> approve it (rename/copy)
4. Next runs: compare "received" vs "approved"

   -> Match = PASS, Differ = FAIL (shows diff)
```

### Approval Tests vs Assertion-Based Tests

Both approaches have their place. The table below captures when each one wins. The big advantage of approval tests is for complex output - once you've approved a 50-line report, any line that changes shows up in the diff immediately.

| Aspect | Assertion Tests | Approval Tests |
| --- | :---: | :---: |
| Setup effort | Write expected values manually | Run once, approve output |
| Complex output (JSON, XML) | Tedious to specify | Natural: approve the blob |
| Refactoring safety | Must update assertions | Diff shows any change |
| Documents behavior? | Only what you assert | Complete output snapshot |
| Discovering legacy behavior | Poor (you need to know) | Excellent (captures what IS) |
| Brittleness | Low (focused assertions) | Higher (any output change fails) |

---

## Self-Assessment

### Q1: Add an Approval Test that captures the output of a legacy function as the approved output

**Answer:**

The example wraps a `SalesReport` generator - a realistic piece of legacy code that produces multi-line formatted output. The first time you run the test, it fails because there's no approved file yet. You look at the received file, confirm the numbers are correct, and approve it. After that, the test protects the exact formatting, values, and structure indefinitely.

The combination approval at the end is worth noting: instead of writing one test per tax rate (five separate tests), you write one test that covers all five rates and produces a single approved file. That's a much better ratio of test-maintenance effort to coverage.

```cpp
// Approval Tests with Catch2 integration
// Requires: ApprovalTests.cpp library + Catch2

#include "ApprovalTests.hpp"
#include <catch2/catch.hpp>
#include <sstream>
#include <string>
#include <vector>
#include <map>
#include <iomanip>

// Legacy code (untested, undocumented)
struct SalesReport {
    struct LineItem {
        std::string product;
        int quantity;
        double unit_price;
    };

    static std::string generate(const std::vector<LineItem>& items,
                                 double tax_rate) {
        std::ostringstream out;
        out << std::fixed << std::setprecision(2);
        out << "=== SALES REPORT ===\n";

        double subtotal = 0;
        for (auto& item : items) {
            double line_total = item.quantity * item.unit_price;
            subtotal += line_total;
            out << item.product << " x" << item.quantity
                << " @ $" << item.unit_price
                << " = $" << line_total << "\n";
        }

        double tax = subtotal * tax_rate;
        double total = subtotal + tax;
        out << "---\n";
        out << "Subtotal: $" << subtotal << "\n";
        out << "Tax (" << (tax_rate * 100) << "%): $" << tax << "\n";
        out << "Total: $" << total << "\n";
        return out.str();
    }
};

// Characterization test - captures ACTUAL output
TEST_CASE("SalesReport generates correct output") {
    std::vector<SalesReport::LineItem> items = {
        {"Widget A", 3, 9.99},
        {"Gadget B", 1, 24.50},
        {"Part C",   10, 1.25},
    };

    std::string report = SalesReport::generate(items, 0.08);

    // First run: creates .received.txt, test FAILS
    // You inspect the output, approve it -> .approved.txt
    // All subsequent runs compare against approved
    ApprovalTests::Approvals::verify(report);
}

// The approved file (SalesReport.generates_correct_output.approved.txt):
// === SALES REPORT ===
// Widget A x3 @ $9.99 = $29.97
// Gadget B x1 @ $24.50 = $24.50
// Part C x10 @ $1.25 = $12.50
// ---
// Subtotal: $66.97
// Tax (8.00%): $5.36
// Total: $72.33

// Combination approvals - test many inputs at once
TEST_CASE("SalesReport tax rates") {
    SalesReport::LineItem item{"Test Item", 1, 100.00};

    auto tax_rates = {0.0, 0.05, 0.08, 0.10, 0.20};

    ApprovalTests::CombinationApprovals::verifyAllCombinations(
        [](double rate) {
            return SalesReport::generate({{"Item", 1, 100.0}}, rate);
        },
        tax_rates
    );
    // Creates ONE approved file with all combinations
}
```

The approved file name is derived from the test name automatically, so you can tell at a glance which test each file belongs to. The approved files live in your repository alongside the source - they are the specification.

### Q2: Run the approval tests after a refactoring and verify behavior is unchanged

**Answer:**

This is approval testing's main purpose: providing a safety net during refactoring. The workflow is simple - approve the output of the legacy code first, then change the implementation, then run the tests. If the output matches, you're done. If it differs, the diff tool shows you exactly what changed.

The example refactors a table formatter from manual string padding to `std::setw`. This looks like it should produce identical output, but small differences in how trailing spaces are handled can slip through. The approval test catches those differences immediately.

```cpp
// Refactoring with Approval Tests as safety net

#include "ApprovalTests.hpp"
#include <catch2/catch.hpp>
#include <sstream>
#include <string>
#include <vector>
#include <iomanip>

// BEFORE refactoring: messy legacy formatting
namespace legacy {
std::string format_table(const std::vector<std::vector<std::string>>& rows) {
    std::ostringstream out;
    // Legacy code: hardcoded widths, manual padding
    for (auto& row : rows) {
        for (size_t i = 0; i < row.size(); ++i) {
            auto& cell = row[i];
            out << cell;
            // Pad to width 15
            for (size_t j = cell.size(); j < 15; ++j) out << ' ';
            if (i < row.size() - 1) out << "| ";
        }
        out << "\n";
    }
    return out.str();
}
} // namespace legacy

// AFTER refactoring: cleaner implementation
namespace refactored {
std::string format_table(const std::vector<std::vector<std::string>>& rows) {
    constexpr size_t col_width = 15;
    std::ostringstream out;
    for (const auto& row : rows) {
        for (size_t i = 0; i < row.size(); ++i) {
            out << std::setw(col_width) << std::left << row[i];
            if (i < row.size() - 1) out << "| ";
        }
        out << "\n";
    }
    return out.str();
}
} // namespace refactored

// The approval test catches behavioral differences
TEST_CASE("format_table output matches approved baseline") {
    std::vector<std::vector<std::string>> data = {
        {"Name", "Age", "City"},
        {"Alice", "30", "NYC"},
        {"Bob", "25", "London"},
        {"Charlie", "35", "Tokyo"},
    };

    // Step 1: Run with legacy:: -> approve the output
    // Step 2: Switch to refactored:: -> if output differs, test FAILS
    // Step 3: Inspect diff -> decide if change is intentional

    std::string output = refactored::format_table(data);
    ApprovalTests::Approvals::verify(output);

    // If setw pads differently than manual padding,
    // the diff tool shows EXACTLY which lines changed
}

// Workflow for safe refactoring
/*
Step 1: Write approval tests BEFORE touching legacy code
        -> Run tests -> approve baseline outputs

Step 2: Refactor the implementation
        -> Change internal structure
        -> DO NOT change behavior

Step 3: Run approval tests
        -> PASS = behavior unchanged
        -> FAIL = diff shows what changed
            -> If intentional: re-approve
            -> If accidental: fix the bug

Step 4: Once all approval tests pass, optionally convert
        some to assertion-based tests for clarity
*/
```

The re-approve step in the workflow is worth spelling out: if you refactored intentionally and the output *should* change (say, you fixed a formatting bug), you simply approve the new output. The approved file becomes the new specification. If the change was accidental, you fix the code until the test passes again.

### Q3: Explain how approval tests differ from assertion-based tests for complex output structures

**Answer:**

The contrast here is clearest with structured output like JSON. An assertion-based test for a JSON response typically checks a handful of fields with individual `REQUIRE` calls, which means the test can pass even if unexpected fields are present, the structure is wrong, or the formatting changed. An approval test captures the entire output in one shot - nothing can change without the test noticing.

```cpp
// Comparison: assertion vs approval for JSON output

#include "ApprovalTests.hpp"
#include <catch2/catch.hpp>
#include <string>

// Complex function that generates JSON
std::string generate_user_json(const std::string& name, int age) {
    return R"({
  "user": {
    "name": ")" + name + R"(",
    "age": )" + std::to_string(age) + R"(,
    "permissions": ["read", "write"],
    "metadata": {
      "created": "2024-01-15",
      "version": 2
    }
  }
})";
}

// Assertion-based: tedious for complex structures
TEST_CASE("User JSON (assertion-based)") {
    std::string json = generate_user_json("Alice", 30);

    // Must manually check every field - error prone!
    REQUIRE(json.find("\"name\": \"Alice\"") != std::string::npos);
    REQUIRE(json.find("\"age\": 30") != std::string::npos);
    REQUIRE(json.find("\"read\"") != std::string::npos);
    REQUIRE(json.find("\"write\"") != std::string::npos);
    REQUIRE(json.find("\"version\": 2") != std::string::npos);
    // 5 assertions and we STILL didn't verify the full structure!
    // What if there's an extra field we didn't check?
}

// Approval-based: capture the ENTIRE output
TEST_CASE("User JSON (approval-based)") {
    std::string json = generate_user_json("Alice", 30);
    ApprovalTests::Approvals::verify(json);
    // One line captures EVERYTHING:
    // - Field names, values, nesting, order, formatting
    // - If ANYTHING changes, test fails and shows a diff
}

// When each approach wins
/*
Use ASSERTION-BASED when:
  - Output is simple (a number, a bool, a short string)
  - You care about specific properties, not full output
  - Output format changes often but semantics stay

Use APPROVAL-BASED when:
  - Output is complex (reports, HTML, JSON, binary formats)
  - You're characterizing legacy code (don't know expected values)
  - You want to protect against ANY behavioral change during refactoring
  - Output is hard to construct manually (multi-line, structured)

Best practice: COMBINE both:

  - Approval tests for full-output regression protection
  - Assertion tests for key semantic invariants

*/
```

The "combine both" recommendation at the end is worth following in practice. Use an approval test to lock down the full output of a complex function, and then add a targeted assertion test for the one invariant that really matters semantically. The approval test catches accidental regressions; the assertion test documents intent.

---

## Notes

- ApprovalTests.cpp: header-only, integrates with Catch2, doctest, Google Test, and CppUnit
- Install: single-header `ApprovalTests.hpp` or via vcpkg/Conan
- `.approved.txt` files **must be committed to version control** - they ARE the specification
- Configure diff tool: `ApprovalTests::DiffReporter::setFrontLoadedReporter(ApprovalTests::DiffReporter::BeyondCompare)`
- Combination approvals reduce test explosion: one test times N inputs = one approved file
