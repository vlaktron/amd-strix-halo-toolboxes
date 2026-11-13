# AI Agent Context: AMD Strix Halo Llama.cpp Toolboxes

**Primary Goal:** This project provides pre-built containers ("toolboxes") for running `llama.cpp` optimally on AMD Ryzen AI Max "Strix Halo" APUs. It simplifies environment setup for ROCm and Vulkan backends, allowing users to leverage up to 124 GiB of unified system memory for LLM inference.

## Core Technologies
*   **Containerization**: [Toolbx](https://containertoolbx.org/) (Fedora) or Distrobox (Ubuntu). Underneath, Docker/Podman is used to build base images.
*   **Inference Engine**: [Llama.cpp](https://github.com/ggerganov/llama.cpp)
*   **Hardware / Drivers**: AMD "Strix Halo" APUs (Gfx1151). Implementations depend on ROCm (v6.4.4, v7.x, v10.x) and Vulkan (Mesa RADV, AMDVLK).
*   **Local AI Deployment**: Docker Compose for running multiple containers on remote networked machines.

---

## Critical Security Patterns (Recent Additions)

The repository now includes hardened containers designed for **production use on remote machines** with enhanced security posture:

### 1. `Dockerfile.rocm-7.2.3-hardened`
**Security Hardening Strategies:**
- **Non-root user execution**: Creates dedicated `llama` user (UID/GID 1000) matching Fedora hardware access conventions
- **No sudo in runtime image**: `sudo` is excluded entirely from the minimal runtime stage to prevent privilege escalation attacks
- **Minimal attack surface**: Uses `fedora-minimal:43` as runtime base, installing only required dependencies
- **Hardware group mapping**: Adds `llama` user to `video` and `render` groups for GPU access without root
- **Fixed user IDs**: UID/GID 1000 ensures consistent permissions when mounting volumes from the host

### 2. `Dockerfile.rocm-7.2.3-comfy-ui`
**Security Hardening Strategies:**
- **Non-root user execution**: Creates dedicated `comfy` user (UID/GID 1000) for ComfyUI application
- **Minimal runtime**: Uses `fedora-minimal:43` base with only essential Python/ROCm dependencies
- **Virtually isolated**: Python virtual environment from builder stage copied to runtime, keeping build tools out of production
- **Proper volume ownership**: `chown -R comfy:comfy /home/comfy/ComfyUI` ensures container user owns runtime outputs

### 3. Security-First Container Design Principles
- **Multi-stage builds**: Keep build dependencies (compilers, headers, development packages) in builder stage only
- **No privileged containers**: Always drop to non-root user with `USER <username>`
- **Explicit device access**: GPU devices (`/dev/kfd`, `/dev/dri`) passed explicitly rather than using `--privileged`
- **No unnecessary capabilities**: Avoid adding Linux capabilities; rely on specific device mounts instead
- **Read-only model volumes**: Docker Compose mounts models as read-only (`:ro`) where possible

---

## Local AI Deployment with docker-compose.yml

The repository includes `toolboxes/docker-compose.yml` for **networked machine deployment** - running AI inference on a separate machine accessible via network.

### Architecture
- **Single host, multiple containers**: One machine runs multiple inference services (Qwen3-Coder, Gemma-4, ComfyUI) simultaneously
- **Network isolation**: Each service binds to a different port (8080, 8081, 8188)
- **Shared GPU resources**: All containers share the same physical GPU via ROCm driver

### Service Configuration Details

#### Qwen3-Coder Service
- **Port**: 8080
- **Model**: Qwen3-Coder-Next-Q5_K_M (quantized, ~28GB VRAM)
- **Context**: 131,072 tokens
- **GPU Layers**: 48 (partial offload)
- **Use Case**: Code generation assistant

#### Gemma-4 Service
- **Port**: 8081
- **Model**: gemma-4-26B-A4B (quantized, ~30GB VRAM)
- **Context**: 49,152 tokens
- **GPU Layers**: 999 (full offload)
- **Parallel**: 2 sequences
- **Use Case**: General purpose assistant

#### ComfyUI Service
- **Port**: 8188
- **Image**: `llama-rocm-7.2.3-comfy-ui` (separate Dockerfile)
- **Models**: Clip, VAE, UNET stored in `/mnt/models/comfy-ui/`
- **Use Case**: Stable Diffusion/Flux image generation
- **Security**: Runs as `comfy` user (UID/GID 1000), no sudo, minimal runtime

### 3. Security-First Container Design Principles

```yaml
security_opt:
  - label:disable  # Disable SELinux/AppArmor labels for ROCm compatibility

group_add:
  - video
  - render      # Match container user's groups for GPU access

devices:
  - /dev/kfd:/dev/kfd
  - /dev/dri:/dev/dri  # Explicit GPU device passthrough

ipc: host          # CRITICAL: Host IPC for ROCm driver shared memory
shm_size: '16gb'   # Sufficient shared memory for GPU operations
```

### Environment Variables (GFX Override)
All services use `HSA_OVERRIDE_GFX_VERSION=11.5.1` to force RDNA 3.5 architecture path, ensuring compatibility.

---

## Repository Structure Overview
*   `/toolboxes/`: Dockerfiles used to build the container images (e.g., `rocm-7.2.3-hardened`, `rocm-7.2.3-comfy-ui`, `vulkan-radv`). These often use multi-stage builds to compile Llama.cpp and extract standalone binaries.
*   `/benchmark/`: Shell scripts and Python utilities (like `generate_results_json.py`, `run_calibrated_campaign.py`) to systematically test Llama.cpp throughput, latency, and RPC performance. Includes calibration data for optimal ubatch sizes.
*   `/docs/`: Markdown documents, along with HTML/CSS/JS (e.g., `index.html`, `assets/`) for the GitHub Pages website (`strix-halo-toolboxes.com`), plus interactive benchmark viewers and documentation on VRAM estimation.
*   `/scripts/`: Python utilities, including `run_distributed_llama.py` for distributed inference across nodes.
*   `/systemd/`: Systemd service files for host-level monitoring (e.g., `gpu-workload-watch` for automatic cooling and TuneD profile switching).
*   `.github/workflows/`: GitHub Actions that automatically rebuild containers whenever the upstream `llama.cpp` master branch updates or when triggered manually.

---

## Critical Technical Quirks (Important for Development)
*   **Flash Attention & no-mmap**: Running `llama-server` or `llama-cli` on Strix Halo *requires* `-fa 1` (flash attention) and `--no-mmap` (or `--load-mode none` for retained-PM4 stacks) to avoid memory fragmentation and crashes.
*   **Kernel memory params**: The optimal Strix Halo host configuration relies on custom boot parameters (`iommu=pt amdgpu.gttsize=126976 ttm.pages_limit=32505856`) to allocate unified RAM to the iGPU.
*   **Kernel Bugs**: Avoid kernels older than 6.18.4, and the specifically broken `linux-firmware-20251125`.
*   **ROCm Shared Memory**: `ipc: host` and `shm_size: '16gb'` are *required* in Docker Compose for ROCm driver buffers. Omitting these causes "out of memory" errors even with sufficient VRAM.
*   **GFX Version Override**: `HSA_OVERRIDE_GFX_VERSION=11.5.1` forces the correct RDNA 3.5 architecture path for Strix Halo (`gfx1151`).

---

## Essential Commands

### Building Containers
```bash
# Build hardened container
cd toolboxes
podman build -t llama-rocm-7.2.3-hardened -f Dockerfile.rocm-7.2.3-hardened .

# Build ComfyUI container
podman build -t llama-rocm-7.2.3-comfy-ui -f Dockerfile.rocm-7.2.3-comfy-ui .
```

### Running with Docker Compose
```bash
cd toolboxes
podman compose up -d  # Start all services in background

# Check logs
podman compose logs -f qwen3-coder
podman compose logs -f gemma-4
podman compose logs -f comfyui

# Stop all services
podman compose down
```

### Using Toolbx (Local Development)
```bash
# Create toolbox from hardened image
toolbox create llama-rocm-7.2.3-hardened --image localhost/llama-rocm-7.2.3-hardened:latest \
  -- --device /dev/dri --device /dev/kfd --group-add video --group-add render \
  --security-opt label:disable

toolbox enter llama-rocm-7.2.3-hardened

# Inside toolbox, verify GPU access
llama-cli --list-devices

# Run inference (requires model files mounted via toolbox --volume)
llama-server -m /models/model.gguf --ctx-size 32768 --n-gpu-layers 999 --no-mmap -fa 1
```

### Refreshing All Toolboxes
```bash
./refresh-toolboxes.sh all  # Updates all known toolboxes to latest images
./refresh-toolboxes.sh llama-rocm-7.2.3-hardened  # Update specific toolbox
```

---

## Code Organization and Patterns

### Dockerfile Conventions
1. **Multi-stage builds**: Builder stage compiles everything, runtime stage copies only necessary binaries
2. **Consistent ROCm repo setup**: All ROCm Dockerfiles use the same `tee /etc/yum.repos.d/rocm.repo` pattern
3. **Environment variable consistency**: `ROCM_PATH`, `HIP_PATH`, `PATH` set identically across images
4. **Non-root users**: Runtime containers always create dedicated user with UID/GID 1000
5. **LD configuration**: Custom libraries added to `/etc/ld.so.conf.d/local.conf` then `ldconfig`

### Benchmarking Patterns
- **Calibration-first**: Use `run_calibrated_campaign.py` with `validate_campaign.py` to find optimal ubatch sizes before full campaigns
- **JSONL output**: Each campaign run produces `.jsonl` files per model per curve (prefill/ generation)
- **Metadata tracking**: `run_metadata.txt` and `campaign_manifest.json` record exact commits and configuration
- **Results aggregation**: `generate_results_json.py` converts campaign outputs to single `results.json` for web visualization

---

## Gotchas and Non-Obvious Patterns

### 1. Container User Permissions
The `llama` and `comfy` users are created with **UID/GID 1000** to match Fedora's default user. When mounting volumes:
- Host files owned by UID/GID 1000 work without permission issues
- Files owned by other UIDs may be unreadable inside the container
- For shared model storage, ensure host files are owned by UID/GID 1000

### 2. ROCm Shared Memory Requirements
`ipc: host` is **non-negotiable** for ROCm containers. The ROCm driver uses System V shared memory for inter-process communication. Without it:
- Containers start but fail with "out of memory" on first GPU allocation
- Error messages are misleading (appear as VRAM exhaustion)
- The fix is always adding `ipc: host` to compose file

### 3. Model Path Consistency
The `docker-compose.yml` uses absolute paths on the host:
- Models must be located at `/mnt/models/` on the host
- ComfyUI models at `/mnt/models/comfy-ui/`
- The container expects these exact paths; no relative paths work

### 4. Read-Only Model Volumes
Models are mounted as read-only (`:ro`) in `docker-compose.yml`:
- Prevents accidental modification of model files
- May cause issues with models that try to write cache files in the same directory
- Solution: Keep model files in a dedicated read-only directory

### 5. Port Conflicts
Services bind to `0.0.0.0:PORT`:
- Port 8080: Qwen3-Coder
- Port 8081: Gemma-4
- Port 8188: ComfyUI
- Ensure these ports are free on the host before running `podman compose up`

### 6. GPU Memory Overcommit
All three services run simultaneously on one GPU:
- Total VRAM required: ~80GB (Qwen3: 28GB + Gemma-4: 30GB + ComfyUI: 22GB)
- Strix Halo has 124GB unified RAM, but ROCm reserves some for system use
- If VRAM runs out, containers may crash or fail to load models
- Solution: Run fewer services simultaneously or use smaller quantized models

---

## Project-Specific Context

### Hardened Container Purpose
The `rocm-7.2.3-hardened` and `rocm-7.2.3-comfy-ui` containers were specifically created for:
1. **Production deployment** on networked machines (not just local development)
2. **Multi-service hosting** - running multiple LLMs and ComfyUI simultaneously
3. **Security hardening** - removing sudo, using non-root users, minimal attack surface
4. **Production monitoring** - services run as systemd `restart: unless-stopped` for automatic recovery

### Local AI Use Case
The docker-compose.yml enables a "home server" workflow:
1. One powerful Strix Halo machine on the home network
2. Runs multiple AI services (code assistant, general assistant, image generator)
3. Accessible from other machines via HTTP endpoints
4. No need to SSH into the machine for each model
5. Centralized GPU management with automatic restarts

### Testing Approach
- **Unit tests**: None in this repository (shell/CLI-based tooling)
- **Integration tests**: Benchmark scripts verify inference throughput and correctness
- **Manual validation**: `llama-cli --list-devices` confirms GPU access after container creation
- **Performance validation**: Benchmark campaigns measure token throughput with calibrated ubatch sizes

---

## General Instructions for Coding Agents
1.  **Branch Naming**: Use `feat/` prefix for feature branches (e.g., `feat/memory-tuning`)
2.  **Conventional Commits**: Use conventional commit messages for all changes
3.  **Container Builds**: When modifying `Dockerfile.*` files inside `/toolboxes`, ensure the build output remains lean and only necessary runtime dependencies and Llama.cpp binaries are carried over.
4.  **Documentation Synchronization**: If adding a new backend or feature, ensure `README.md` is updated simultaneously.
5.  **Scripts**: Benchmarking and utility scripts are expected to integrate with standard `toolbox` execution. Use `/dev/dri` and `/dev/kfd` mounts for device access.
4.  **Security First**: For new hardened containers, always:
    - Use `fedora-minimal` as runtime base
    - Create dedicated non-root user (UID/GID 1000)
    - Exclude `sudo` from runtime dependencies
    - Add user to `video,render` groups
    - Use `USER <username>` directive before `ENTRYPOINT`
5.  **Docker Compose**: When adding new services:
    - Use unique ports (8080, 8081, 8188, etc.)
    - Include `ipc: host` and `shm_size: '16gb'` for ROCm
    - Mount models read-only (`:ro`) where possible
    - Match `HSA_OVERRIDE_GFX_VERSION=11.5.1` for all services

---

## Conventional Commits

This project follows conventional commit messages for clear changelog generation:

```
<type>[optional scope]: <description>

[optional body]

[optional footer]
```

### Types
- `feat`: New feature for the user
- `fix`: Bug fix for the user
- `docs`: Documentation-only changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `perf`: Performance improvement
- `test`: Adding or correcting tests
- `chore`: Changes to the build process or auxiliary tools

### Examples
```
feat: add memory tuning with VRAM caps
fix: correct vram allocation for gemma-4 service
docs: update AGENTS.md with branch naming conventions
```
