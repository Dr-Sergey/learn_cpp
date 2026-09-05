# Compile C++ to WebAssembly with Emscripten (Part 3: I/O, Debugging, and Optimization)

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://emscripten.org>  

---

## Topic Overview

This part covers Emscripten's **I/O mapping** (how std::cout/cin work in the browser), **debugging** Wasm, and **production optimization** techniques.

### Emscripten I/O Mapping

One of the first things people wonder about is where their `printf` output goes. Emscripten maps the standard C/C++ I/O streams to the closest browser or Node.js equivalents, so your existing logging code usually works without changes:

| C++ I/O | Browser Target | Node.js Target |
| --- | --- | --- |
| `std::cout` / `printf` | `console.log()` | `process.stdout` |
| `std::cerr` | `console.error()` | `process.stderr` |
| `std::cin` | `window.prompt()` (blocking) | `process.stdin` |
| File I/O (`fopen`) | Virtual FS (MEMFS) | NODEFS (real disk) |
| `std::this_thread::sleep_for` | Needs Asyncify | `setTimeout` via Asyncify |

### Optimization Levels

Choosing the right optimization level is a trade-off between compile time and output quality. The table below shows the flags and when to reach for each one:

| Flag | Use Case | Size | Speed |
| --- | --- | --- | --- |
| `-O0` | Debug (fast compile) | Large | Slow |
| `-O1` | Dev builds | Medium | Moderate |
| `-O2` | Production (recommended) | Small | Fast |
| `-O3` | Maximum speed | Small | Fastest |
| `-Os` | Minimize size | Smallest | Good |
| `-Oz` | Aggressively minimize size | Tiny | Acceptable |

---

## Self-Assessment

### Q1: Compile a C++ function to WASM using emcc and call it from JavaScript

**Answer:**

A game loop is a great example of how Wasm + Emscripten differs from normal desktop C++. You can't just write `while (true)` and update the screen - browsers don't work that way. Instead, you hand control over to `emscripten_set_main_loop_arg`, which schedules your update function using `requestAnimationFrame` under the hood:

<!-- compile: needs third-party library header `emscripten.h` -->
```cpp
// game_engine.cpp - simple game loop compiled to Wasm
#include <emscripten.h>
#include <emscripten/html5.h>
#include <cstdio>
#include <cmath>

struct GameState {
    double x = 400.0, y = 300.0;
    double vx = 2.0, vy = 1.5;
    int width = 800, height = 600;
    int frame_count = 0;
};

GameState state;

void game_loop(void* arg) {
    auto* gs = static_cast<GameState*>(arg);

    gs->x += gs->vx;
    gs->y += gs->vy;

    // Bounce off walls
    if (gs->x <= 0 || gs->x >= gs->width)  gs->vx = -gs->vx;
    if (gs->y <= 0 || gs->y >= gs->height) gs->vy = -gs->vy;

    gs->frame_count++;
    if (gs->frame_count % 60 == 0) {
        printf("Frame %d: ball at (%.1f, %.1f)\n", gs->frame_count, gs->x, gs->y);
        // printf -> console.log in browser
    }
}

int main() {
    printf("Game starting...\n");   // -> console.log("Game starting...")

    // Request animation frame loop - doesn't block!
    emscripten_set_main_loop_arg(game_loop, &state, 60, true);
    // 60 FPS, simulate_infinite_loop=true

    return 0;  // Never reached (main loop takes over)
}
```

```bash
emcc game_engine.cpp -o game.html -O2 \
    -sUSE_SDL=0 \
    --shell-file minimal_shell.html
# Produces: game.html, game.js, game.wasm
# Open game.html -> see console.log output at 60 FPS
```

### Q2: Use EMSCRIPTEN_BINDINGS to expose a C++ class to JavaScript without manual glue code

**Answer:**

Here's a physics engine example that shows how naturally a C++ class can map to a JavaScript API with Embind. Notice the `.property()` binding for `gravity` - it creates a JS property backed by a C++ getter/setter pair, so the JS side reads `world.gravity` just like any other object property:

```cpp
// physics.cpp - physics engine exposed to JS
#include <emscripten/bind.h>
#include <cmath>
#include <vector>
#include <string>

struct Particle {
    double x, y;
    double vx, vy;
    double mass;
};

class PhysicsWorld {
    std::vector<Particle> particles_;
    double gravity_ = -9.81;
    double dt_ = 1.0 / 60.0;
public:
    void set_gravity(double g) { gravity_ = g; }
    double get_gravity() const { return gravity_; }

    int add_particle(double x, double y, double mass) {
        particles_.push_back({x, y, 0, 0, mass});
        return particles_.size() - 1;
    }

    void apply_force(int id, double fx, double fy) {
        auto& p = particles_.at(id);
        p.vx += fx / p.mass * dt_;
        p.vy += fy / p.mass * dt_;
    }

    void step() {
        for (auto& p : particles_) {
            p.vy += gravity_ * dt_;
            p.x += p.vx * dt_;
            p.y += p.vy * dt_;
            if (p.y < 0) { p.y = 0; p.vy = -p.vy * 0.8; }  // Bounce
        }
    }

    double get_x(int id) const { return particles_.at(id).x; }
    double get_y(int id) const { return particles_.at(id).y; }
    int count() const { return particles_.size(); }
};

EMSCRIPTEN_BINDINGS(physics) {
    emscripten::class_<PhysicsWorld>("PhysicsWorld")
        .constructor<>()
        .property("gravity", &PhysicsWorld::get_gravity, &PhysicsWorld::set_gravity)
        .function("addParticle", &PhysicsWorld::add_particle)
        .function("applyForce", &PhysicsWorld::apply_force)
        .function("step", &PhysicsWorld::step)
        .function("getX", &PhysicsWorld::get_x)
        .function("getY", &PhysicsWorld::get_y)
        .function("count", &PhysicsWorld::count);
}
```

On the JavaScript side, the result is a very natural API - the physics world looks and feels like a plain JS object, with no visible glue code at all:

```javascript
// JavaScript: use C++ physics engine with zero glue code
const world = new Module.PhysicsWorld();
world.gravity = -9.81;

const ball = world.addParticle(0, 100, 1.0);
world.applyForce(ball, 5.0, 0);

for (let i = 0; i < 300; i++) {
    world.step();
    if (i % 60 === 0) {
        console.log(`t=${i/60}s: x=${world.getX(ball).toFixed(2)}, y=${world.getY(ball).toFixed(2)}`);
    }
}
world.delete();
```

### Q3: Show how Emscripten maps std::cout to the browser console and std::cin to JS prompts

**Answer:**

For most output, Emscripten's I/O mapping just works. For input, though, `std::cin` becomes a `window.prompt()` dialog which is blocking and ugly - definitely not what you want in a real web app. The better approach is to use Embind to expose a function that JavaScript can call directly with DOM input:

```cpp
// io_demo.cpp
#include <iostream>
#include <string>
#include <cstdio>

int main() {
    // stdout -> console.log
    std::cout << "Hello from C++!" << std::endl;
    // Browser: appears in DevTools Console as:
    //   Hello from C++!

    printf("printf works too: %d + %d = %d\n", 2, 3, 5);
    // Browser: console.log("printf works too: 2 + 3 = 5")

    // stderr -> console.error
    std::cerr << "This is an error!" << std::endl;
    // Browser: appears in Console as red error text

    // stdin -> window.prompt()
    // WARNING: std::cin blocks -> triggers window.prompt() dialog
    // This is ugly and blocking - prefer Embind callbacks instead
    std::cout << "Enter your name: ";
    std::string name;
    std::cin >> name;
    // Browser: shows a prompt dialog "Enter your name:"
    // User types in the dialog -> result goes to 'name'

    std::cout << "Hello, " << name << "!" << std::endl;

    return 0;
}
```

```bash
emcc io_demo.cpp -o io_demo.html -O2
# Open io_demo.html in browser
# Console shows: "Hello from C++!"
# A prompt dialog appears for std::cin
```

If you need to redirect stdout to a DOM element instead of the console, you can override Emscripten's print function before the module loads:

**Custom output redirection:**

```javascript
// Override Emscripten's print functions
var Module = {
    print: function(text) {
        // Redirect stdout to a DOM element instead of console
        document.getElementById('output').textContent += text + '\n';
    },
    printErr: function(text) {
        document.getElementById('errors').textContent += text + '\n';
    }
};
```

For input, the better pattern avoids `std::cin` entirely and exposes a C++ function that JS can call with the value from an `<input>` element:

**Better alternative for input (no prompt dialogs):**

```cpp
// Use Embind + val to interact with DOM directly
#include <emscripten/bind.h>
#include <emscripten/val.h>
#include <string>

std::string process_input(const std::string& input) {
    return "Processed: " + input;
}

EMSCRIPTEN_BINDINGS(io) {
    emscripten::function("processInput", &process_input);
}
```

```javascript
// JS calls C++ directly - no stdin/prompt needed
document.getElementById('btn').onclick = () => {
    const input = document.getElementById('input').value;
    const result = Module.processInput(input);
    document.getElementById('output').textContent = result;
};
```

---

## Notes

- Prefer Embind + DOM interaction over `std::cin`/`cout` for web UIs.
- `emscripten_set_main_loop()` replaces `while(true)` game loops - you can't block in a browser.
- Debugging: `emcc -g -gsource-map` generates source maps for step-through debugging in DevTools.
- `wasm-opt` (from Binaryen) can further shrink `.wasm` files: `wasm-opt -O3 out.wasm -o opt.wasm`.
- Use `-sASSERTIONS=1` during development for helpful error messages (stripped in release).
- For large projects, `-flto` (link-time optimization) can reduce `.wasm` size by 10-30%.
