# Understand and prevent race conditions in security-critical code

**Category:** Safety and Security  
**Standard:** C++11  
**Reference:** <https://clang.llvm.org/docs/ThreadSanitizer.html>  

---

## Topic Overview

Race conditions in security-critical code allow attackers to exploit the time gap between a security check and the use of the checked resource. The most dangerous class is TOCTOU (Time-Of-Check-To-Time-Of-Use), where the state of a resource changes between checking and acting. In multithreaded code, unsynchronized access to shared credentials, authorization flags, or security tokens creates exploitable data races.

The core problem is that "check" and "use" feel like a single logical operation in code, but the CPU executes them as two separate events with time between them. That time gap is the attack window.

### TOCTOU Attack Model

Here's the file system version of TOCTOU, which is the classic example. The attacker's entire strategy is to squeeze actions into the gap between step 1 and step 3:

```cpp
TOCTOU (file system example):

Thread/Process A (victim):         Attacker:
---------------------------        ---------

1. access("/tmp/data", R_OK)

   -> returns OK (file is readable)

                                   2. rm /tmp/data

                                      ln -s /etc/shadow /tmp/data

3. open("/tmp/data", O_RDONLY)

   -> opens /etc/shadow!
   -> reads password hashes!

Time gap between CHECK (step 1) and USE (step 3) is the attack window.
```

The fix is always the same: eliminate the gap by either making the check-and-use atomic, or by performing the check on the already-opened resource rather than on the path.

### Race Condition Categories in Security Code

| Category | Example | Impact |
| --- | --- | --- |
| File TOCTOU | Check permission then open | Privilege escalation |
| Credential race | Check auth flag then grant access | Authorization bypass |
| Double-fetch | Read user input twice from shared memory | Input validation bypass |
| Signal handler race | Async signal modifies shared state | Memory corruption |
| Lazy init race | `if (!init) { init = true; ... }` | Double init, state corruption |

### Core Example

Here's the vulnerable pattern and its fix side by side. The key difference is that the safe version gets a file handle first and validates the opened handle, rather than validating the path and then opening it:

```cpp
#include <filesystem>
#include <fstream>
#include <iostream>
#include <mutex>
#include <string>

// VULNERABLE: TOCTOU race
bool unsafe_read_file(const std::string& path) {
    // CHECK: does file exist and is it a regular file?
    if (std::filesystem::is_regular_file(path)) {
        // <- RACE WINDOW: attacker replaces file with symlink here ->
        // USE: open and read the file
        std::ifstream file(path);   // may open a different file!
        std::string content;
        std::getline(file, content);
        std::cout << content << "\n";
        return true;
    }
    return false;
}

// SAFE: open first, then validate
bool safe_read_file(const std::string& path) {
    // Open FIRST - get a handle to the actual file
    std::ifstream file(path);
    if (!file) return false;

    // Now validate properties of the OPENED file (not the path)
    // In production: use fstat() on the file descriptor to check ownership,
    // permissions, and that it's not a symlink - all on the already-opened handle.

    std::string content;
    std::getline(file, content);
    std::cout << content << "\n";
    return true;
}
```

Once you have a file descriptor (from `open()` or an `ifstream`), the file it refers to cannot be swapped out from under you. That's the invariant that makes the safe version work.

---

## Self-Assessment

### Q1: Show a TOCTOU (time-of-check-to-time-of-use) race in a file permission check

**Answer:**

This example makes the race window explicit with a `sleep_for` call. In a real attack, the attacker doesn't need the sleep to exist - they just need any time gap, and modern systems have plenty of them between system calls:

```cpp
#include <iostream>
#include <fstream>
#include <cstring>
#include <string>
#include <thread>
#include <chrono>

// On POSIX systems, this would use:
//   #include <unistd.h>
//   #include <sys/stat.h>
// For portability, we simulate with std::filesystem

#include <filesystem>
namespace fs = std::filesystem;

// VULNERABLE: Classic TOCTOU

bool vulnerable_write(const std::string& filepath, const std::string& data) {
    // STEP 1 (CHECK): Is this a regular file we own?
    auto status = fs::status(filepath);

    if (!fs::is_regular_file(status)) {
        std::cerr << "Not a regular file!\n";
        return false;
    }

    // +--------------------------------------------------+
    // | RACE WINDOW: attacker can do:                     |
    // |   rm /tmp/output.txt                              |
    // |   ln -s /etc/passwd /tmp/output.txt               |
    // | Now the path points to /etc/passwd!               |
    // +--------------------------------------------------+

    // Simulate the race window
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    // STEP 2 (USE): Open and write - but the file may have changed!
    std::ofstream out(filepath);
    if (!out) return false;
    out << data;
    // If attacker swapped the file, we just wrote to /etc/passwd!

    return true;
}

// SAFE: Eliminate the race

// Approach 1: Open first, check the opened fd
// (requires POSIX fstat on the file descriptor)

// Approach 2: Use O_NOFOLLOW to refuse symlinks
// int fd = open(path, O_WRONLY | O_NOFOLLOW);
// if (fd < 0 && errno == ELOOP) -> it's a symlink, reject!

// Approach 3: Use a directory fd + openat for relative operations
// int dirfd = open("/tmp", O_RDONLY | O_DIRECTORY);
// int fd = openat(dirfd, "output.txt", O_WRONLY | O_CREAT | O_NOFOLLOW, 0600);

// Approach 4: For temporary files, use mkstemp (creates + opens atomically)
// char tmpl[] = "/tmp/myapp.XXXXXX";
// int fd = mkstemp(tmpl);  // atomic create + open, unique name

int main() {
    // Create a test file
    const std::string path = "test_toctou.txt";
    {
        std::ofstream f(path);
        f << "original content";
    }

    // Show the vulnerability
    std::cout << "File exists: " << fs::exists(path) << "\n";
    std::cout << "Is regular: " << fs::is_regular_file(path) << "\n";

    // In a real attack, another process would replace the file
    // between the check and the use above.

    // Clean up
    fs::remove(path);

    std::cout << "TOCTOU demo complete. In real attacks, the time gap\n"
              << "between check and use allows file substitution.\n";
    // Output:
    // File exists: 1
    // Is regular: 1
    // TOCTOU demo complete. In real attacks, the time gap
    // between check and use allows file substitution.
}
```

The vulnerable code checks `is_regular_file(path)` and then later opens the same path. Between those two operations, an attacker replaces the file with a symlink to a sensitive target. The fix is to never separate "check" from "use" - instead, open the file first (getting a file descriptor), then validate properties of the opened handle using `fstat()`.

### Q2: Fix the race using O_CREAT | O_EXCL atomic file creation

**Answer:**

The key insight here is that checking "does it exist?" and then creating it are two separate operations with a gap. The kernel provides a way to do both atomically - and C++23 finally exposes it portably:

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <cerrno>
#include <cstring>
#include <filesystem>

// POSIX-specific (Linux/macOS)
#ifdef __linux__
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#endif

// VULNERABLE: Check-then-create race

bool vulnerable_create(const std::string& path) {
    namespace fs = std::filesystem;

    // CHECK: does file already exist?
    if (fs::exists(path)) {
        std::cerr << "File already exists!\n";
        return false;
    }

    // <- RACE WINDOW ->
    // Attacker creates a symlink at `path` pointing to /etc/crontab

    // CREATE: open for writing - but may now follow symlink!
    std::ofstream out(path);
    out << "cron job content";  // writes to /etc/crontab!
    return true;
}

// SAFE: O_CREAT | O_EXCL - atomic create

#ifdef __linux__
int safe_create_posix(const char* path) {
    // O_CREAT: create file if it doesn't exist
    // O_EXCL:  FAIL if file already exists (atomic with O_CREAT)
    // O_WRONLY: write-only
    // O_NOFOLLOW: refuse to follow symlinks (extra safety)
    // 0600: owner read/write only

    int fd = ::open(path, O_CREAT | O_EXCL | O_WRONLY | O_NOFOLLOW, 0600);

    if (fd < 0) {
        if (errno == EEXIST) {
            std::cerr << "File already exists (atomic check)\n";
        } else if (errno == ELOOP) {
            std::cerr << "Path is a symlink (O_NOFOLLOW)\n";
        } else {
            std::cerr << "Error: " << std::strerror(errno) << "\n";
        }
        return -1;
    }

    // fd is guaranteed to be a NEW file, not a pre-existing one.
    // No race window between check and create!

    // Write data via the file descriptor
    const char* data = "safe content\n";
    ::write(fd, data, std::strlen(data));
    ::close(fd);

    return 0;
}
#endif

// C++ cross-platform approximation

bool safe_create_cpp(const std::string& path) {
    namespace fs = std::filesystem;

    // std::ofstream with std::ios::noreplace (C++23!)
    // This is the C++ equivalent of O_CREAT | O_EXCL
    #if __cplusplus >= 202302L
    std::ofstream out(path, std::ios::out | std::ios::noreplace);
    if (!out) {
        std::cerr << "File already exists or cannot create\n";
        return false;
    }
    out << "safe content\n";
    return true;
    #else
    // Pre-C++23: no portable atomic create.
    // Use platform API (open with O_EXCL on POSIX, CreateFile with CREATE_NEW on Windows)
    std::cerr << "Need C++23 for ios::noreplace\n";
    return false;
    #endif
}

int main() {
    // Why O_CREAT | O_EXCL works
    //
    // The kernel performs the existence check AND creation in a single
    // atomic system call. There is NO gap between check and create.
    //
    // Timeline comparison:
    //
    // VULNERABLE (two operations):
    //   1. stat(path) -> "doesn't exist"        <- CHECK
    //      [attacker creates symlink here]
    //   2. open(path, O_CREAT) -> follows symlink <- USE
    //
    // SAFE (single atomic operation):
    //   1. open(path, O_CREAT|O_EXCL) -> kernel checks + creates atomically
    //      [attacker cannot interpose between check and create]
    //      If file exists -> EEXIST error, no file opened
    //
    // C++23 equivalent: std::ios::noreplace flag for ofstream

    #ifdef __linux__
    safe_create_posix("/tmp/test_safe_create.txt");
    // Second call fails atomically:
    safe_create_posix("/tmp/test_safe_create.txt");
    ::unlink("/tmp/test_safe_create.txt");
    #endif

    safe_create_cpp("test_noreplace.txt");

    std::cout << "Demo complete.\n";
    // Output (Linux):
    // File already exists (atomic check)
    // Demo complete.
}
```

`O_CREAT | O_EXCL` makes the kernel perform the existence check and file creation as a single atomic operation - there is no time gap for an attacker to exploit. In C++23, `std::ios::noreplace` provides the same guarantee portably. Combined with `O_NOFOLLOW` (refuse symlinks), this eliminates both TOCTOU races and symlink attacks.

### Q3: Use ThreadSanitizer to detect a race between a credential check and a capability use

**Answer:**

Shared auth state is especially dangerous because a race here doesn't just corrupt data - it corrupts your security model. An attacker who can race two threads can potentially gain access they shouldn't have:

```cpp
#include <iostream>
#include <thread>
#include <string>
#include <atomic>
#include <mutex>

// Compile with TSan: clang++ -fsanitize=thread -g -O1 -std=c++20 race.cpp

// VULNERABLE: Race on shared auth state

struct UnsafeAuthContext {
    bool authenticated = false;    // SHARED, no synchronization!
    std::string username;          // SHARED, no synchronization!
    int privilege_level = 0;       // SHARED, no synchronization!

    void login(const std::string& user, const std::string& password) {
        if (password == "secret123") {
            username = user;        // DATA RACE: unsynchronized write
            privilege_level = 1;    // DATA RACE
            authenticated = true;   // DATA RACE
            // Note: even if this were atomic, the 3-step update is not atomic
        }
    }

    void perform_admin_action() {
        if (authenticated) {               // DATA RACE: unsynchronized read
            // <- Another thread may be mid-login, setting fields ->
            if (privilege_level >= 2) {     // DATA RACE
                std::cout << "Admin action by " << username << "\n"; // DATA RACE
            }
        }
    }

    void logout() {
        authenticated = false;     // DATA RACE
        privilege_level = 0;       // DATA RACE
        username.clear();          // DATA RACE
    }
};

// SAFE: Properly synchronized auth state

struct SafeAuthContext {
    mutable std::mutex mtx;
    bool authenticated = false;
    std::string username;
    int privilege_level = 0;

    void login(const std::string& user, const std::string& password) {
        std::lock_guard lock(mtx);  // atomic state transition
        if (password == "secret123") {
            username = user;
            privilege_level = 1;
            authenticated = true;
            // ALL fields updated under ONE lock - consistent state guaranteed
        }
    }

    bool perform_admin_action() {
        std::lock_guard lock(mtx);  // read consistent snapshot
        if (authenticated && privilege_level >= 2) {
            std::cout << "Admin action by " << username << "\n";
            return true;
        }
        return false;
    }

    void elevate(int level) {
        std::lock_guard lock(mtx);
        if (authenticated) {
            privilege_level = level;
        }
    }

    void logout() {
        std::lock_guard lock(mtx);  // atomic state transition
        authenticated = false;
        privilege_level = 0;
        username.clear();
    }
};

int main() {
    // Trigger TSan detection on unsafe version

    UnsafeAuthContext unsafe_ctx;

    std::thread t1([&] {
        unsafe_ctx.login("admin", "secret123");
    });

    std::thread t2([&] {
        unsafe_ctx.perform_admin_action();
    });

    std::thread t3([&] {
        unsafe_ctx.logout();
    });

    t1.join();
    t2.join();
    t3.join();

    // TSan output (abbreviated):
    // ==================
    // WARNING: ThreadSanitizer: data race (pid=12345)
    //   Write of size 1 at 0x... by thread T1:
    //     #0 UnsafeAuthContext::login() at race.cpp:15
    //   Previous read of size 1 at 0x... by thread T2:
    //     #0 UnsafeAuthContext::perform_admin_action() at race.cpp:22
    //   Location is global 'unsafe_ctx' of size 72 at 0x...
    // ==================

    // Safe version: no TSan warnings

    SafeAuthContext safe_ctx;

    std::thread s1([&] {
        safe_ctx.login("admin", "secret123");
        safe_ctx.elevate(2);
    });

    std::thread s2([&] {
        safe_ctx.perform_admin_action();
    });

    std::thread s3([&] {
        safe_ctx.logout();
    });

    s1.join();
    s2.join();
    s3.join();

    std::cout << "Race condition demo complete.\n";
    // Output (with TSan): Multiple data race warnings for unsafe_ctx
    // Safe version: clean, no warnings
}
```

TSan instruments every memory access and tracks which thread last wrote or read each byte. When it detects two threads accessing the same memory location with at least one write and no synchronization between them, it reports a data race with full stack traces. The fix for security-critical auth state: protect ALL related fields with a single mutex so state transitions are atomic and consistent. TSan is a runtime tool - compile with `-fsanitize=thread` and run your test suite.

---

## Notes

- **CWE-367 (TOCTOU Race Condition)** is in the Top 25 most dangerous software weaknesses.
- **`O_NOFOLLOW`** is critical for setuid/setgid programs that operate on files in world-writable directories like `/tmp`.
- **`std::ios::noreplace`** (C++23) is the portable equivalent of `O_CREAT | O_EXCL`.
- **TSan cannot detect TOCTOU** - it only detects data races in shared memory. TOCTOU is a file system race, not a threading race.
- **Double-fetch bugs:** Reading user data from shared memory twice - once to validate, once to use. Fix: copy data to local storage, validate and use the copy.
- Compile with `-fsanitize=thread -g -O1` for TSan (incompatible with ASan - use separately).
