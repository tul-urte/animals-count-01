# CUDA model-experiment environment

Created 2026-11-12 in WSL2 `Ubuntu-22.04`, as user `imx`, separate from the
existing IMX500 conversion environment.

## Environments are deliberately separate

| Purpose | WSL path | Current role |
|---|---|---|
| Model experiments / CUDA inference | `/home/imx/model-experiments-cuda` | PyTorch training, evaluation, GPU inference, and quantization experiments |
| Sony IMX500 conversion | `/home/imx/imx500-env` | Existing known-working MCT + `imxconv-pt` packaging environment; do not upgrade it in place |

The experiment environment uses Python 3.11, **torch 2.11.0+cu128**,
**torchvision 0.26.0+cu128**, ONNX Runtime GPU 1.31.1, MCT 2.5.1, and
edge-mdt-tpc 1.3.0. It is not a replacement for the converter environment:
conversion compatibility must be checked separately before deploying an rpk.

## Activate

From Windows:

```powershell
wsl -d Ubuntu-22.04 -u imx -- bash -lc 'source ~/model-experiments-cuda/bin/activate; python your_script.py'
```

From inside WSL:

```bash
source ~/model-experiments-cuda/bin/activate
```

The GPU is the NVIDIA RTX Spark N1X (CUDA capability 12.1, approximately
31 GB VRAM). CUDA tensor matmul and SSDLite320 inference were verified after
installation. The original `imx500-env` package fingerprint was identical before
and after this environment was created.

## Rebuild

```bash
python3.11 -m venv ~/model-experiments-cuda
source ~/model-experiments-cuda/bin/activate
python -m pip install --upgrade pip
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128
pip install onnx onnxruntime-gpu model-compression-toolkit==2.5.1 edge-mdt-tpc==1.3.0
```

Do not install packages into `imx500-env` for experiments. Do not assume that
MCT output which works in this environment is deployable: run the Sony converter
and evaluate the resulting rpk against the saved-image regression set first.

The failed ep29 SSD INT8 deployment variants and the current production warning
are recorded in the session notes `notes/2026-11-12 S070.md` +
`notes/2026-11-12 S071.md` (the 2026-11-12 quantization investigations). In
particular, the current unsigned 8-bit classification output
(scale 0.0078125) is not acceptable for normal wildlife detection.
