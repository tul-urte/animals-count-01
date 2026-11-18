# System Capabilities Report

This document outlines the hardware and software capabilities of the training machine, used to optimize the object detection training pipeline.

## 1. Compute Acceleration (GPU)
- **Model**: NVIDIA RTX Spark N1X
- **Architecture**: Blackwell (compute capability sm_121, confirmed via `torch.cuda.get_device_capability`)
- **Cores**: 6144 CUDA cores
- **VRAM**: ~48 GB total (torch reports ~48.7 GB; nvidia-smi shows ~31 GB visible — treat ~48 GB as the addressable ceiling, verify with actual batch sizes)
- **CUDA Version**: 13.4 (driver UMD 13.4, driver 617.69)
- **Notes**: 
    - High-performance Blackwell architecture. GPU compute verified working under CUDA (matmul + Conv2d on `cuda:0`).
    - `nvcc` is not in PATH (CUDA Toolkit not installed) — **this is fine**; the PyTorch wheel bundles the CUDA runtime. The Toolkit is only needed to compile custom CUDA kernels, which we are not doing.

## 2. System Resources (CPU & RAM)
- **CPU**: NVIDIA RTX Spark N1X integrated CPU (or associated host CPU)
- **Core Count**: 20 Cores / 20 Logical Processors
- **System RAM**: 64 GB
- **Implications for Training**:
    - **Data Loading**: With 20 cores, we can set `num_workers` in the PyTorch DataLoader to a high value (e.g., 8-16) to ensure the GPU is never starved of data.
    - **Memory**: 64GB of system RAM allows for significant caching of the dataset if necessary.

## 3. Software Environment (as of this session)
- **OS**: **Windows on ARM64** (`ARM 64-bit`, `Get-CimInstance` Win32_Processor Architecture=12). This is the critical detail that was previously missed.
- **GPU driver**: NVIDIA 617.69, CUDA UMD 13.4.
- **Python**: 3.14.9 (ARM64) installed per-user for this project; system Python is 3.15.1 (no CUDA wheels for it).
- **Project venv**: `c:\projects\animals-count\.venv` (Python 3.14.9, ARM64)
- **Installed in venv**:
  - `torch 2.14.0+cu134`, `torchvision 0.29.0+cu134`, `torchaudio 2.11.0+cu134` — from NVIDIA's Windows-ARM64 index `https://pypi.nvidia.com/nvtorch_oot/`
  - `numpy 2.5.4`, `pillow 13.0.0`, `networkx`, `mpmath`, `sympy`, `filelock`, `fsspec`, `jinja2`, `MarkupSafe`, `typing-extensions`
- **Framework**: **PyTorch** (chosen). TensorFlow has no Windows-ARM64 GPU path.
- **Not installed**: Ultralytics — deferred. `opencv-python` has **zero `win_arm64` wheels** (NVIDIA lists this as a known Windows-ARM64 gap), which is the main blocker for `pip install ultralytics`. Pure-PyTorch training sidesteps this.
- **CUDA status (verified)**: `torch.cuda.is_available() == True`, device `NVIDIA RTX Spark N1X`, compute capability `(12, 1)` (sm_121), GPU matmul + Conv2d run on `cuda:0`.

## 4. Training Optimization Strategy
Based on these capabilities, the following strategy is recommended:
- **Batch Size**: Can be set significantly higher than default (e.g., 32–128) due to the large VRAM. Verify empirically.
- **Precision**: Use `AMP` (Automatic Mixed Precision, `torch.autocast('cuda')`) to leverage Blackwell's Tensor Cores for faster training and lower memory usage.
- **Parallelism**: Utilize the 20 CPU cores for aggressive data augmentation and preprocessing (`num_workers` 8–16).
- **Model path options**:
  - **A. Pure PyTorch** (recommended default now): implement/adopt a lightweight YOLOv8/v11-style detector in `torch.nn`; no OpenCV needed for training; export to ONNX via `torch.onnx.export` for the IMX500 step.
  - **B. Ultralytics**: requires resolving the OpenCV ARM64 gap (build `opencv-python` from source, or confirm whether ultralytics training can run without cv2).
