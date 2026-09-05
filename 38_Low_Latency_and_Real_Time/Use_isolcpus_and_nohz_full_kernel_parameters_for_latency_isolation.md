# Use isolcpus and nohz_full kernel parameters for latency isolation

**Category:** Low Latency and Real Time  
**Standard:** C++17  
**Reference:** <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html>  

---

## Topic Overview

### The Problem: OS Jitter

Even on an otherwise idle system, the Linux kernel periodically interrupts every CPU core with bookkeeping work. This creates unpredictable latency spikes on cores you thought were dedicated to your real-time thread. The interruptions are invisible unless you specifically measure for them, which is one reason jitter bugs are so frustrating to track down.

Here is what the timeline looks like with and without isolation. The uninterrupted version is what you're aiming for:

```cpp
Without isolation:
CPU Core 2: [your_thread][timer_tick][your_thread][scheduler][your_thread][RCU][...]
             ^ jitter: 1-50 µs                    ^ jitter        ^ jitter

With isolation:
CPU Core 2: [your_thread────────────────────────────────────────────────────────]
             ^ uninterrupted execution
```

The main sources of jitter on an untuned system are:

- **Timer ticks** (HZ=1000 means an interrupt every 1 ms)
- **Scheduler** load balancing across cores
- **RCU callbacks** (Read-Copy-Update maintenance work)
- **Kernel workqueues** and softirqs
- **Hardware interrupts** (NIC, disk, USB) routed to all cores

### Kernel Boot Parameters

All of these sources can be tamed with three kernel command-line parameters that work together. You add them to the GRUB configuration and they take effect on the next boot:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7"
```

| Parameter | Effect |
| --- | --- |
| `isolcpus=2-7` | Remove CPUs 2-7 from the general scheduler. No process will be scheduled on them unless explicitly pinned. |
| `nohz_full=2-7` | Disable timer ticks on CPUs 2-7 when only one task is running (adaptive-ticks mode). |
| `rcu_nocbs=2-7` | Offload RCU callbacks from CPUs 2-7 to housekeeping CPUs (0-1). |

After editing the GRUB config, apply it and reboot. You can verify isolation took effect by checking the sysfs file that reports which CPUs the kernel has excluded from scheduling:

```bash
# Apply and reboot
sudo update-grub
sudo reboot

# Verify
cat /sys/devices/system/cpu/isolated
# Output: 2-7
```

### Pinning Threads to Isolated Cores in C++

With the cores isolated at the kernel level, you then pin your real-time thread to one of them and optionally set a real-time scheduling policy. The three-step pattern here - pin, set priority, lock pages - is the standard setup recipe for a low-latency thread:

<!-- compile: needs POSIX header `pthread.h` -->
```cpp
#include <pthread.h>
#include <sched.h>
#include <thread>

// Pin current thread to a specific CPU core
void pin_to_core(int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);

    int rc = pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
    if (rc != 0) {
        throw std::runtime_error("Failed to pin thread to core " +
                                 std::to_string(core_id));
    }
}

// Set real-time scheduling priority
void set_realtime_priority(int priority = 90) {
    struct sched_param param;
    param.sched_priority = priority;
    int rc = pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
    if (rc != 0) {
        throw std::runtime_error("Failed to set RT priority (run as root?)");
    }
}

// Complete setup for a latency-critical thread
void setup_low_latency_thread(int core_id) {
    pin_to_core(core_id);           // Keep on isolated core
    set_realtime_priority(90);       // Preempt everything
    mlockall(MCL_CURRENT | MCL_FUTURE);  // No page faults
}

// Usage
int main() {
    std::thread critical_thread([]{
        setup_low_latency_thread(4);  // Pin to isolated core 4

        while (true) {
            // Hot loop - runs uninterrupted
            auto msg = receive_market_data();
            auto order = compute_strategy(msg);
            send_order(order);
        }
    });

    critical_thread.join();
}
```

### IRQ Affinity - Move Interrupts Away

Isolating CPUs from the scheduler still leaves hardware interrupts as a potential jitter source. You need to explicitly move all IRQs to the housekeeping CPUs (0-1 in this example) so that no NIC, disk, or USB interrupt fires on your isolated cores. The one exception is your NIC's RX IRQ, which you may want to pin to a specific core for lowest-latency networking:

```bash
# Move all IRQs to housekeeping CPUs (0-1)
for irq in /proc/irq/*/smp_affinity_list; do
    echo "0-1" > "$irq" 2>/dev/null
done

# Except: bind NIC IRQ to a specific core for lowest-latency networking
# Find NIC IRQ numbers
cat /proc/interrupts | grep eth0

# Pin NIC RX queue 0 to core 2 (the same core as the trading thread)
echo 2 > /proc/irq/42/smp_affinity_list
```

### Complete System Tuning Script

Here is a complete setup script that applies all the tuning steps in the right order. Run it as root before starting your application. Step 4 (performance governor) is especially important - frequency scaling can add 50-200ns of jitter all by itself when the CPU ramps up from a low-power state:

```bash
#!/bin/bash
# low_latency_setup.sh - Run as root before starting the application

ISOLATED_CORES="2-7"
HOUSEKEEPING="0-1"

# 1. Verify kernel parameters
if ! grep -q "isolcpus=$ISOLATED_CORES" /proc/cmdline; then
    echo "WARNING: isolcpus not set! Add to kernel command line."
fi

# 2. Move IRQs to housekeeping cores
for irq in /proc/irq/*/smp_affinity_list; do
    echo "$HOUSEKEEPING" > "$irq" 2>/dev/null
done

# 3. Disable kernel thread migration to isolated cores
for cpu in 2 3 4 5 6 7; do
    echo 0 > /sys/devices/system/cpu/cpu${cpu}/cpufreq/scaling_governor 2>/dev/null
done

# 4. Set CPU frequency to maximum (disable frequency scaling)
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo "performance" > "$cpu"
done

# 5. Disable transparent hugepages (THP defrag causes jitter)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 6. Set watchdog off for isolated cores (prevents NMI interrupts)
echo 0 > /proc/sys/kernel/watchdog

# 7. Disable ASLR (optional, for deterministic memory layout)
# echo 0 > /proc/sys/kernel/randomize_va_space

echo "Low-latency setup complete."
```

### Measuring Jitter

The only way to know if your isolation is working is to measure it. This function runs a tight loop taking timestamps and records the gap between consecutive measurements. Any gap larger than the expected loop overhead is jitter - it means the CPU was doing something else. You want p99 in the hundreds of nanoseconds, not the tens of microseconds:

```cpp
#include <chrono>
#include <vector>
#include <algorithm>

struct JitterStats {
    double min_ns, max_ns, avg_ns, p99_ns, p999_ns;
};

JitterStats measure_jitter(int iterations = 1'000'000) {
    std::vector<int64_t> deltas;
    deltas.reserve(iterations);

    auto prev = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        auto now = std::chrono::high_resolution_clock::now();
        auto delta = std::chrono::duration_cast<std::chrono::nanoseconds>(now - prev).count();
        deltas.push_back(delta);
        prev = now;

        // Simulate work
        asm volatile("" ::: "memory");
    }

    std::sort(deltas.begin(), deltas.end());

    JitterStats stats;
    stats.min_ns = deltas.front();
    stats.max_ns = deltas.back();
    stats.avg_ns = std::accumulate(deltas.begin(), deltas.end(), 0.0) / iterations;
    stats.p99_ns = deltas[iterations * 99 / 100];
    stats.p999_ns = deltas[iterations * 999 / 1000];
    return stats;

    // Without isolation: p99 = 5-50 µs, max = 100+ µs
    // With full isolation: p99 = 50-200 ns, max = 1-5 µs
}
```

Run this before and after applying your isolation settings. The difference in p99 and max is the quantitative proof that the tuning worked.

---

## Self-Assessment

### Q1: What is the difference between `isolcpus` and `nohz_full`

`isolcpus` removes CPUs from the general scheduler - no processes will be scheduled on them unless explicitly pinned with `sched_setaffinity`. `nohz_full` disables the periodic timer tick on those CPUs when only one runnable task exists (adaptive-ticks mode). They complement each other: `isolcpus` prevents unwanted tasks, and `nohz_full` prevents timer interrupts. Both are needed for true isolation.

### Q2: Why do you need `rcu_nocbs` in addition to `isolcpus`

RCU (Read-Copy-Update) is a kernel synchronization mechanism that needs periodic maintenance (callbacks). By default, these callbacks run on every CPU. `rcu_nocbs=2-7` offloads RCU callback processing to dedicated kernel threads on housekeeping CPUs, preventing RCU from interrupting isolated cores. Without it, you'll see intermittent jitter spikes of ~10-50 µs from RCU processing.

### Q3: How do you verify that isolation is working

1. **Check kernel parameters**: `cat /proc/cmdline` should show `isolcpus`, `nohz_full`, `rcu_nocbs`
2. **Check isolated CPUs**: `cat /sys/devices/system/cpu/isolated` should list isolated cores
3. **Measure jitter**: Run the jitter measurement tool - p99 should drop from ~10-50 µs to < 1 µs
4. **Monitor IRQs**: `watch cat /proc/interrupts` - isolated cores should show minimal interrupt counts
5. **Check scheduling**: `ps -eo pid,psr,comm` - no unexpected processes on isolated cores

---

## Notes

- `isolcpus` is being deprecated in favor of cgroups cpuset - check your kernel version before relying on it.
- `tuned` and `tuned-adm` provide pre-built low-latency profiles: `tuned-adm profile latency-performance`.
- PREEMPT_RT kernel patches reduce worst-case latency from milliseconds to microseconds for the cases where the stock kernel isn't enough.
- On NUMA systems, pin threads to cores on the same NUMA node as the NIC for lowest latency.
- Disable SMT (hyperthreading) for isolated cores - the sibling thread can cause L1/L2 cache evictions even when your thread is the only one running.
