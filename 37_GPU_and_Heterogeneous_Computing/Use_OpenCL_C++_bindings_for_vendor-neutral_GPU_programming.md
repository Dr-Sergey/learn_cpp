# Use OpenCL C++ Bindings for Vendor-Neutral GPU Programming

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** <https://www.khronos.org/opencl/>  

---

## Topic Overview

OpenCL is the most widely supported open standard for heterogeneous computing, running on GPUs from NVIDIA, AMD, Intel, ARM, and even FPGAs. If you need your GPU code to run on hardware you don't control - or on hardware that doesn't support CUDA at all - OpenCL is your best bet for genuine vendor neutrality.

The OpenCL C++ bindings (`cl2.hpp` / `opencl.hpp`) wrap the verbose C API in RAII-managed C++ classes: `cl::Platform`, `cl::Device`, `cl::Context`, `cl::CommandQueue`, `cl::Buffer`, `cl::Kernel`, and `cl::Program`. These wrappers use reference counting and automatic resource release, eliminating the manual `clRelease*` calls that plague C-based OpenCL code. If you've ever written raw OpenCL C, you know how much ceremony those release calls add - the C++ bindings handle all of that for you.

OpenCL 3.0 restructured the specification so that OpenCL 1.2 features are mandatory and everything else is optional, queryable via feature macros. This means vendor-neutral code must check capabilities at runtime rather than assuming them. The C++ wrapper makes this ergonomic with `device.getInfo<CL_DEVICE_...>()` queries. For kernel development, OpenCL C (a C99 dialect with extensions) is compiled at runtime by the driver, enabling JIT optimization for the target device. Alternatively, SPIR-V intermediate representation allows ahead-of-time compilation.

| Aspect | OpenCL C API | OpenCL C++ Bindings (`cl2.hpp`) |
| --- | --- | --- |
| Resource mgmt | Manual `clCreate/clRelease` | RAII reference counting |
| Error handling | Error codes per call | Exceptions (opt-in via `#define`) |
| Kernel args | `clSetKernelArg` per arg | `kernel.setArg(i, val)` or functors |
| Buffer creation | `clCreateBuffer` + flags | `cl::Buffer(context, flags, size)` |
| Platform query | `clGetPlatformIDs` + arrays | `cl::Platform::get(&platforms)` |
| Event profiling | Manual `clGetEventProfilingInfo` | `event.getProfilingInfo<...>()` |

Here's how the pieces fit together at runtime. The kernel source is compiled by the driver, then loaded into a command queue that reads and writes device buffers:

```cpp
OpenCL Execution Model:
┌────────────┐    compile     ┌──────────┐   enqueue   ┌──────────┐
│ Kernel src │───────────────>│ Program  │────────────>│ Command  │
│ (OpenCL C) │  or SPIR-V    │ (binary) │   kernel    │  Queue   │
└────────────┘               └──────────┘             └────┬─────┘
                                                          │
                              ┌──────────┐                │
                              │ Buffers  │<───────────────┘
                              │ (device  │  read/write
                              │  memory) │  operations
                              └──────────┘
```

---

## Self-Assessment

### Q1: Set up an OpenCL C++ environment, compile a kernel at runtime, and execute a vector addition

This example walks through the full OpenCL workflow from start to finish: discover the platform, pick a device, create a context and queue, compile your kernel from a source string, allocate device buffers, set arguments, launch, and read back the results. Notice that the kernel source is just a plain string - the OpenCL driver compiles it at runtime.

<!-- compile: needs third-party library header `CL/opencl.hpp` -->
```cpp
#define CL_HPP_ENABLE_EXCEPTIONS
#define CL_HPP_TARGET_OPENCL_VERSION 300
#include <CL/opencl.hpp>
#include <iostream>
#include <vector>
#include <string>

// Kernel source (OpenCL C) -- compiled at runtime by the driver
const std::string kernel_source = R"CLC(
__kernel void vec_add(__global const float* a,
                      __global const float* b,
                      __global float* c,
                      const int n) {
    int gid = get_global_id(0);
    if (gid < n) {
        c[gid] = a[gid] + b[gid];
    }
}
)CLC";

int main() {
    try {
        // Discover platforms and devices
        std::vector<cl::Platform> platforms;
        cl::Platform::get(&platforms);

        cl::Platform platform = platforms.front();
        std::cout << "Platform: "
                  << platform.getInfo<CL_PLATFORM_NAME>() << '\n';

        std::vector<cl::Device> devices;
        platform.getDevices(CL_DEVICE_TYPE_GPU, &devices);
        cl::Device device = devices.front();
        std::cout << "Device: "
                  << device.getInfo<CL_DEVICE_NAME>() << '\n';

        // Context and command queue (RAII-managed)
        cl::Context context(device);
        cl::CommandQueue queue(context, device,
                               CL_QUEUE_PROFILING_ENABLE);

        // Compile kernel from source
        cl::Program program(context, kernel_source);
        program.build({device});
        cl::Kernel kernel(program, "vec_add");

        // Host data
        constexpr int N = 1 << 20;
        std::vector<float> ha(N, 1.0f), hb(N, 2.0f), hc(N);

        // Device buffers
        cl::Buffer da(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
                      N * sizeof(float), ha.data());
        cl::Buffer db(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
                      N * sizeof(float), hb.data());
        cl::Buffer dc(context, CL_MEM_WRITE_ONLY, N * sizeof(float));

        // Set kernel arguments and launch
        kernel.setArg(0, da);
        kernel.setArg(1, db);
        kernel.setArg(2, dc);
        kernel.setArg(3, N);

        cl::Event event;
        queue.enqueueNDRangeKernel(kernel,
                                    cl::NullRange,       // offset
                                    cl::NDRange(N),      // global
                                    cl::NDRange(256),    // local
                                    nullptr, &event);

        // Read result back
        queue.enqueueReadBuffer(dc, CL_TRUE, 0,
                                N * sizeof(float), hc.data());

        // Profile the kernel execution time
        event.wait();
        cl_ulong start = event.getProfilingInfo<
            CL_PROFILING_COMMAND_START>();
        cl_ulong end = event.getProfilingInfo<
            CL_PROFILING_COMMAND_END>();
        double ms = (end - start) * 1e-6;
        printf("Kernel time: %.3f ms\n", ms);
        printf("c[0] = %f (expected 3.0)\n", hc[0]);

    } catch (const cl::Error& e) {
        std::cerr << "OpenCL error: " << e.what()
                  << " (" << e.err() << ")\n";
        return 1;
    }
    return 0;
}
```

The `CL_MEM_COPY_HOST_PTR` flag on the buffer creation is doing the host-to-device transfer implicitly. Alternatively you can create the buffer first and then call `enqueueWriteBuffer` explicitly, which gives you more control over timing and asynchrony.

### Q2: Use OpenCL profiling events to measure and compare transfer times vs kernel execution time

One of the most common mistakes when optimizing GPU code is optimizing the kernel when the real bottleneck is actually data transfer. This example measures each phase separately - the host-to-device write, the kernel, and the device-to-host read - so you can see where your time is actually going.

<!-- compile: needs third-party library header `CL/opencl.hpp` -->
```cpp
#define CL_HPP_ENABLE_EXCEPTIONS
#define CL_HPP_TARGET_OPENCL_VERSION 300
#include <CL/opencl.hpp>
#include <iostream>
#include <vector>

double event_ms(cl::Event& e) {
    e.wait();
    cl_ulong t0 = e.getProfilingInfo<CL_PROFILING_COMMAND_START>();
    cl_ulong t1 = e.getProfilingInfo<CL_PROFILING_COMMAND_END>();
    return (t1 - t0) * 1e-6;
}

int main() {
    try {
        cl::Platform platform = cl::Platform::getDefault();
        cl::Device device = cl::Device::getDefault();
        cl::Context context(device);
        cl::CommandQueue queue(context, device,
                               CL_QUEUE_PROFILING_ENABLE);

        constexpr int N = 1 << 22;  // 16M floats
        std::vector<float> host(N, 1.0f);

        // Measure write (H2D)
        cl::Buffer buf(context, CL_MEM_READ_WRITE, N * sizeof(float));
        cl::Event write_ev;
        queue.enqueueWriteBuffer(buf, CL_FALSE, 0,
                                  N * sizeof(float), host.data(),
                                  nullptr, &write_ev);

        // Trivial kernel (scale by 2)
        const char* src = R"(
            __kernel void scale(__global float* d, int n) {
                int i = get_global_id(0);
                if (i < n) d[i] *= 2.0f;
            })";
        cl::Program prog(context, src);
        prog.build({device});
        cl::Kernel kern(prog, "scale");
        kern.setArg(0, buf);
        kern.setArg(1, N);

        cl::Event kern_ev;
        queue.enqueueNDRangeKernel(kern, cl::NullRange,
                                    cl::NDRange(N), cl::NDRange(256),
                                    nullptr, &kern_ev);

        // Read back (D2H)
        cl::Event read_ev;
        queue.enqueueReadBuffer(buf, CL_FALSE, 0,
                                 N * sizeof(float), host.data(),
                                 nullptr, &read_ev);
        queue.finish();

        printf("%-20s %8.3f ms\n", "Write (H2D):", event_ms(write_ev));
        printf("%-20s %8.3f ms\n", "Kernel:", event_ms(kern_ev));
        printf("%-20s %8.3f ms\n", "Read (D2H):", event_ms(read_ev));
        printf("%-20s %8.3f ms\n", "Total:",
               event_ms(write_ev) + event_ms(kern_ev) +
               event_ms(read_ev));

    } catch (const cl::Error& e) {
        std::cerr << "Error: " << e.what() << " (" << e.err() << ")\n";
    }
    return 0;
}
```

Notice that `CL_QUEUE_PROFILING_ENABLE` must be set at queue creation time - you can't add it later. If you forget it, the profiling info calls will return zeros or errors.

### Q3: Enumerate all OpenCL devices, query detailed capabilities, and select the best device programmatically

In a real application you often don't know what hardware the user has. This example scans all platforms and all devices, prints their capabilities, scores them with a simple heuristic, and picks the best one. The scoring function is deliberately simple - in a production system you might add weights for specific features your workload needs.

<!-- compile: needs third-party library header `CL/opencl.hpp` -->
```cpp
#define CL_HPP_ENABLE_EXCEPTIONS
#define CL_HPP_TARGET_OPENCL_VERSION 300
#include <CL/opencl.hpp>
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

struct DeviceScore {
    cl::Device device;
    std::string name;
    int score;
};

int score_device(const cl::Device& d) {
    int s = 0;
    auto type = d.getInfo<CL_DEVICE_TYPE>();
    if (type & CL_DEVICE_TYPE_GPU) s += 1000;

    s += static_cast<int>(
        d.getInfo<CL_DEVICE_MAX_COMPUTE_UNITS>() * 10);
    s += static_cast<int>(
        d.getInfo<CL_DEVICE_GLOBAL_MEM_SIZE>() / (1024 * 1024));
    s += static_cast<int>(
        d.getInfo<CL_DEVICE_MAX_CLOCK_FREQUENCY>());

    return s;
}

int main() {
    try {
        std::vector<cl::Platform> platforms;
        cl::Platform::get(&platforms);

        std::vector<DeviceScore> all_devices;

        for (auto& plat : platforms) {
            std::cout << "=== Platform: "
                      << plat.getInfo<CL_PLATFORM_NAME>() << " ===\n";

            std::vector<cl::Device> devs;
            plat.getDevices(CL_DEVICE_TYPE_ALL, &devs);

            for (auto& d : devs) {
                auto name = d.getInfo<CL_DEVICE_NAME>();
                auto version = d.getInfo<CL_DEVICE_OPENCL_C_VERSION>();
                auto cu = d.getInfo<CL_DEVICE_MAX_COMPUTE_UNITS>();
                auto gmem = d.getInfo<CL_DEVICE_GLOBAL_MEM_SIZE>();
                auto lmem = d.getInfo<CL_DEVICE_LOCAL_MEM_SIZE>();
                auto wg = d.getInfo<CL_DEVICE_MAX_WORK_GROUP_SIZE>();
                auto freq = d.getInfo<CL_DEVICE_MAX_CLOCK_FREQUENCY>();

                printf("  %-40s\n", name.c_str());
                printf("    OpenCL C:      %s\n", version.c_str());
                printf("    Compute units: %u\n", cu);
                printf("    Global mem:    %lu MB\n",
                       gmem / (1024 * 1024));
                printf("    Local mem:     %lu KB\n", lmem / 1024);
                printf("    Max WG size:   %zu\n", wg);
                printf("    Clock:         %u MHz\n", freq);

                int sc = score_device(d);
                printf("    Score:         %d\n\n", sc);
                all_devices.push_back({d, name, sc});
            }
        }

        // Select best device
        auto best = std::max_element(all_devices.begin(),
                                      all_devices.end(),
            [](const DeviceScore& a, const DeviceScore& b) {
                return a.score < b.score;
            });

        std::cout << "Selected device: " << best->name
                  << " (score: " << best->score << ")\n";

    } catch (const cl::Error& e) {
        std::cerr << "Error: " << e.what() << " (" << e.err() << ")\n";
    }
    return 0;
}
```

The GPU type check (`CL_DEVICE_TYPE_GPU`) adds 1000 points, which effectively always prefers a GPU over a CPU OpenCL device. That bias is intentional - for compute workloads, a GPU is almost always the right choice when one is available.

---

## Notes

- Define `CL_HPP_ENABLE_EXCEPTIONS` before including `opencl.hpp` to get exception-based error handling instead of error codes - this makes error paths much cleaner.
- OpenCL 3.0 made most 2.x features optional - always query `CL_DEVICE_*` capabilities before using SVM, pipes, or device-side enqueue, or you'll get mysterious failures on some hardware.
- Runtime kernel compilation enables device-specific JIT optimization but adds startup latency - cache compiled binaries with `program.getInfo<CL_PROGRAM_BINARIES>()` to avoid recompiling on every launch.
- SPIR-V input (`clCreateProgramWithIL`) allows offline compilation and is the recommended path for production deployments where startup time matters.
- `CL_QUEUE_PROFILING_ENABLE` must be set at queue creation - profiling timestamps are not available without it, and there is no way to add it after the fact.
- The C++ wrapper's reference counting means copying a `cl::Buffer` increments the ref count, not the data - be explicit about ownership to avoid confusing lifetime bugs.
