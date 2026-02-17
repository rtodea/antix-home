# GPU Configuration and Usage

## System Overview
- **OS:** Debian 12 (bookworm)
- **Graphics Setup:** Hybrid (Optimus)
- **Integrated GPU:** Intel(R) UHD Graphics 630
- **Dedicated GPU:** NVIDIA Quadro RTX 4000 Mobile / Max-Q
- **Driver Version:** 535.261.03

## Verifying GPU Usage

### 1. Check Active Renderer
To see which GPU is currently rendering graphics for your session:
```bash
glxinfo | grep "OpenGL renderer"
```
*Currently, this defaults to the Intel integrated graphics.*

### 2. Check NVIDIA Status
To see if the NVIDIA GPU is recognized and check its current load:
```bash
nvidia-smi
```

## Running Applications on NVIDIA GPU
Since this is a hybrid setup, you must explicitly tell the system to offload tasks to the NVIDIA GPU using PRIME environment variables.

### Command Template
```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <application-command>
```

### Example Test
```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```
This should return `Quadro RTX 4000`.

## Live Monitoring
To monitor GPU usage, memory, and temperature in real-time while running an application:
```bash
watch -n 1 nvidia-smi
```
