# RAVEN (Rockchip Accelerated Vision Edge Node)

**High-Performance Edge Perception System based on RK3588 & Qwen3-VL**

![Platform](https://img.shields.io/badge/Platform-Rockchip%20RK3588-blue) ![OS](https://img.shields.io/badge/OS-Linux%20Embeddded-green) ![Framework](https://img.shields.io/badge/Framework-RKNN%20%7C%20MPP%20%7C%20RGA-orange) 

## 📖 Introduction
**RAVEN** is a full-stack embedded AI solution designed for the Rockchip RK3588 platform. It integrates a custom-built Linux BSP, a zero-copy hardware acceleration pipeline, and state-of-the-art Visual Language Model (VLM) deployment. 

The project aims to solve the bottleneck of CPU memory copying in traditional edge AI pipelines by leveraging **DMA-BUF (DRM)** mechanisms to achieve zero-copy data transmission between VPU, RGA, and NPU, enabling real-time semantic understanding of video streams with minimal CPU overhead.

## 🚀 Key Features

### 1. Custom Linux BSP Construction
- **U-Boot & Kernel Porting**: Built from Rockchip SDK source; customized Device Tree (DTS) for specific carrier boards.
- **System Optimization**: Tailored RootFS with PREEMPT_RT patch for real-time control capability.
- **Driver Development**: Integrated drivers for MIPI-CSI cameras and NPU/RGA kernel modules.

### 2. Zero-Copy Hardware Pipeline
- **MPP (Media Process Platform)**: Hardware decoding for up to 8-channel 1080p video streams.
- **RGA (Raster Graphic Acceleration)**: Hardware-accelerated color space conversion (NV12 -> RGB) and resizing.
- **DRM Zero-Copy**: Implemented physical address mapping to share memory between decoder (VPU) and inference engine (NPU), eliminating CPU `memcpy` overhead.

### 3. Edge VLM Deployment (Qwen3-VL)
- **Model**: Qwen3-VL-4B-Instruct.
- **Quantization**: W4A16 (4-bit weights, 16-bit activation) hybrid quantization using RKNN-Toolkit2.
- **Performance**: Achieves **~12 tokens/s** inference speed on RK3588 NPU (3-core utilization).

## 🏗 Architecture

```mermaid
graph LR
    Cam[Camera/RTSP] -->|Encoded Stream| VPU[MPP Decoder]
    VPU -->|DMA-BUF (NV12)| RGA[RGA Hardware]
    RGA -->|DMA-BUF (RGB)| NPU[RKNN NPU]
    
    subgraph "Zero-Copy Pipeline (DRM)"
    VPU -.-> RGA -.-> NPU
    end
    
    NPU -->|Tensor| CPU[Post-Process]
    CPU -->|Result| App[VLM Agent]
```

## 📊 Performance Benchmarks

| Module | Task | Method | Latency / FPS | CPU Usage |
| :--- | :--- | :--- | :--- | :--- |
| **Pre-process** | 1080p Decode + Resize | OpenCV (CPU) | ~15ms | 35% (Single Core) |
| **Pre-process** | 1080p Decode + Resize | **RAVEN (MPP+RGA)** | **< 2ms** | **< 5%** |
| **Inference** | Qwen2-VL-2B | FP16 | OOM / Slow | - |
| **Inference** | Qwen2-VL-2B | **W4A16 (RKNN)** | **12 tokens/s** | NPU High / CPU Low |

## 🛠️ Build & Usage

### Prerequisites
*   Hardware: RK3588 / RK3588S EVB or Custom Board.
*   Host: Ubuntu 20.04/22.04 (Cross-compilation environment).
*   SDK: Rockchip Linux SDK 5.10+.

### 1. Build BSP (Optional)
If you need to build the OS image from scratch:
```bash
./build.sh uboot
./build.sh kernel
./build.sh rootfs
./build.sh firmware
```

### 2. Compile Application
```bash
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/rk3588_linux.cmake
make -j8
```

### 3. Run Demo
Transfer the executable and model (`qwen2vl_w4a16.rknn`) to the board:
```bash
# On RK3588 device
sudo ./raven_vlm_demo --model ./models/qwen2vl_w4a16.rknn --input video_sample.mp4
```

## 📂 Project Structure
```text
RAVEN
├── bsp/                # Kernel config, DTS overlays, and U-Boot patches
├── src/
│   ├── core/           # Main pipeline logic
│   ├── hardware/       # MPP and RGA wrapper classes
│   ├── model/          # RKNN inference engine interface
│   └── utils/          # DRM memory management utils
├── models/             # Quantization scripts and configs for Qwen2-VL
├── docs/               # Architecture diagrams and performance reports
└── README.md
```
