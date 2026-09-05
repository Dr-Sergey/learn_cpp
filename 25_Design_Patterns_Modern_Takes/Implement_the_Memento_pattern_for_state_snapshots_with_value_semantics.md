# Implement the Memento pattern for state snapshots with value semantics

**Category:** Design Patterns Modern Takes  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Memento_pattern>  

---

## Topic Overview

This second Memento file focuses on **value semantics**, **copy-on-write optimization**, and **move semantics** for efficient snapshots. Unlike the first Memento file (which covered basic snapshot/restore with friend classes), this one addresses performance and memory considerations for large state objects.

The key question is: when you save a snapshot, how much does it cost? For small objects the answer doesn't matter. For something like a canvas with megabytes of pixel data, taking a deep copy on every save is expensive. This file covers three strategies and explains when each one is appropriate.

### Snapshot Strategies

| Strategy | Cost to Save | Cost to Restore | Memory | Best For |
| --- | --- | --- | --- | --- |
| Deep copy | O(n) | O(n) | Full copy per snapshot | Small state |
| Copy-on-write (CoW) | O(1) share | O(n) on first modify | Shared until modified | Many snapshots, rare modifications |
| Move capture | O(1) | O(1) | Original moved out | One-shot save (no further use of original) |

---

## Self-Assessment

### Q1: Use deep copy to implement a Memento that captures the complete state of an object

**Answer:**

Deep copy is the simplest and most straightforward approach. The memento holds its own independent copy of all the state, so the original and the snapshot can evolve independently. The downside is that saving is O(n) in the size of the state - for a 100x100 canvas at 4 bytes per pixel, that's 40KB per snapshot.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

// Canvas with large pixel data
class Canvas {
    std::vector<uint8_t> pixels_;
    int width_, height_;
    std::string label_;

public:
    Canvas(int w, int h, const std::string& label)
        : pixels_(w * h * 4, 0), width_(w), height_(h), label_(label) {}

    void draw_pixel(int x, int y, uint8_t r, uint8_t g, uint8_t b) {
        int i = (y * width_ + x) * 4;
        pixels_[i] = r; pixels_[i+1] = g; pixels_[i+2] = b; pixels_[i+3] = 255;
    }

    // Memento = deep copy of entire state
    struct Memento {
        std::vector<uint8_t> pixels;
        int width, height;
        std::string label;
    };

    Memento save() const {
        return Memento{pixels_, width_, height_, label_};  // Deep copy
    }

    void restore(const Memento& m) {
        pixels_ = m.pixels;  // Deep copy back
        width_ = m.width;
        height_ = m.height;
        label_ = m.label;
    }

    size_t pixel_count() const { return pixels_.size(); }
    std::string info() const {
        return label_ + " (" + std::to_string(width_) + "x" + std::to_string(height_) + ")";
    }
};

int main() {
    Canvas canvas(100, 100, "Drawing1");
    canvas.draw_pixel(50, 50, 255, 0, 0);

    // Save complete state (deep copy: 40KB of pixel data copied)
    auto snapshot = canvas.save();
    std::cout << "Saved: " << snapshot.pixels.size() << " bytes\n";

    canvas.draw_pixel(0, 0, 0, 255, 0);  // Modify after save

    // Restore to saved state
    canvas.restore(snapshot);
    std::cout << "Restored: " << canvas.info() << '\n';
}
```

Because the snapshot holds its own copy of the pixel data, modifying the canvas after saving doesn't affect the snapshot at all. That independence is the defining property of a deep copy - and also why it costs O(n) both to save and to restore.

### Q2: Optimize by using copy-on-write (shared\_ptr + lazy copy) for large state objects

**Answer:**

Copy-on-write (CoW) exploits the observation that most snapshots are never actually modified - you save them, keep them as a safety net, and eventually throw them away. Until a modification actually happens, there's no need for a separate copy. CoW makes saving O(1) by just sharing the underlying data. The copy only happens when someone actually writes to the shared data - hence "copy on write."

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

// Copy-on-Write (CoW) buffer
class CowBuffer {
    struct Data {
        std::vector<int> values;
    };
    std::shared_ptr<Data> data_;

    // Detach: make a private copy if shared
    void detach() {
        if (data_.use_count() > 1) {
            data_ = std::make_shared<Data>(*data_);  // Copy only when modifying
        }
    }

public:
    CowBuffer() : data_(std::make_shared<Data>()) {}

    // Read: no copy needed - return const reference
    const std::vector<int>& values() const { return data_->values; }

    // Write: detach first (copy if shared), then modify
    void push(int val) {
        detach();
        data_->values.push_back(val);
    }

    void set(size_t i, int val) {
        detach();
        data_->values[i] = val;
    }

    // Memento = share, not copy
    struct Memento {
        std::shared_ptr<Data> snapshot;  // Shared ownership - O(1)!
    };

    Memento save() const {
        return Memento{data_};  // Just increment ref count - O(1)
    }

    void restore(const Memento& m) {
        data_ = m.snapshot;  // Point to shared data - O(1)
    }

    size_t size() const { return data_->values.size(); }
    long use_count() const { return data_.use_count(); }
};

int main() {
    CowBuffer buf;
    for (int i = 0; i < 1'000'000; ++i) buf.push(i);

    // Save: O(1) - just shares the pointer
    auto snap1 = buf.save();
    std::cout << "After save, ref count: " << buf.use_count() << '\n';  // 2

    // Modify: triggers copy (detach)
    buf.set(0, 999);
    std::cout << "After modify, ref count: " << buf.use_count() << '\n';  // 1

    // Restore: O(1) - just share pointer again
    buf.restore(snap1);
    std::cout << "Restored buf[0] = " << buf.values()[0] << '\n';  // 0

    // Multiple snapshots share same data until any is modified
    auto snap2 = buf.save();
    auto snap3 = buf.save();
    std::cout << "3 snapshots, ref count: " << buf.use_count() << '\n';  // 4
}
```

The `detach()` call inside `push()` and `set()` is the heart of the CoW mechanism. It checks whether the reference count is greater than 1, which means someone else (a snapshot) also points to this data. If so, it makes a private copy before modifying. If the ref count is already 1, no copy is needed - you're the sole owner.

### Q3: Show that move semantics enables efficient Memento capture without unnecessary copying

**Answer:**

Move-based snapshots take O(1) to save by transferring ownership of the data entirely to the checkpoint. The downside is that the original loses its state in the process - it's a one-shot capture. This is appropriate for temporary checkpoints where you don't need to continue using the original until you restore from the checkpoint.

```cpp
#include <iostream>
#include <string>
#include <vector>

// Move-based Memento: zero-copy snapshot
class GameState {
    std::vector<float> world_data_;
    std::string level_name_;

public:
    GameState(std::string level, size_t data_size)
        : world_data_(data_size, 1.0f), level_name_(std::move(level)) {}

    // Move constructor/assignment for efficient transfer
    GameState(GameState&&) = default;
    GameState& operator=(GameState&&) = default;
    GameState(const GameState&) = default;
    GameState& operator=(const GameState&) = default;

    bool empty() const { return world_data_.empty(); }
    size_t size() const { return world_data_.size(); }
    const std::string& level() const { return level_name_; }

    void modify(float val) { world_data_[0] = val; }
};

// Checkpoint system using move
class Checkpoint {
    GameState saved_state_;
public:
    // Move the state INTO the checkpoint (originator loses it)
    explicit Checkpoint(GameState&& state)
        : saved_state_(std::move(state)) {}

    // Move the state OUT (one-shot restore)
    GameState restore() && {
        return std::move(saved_state_);
    }
};

int main() {
    GameState game("Level 5", 1'000'000);
    std::cout << "Before save: " << game.size() << " floats\n";

    // Save via move - O(1), no copy of 4MB data
    Checkpoint cp(std::move(game));
    std::cout << "After move-save: game empty? " << game.empty() << '\n';  // true

    // Restore via move - O(1)
    game = std::move(cp).restore();
    std::cout << "After restore: " << game.size() << " floats, level: "
              << game.level() << '\n';

    // For undo history: use deep copy for save, move for temporary captures
    std::vector<GameState> history;
    history.push_back(game);           // Deep copy for persistent snapshot
    game.modify(42.0f);               // Modify original
    history.push_back(game);           // Another deep copy

    // Restore from history
    game = std::move(history.back());  // Move from history (destroys that snapshot)
    history.pop_back();
    std::cout << "Restored from history: " << game.size() << " floats\n";
}
```

The `restore() &&` qualifier on the `Checkpoint::restore()` method means it can only be called on an rvalue `Checkpoint` - you have to write `std::move(cp).restore()`. This makes it explicit at the call site that the checkpoint is being consumed.

---

## Notes

- **Deep copy** is simplest and safest - use it when state is small (a few kilobytes or less). It's easy to reason about and has no shared ownership to track.
- **Copy-on-write** shines when you take many snapshots but rarely restore - common in version-control-like systems where most snapshots are never actually used.
- **Move-based save** is O(1) but destructive - the originator loses its state until restore. Don't use this if you need to keep using the object after saving.
- Combine strategies where it makes sense: use CoW for a persistent undo history and move for temporary quick-save/quick-load checkpoints.
