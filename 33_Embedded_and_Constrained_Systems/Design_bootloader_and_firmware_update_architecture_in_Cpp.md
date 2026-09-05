# Design bootloader and firmware update architecture in C++

**Category:** Embedded and Constrained Systems  
**Standard:** C++17  
**Reference:** -  

---

## Topic Overview

### Why Bootloader Architecture Matters

A bootloader is the first code that runs on a microcontroller. It initializes minimal hardware, validates the application image, and jumps to it. In production systems, you also need a reliable firmware update (OTA or wired) path that can **never brick** the device. C++ gives you type-safe abstractions over the raw memory operations involved.

### Two-Stage Boot Architecture

The memory map below shows a typical arrangement. The bootloader lives at the very start of flash and never gets updated in the field - it is your root of trust. Everything below it in flash is fair game for updates.

```text
+==================+  0x0800_0000  Flash start
|   Bootloader     |  (16-64 KB, never updated in the field)
|   - HW init      |
|   - Image verify  |
|   - Jump to app   |
+==================+  0x0801_0000  Application slot A
|   Application A  |
|   (main firmware) |
+==================+  0x0804_0000  Application slot B (optional, A/B scheme)
|   Application B  |
|   (backup/update) |
+==================+  0x0807_F000  Persistent config / flags
|   Update flags   |
+==================+
```

### Image Header

Every firmware image starts with a fixed-size header the bootloader can inspect. Making the size a `static_assert` is important - if you ever accidentally add a field and shift the layout, the build breaks immediately rather than silently corrupting the flash write.

```cpp
#include <cstdint>
#include <array>

struct ImageHeader {
    uint32_t magic;           // e.g. 0xDEADBEEF
    uint32_t version;         // semantic version packed: major.minor.patch
    uint32_t image_size;      // bytes, excluding header
    uint32_t crc32;           // CRC-32 of image body
    uint32_t vector_table;    // offset to the real vector table
    uint32_t flags;           // bit 0 = signed, bit 1 = encrypted
    std::array<uint8_t, 64> signature; // Ed25519 or ECDSA signature
    std::array<uint8_t, 32> reserved;
};
static_assert(sizeof(ImageHeader) == 120, "Header must be fixed size");
```

### Bootloader Main Logic

The boot flow validates the primary slot, falls back to the secondary slot if validation fails, and drops into DFU recovery mode only if both are invalid. The `jump_to_app` function has to do some low-level ARM work: it reads the vector table to get the initial stack pointer and reset handler address, then sets up the processor registers and hands off execution.

```cpp
#include <cstdint>

// Memory-mapped flash regions
inline constexpr uintptr_t SLOT_A_ADDR = 0x0801'0000;
inline constexpr uintptr_t SLOT_B_ADDR = 0x0804'0000;
inline constexpr uintptr_t FLAGS_ADDR  = 0x0807'F000;

enum class BootSlot : uint8_t { A = 0, B = 1 };

struct UpdateFlags {
    BootSlot  pending_slot;
    uint8_t   update_pending;  // 1 = new image written, needs validation
    uint8_t   boot_attempts;   // incremented each boot, cleared on success
    uint8_t   reserved;
};

[[nodiscard]] const ImageHeader* get_header(BootSlot slot) {
    auto addr = (slot == BootSlot::A) ? SLOT_A_ADDR : SLOT_B_ADDR;
    return reinterpret_cast<const ImageHeader*>(addr);
}

[[nodiscard]] bool validate_image(const ImageHeader* hdr) {
    if (hdr->magic != 0xDEADBEEF) return false;
    if (hdr->image_size == 0 || hdr->image_size > 256 * 1024) return false;

    // Compute CRC-32 over image body (after header)
    auto body = reinterpret_cast<const uint8_t*>(hdr) + sizeof(ImageHeader);
    uint32_t computed_crc = crc32(body, hdr->image_size);
    return computed_crc == hdr->crc32;
}

using EntryPoint = void(*)();

[[noreturn]] void jump_to_app(const ImageHeader* hdr) {
    auto vector_table = reinterpret_cast<const uint32_t*>(
        reinterpret_cast<uintptr_t>(hdr) + hdr->vector_table
    );

    // ARM Cortex-M: first word is initial SP, second is Reset_Handler
    uint32_t sp    = vector_table[0];
    uint32_t entry = vector_table[1];

    // Disable interrupts before jump
    __disable_irq();

    // Set vector table offset register (SCB->VTOR)
    *reinterpret_cast<volatile uint32_t*>(0xE000ED08) =
        reinterpret_cast<uint32_t>(vector_table);

    // Set stack pointer and jump
    __set_MSP(sp);
    auto app_entry = reinterpret_cast<EntryPoint>(entry);
    app_entry();

    __builtin_unreachable();
}

[[noreturn]] void bootloader_main() {
    hw_early_init();  // clocks, watchdog

    auto* flags = reinterpret_cast<volatile UpdateFlags*>(FLAGS_ADDR);

    BootSlot primary   = BootSlot::A;
    BootSlot secondary = BootSlot::B;

    // Check if an update is pending
    if (flags->update_pending && flags->boot_attempts < 3) {
        primary   = flags->pending_slot;
        secondary = (primary == BootSlot::A) ? BootSlot::B : BootSlot::A;
        flags->boot_attempts++;
    }

    // Try primary slot
    auto* hdr = get_header(primary);
    if (validate_image(hdr)) {
        jump_to_app(hdr);
    }

    // Fallback to secondary slot
    hdr = get_header(secondary);
    if (validate_image(hdr)) {
        jump_to_app(hdr);
    }

    // Both slots invalid - enter DFU recovery mode
    enter_dfu_mode();
    __builtin_unreachable();
}
```

Notice the `boot_attempts` counter. This is the rollback mechanism: if the new firmware crashes immediately on every boot, the watchdog fires, the counter increments, and after three tries the bootloader gives up on the new slot and falls back to the old one.

### A/B Update Strategy

The **A/B (ping-pong) scheme** is the safest approach:

1. Application writes new firmware to the **inactive** slot.
2. Sets `update_pending = 1` and `pending_slot` to the new slot.
3. Reboots.
4. Bootloader validates and boots the new slot.
5. If the new application runs successfully, it clears `update_pending` and resets `boot_attempts`.
6. If it crashes (watchdog reset), `boot_attempts` increments. After 3 failures, bootloader falls back.

The application calls `confirm_boot` after its own startup checks pass - that's the "commit" that tells the bootloader the new firmware is good.

```cpp
// Called from the application after successful startup
void confirm_boot() {
    auto* flags = reinterpret_cast<volatile UpdateFlags*>(FLAGS_ADDR);
    flags->update_pending = 0;
    flags->boot_attempts  = 0;
}
```

### Flash Programming RAII Wrapper

Flash programming has strict rules: unlock before write, erase before write, re-lock when done. The RAII wrapper makes it impossible to forget the lock - the destructor always fires, even if an error causes an early return.

```cpp
class FlashWriter {
    uintptr_t base_;
    size_t    offset_ = 0;
    bool      erased_ = false;

public:
    explicit FlashWriter(uintptr_t base) : base_(base) {}

    ~FlashWriter() {
        flash_lock();  // Always re-lock flash on scope exit
    }

    FlashWriter(const FlashWriter&) = delete;
    FlashWriter& operator=(const FlashWriter&) = delete;

    bool erase(size_t num_sectors) {
        flash_unlock();
        for (size_t i = 0; i < num_sectors; ++i) {
            if (!flash_erase_sector(base_ + i * SECTOR_SIZE)) {
                flash_lock();
                return false;
            }
        }
        erased_ = true;
        return true;
    }

    bool write(const uint8_t* data, size_t len) {
        if (!erased_) return false;
        bool ok = flash_write(base_ + offset_, data, len);
        if (ok) offset_ += len;
        return ok;
    }

    [[nodiscard]] size_t bytes_written() const { return offset_; }
};
```

### Secure Boot Considerations

For devices where firmware tampering is a concern, you embed the signing public key in the bootloader (in read-only flash) and verify an Ed25519 or ECDSA signature over the image body before booting. An attacker can write any bytes they like to the application slot - they can't produce a valid signature without the private key.

```cpp
#include <array>

// Public key embedded in bootloader (read-only flash, not updateable)
inline constexpr std::array<uint8_t, 32> SIGNING_PUBLIC_KEY = {
    0x3b, 0x6a, /* ... 30 more bytes ... */
};

[[nodiscard]] bool verify_signature(const ImageHeader* hdr) {
    auto body = reinterpret_cast<const uint8_t*>(hdr) + sizeof(ImageHeader);
    // Ed25519 signature verification
    return ed25519_verify(
        hdr->signature.data(),
        body,
        hdr->image_size,
        SIGNING_PUBLIC_KEY.data()
    );
}
```

---

## Self-Assessment

### Q1: Why is A/B update safer than in-place update

**In-place update** overwrites the running firmware. If power is lost mid-write, or the new image is corrupt, the device is **bricked** - there is no valid image to fall back to.

**A/B update** writes to the inactive slot while the current slot remains untouched. If the new image fails validation or crashes at runtime, the bootloader can fall back to the old image. The device is **never bricked** as long as at least one slot contains a valid image.

Key advantages:

- **Atomic updates** - the slot switch is a single flag write.
- **Rollback** - automatic fallback after N failed boot attempts.
- **No downtime** - the device runs the old firmware until the new one is confirmed.
- Trade-off: requires 2x the flash for application storage.

### Q2: How do you prevent a corrupt update from being booted

You layer multiple checks in the bootloader. Each layer catches a different failure mode: the magic number quickly rejects erased or random flash, the size check prevents reading beyond allocated flash, the CRC catches incomplete writes and bit rot, and the cryptographic signature catches deliberate tampering.

```cpp
// Multi-layer validation in the bootloader:
[[nodiscard]] bool validate_image_full(const ImageHeader* hdr) {
    // 1. Magic number check - fast reject of erased/random flash
    if (hdr->magic != 0xDEADBEEF) return false;

    // 2. Size sanity - prevent reading beyond flash
    if (hdr->image_size > MAX_IMAGE_SIZE) return false;

    // 3. CRC-32 integrity - catches bit rot and incomplete writes
    auto body = reinterpret_cast<const uint8_t*>(hdr) + sizeof(ImageHeader);
    if (crc32(body, hdr->image_size) != hdr->crc32) return false;

    // 4. Cryptographic signature - catches tampering
    if (!verify_signature(hdr)) return false;

    // 5. Version monotonicity - prevent rollback attacks
    auto* current = get_header(get_running_slot());
    if (hdr->version <= current->version) return false;

    return true;
}
```

### Q3: Show proper flash write RAII that handles errors and power loss

The `FlashWriter` class above demonstrates this. Key points:

- **Constructor**: stores base address, no side effects.
- **`erase()`**: unlocks flash, erases sectors, returns false on failure.
- **`write()`**: writes data sequentially, tracks offset.
- **Destructor**: always re-locks flash (RAII guarantee).
- Power loss during write: the `update_pending` flag is only set **after** the complete image is written and validated, so a partial write is simply ignored on next boot.

---

## Notes

- Never update the bootloader itself in the field - it is the root of trust.
- Use hardware watchdog to detect application hangs and trigger rollback.
- Store update flags in a separate flash sector with wear leveling.
- Consider encrypted firmware images if IP protection is needed.
- Test the rollback path as thoroughly as the happy path.
