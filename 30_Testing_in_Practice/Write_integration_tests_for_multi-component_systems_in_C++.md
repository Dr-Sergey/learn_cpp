# Write integration tests for multi-component systems in C++

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Integration tests** verify that separately developed modules work together correctly. Unlike unit tests (isolated, fast) or system tests (full stack), integration tests exercise real interactions between 2-3 components - databases, network layers, file systems, or inter-module APIs.

The reason integration tests exist as a separate tier is that unit tests, by design, mock out dependencies. You can have a perfectly green unit test suite and still have a wiring bug where component A produces output in a format that component B cannot parse. Integration tests catch exactly that class of problem - the mismatches at the seams.

### Test Pyramid Positioning

```cpp
         /  System  \         <- Few, slow, end-to-end
        / Integration \       <- Moderate, verify wiring
       /    Unit Tests  \     <- Many, fast, isolated
      /___________________\
```

| Aspect | Unit | Integration | System |
| --- | --- | --- | --- |
| Components | 1 | 2-5 | All |
| Mocking | Heavy | Minimal | None |
| Speed | ms | seconds | minutes |
| Flakiness | Rare | Moderate | Common |
| Setup cost | Low | Medium | High |

---

## Self-Assessment

### Q1: Write integration tests for components with real dependencies

**Answer:**

This test exercises the real interaction between `ConfigParser` and `ConnectionManager`. There is no mocking of file I/O - the test actually creates a temporary file on disk and reads it. That is intentional: you want to verify that the real parser, reading a real file, produces output that the connection manager can actually use. The fixture handles cleanup so no test artifacts remain after the run.

```cpp
#include <gtest/gtest.h>
#include <filesystem>
#include <fstream>
#include <memory>
#include <string>

namespace fs = std::filesystem;

// === Components under test ===
class ConfigParser {
public:
    struct Config {
        std::string host;
        int port;
        bool tls_enabled;
    };

    Config parse(const fs::path& path) {
        std::ifstream file(path);
        if (!file) throw std::runtime_error("Cannot open: " + path.string());

        Config cfg;
        std::string line;
        while (std::getline(file, line)) {
            auto eq = line.find('=');
            if (eq == std::string::npos) continue;
            auto key = line.substr(0, eq);
            auto val = line.substr(eq + 1);
            if (key == "host") cfg.host = val;
            else if (key == "port") cfg.port = std::stoi(val);
            else if (key == "tls") cfg.tls_enabled = (val == "true");
        }
        return cfg;
    }
};

class ConnectionManager {
public:
    struct Connection {
        std::string address;
        bool connected = false;
    };

    Connection connect(const ConfigParser::Config& cfg) {
        std::string addr = (cfg.tls_enabled ? "tls://" : "tcp://")
                           + cfg.host + ":" + std::to_string(cfg.port);

        return {addr, true};
    }
};

// === Integration test fixture ===
class ConfigToConnectionTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Create temporary directory for test files
        test_dir_ = fs::temp_directory_path() / "integration_test";
        fs::create_directories(test_dir_);
    }

    void TearDown() override {
        fs::remove_all(test_dir_);
    }

    fs::path write_config(const std::string& content) {
        auto path = test_dir_ / "test.conf";
        std::ofstream(path) << content;
        return path;
    }

    fs::path test_dir_;
    ConfigParser parser_;
    ConnectionManager conn_mgr_;
};

// Integration: Config file -> Parser -> ConnectionManager
TEST_F(ConfigToConnectionTest, TlsConfigProducesTlsConnection) {
    auto path = write_config("host=example.com\nport=443\ntls=true");

    auto config = parser_.parse(path);  // Real file I/O
    auto conn = conn_mgr_.connect(config);  // Real component interaction

    EXPECT_EQ(conn.address, "tls://example.com:443");
    EXPECT_TRUE(conn.connected);
}

TEST_F(ConfigToConnectionTest, PlainConfigProducesTcpConnection) {
    auto path = write_config("host=localhost\nport=8080\ntls=false");

    auto config = parser_.parse(path);
    auto conn = conn_mgr_.connect(config);

    EXPECT_EQ(conn.address, "tcp://localhost:8080");
}

TEST_F(ConfigToConnectionTest, MissingFileThrows) {
    EXPECT_THROW(parser_.parse(test_dir_ / "nonexistent.conf"),
                 std::runtime_error);
}
```

The `TearDown` calling `fs::remove_all` is essential. Without it, repeated test runs accumulate temp directories, and the tests may fail if a stale file from a previous run confuses the parser. Always clean up in `TearDown`, not in `SetUp` - if setup crashes midway, `TearDown` still runs.

### Q2: Integration test a pipeline of processing stages

**Answer:**

This test exercises a four-stage pipeline end to end. Notice that intermediate results are verified at each stage boundary - this is a good practice because if the pipeline produces a wrong final result, the per-stage assertions tell you which stage broke.

```cpp
#include <gtest/gtest.h>
#include <vector>
#include <string>
#include <sstream>
#include <algorithm>
#include <numeric>

// === Processing pipeline: Read -> Transform -> Aggregate -> Format ===
class DataReader {
public:
    std::vector<double> read(std::istream& input) {
        std::vector<double> data;
        double val;
        while (input >> val) data.push_back(val);
        return data;
    }
};

class DataTransformer {
public:
    std::vector<double> normalize(const std::vector<double>& data) {
        if (data.empty()) return {};
        auto [min_it, max_it] = std::minmax_element(data.begin(), data.end());
        double range = *max_it - *min_it;
        if (range == 0.0) return std::vector<double>(data.size(), 0.0);

        std::vector<double> result;
        result.reserve(data.size());
        for (double v : data)
            result.push_back((v - *min_it) / range);
        return result;
    }
};

class Aggregator {
public:
    struct Stats {
        double mean;
        double min;
        double max;
        size_t count;
    };

    Stats compute(const std::vector<double>& data) {
        if (data.empty()) return {0, 0, 0, 0};
        auto [min_it, max_it] = std::minmax_element(data.begin(), data.end());
        double sum = std::accumulate(data.begin(), data.end(), 0.0);
        return {sum / data.size(), *min_it, *max_it, data.size()};
    }
};

class ReportFormatter {
public:
    std::string format(const Aggregator::Stats& stats) {
        std::ostringstream os;
        os << "Count: " << stats.count
           << ", Mean: " << stats.mean
           << ", Range: [" << stats.min << ", " << stats.max << "]";
        return os.str();
    }
};

// === Integration test: full pipeline ===
class DataPipelineTest : public ::testing::Test {
protected:
    DataReader reader_;
    DataTransformer transformer_;
    Aggregator aggregator_;
    ReportFormatter formatter_;
};

TEST_F(DataPipelineTest, FullPipelineProducesCorrectReport) {
    std::istringstream input("10 20 30 40 50");

    // Stage 1: Read
    auto data = reader_.read(input);
    ASSERT_EQ(data.size(), 5);

    // Stage 2: Normalize
    auto normalized = transformer_.normalize(data);
    EXPECT_DOUBLE_EQ(normalized.front(), 0.0);
    EXPECT_DOUBLE_EQ(normalized.back(), 1.0);

    // Stage 3: Aggregate
    auto stats = aggregator_.compute(normalized);
    EXPECT_EQ(stats.count, 5);
    EXPECT_DOUBLE_EQ(stats.min, 0.0);
    EXPECT_DOUBLE_EQ(stats.max, 1.0);

    // Stage 4: Format
    auto report = formatter_.format(stats);
    EXPECT_THAT(report, ::testing::HasSubstr("Count: 5"));
    EXPECT_THAT(report, ::testing::HasSubstr("Mean: 0.5"));
}

TEST_F(DataPipelineTest, EmptyInputProducesEmptyReport) {
    std::istringstream input("");
    auto data = reader_.read(input);
    auto normalized = transformer_.normalize(data);
    auto stats = aggregator_.compute(normalized);
    auto report = formatter_.format(stats);

    EXPECT_THAT(report, ::testing::HasSubstr("Count: 0"));
}
```

Using `HasSubstr` rather than `EXPECT_EQ` for the formatted report is deliberate. You want the test to survive formatting changes (adding decimal places, reordering fields) as long as the semantic content is correct. Testing the full string literal would make the test brittle.

### Q3: Manage test environments and resource lifecycle

**Answer:**

When integration tests need a resource that is expensive to create - a real database schema, a server process, a large test data set - you want to create it once and share it across the entire test binary. `::testing::Environment` is the right tool for that. Per-test isolation is then achieved by using transactions and rolling back after each test.

```cpp
#include <gtest/gtest.h>
#include <filesystem>
#include <memory>

namespace fs = std::filesystem;

// === RAII test environment for expensive shared resources ===
class DatabaseEnvironment : public ::testing::Environment {
public:
    void SetUp() override {
        // One-time setup for all tests in this binary
        db_path_ = fs::temp_directory_path() / "test_integration.db";
        // In real code: create database, run migrations
    }

    void TearDown() override {
        fs::remove(db_path_);
    }

    static fs::path db_path_;
};

fs::path DatabaseEnvironment::db_path_;

// Register globally
auto* const db_env = ::testing::AddGlobalTestEnvironment(
    new DatabaseEnvironment);

// === Fixture with per-test isolation ===
class DatabaseIntegrationTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Per-test: begin transaction for isolation
        // In real code: db.begin_transaction();
    }

    void TearDown() override {
        // Per-test: rollback to ensure isolation
        // In real code: db.rollback();
    }
};

// === Retry flaky integration tests (CTest) ===
// CMakeLists.txt:
//   set_tests_properties(integration_tests PROPERTIES
//       ENVIRONMENT "TEST_DB_PATH=/tmp/test.db"
//       TIMEOUT 30
//       LABELS "integration"
//   )
//
// Run only integration tests:
//   ctest --label-regex integration
//   ctest --rerun-failed --output-on-failure
```

The `LABELS "integration"` CTest property is important for team workflow. Developers running tests locally can skip the slow integration suite with `ctest --exclude-labels integration`, while CI always runs everything. The label makes it easy to enforce that policy without any manual filtering.

---

## Notes

- Integration tests should use **real components** but may stub external services (databases, networks).
- Use temp directories with RAII cleanup - never leave test artifacts on the filesystem.
- Label integration tests separately (`LABELS "integration"`) so devs can skip them locally.
- Integration tests are inherently slower - keep the count lower than unit tests.
- Use `::testing::Environment` for expensive shared setup (database creation, server startup).
- Use per-test transactions/rollbacks for isolation between tests in the same suite.
- Flaky integration tests often indicate race conditions - fix the code, don't just retry.
- Run integration tests in CI with longer timeouts than unit tests.
