# Use Kernel Bypass Networking (DPDK) for Ultra-Low Latency

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** [DPDK Documentation](https://doc.dpdk.org/guides/), [DPDK Programmer's Guide](https://doc.dpdk.org/guides/prog_guide/)  

---

## Topic Overview

The Linux kernel network stack adds 5-15µs of latency per packet hop through interrupt handling, context switches, socket buffer copies, and protocol processing. Kernel bypass frameworks like **DPDK** (Data Plane Development Kit) eliminate this by mapping NIC memory directly into userspace, polling for packets in a tight loop, and processing them without ever entering the kernel.

DPDK uses **poll-mode drivers (PMDs)** that busy-poll the NIC's RX ring buffer. Packets arrive as **mbufs** - reference-counted, pool-allocated packet buffers - and are processed in zero-copy fashion. The typical DPDK application dedicates one or more CPU cores exclusively to packet processing (using `isolcpus` and `pthread_setaffinity`), so there is no context-switch overhead.

The reason this trips people up from a C++ perspective is that DPDK is a C library with manual memory management. Every mbuf you receive must be returned to the pool after processing - via `rte_pktmbuf_free()` on the RX path, or automatically after TX completion. Failing to return mbufs is a resource leak that eventually starves the mempool and stalls the NIC. The solution is to wrap mbufs in an RAII class that returns them automatically on scope exit.

| Metric | Kernel Stack | DPDK (Kernel Bypass) |
| --- | --- | --- |
| Per-packet latency | 5-15 µs | 0.5-2 µs |
| Throughput (64B pkts) | ~1-5 Mpps | 40+ Mpps (per core) |
| Context switches | Per-packet interrupt | Zero |
| CPU usage | Low (idle when no traffic) | 100% (busy-poll) |
| Buffer copies | 1-2 (kernel <-> user) | Zero |
| Setup complexity | Low (`socket()`) | High (hugepages, VFIO, PMD) |

Here is the architectural contrast in one picture. The kernel path has several hops between the NIC and your application; the DPDK path is a direct poll with zero copies:

```cpp
KERNEL PATH:                          DPDK PATH:
NIC -> IRQ -> Kernel -> sk_buff ->    NIC -> RX Ring -> PMD Poll ->
  copy -> Socket -> read() -> User      User (mbuf, zero-copy)

Latency: ~10 µs                       Latency: ~1 µs
```

---

## Self-Assessment

### Q1: Write an RAII wrapper for DPDK `rte_mbuf` that returns the buffer to its pool on destruction and supports zero-copy access to packet data

The `MbufPtr` class is move-only, just like `unique_ptr`. When the `MbufPtr` goes out of scope, it calls `rte_pktmbuf_free` to return the mbuf to its pool. The `release()` method lets you hand ownership to the TX path (where the NIC itself frees the buffer after sending), without calling free prematurely:

```cpp
#include <rte_mbuf.h>
#include <rte_ether.h>
#include <rte_ip.h>
#include <cstdint>
#include <utility>
#include <cstring>
#include <span>

// RAII wrapper: automatically frees mbuf to mempool on scope exit
class MbufPtr {
    rte_mbuf* mbuf_ = nullptr;

public:
    MbufPtr() = default;
    explicit MbufPtr(rte_mbuf* m) noexcept : mbuf_(m) {}

    ~MbufPtr() {
        if (mbuf_) rte_pktmbuf_free(mbuf_);
    }

    // Move-only semantics
    MbufPtr(MbufPtr&& other) noexcept
        : mbuf_(std::exchange(other.mbuf_, nullptr)) {}

    MbufPtr& operator=(MbufPtr&& other) noexcept {
        if (this != &other) {
            if (mbuf_) rte_pktmbuf_free(mbuf_);
            mbuf_ = std::exchange(other.mbuf_, nullptr);
        }
        return *this;
    }

    MbufPtr(const MbufPtr&) = delete;
    MbufPtr& operator=(const MbufPtr&) = delete;

    // Release ownership without freeing (for TX - NIC frees after send)
    rte_mbuf* release() noexcept { return std::exchange(mbuf_, nullptr); }

    // Zero-copy access to packet data
    std::span<const uint8_t> data() const noexcept {
        return {rte_pktmbuf_mtod(mbuf_, const uint8_t*), mbuf_->data_len};
    }

    std::span<uint8_t> mutable_data() noexcept {
        return {rte_pktmbuf_mtod(mbuf_, uint8_t*), mbuf_->data_len};
    }

    // Typed header access (zero-copy cast)
    template <typename Header>
    const Header* header_at(std::size_t offset) const noexcept {
        if (offset + sizeof(Header) > mbuf_->data_len) return nullptr;
        return reinterpret_cast<const Header*>(
            rte_pktmbuf_mtod(mbuf_, const uint8_t*) + offset);
    }

    rte_mbuf* get() const noexcept { return mbuf_; }
    explicit operator bool() const noexcept { return mbuf_ != nullptr; }
    uint16_t length() const noexcept { return mbuf_ ? mbuf_->data_len : 0; }
};

// Usage in a packet processing function
void process_packet(MbufPtr pkt) {
    auto* eth = pkt.header_at<rte_ether_hdr>(0);
    if (!eth) return;

    if (rte_be_to_cpu_16(eth->ether_type) == RTE_ETHER_TYPE_IPV4) {
        auto* ip = pkt.header_at<rte_ipv4_hdr>(sizeof(rte_ether_hdr));
        if (ip) {
            // Process IPv4 packet - zero copies made
            (void)ip->src_addr;
        }
    }
    // mbuf automatically returned to pool here
}
```

The `header_at<T>` template does a bounds-checked cast into the raw packet data - zero copy, no deserialization step.

### Q2: Implement a DPDK poll-mode RX loop that receives packets in bursts, dispatches by EtherType, and tracks per-type counters

The `PacketDispatcher` template uses compile-time dispatch via lambda parameters, so there are no virtual calls and the compiler can inline the handlers. The main loop calls `rte_eth_rx_burst` which fills an array of mbuf pointers in one go - processing in bursts amortizes the cost of fetching NIC descriptor ring entries from memory:

```cpp
#include <rte_ethdev.h>
#include <rte_mbuf.h>
#include <rte_ether.h>
#include <rte_ip.h>
#include <cstdint>
#include <cstdio>
#include <array>
#include <atomic>

struct PacketCounters {
    uint64_t ipv4 = 0;
    uint64_t ipv6 = 0;
    uint64_t arp  = 0;
    uint64_t other = 0;
    uint64_t total = 0;
};

// Template-based dispatch: zero overhead, no virtual calls
template <typename IPv4Handler, typename ARPHandler>
class PacketDispatcher {
    IPv4Handler ipv4_handler_;
    ARPHandler  arp_handler_;

public:
    PacketDispatcher(IPv4Handler h4, ARPHandler ha)
        : ipv4_handler_(std::move(h4)), arp_handler_(std::move(ha)) {}

    void dispatch(rte_mbuf* mbuf, PacketCounters& counters) noexcept {
        auto* eth = rte_pktmbuf_mtod(mbuf, const rte_ether_hdr*);
        uint16_t etype = rte_be_to_cpu_16(eth->ether_type);

        switch (etype) {
            case RTE_ETHER_TYPE_IPV4:
                ++counters.ipv4;
                ipv4_handler_(mbuf);
                break;
            case RTE_ETHER_TYPE_ARP:
                ++counters.arp;
                arp_handler_(mbuf);
                break;
            case RTE_ETHER_TYPE_IPV6:
                ++counters.ipv6;
                break;
            default:
                ++counters.other;
                break;
        }
        ++counters.total;
    }
};

// Main poll loop - runs on a dedicated core
void rx_poll_loop(uint16_t port_id, uint16_t queue_id,
                  std::atomic<bool>& running) {
    constexpr uint16_t BURST_SIZE = 32;
    std::array<rte_mbuf*, BURST_SIZE> rx_pkts;
    PacketCounters counters{};

    auto ipv4_handler = [](rte_mbuf* m) noexcept {
        // Fast-path IPv4 processing
        (void)m;
    };
    auto arp_handler = [](rte_mbuf* m) noexcept {
        // ARP response logic
        (void)m;
    };

    PacketDispatcher dispatcher(ipv4_handler, arp_handler);

    while (running.load(std::memory_order_relaxed)) {
        uint16_t nb_rx = rte_eth_rx_burst(port_id, queue_id,
                                           rx_pkts.data(), BURST_SIZE);
        for (uint16_t i = 0; i < nb_rx; ++i) {
            dispatcher.dispatch(rx_pkts[i], counters);
            rte_pktmbuf_free(rx_pkts[i]);
        }
        // No sleep, no yield - pure busy-poll for minimum latency
    }

    std::printf("Total: %lu  IPv4: %lu  ARP: %lu  Other: %lu\n",
                counters.total, counters.ipv4, counters.arp, counters.other);
}
```

The comment "no sleep, no yield" is the key design decision: this loop burns 100% of one CPU core. That's the explicit tradeoff you make for sub-microsecond latency.

### Q3: Set up DPDK EAL initialization, mempool creation, and port configuration from C++ with proper error handling

EAL initialization is the mandatory first step in any DPDK application - it sets up hugepage mappings, binds devices, and initializes per-core memory. The `std::expected` return type lets you propagate errors cleanly without exceptions. The mempool parameters matter: the cache size per core (256 here) keeps recently freed mbufs in a per-core cache to avoid contention on the global pool:

<!-- compile: needs third-party library header `rte_eal.h` -->
```cpp
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>
#include <cstdio>
#include <cstdlib>
#include <expected>
#include <string_view>
#include <array>

enum class DpdkError { EalInitFailed, PoolCreateFailed, PortConfigFailed };

struct DpdkContext {
    rte_mempool* mbuf_pool = nullptr;
    uint16_t port_id = 0;
};

std::expected<DpdkContext, DpdkError>
init_dpdk(int argc, char** argv) {
    // EAL initialization
    int ret = rte_eal_init(argc, argv);
    if (ret < 0) return std::unexpected(DpdkError::EalInitFailed);

    DpdkContext ctx;
    ctx.port_id = 0;  // first available port

    // Create mbuf pool: pre-allocate 8192 packet buffers
    ctx.mbuf_pool = rte_pktmbuf_pool_create(
        "MBUF_POOL",
        8192,                          // number of mbufs
        256,                           // cache size per core
        0,                             // priv_size
        RTE_MBUF_DEFAULT_BUF_SIZE,     // data room size
        rte_socket_id()                // NUMA socket
    );
    if (!ctx.mbuf_pool)
        return std::unexpected(DpdkError::PoolCreateFailed);

    // Port configuration
    rte_eth_conf port_conf{};
    port_conf.rxmode.mq_mode = RTE_ETH_MQ_RX_NONE;

    ret = rte_eth_dev_configure(ctx.port_id, 1, 1, &port_conf);
    if (ret < 0) return std::unexpected(DpdkError::PortConfigFailed);

    // Setup RX queue
    ret = rte_eth_rx_queue_setup(ctx.port_id, 0, 512,
                                  rte_eth_dev_socket_id(ctx.port_id),
                                  nullptr, ctx.mbuf_pool);
    if (ret < 0) return std::unexpected(DpdkError::PortConfigFailed);

    // Setup TX queue
    ret = rte_eth_tx_queue_setup(ctx.port_id, 0, 512,
                                  rte_eth_dev_socket_id(ctx.port_id),
                                  nullptr);
    if (ret < 0) return std::unexpected(DpdkError::PortConfigFailed);

    // Start the port
    ret = rte_eth_dev_start(ctx.port_id);
    if (ret < 0) return std::unexpected(DpdkError::PortConfigFailed);

    // Enable promiscuous mode
    rte_eth_promiscuous_enable(ctx.port_id);

    return ctx;
}

int main(int argc, char** argv) {
    auto result = init_dpdk(argc, argv);
    if (!result) {
        std::fprintf(stderr, "DPDK init failed: error %d\n",
                     static_cast<int>(result.error()));
        return 1;
    }

    auto& ctx = *result;
    std::printf("DPDK ready: port %u, pool %p\n",
                ctx.port_id, static_cast<void*>(ctx.mbuf_pool));

    // Run poll loop...
    rte_eth_dev_stop(ctx.port_id);
    rte_eth_dev_close(ctx.port_id);
    rte_eal_cleanup();
}
```

Notice that the mbuf pool is created on `rte_socket_id()` - the NUMA node of the current CPU. This ensures the pool's memory is local to the core that will be doing the polling.

---

## Notes

- **DPDK requires hugepages** (typically 1GB or 2MB) and VFIO/UIO kernel modules to bind the NIC to userspace drivers.
- **mbufs must always be returned** to their pool - via `rte_pktmbuf_free()` after RX processing, or automatically after TX completion.
- **Burst processing** (32-64 packets at a time) amortizes the cost of cache-line fetches from the NIC descriptor ring.
- Use **RSS (Receive Side Scaling)** to distribute packets across multiple RX queues, each handled by a pinned core.
- Alternatives to DPDK: **io_uring** (lower complexity, higher latency ~3-5µs), **XDP/eBPF** (in-kernel fast path), **Solarflare OpenOnload** (socket-compatible bypass).
- Profile with `rte_rdtsc()` for cycle-accurate latency measurement within DPDK; avoid `clock_gettime` in tight loops.
