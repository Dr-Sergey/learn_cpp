# Use Tracy, Orbit, or Superluminal for real-time profiling

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://github.com/wolfpld/tracy>  

---

## Topic Overview

When you have found a performance problem and need to understand exactly where time is going, a real-time profiler gives you a timeline view of your application's CPU activity as it runs. These tools are different from simple benchmarks: instead of measuring a single function in isolation, they show you the whole picture - which functions overlap, how long each frame took, where threads are waiting on each other, and how memory usage evolves.

The landscape of real-time profilers for C++ breaks down by platform and approach:

| Tool | Platform | Type | License |
| --- | --- | --- | --- |
| **Tracy** | Cross-platform | Instrumentation + sampling | MIT |
| **Orbit** | Linux/Windows | Sampling + dynamic instrumentation | BSD |
| **Superluminal** | Windows | Sampling + context switches | Commercial |
| **perf** | Linux | Sampling | GPL |
| **Intel VTune** | Cross-platform | HW counters + sampling | Free |

The distinction between sampling and instrumentation matters in practice. Sampling profilers interrupt the process periodically and record the call stack - they need no code changes and see everything, including third-party libraries. Instrumentation profilers require you to mark regions of code explicitly, but they give you exact timing for those regions and can show you a detailed timeline. In practice, you often want both: sample to find the hotspot, then instrument it for precise measurement.

---

## Self-Assessment

### Q1: Instrument with Tracy zones and visualize

Tracy's API is built around "zones" - regions of code that you explicitly mark with macros. When you add `ZoneScoped` at the top of a function, Tracy records the entry and exit times and sends them to the UI over a network connection. The UI then renders a real-time flame chart. The macro-based approach means the overhead is nearly zero when a zone is not being recorded.

<!-- compile: needs third-party library header `tracy/Tracy.hpp` -->
```cpp
// tracy_demo.cpp
// Link with Tracy: add tracy as a dependency or include TracyClient.cpp
#include <tracy/Tracy.hpp>
#include <vector>
#include <algorithm>
#include <random>
#include <thread>
#include <chrono>

void parse_data(const std::vector<int>& data) {
    ZoneScoped;  // Tracy: auto-named zone from function name
    // Parsing work...
    int sum = 0;
    for (int x : data) sum += x;
    (void)sum;
}

void sort_data(std::vector<int>& data) {
    ZoneScopedN("CustomSort");  // Tracy: named zone
    std::sort(data.begin(), data.end());
}

void process_frame(int frame_num) {
    ZoneScoped;
    ZoneValue(frame_num);  // attach frame number to zone

    std::vector<int> data(100000);
    {
        ZoneScopedN("Generate");  // nested zone
        std::mt19937 rng(frame_num);
        std::generate(data.begin(), data.end(), rng);
    }

    parse_data(data);
    sort_data(data);

    FrameMark;  // Tracy: mark end of frame
}

int main() {
    for (int i = 0; i < 100; ++i) {
        process_frame(i);
    }
}
```

Integrate Tracy into your build system like this:

```cmake
# CMake integration:
find_package(Tracy CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
target_compile_definitions(myapp PRIVATE TRACY_ENABLE)

# Or via vcpkg:
# vcpkg install tracy
```

Once running, the Tracy UI streams data in real time and shows a timeline like this:

```cpp
Tracy profiler UI shows:
┌─────────────────────────────────────────────────┐
│ Timeline (horizontal = time)                    │
│                                                 │
│ Thread 1: █process_frame██████████████████████│
│           │Generate│parse  │ CustomSort       │  │
│                                                 │
│ Frame: 42  Duration: 2.3ms                      │
│ Zones: 4  CPU: 98%                              │
└─────────────────────────────────────────────────┘
```

You can click on any zone to see its exact duration and which child zones contributed most. The `FrameMark` call tells Tracy where one game/simulation frame ends and the next begins, which enables per-frame statistics.

### Q2: Detect lock contention with a profiler

Lock contention is one of the most common causes of multi-threaded performance problems, and it is almost impossible to spot by just reading code. When eight threads all need the same lock, seven of them spend most of their time waiting. Tracy can track this directly.

<!-- compile: needs third-party library header `tracy/Tracy.hpp` -->
```cpp
// contention.cpp
#include <tracy/Tracy.hpp>
#include <mutex>
#include <thread>
#include <vector>

std::mutex global_mutex;
std::vector<int> shared_data;

void worker(int id) {
    ZoneScoped;
    for (int i = 0; i < 100000; ++i) {
        {
            // Tracy tracks lock contention:
            std::lock_guard lock(global_mutex);
            LockMark(global_mutex);  // Tracy: mark lock acquisition
            shared_data.push_back(id * 100000 + i);
        }
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i)
        threads.emplace_back(worker, i);
    for (auto& t : threads)
        t.join();
}

// Tracy shows:
// - Lock wait time per thread (red bars = contention)
// - Which thread held the lock when others were waiting
// - Total lock contention as percentage of wall time
//
// Fix: shard the data, use lock-free queue, reduce critical section
```

For more detailed tracking, you can wrap your mutex type itself with Tracy's annotation:

```cpp
// Tracy lock tracking (advanced):
#include <tracy/Tracy.hpp>

// Annotate your mutex:
TracyLockable(std::mutex, my_mutex);  // wraps mutex with tracking

void worker() {
    std::lock_guard<LockableBase(std::mutex)> lock(my_mutex);
    // Tracy now tracks: wait time, hold time, contention count
}
```

The wrapped mutex records acquisition and release times automatically, so the timeline shows exactly how long each thread spent waiting versus working.

### Q3: Sampling vs instrumentation profilers

The reason this trips people up is that both types of profiler have compelling advantages, and neither is strictly better. Understanding the trade-offs helps you pick the right tool for the job.

| Aspect | Sampling (perf, VTune) | Instrumentation (Tracy, Orbit) |
| --- | --- | --- |
| **How** | Periodically samples stack | Code explicitly marks zones |
| **Overhead** | Low (~1-5%) | Variable (per instrumentation point) |
| **Precision** | Statistical (more samples = better) | Exact timing for each zone |
| **Setup** | No code changes | Must add macros/annotations |
| **Timeline** | Post-mortem analysis | Real-time live streaming (Tracy) |
| **Visibility** | All functions (even 3rd party) | Only instrumented functions |
| **Best for** | Finding unknown hotspots | Understanding known hot paths |
| **Frame-based** | Not natively | Yes (Tracy FrameMark) |

The typical workflow that gets the best of both is a two-pass approach:

```cpp
Typical workflow:

1. Use sampling profiler (perf) to FIND the hotspot
2. Instrument the hotspot with Tracy for DETAILED analysis
3. Optimize based on Tracy's precise timing
4. Verify with perf that overall time decreased
```

Start with sampling because it shows you everything without any code changes. Once you know which function is slow, add Tracy zones to understand the internal structure. Then verify your optimization actually helped by going back to the sampling profiler for a before/after comparison.

---

## Notes

- Tracy has near-zero overhead for non-active zones (roughly 1 ns per zone entry).
- Tracy supports GPU profiling for Vulkan, OpenGL, and D3D12.
- Orbit (Google) can attach to running processes without recompilation, which is useful for profiling production builds.
- Superluminal excels at context switch and thread migration analysis on Windows.
- Use `TracyAllocS`/`TracyFreeS` to track memory allocations in the timeline alongside CPU work.
