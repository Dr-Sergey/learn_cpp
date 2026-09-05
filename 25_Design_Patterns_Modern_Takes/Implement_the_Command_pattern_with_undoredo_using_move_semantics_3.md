# Implement Command undo/redo with lambda commands instead of virtual interfaces

**Category:** Design Patterns Modern Takes  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Command_pattern>  

---

## Topic Overview

This third Command pattern file focuses on **lambda-based commands** (avoiding the virtual interface entirely), **command merging** (coalescing consecutive similar commands), and practical considerations for production undo/redo systems.

The reason you might want lambda-based commands is sheer ergonomics. If you have twenty different action types in a small editor, defining twenty subclasses is a lot of boilerplate. Lambdas let you express an action and its inverse inline at the call site, with no separate class needed at all.

### Lambda vs Virtual Command Comparison

```cpp
Virtual Command:                Lambda Command:
  class InsertCmd : Command       auto cmd = MakeCmd(
    void execute() override         [&]{ doc.insert(pos, s); },
    void undo() override            [&]{ doc.erase(pos, s.size()); }
  };                              );
  // Separate class per action    // Inline, no class needed
```

---

## Self-Assessment

### Q1: Model each command as a struct with execute() and undo() methods, stored in a deque\<unique\_ptr\<Command\>\>

**Answer:**

Instead of a class hierarchy, `LambdaCommand` stores two `std::function<void()>` objects - one for execute and one for undo. The `make_command` factory keeps construction tidy. All the capturing of needed state happens inside the lambdas themselves.

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <deque>
#include <functional>

// Lambda-based Command - no virtual interface
struct LambdaCommand {
    std::function<void()> exec;
    std::function<void()> undo_fn;
    std::string description;

    void execute() { exec(); }
    void undo()    { undo_fn(); }
};

// Factory: create command from two lambdas
auto make_command(std::string desc,
                  std::function<void()> exec,
                  std::function<void()> undo)
{
    return std::make_unique<LambdaCommand>(
        LambdaCommand{std::move(exec), std::move(undo), std::move(desc)}
    );
}

int main() {
    std::string text;
    std::deque<std::unique_ptr<LambdaCommand>> history;

    // Create commands with lambdas - no subclasses needed
    auto cmd1 = make_command("type Hello",
        [&]{ text += "Hello"; },
        [&]{ text.erase(text.size() - 5); }
    );
    auto cmd2 = make_command("type World",
        [&]{ text += " World"; },
        [&]{ text.erase(text.size() - 6); }
    );

    cmd1->execute();
    history.push_back(std::move(cmd1));

    cmd2->execute();
    history.push_back(std::move(cmd2));

    std::cout << text << '\n';  // Hello World

    // Undo last
    history.back()->undo();
    history.pop_back();
    std::cout << text << '\n';  // Hello
}
```

One thing to watch out for with capture-by-reference (`[&]`): the lambda must not outlive the captured variables. Here `text` lives in `main`, so it's fine. In a longer-lived system, prefer capturing by value or use a shared data structure.

### Q2: Use std::move to transfer commands into the history without copying

**Answer:**

Command merging is a quality-of-life feature you'll appreciate as a user: typing "Hello World" one character at a time produces eleven separate history entries by default, but after merging they collapse into a single undo unit. One Ctrl+Z removes the whole word, not one letter. The trick is to check whether the new command can be merged into the previous one before pushing it onto the stack.

```cpp
#include <iostream>
#include <memory>
#include <deque>
#include <string>
#include <functional>

// Command merging - coalesce consecutive similar commands
struct TypeCommand {
    std::string& target;
    std::string typed;

    void execute() { target += typed; }
    void undo()    { target.erase(target.size() - typed.size()); }

    // Merge: combine consecutive typing into one command
    bool can_merge(const TypeCommand& other) const {
        return true;  // Any consecutive typing can merge
    }
    void merge(TypeCommand&& other) {
        typed += other.typed;  // Combine text
    }
};

class MergingHistory {
    std::deque<std::unique_ptr<TypeCommand>> stack_;
public:
    void execute(std::unique_ptr<TypeCommand> cmd) {
        cmd->execute();

        // Try to merge with previous command
        if (!stack_.empty() && stack_.back()->can_merge(*cmd)) {
            stack_.back()->merge(std::move(*cmd));
            // cmd is consumed - no need to push
        } else {
            stack_.push_back(std::move(cmd));
        }
    }

    void undo() {
        if (stack_.empty()) return;
        stack_.back()->undo();  // Undoes entire merged chunk
        stack_.pop_back();
    }

    size_t depth() const { return stack_.size(); }
};

int main() {
    std::string buffer;
    MergingHistory history;

    // Type character by character
    for (char c : std::string("Hello World")) {
        auto cmd = std::make_unique<TypeCommand>(TypeCommand{buffer, std::string(1, c)});
        history.execute(std::move(cmd));
    }

    std::cout << buffer << '\n';             // Hello World
    std::cout << "History depth: " << history.depth() << '\n';  // 1 (all merged!)

    history.undo();  // Undoes entire "Hello World" in one step
    std::cout << "After undo: \"" << buffer << "\"\n";  // ""
}
```

After typing all eleven characters, the history depth is still 1 because every new `TypeCommand` was merged into the previous one. A real editor would break the merge when the user pauses typing, switches to a different location, or presses a non-character key - but the merge mechanism itself is the same.

### Q3: Show that redo is implemented by moving commands from a redo stack back to the undo stack

**Answer:**

This example combines lambda commands with a `TransactionManager` that supports grouping multiple commands into a single atomic undo unit. The transaction concept is important in real applications: "Replace All" might change hundreds of locations, but from the user's perspective it should be one single undo step.

```cpp
#include <iostream>
#include <memory>
#include <deque>
#include <functional>
#include <string>

struct Command {
    std::function<void()> exec;
    std::function<void()> undo_fn;
    void execute() { exec(); }
    void undo()    { undo_fn(); }
};

// Production undo/redo with transaction groups
class TransactionManager {
    std::deque<std::unique_ptr<Command>> undo_stack_;
    std::deque<std::unique_ptr<Command>> redo_stack_;

    // Transaction group: multiple commands as one undo unit
    std::deque<std::unique_ptr<Command>>* active_group_ = nullptr;

public:
    void execute(std::unique_ptr<Command> cmd) {
        cmd->execute();
        if (active_group_) {
            active_group_->push_back(std::move(cmd));
        } else {
            undo_stack_.push_back(std::move(cmd));
            redo_stack_.clear();
        }
    }

    // Begin/end transaction: groups multiple commands as one undo step
    void begin_transaction() {
        undo_stack_.push_back(nullptr);  // Marker
        active_group_ = &undo_stack_;
    }

    void undo() {
        if (undo_stack_.empty()) return;
        auto cmd = std::move(undo_stack_.back());
        undo_stack_.pop_back();
        cmd->undo();
        redo_stack_.push_back(std::move(cmd));   // -> redo stack
    }

    void redo() {
        if (redo_stack_.empty()) return;
        auto cmd = std::move(redo_stack_.back());
        redo_stack_.pop_back();
        cmd->execute();
        undo_stack_.push_back(std::move(cmd));   // -> undo stack
    }
};

int main() {
    int x = 0, y = 0;
    TransactionManager mgr;

    auto make_add = [](int& val, int amt) {
        return std::make_unique<Command>(Command{
            [&val, amt]{ val += amt; },
            [&val, amt]{ val -= amt; }
        });
    };

    mgr.execute(make_add(x, 10));
    mgr.execute(make_add(y, 20));
    std::cout << "x=" << x << " y=" << y << '\n';  // x=10 y=20

    mgr.undo();  // undo y+=20
    mgr.undo();  // undo x+=10
    std::cout << "x=" << x << " y=" << y << '\n';  // x=0 y=0

    mgr.redo();  // redo x+=10
    mgr.redo();  // redo y+=20
    std::cout << "x=" << x << " y=" << y << '\n';  // x=10 y=20
}
```

The redo mechanics here are the same as in the virtual-command version - commands move from the redo stack back to the undo stack via `std::move`. The difference is just that the commands themselves are plain structs holding lambdas rather than polymorphic class instances.

---

## Notes

- **Lambda commands** eliminate boilerplate - ideal when you have many small action types and don't want a separate class per action.
- **Command merging** reduces history depth: typing "Hello" character-by-character becomes one undo unit instead of five. Real editors typically also break merges on a timer or when the cursor position changes.
- **Transaction groups** let multi-step operations (for example, "Replace All") undo as a single action from the user's perspective.
- `std::function` has some overhead from type erasure; for performance-sensitive hot paths, consider templates or `std::move_only_function` (C++23).
