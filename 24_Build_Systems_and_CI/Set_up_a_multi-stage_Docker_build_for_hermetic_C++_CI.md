# Set up a multi-stage Docker build for hermetic C++ CI

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://docs.docker.com/build/guide/multi-stage/>  

---

## Topic Overview

A **hermetic build** means every dependency - compiler, libraries, system headers - is locked inside a container. Nothing from the host machine leaks in. Multi-stage Docker builds take this further by separating the heavy toolchain image (the builder) from a tiny runtime image (the runner). The result is small, reproducible artifacts: the multi-gigabyte compiler toolchain never ships to production.

### Multi-Stage Build Architecture

The key insight is that you only need the compiler to *build* the binary. You do not need it to *run* the binary. Multi-stage builds exploit this by using two separate images.

```cpp
+-------------------------------------------------------+
| Stage 1: builder (ubuntu:22.04 + GCC + vcpkg + deps)  |
|  +-- Install compiler toolchain                        |
|  +-- Install vcpkg + project dependencies             |
|  +-- cmake --build -> produces binary                  |
|  +-- Image size: ~2-4 GB                              |
+-------------------------------------------------------+
| Stage 2: runner (ubuntu:22.04-minimal or distroless)   |
|  +-- COPY --from=builder /app/build/myapp /app/       |
|  +-- Only runtime libs (libstdc++, etc.)              |
|  +-- Image size: ~50-150 MB                           |
+-------------------------------------------------------+
```

### Why Hermetic Builds

If the table feels like a lot, the core idea boils down to one thing: you get the same binary on every machine, every time, because the build environment is completely defined in code.

| Benefit | Explanation |
| --- | --- |
| Reproducibility | Same compiler, same libs, same result - on any machine |
| No "works on my machine" | CI and dev use identical images |
| Small deployments | Runner image has only the binary + runtime deps |
| Security | Smaller attack surface (no compiler, no build tools in prod) |
| Caching | Docker layer cache speeds up rebuilds |

---

## Self-Assessment

### Q1: Write a Dockerfile with a builder stage (toolchain) and a runner stage (minimal binary)

**Answer:**

Here is a complete two-stage Dockerfile. Notice the structure: the builder stage does all the heavy work, and the runner stage does nothing except copy the compiled binary and set an entrypoint.

```dockerfile
# ======= Stage 1: Builder =======
FROM ubuntu:22.04 AS builder

# Prevent apt interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install build essentials + CMake + Ninja
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc-12 g++-12 \
    cmake \
    ninja-build \
    git \
    ca-certificates \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Set GCC 12 as default
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100 \
    && update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-12 100

# Copy source code
WORKDIR /src
COPY CMakeLists.txt .
COPY src/ src/
COPY include/ include/
COPY tests/ tests/

# Configure and build
RUN cmake -B build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DBUILD_TESTING=ON

RUN cmake --build build --parallel $(nproc)

# Run tests inside the builder (fail build if tests fail)
RUN cd build && ctest --output-on-failure

# ======= Stage 2: Runner =======
FROM ubuntu:22.04 AS runner

RUN apt-get update && apt-get install -y --no-install-recommends \
    libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd --create-home --shell /bin/bash appuser

WORKDIR /app
# Copy only the built binary from builder
COPY --from=builder /src/build/myapp .

# Run as non-root
USER appuser

ENTRYPOINT ["./myapp"]
```

To build and run the image, you only need two commands:

```bash
# Build the image
docker build -t myapp:latest .

# Layer sizes:
# builder stage: ~2 GB (not shipped)
# runner stage:  ~80 MB (shipped to production)

# Run the app
docker run --rm myapp:latest --help
```

The builder stage contains the full toolchain (~2 GB) and runs the build plus tests. The runner stage uses `COPY --from=builder` to extract only the compiled binary. The toolchain is never shipped to production, reducing image size by roughly 95% and shrinking the attack surface significantly.

### Q2: Pin the compiler version and vcpkg baseline hash for a fully reproducible CI image

**Answer:**

"Reproducible" means the same source always produces the same binary. To get there, every tool in the chain needs a pinned version - even the base OS image. The reason this trips people up is that Docker tags like `ubuntu:22.04` are mutable: Ubuntu can push a new image to that tag at any time, silently changing your build environment.

```dockerfile
# ======= Pinned, Reproducible Builder =======
FROM ubuntu:22.04@sha256:abc123... AS builder
# Pin the base image by SHA256 digest, not just tag.
# Tags like "22.04" can change when Ubuntu pushes updates.

ENV DEBIAN_FRONTEND=noninteractive

# Pin exact compiler packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc-12=12.3.0-1ubuntu1~22.04 \
    g++-12=12.3.0-1ubuntu1~22.04 \
    cmake=3.22.1-1ubuntu1.22.04.2 \
    ninja-build=1.10.1-1 \
    git \
    ca-certificates \
    curl \
    zip unzip tar \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# ======= Pin vcpkg to exact commit =======
ENV VCPKG_ROOT=/opt/vcpkg
ENV VCPKG_DEFAULT_BINARY_CACHE=/opt/vcpkg-cache

RUN git clone https://github.com/microsoft/vcpkg.git $VCPKG_ROOT \
    && cd $VCPKG_ROOT \
    && git checkout a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6 \
    && ./bootstrap-vcpkg.sh -disableMetrics

RUN mkdir -p $VCPKG_DEFAULT_BINARY_CACHE

# Copy manifest first (for layer caching)
WORKDIR /src
COPY vcpkg.json vcpkg-configuration.json ./

# Install dependencies (cached unless vcpkg.json changes)
RUN $VCPKG_ROOT/vcpkg install --triplet x64-linux

# Copy source and build
COPY CMakeLists.txt .
COPY src/ src/
COPY include/ include/

RUN cmake -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    -DCMAKE_CXX_STANDARD=20

RUN cmake --build build --parallel $(nproc)
```

The `vcpkg-configuration.json` file is what locks down the library versions:

```json
// vcpkg-configuration.json - pin the baseline
{
  "default-registry": {
    "kind": "git",
    "baseline": "a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6",
    "repository": "https://github.com/microsoft/vcpkg"
  }
}
```

When every version is locked, the build becomes truly reproducible:

- Base image: pinned by `@sha256:` digest
- Compiler: exact `apt` package versions (e.g., `gcc-12=12.3.0-...`)
- CMake: exact version
- vcpkg: pinned by git commit hash
- vcpkg packages: pinned by `vcpkg-configuration.json` baseline

### Q3: Use BuildKit cache mounts to cache vcpkg installs between CI runs

**Answer:**

Regular Docker layer caching is good, but it has a fragility problem: if any earlier layer is invalidated (say, you add one apt package), every layer after it is also invalidated, including your vcpkg install. BuildKit cache mounts solve this by storing the cache in a persistent volume that survives layer invalidation.

```dockerfile
# syntax=docker/dockerfile:1.4
# Required for BuildKit features

FROM ubuntu:22.04 AS builder
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc-12 g++-12 cmake ninja-build git curl \
    ca-certificates zip unzip tar pkg-config \
    && rm -rf /var/lib/apt/lists/*

ENV VCPKG_ROOT=/opt/vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git $VCPKG_ROOT \
    && cd $VCPKG_ROOT \
    && git checkout a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6 \
    && ./bootstrap-vcpkg.sh -disableMetrics

WORKDIR /src
COPY vcpkg.json vcpkg-configuration.json ./

# ======= BuildKit cache mount for vcpkg =======
# --mount=type=cache persists the directory across builds.
# Even if the Docker layer cache is cleared, the mount survives.
RUN --mount=type=cache,target=/opt/vcpkg-cache \
    VCPKG_DEFAULT_BINARY_CACHE=/opt/vcpkg-cache \
    $VCPKG_ROOT/vcpkg install --triplet x64-linux

COPY CMakeLists.txt .
COPY src/ src/
COPY include/ include/

# ======= BuildKit cache mount for CMake build =======
# Cache the build directory for incremental compilation
RUN --mount=type=cache,target=/src/build-cache \
    cmake -B build-cache -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    && cmake --build build-cache --parallel $(nproc) \
    && cp build-cache/myapp /src/myapp

FROM ubuntu:22.04 AS runner
RUN apt-get update && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /src/myapp /app/myapp
ENTRYPOINT ["/app/myapp"]
```

Enable BuildKit when you run the build:

```bash
# Build with BuildKit enabled
DOCKER_BUILDKIT=1 docker build -t myapp:latest .

# Or in CI (GitHub Actions):
# - name: Build Docker image
#   env:
#     DOCKER_BUILDKIT: 1
#   run: docker build -t myapp:latest

# Cache mount benefits:
# First build:  vcpkg install -> 5 min (downloads + builds)
# Second build:  vcpkg install -> 10 sec (binary cache hit)
# CMake:         Only recompiles changed files
```

The difference between the two caching mechanisms matters in practice:

- **Layer cache** is invalidated when ANY earlier layer changes (e.g., a new apt package). Everything downstream gets rebuilt.
- **Cache mount** persists independently - it survives layer invalidation entirely. The `--mount=type=cache,target=/path` syntax mounts a persistent volume at that path only during the `RUN` command that declares it.
- Combined, vcpkg binary cache and CMake object cache together save 5-10 minutes per CI run on a typical project.

---

## Notes

- **Distroless runner** (`gcr.io/distroless/cc-debian12`) has no shell at all - it's the smallest possible image but harder to debug because you can't exec into it interactively.
- **Security scanning:** Run `docker scout cves myapp:latest` or Trivy to check for CVEs in the runner image. The runner image is what ships to production, so it's the one that matters for security.
- **`.dockerignore`** is critical - exclude `.git/`, `build/`, and any generated directories to keep the build context small. A large build context slows down every `docker build` call.
- **Layer ordering:** Put rarely-changing steps (apt install) first, frequently-changing steps (COPY source) last to maximize cache reuse. This is the most impactful Dockerfile optimization you can make.
- **Multi-platform:** Use `docker buildx build --platform linux/amd64,linux/arm64` to build images for multiple architectures from a single Dockerfile.
