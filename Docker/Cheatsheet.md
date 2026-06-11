# Docker Interview Cheat Sheet

### 🔌 Port Mapping & Variables (`-p` vs `-P` vs `ENV` vs `ARG`)

*   **`-p host:container`**: Manual port mapping. Fixed, predictable URLs. Can cause host port conflicts.
*   **`-P`**: Automatic port mapping. Maps all `EXPOSE` ports to random high ports. Prevents conflicts.
*   **`ENV`**: Runtime configuration. Persists in image metadata. Accessible to app code via `os.Getenv()`.
*   **`ARG`**: Build-time configuration. Erased after build. Viewable in `docker history`. Never use for secrets.

---

### ⏱️ Build-Time vs Runtime Instructions

| Instruction | Phase | Execution Context | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **`RUN`** | **Build Time** | Creates a new permanent image layer. | Installing packages (`apk add`), compiling binaries. |
| **`ENTRYPOINT`** | **Runtime** | Sets the un-overridable target executable. | The main binary or script (`["./product-catalog"]`). |
| **`CMD`** | **Runtime** | Provides default arguments to `ENTRYPOINT`. | Default flags or configurations (`["--help"]`). |

*   **The Interview Formula**: `ENTRYPOINT` (Executable) + `CMD` (Default Arguments) = Final Command.
*   **Exec Form vs Shell Form**: Always use **Exec Form** (`["nginx"]`) so your application runs as `PID 1` and can gracefully process lifecycle signals like `SIGTERM`.

---

### 🗂️ Production-Grade Go Multi-Stage Dockerfile Blueprint

```dockerfile
# --- Stage 1: Build Phase ---
FROM golang:1.22-alpine AS builder
WORKDIR /usr/src/app

# Leverage layer caching for dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy source and compile
COPY . .
RUN go build -o product-catalog ./

# --- Stage 2: Tiny, Secure Release Phase ---
FROM alpine:latest AS release
WORKDIR /usr/src/app

# Copy only the compiled asset from the builder stage
COPY --from=builder /usr/src/app/product-catalog ./

# Dynamic configuration setup
ENV PRODUCT_CATALOG_PORT=8088
EXPOSE \${PRODUCT_CATALOG_PORT}

ENTRYPOINT ["./product-catalog"]
```

---

### 🏎️ Essential CLI Troubleshooting Commands

*   **`docker ps`**: Lists running containers. Used to check if status is `Up` and verify active port mappings.
*   **`docker logs <container-id>`**: Displays application stdout/stderr. Used to check if the app crashed or bound to the wrong port.
*   **`docker history <image-id>`**: Shows the build steps of an image. Used to verify layer sizes or inspect accidentally leaked `ARG` secrets.
*   **`docker inspect <container-id>`**: Dumps full container metadata JSON. Used to inspect active runtime `ENV` variables.

---

### 🔐 3 Golden Rules of Container Security

1.  **Never bake secrets** into images via `ENV` or `ARG`. Inject them dynamically at runtime via secrets managers or read-only volume mounts.
2.  **Bind applications to `0.0.0.0`**, never `127.0.0.1` (localhost). `127.0.0.1` forces the app to look only inside its own container sandbox, dropping all traffic routed through Docker port mappings.
3.  **Use Multi-Stage Builds** to keep build tools, source code, and package managers out of your final image, dramatically reducing the attack surface.


---

### 📦 Optimizing Image Size & Layer Caching

*   **Order Matters**: Place instructions that change frequently (like `COPY . .`) at the very bottom of the Dockerfile. Place instructions that rarely change (like `RUN apk add`) near the top. This prevents Docker from busting the cache unnecessarily.
*   **Chain Commands**: Combine commands in a single `RUN` instruction using `&&` to reduce layer count. Clean up package caches in the same layer to avoid keeping useless files in the final image.
    *   *Bad (Creates 2 heavy layers):* 
        ```dockerfile
        RUN apt-get update
        RUN apt-get install -y curl
        ```
    *   *Good (Creates 1 optimized layer):* 
        ```dockerfile
        RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
        ```
*   **Use `.dockerignore`**: Always add a `.dockerignore` file next to your Dockerfile. Explicitly exclude folders like `.git`, `node_modules`, `vendor`, and local logs. This stops heavy or sensitive local files from accidentally copying into the image build context.

---

### 🌐 Docker Networking Basics

*   **Bridge Network (Default)**: The default driver for standalone containers. It creates an isolated network inside your host machine. Containers can talk to each other using their private container IP addresses.
*   **Host Network**: Removes network isolation between the container and the Docker host. The container shares the host’s networking namespace directly (e.g., a web app running on port 80 in the container is immediately available on host port 80 without using `-p`).
*   **Overlay Network**: Used to connect containers across multiple different host machines running Docker Swarm.
*   **DNS Resolution**: Inside a custom Docker Network or Docker Compose setup, containers do not need to use IP addresses to talk to each other. They can resolve each other's network addresses automatically using their **service names** as the hostname (e.g., `http://database-service:5432`).

---

### 💾 Storage & Data Persistence (Volumes vs Binds)

*   **Anonymous Volumes**: Managed entirely by Docker. Located in a hidden system folder (`/var/lib/docker/volumes/`). If the container is deleted, the data persists, but finding it is difficult.
*   **Named Volumes (`volumes:`)**: The production standard. Docker manages the lifecycle and folder path, but you give it an explicit name (e.g., `db_data:/var/lib/postgresql/data`). Perfect for persistent databases.
*   **Bind Mounts**: Maps a specific, absolute path on your host laptop directly to a path inside the container (e.g., `-v /Users/me/project:/app`). Any code changes made on your laptop instantly update inside the container. Best used for active local development, never in production.

---

### 🩺 Container Lifecycle & Health Monitoring

*   **`HEALTHCHECK`**: An instruction built into the Dockerfile that tells Docker how to test if the app is actually functioning, not just running.
    *   *Example:* `HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8088/health || exit 1`
    *   If the endpoint fails, Docker marks the container status as `unhealthy`, alerting orchestrators like Kubernetes to restart it.
*   **Restart Policies**: Defines how Docker should handle container crashes or host reboots.
    *   `no`: Do not automatically restart (Default).
    *   `on-failure`: Restart only if the app exits with a non-zero error code.
    *   `always`: Always restart the container if it stops, even if manually stopped.
    *   `unless-stopped`: Always restart unless it was explicitly stopped by the user before the Docker daemon restarted.

---

### 🛡️ Non-Root User Execution (Security Hardening)

*   **The Default Risk**: By default, containers run apps as the `root` user. If an attacker exploits a vulnerability in your Go or Node.js app, they gain root access inside the container, increasing the risk of a container breakout to the host machine.
*   **The Fix**: Always create and switch to a non-privileged system user before running your executable.
*   **Example Setup**:
    ```dockerfile
    FROM alpine:latest
    RUN addgroup -S appgroup && adduser -S appuser -G appgroup
    WORKDIR /app
    COPY --from=builder /build/app .
    # Change file ownership to the non-root user
    RUN chown -R appuser:appgroup /app
    
    USER appuser
    ENTRYPOINT ["./app"]
    ```

---

### 🪵 Logging Drivers & Handling I/O

*   **Standard Output Standard**: Containers should always log directly to `stdout` and `stderr`. Do not write log files to the container file system (which bloats image size and disappears when the container terminates).
*   **`json-file` (Default Driver)**: Docker intercepts stdout/stderr and writes it to a JSON file on the host machine. This is what you see when you run `docker logs`.
*   **Production Log Forwarding**: In production clusters, change the logging driver in `daemon.json` or Docker Compose to forward logs directly to central aggregators like Fluentd, Splunk, or AWS CloudWatch without filling up local disk space.
    *   *Compose Example:*
        ```yaml
        logging:
          driver: "json-file"
          options:
            max-size: "10m" # Prevents logs from consuming the entire host hard drive
            max-file: "3"
        ```

---

### 🧹 Garbage Collection & Image Cleanup

*   **Dangling Images (`<none>:<none>`)**: These occur when you rebuild an image with the same tag. The old image layers lose their name tag but remain on disk.
*   **`docker image prune`**: Deletes all dangling images to free up space.
*   **`docker system prune -a --volumes`**: The ultimate cleanup command. Deletes **all** stopped containers, unused networks, dangling/unused images, and build caches. Use with extreme caution as it wipes out volume data if appended with `--volumes`.

---

### 🏗️ Multi-Platform Builds (Buildx)

*   **The Architecture Clashing Problem**: If you build a Docker image on an Apple Silicon Mac (M1/M2/M3 uses ARM64), it will fail to run on an AWS EC2 production server running Intel/AMD processors (x86_64) with an `exec format error`.
*   **The Solution (`docker buildx`)**: Docker's modern build tool leverages QEMU emulator layers to compile and output binaries for multiple CPU architectures simultaneously out of a single command.
*   **CLI Command**:
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t my-app:latest . --push
    ```

---

### 🧟 The Zombie Process Problem & Tini (Init System)

*   **The Problem**: If your application creates child processes (common in Node.js, Python, or Ruby) and runs as `PID 1` inside the container, it might fail to reap dead child processes. These become "Zombie Processes" that slowly consume system memory tables until the host freezes.
*   **The Solution (`--init`)**: Pass the `--init` flag at runtime, or use a tiny init system inside your Dockerfile like `tini`. It acts as a lightweight `PID 1` that correctly routes system signals and reaps dead zombie processes safely.
*   **Example Setup**:
    ```dockerfile
    RUN apk add --no-cache tini
    ENTRYPOINT ["/sbin/tini", "--", "./my-app"]
    ```

---

### 🚀 Advanced BuildKit Cache Mounts (Eliminating `go mod download` Slowness)

*   **The Production Problem:** In a large microservice repo, running `RUN go mod download` or `RUN npm ci` during every CI/CD pipeline run wastes massive amounts of time downloading the exact same dependencies over and over whenever a file changes.
*   **The Industry Solution:** Docker BuildKit introduces **cache mounts**. Instead of relying purely on image layer caching, it creates a persistent, isolated cache directory on the host machine that survives across different builds.
*   **Example Setup:**
    ```dockerfile
    # syntax=docker/dockerfile:1
    FROM golang:1.22-alpine AS builder
    WORKDIR /usr/src/app
    COPY go.mod go.sum ./
    
    # Mounts the Go module cache dynamically across builds
    RUN --mount=type=cache,target=/go/pkg/mod/ \
        go mod download
        
    COPY . .
    RUN --mount=type=cache,target=/go/pkg/mod/ \
        go build -o product-catalog ./
    ```
*   *Why it impresses:* It shows you know how to optimize enterprise CI/CD pipeline speeds, cutting build times down from minutes to seconds.

---

### 🔧 Distroless & Scratch Images (Enterprise Security Hardening)

*   **The Production Problem:** Even `alpine` images contain a shell (`/bin/sh`) and basic package managers (`apk`). If an attacker gains access to your container, they can use these built-in tools to download malware, scan your network, and escalate privileges.
*   **The Industry Solution:** Production teams use **`scratch`** (an explicitly empty 0-byte image) or Google’s **`distroless`** images for the runtime stage. These images contain *only* your compiled binary and its runtime dependencies (like SSL certificates). They have no shell, no package manager, and no standard utilities.
*   **Example Setup:**
    ```dockerfile
    FROM golang:1.22-alpine AS builder
    # ... compile steps ...

    # The absolute minimum attack surface
    FROM gcr.io/distroless/static-debian12 AS release
    WORKDIR /
    COPY --from=builder /usr/src/app/product-catalog .
    ENTRYPOINT ["/product-catalog"]
    ```
*   *The Catch:* You cannot use `docker exec -it <container> sh` to debug it because no shell exists. You must use ephemeral debug containers (`kubectl debug` or specialized tools).

---

### 🦏 Cgroups v2 & The OOM-Killer Memory Trap

*   **The Production Problem:** In production clusters (Kubernetes/ECS), you set memory limits on containers. If your Go/Java application exceeds this limit, the Linux kernel instantly terminates the process with an **OOMKilled (Exit Code 137)** error. 
*   **The Industry Nuance:** Many runtimes (especially Java or older Go versions) do not natively understand they are running inside a container. They look at the host machine's total RAM (e.g., 64GB) instead of the container's allocated limit (e.g., 512MB), causing them to aggressively allocate memory and trigger the OOM-killer.
*   **The Fix:** 
    *   For **Go 1.19+**, set the environment variable `GOMEMLIMIT=450MiB` (leave a 10% buffer for runtime overhead) so the Go garbage collector triggers *before* the Linux kernel kills the container.
    *   Ensure your underlying host OS uses **Cgroups v2**, which accurately propagates memory and CPU throttling metrics down into the container file system (`/sys/fs/cgroup/`).

---

### 🛡️ Secure Secret SSH Forwarding during Builds

*   **The Production Problem:** Your Go application needs to fetch code from a **private internal GitHub/GitLab repository** during the build phase (`go get ://github.com`). Junior developers often bake GitHub SSH keys or Private Access Tokens (PATs) into `ARG` variables, leaking them to the image history.
*   **The Industry Solution:** Use BuildKit's native SSH agent forwarding. It temporarily forwards your local machine's or CI/CD runner's logged-in SSH session into the container *only* during that specific `RUN` command, without writing any keys to disk.
*   **Example Setup:**
    ```dockerfile
    # syntax=docker/dockerfile:1
    FROM golang:1.22-alpine AS builder
    RUN apk add --no-cache git openssh-client
    
    # Add GitHub to known hosts safely
    RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
    
    # Mount the SSH socket dynamically for this step only
    RUN --mount=type=ssh go mod download
    ```
*   *How to execute:* `docker build --ssh default .`

---

### 🛑 Graceful Shutdowns & The PID 1 Trap

*   **The Production Problem:** When you run `docker stop` or when Kubernetes rolls out an update, Docker sends a **`SIGTERM`** signal to `PID 1` inside the container, giving it 10 seconds to finish active user requests and close database connections before executing a forced `SIGKILL`. If your app script wraps the binary inside a shell script, the shell swallows the signal, resulting in dropped database writes and broken user requests.
*   **The Industry Solution:** If you *must* use a wrapper shell script (`entrypoint.sh`) to do configuration before starting your app, you cannot just call `./product-catalog`. You must use the Linux **`exec`** command.
*   **Example Setup (`entrypoint.sh`):**
    ```bash
    #!/bin/sh
    # Do some initial setup, template reading, etc.
    echo "Setup complete!"
    
    # 'exec' replaces the shell process entirely with the Go binary.
    # This promotes your binary to PID 1 so it receives SIGTERM directly.
    exec ./product-catalog 
    ```
