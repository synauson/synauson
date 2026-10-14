# NVIDIA GPUs

**Synauson does not use NVIDIA GPUs yet.** GPU support is planned but not yet
released. Every release so far runs all inference on the CPU, through ONNX Runtime's
CPU execution provider, and loads only the CPU build of ONNX Runtime. On a machine
with an NVIDIA GPU, Synauson runs on the CPU and needs no NVIDIA driver, CUDA or cuDNN.
Installing them changes nothing for Synauson today.

The rest of this page is for preparing a host before GPU support ships. GPU support
will use ONNX Runtime's CUDA execution provider on Linux and Windows, in a separate
Synauson process, the model host. The model host's downloads (a binary beside the
native server and an optional Java SDK artifact) and its container image are planned to
carry the CUDA and cuDNN libraries it needs, at a version Synauson has tested, so you
would install only the NVIDIA driver. The server image, the native archives and the Java
SDK's CPU natives jars won't carry NVIDIA's files. The
requirements below are ONNX Runtime's and NVIDIA's, for the ONNX Runtime release
Synauson ships, 1.24.4. When GPU support is released, its release notes and this page will say which build Synauson loads;
until then, treat this as a checklist, not a supported setup.

## Contents

- [What the CUDA execution provider needs](#what-the-cuda-execution-provider-needs)
- [Linux: driver](#linux-driver)
- [Linux: CUDA and cuDNN libraries](#linux-cuda-and-cudnn-libraries)
- [Containers: NVIDIA Container Toolkit](#containers-nvidia-container-toolkit)
- [Windows](#windows)
- [Check the installation](#check-the-installation)
- [Sources](#sources)

## What the CUDA execution provider needs

Microsoft publishes two GPU builds of ONNX Runtime 1.24.4 for each platform, beside
the CPU build Synauson uses now:

| Archive (Linux / Windows) | CUDA | cuDNN | Minimum NVIDIA driver |
|---|---|---|---|
| `onnxruntime-linux-x64-gpu-1.24.4.tgz` / `onnxruntime-win-x64-gpu-1.24.4.zip` | 12.x, 12.8 or newer | 9.x | R580 or newer recommended (525 or newer with CUDA's minor version compatibility) |
| `onnxruntime-linux-x64-gpu_cuda13-1.24.4.tgz` / `onnxruntime-win-x64-gpu_cuda13-1.24.4.zip` | 13.x | 9.x | R580 or newer |

The CUDA libraries the CUDA execution provider links are cuBLAS, cuBLASLt, cuFFT,
cuRAND and the CUDA runtime, plus cuDNN 9; the Linux builds name them as
`libcudart.so.12`, `libcublas.so.12`, `libcublasLt.so.12`, `libcufft.so.11`,
`libcurand.so.10` and `libcudnn.so.9` (CUDA 12 build) or `libcudart.so.13`,
`libcublas.so.13`, `libcufft.so.12`, `libcurand.so.10` and `libcudnn.so.9` (CUDA 13
build). The Windows CUDA 12 build links `cudart64_12.dll`, `cublasLt64_12.dll`,
`cublas64_12.dll`, `cufft64_11.dll` and `cudnn64_9.dll` (no cuRAND). A build made
for CUDA 12 does not load CUDA 13 libraries, and the reverse, so match the libraries
to the build. cuDNN also loads its own sub-libraries (`libcudnn_graph.so.9`,
`libcudnn_ops.so.9`, `libcudnn_cnn.so.9` and the rest, from cuDNN's folder) when a
model first runs, so install cuDNN whole: a missing sub-library passes the `ldd`
check below.

Points to plan for:

- **GPU generation.** CUDA 13 dropped Maxwell, Pascal and Volta GPUs (compute
  capability below 7.5, Turing). The GPU code in the 1.24.4 archives starts at
  compute capability 7.0 (Volta) in the CUDA 12 build and 7.5 (Turing) in the CUDA 13
  build (read from the `.target` lines of the PTX they embed). Synauson's GPU support
  is planned for Turing or newer on both builds: cuDNN 9.12 and later no longer support
  Volta, so a V100 runs only with cuDNN 9.11 or older and won't be supported, and
  Pascal and older won't run at all. `nvidia-smi
  --query-gpu=compute_cap --format=csv` shows a GPU's compute capability.
- **cuDNN version.** Use a cuDNN release from 9.8 to 9.26 with the CUDA 12 build:
  Synauson tested that range on Linux with Turing, Ampere and Hopper GPUs (every
  release on an RTX 3090, a subset on a T4 and an H100), and 9.10.2 is the release its
  GPU measurements used. cuDNN 9.27 fails a speech recognition
  model in one configuration on an H100, so releases after 9.26 count as untested until
  they pass the same check. Ada GPUs, Windows and the CUDA 13 build have not been tested
  with any release.
- **Driver.** One driver serves every CUDA version up to the one it shipped with.
  An R580 or newer driver runs both builds; the CUDA 13 build needs it. Volta GPUs
  (V100) are the exception: NVIDIA supports them only up to the 580 branch, and only
  with its proprietary kernel modules. Driver 590 or newer, or the open modules,
  ignore a V100 (`nvidia-smi` then fails, "supported through the NVIDIA 580.xx Legacy
  drivers"). On Ubuntu that is `nvidia-driver-580-server`; checked on a V100.
- **Toolkit or libraries.** Running the CUDA execution provider needs the runtime
  libraries only, not the full CUDA toolkit (no compiler). On Linux, NVIDIA's
  package repository has them as separate packages.
- **Debian 13.** NVIDIA's `debian13` repository carries CUDA 13 only, so a Debian 13
  host needs the CUDA 13 build.

## Linux: driver

Ubuntu 24.04, with Ubuntu's signed drivers (they work with Secure Boot):

```bash
sudo ubuntu-drivers list --gpgpu
sudo ubuntu-drivers install --gpgpu                  # or name a branch, e.g. nvidia:580-server
sudo apt-get install -y nvidia-utils-<branch>-server  # nvidia-smi for that branch, e.g. nvidia-utils-580-server
sudo reboot
```

Ubuntu 24.04 or Debian 13, from NVIDIA's repository (on Debian, enable the `contrib`
component first):

```bash
sudo apt-get install -y linux-headers-$(uname -r)
distro=ubuntu2404   # or debian13
wget https://developer.download.nvidia.com/compute/cuda/repos/${distro}/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
apt-cache search nvidia-driver-pinning                # the driver branches on offer
sudo apt-get install -y nvidia-driver-pinning-<branch>  # optional: stay on one branch, e.g. 615
sudo apt-get install -y nvidia-open
sudo reboot
```

`nvidia-open` installs the open kernel modules, which NVIDIA recommends for Turing
and newer GPUs. Check with `nvidia-smi` after the reboot.

## Linux: CUDA and cuDNN libraries

You won't need this once the model host ships with its bundled libraries; it applies
only to running ONNX Runtime's GPU build yourself.

With NVIDIA's repository from the previous step, install only the runtime libraries
the build needs. For the CUDA 12 build on Ubuntu 24.04:

```bash
sudo apt-get install -y --no-install-recommends \
    cuda-cudart-12-9 libcublas-12-9 libcufft-12-9 libcurand-12-9 libcudnn9-cuda-12=9.10.2.21-1
```

Pin cuDNN as shown: without a version apt installs the newest release, which can be
past the [tested range](#what-the-cuda-execution-provider-needs). `apt-cache policy
libcudnn9-cuda-12` lists the versions the repository offers.

For the CUDA 13 build (Ubuntu 24.04 or Debian 13), with the newest CUDA 13 minor
version the repository offers (`apt-cache search cuda-cudart-13` lists them):

```bash
sudo apt-get install -y --no-install-recommends \
    cuda-cudart-13-4 libcublas-13-4 libcufft-13-4 libcurand-13-4 libcudnn9-cuda-13
```

The packages register their folders with the dynamic linker, so no
`LD_LIBRARY_PATH` is needed. `cuda-toolkit-12-9` or `cuda-toolkit-13-4` installs the
whole toolkit instead, and `cudnn9-cuda-12` or `cudnn9-cuda-13` adds cuDNN's
development files; neither is needed to run.

## Containers: NVIDIA Container Toolkit

A container reaches the GPU through the host's driver and the NVIDIA Container
Toolkit. The host needs the [driver](#linux-driver); the CUDA and cuDNN libraries
then belong inside the image. The planned model host image will carry them; the server
image stays CPU only, so giving it a GPU has no effect.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Check it with NVIDIA's sample workload:

```bash
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

Podman uses CDI instead. Toolkit 1.18 and newer generate the CDI specification
automatically (the `nvidia-cdi-refresh` service); `nvidia-ctk cdi list` shows the
devices:

```bash
podman run --rm --device nvidia.com/gpu=all --security-opt=label=disable ubuntu nvidia-smi -L
```

In Docker Compose a service asks for GPUs like this:

```yaml
services:
  model-host:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

## Windows

Step 1, the driver, is all the model host is planned to need. Steps 2 to 4 apply only
to running ONNX Runtime's GPU build yourself.

1. Install the driver for your GPU from [nvidia.com/drivers](https://www.nvidia.com/drivers).
   Since CUDA 13.1 the CUDA installer no longer includes a driver.
2. Install CUDA 12.x (12.8 or newer) or 13.x from NVIDIA's CUDA downloads, matching
   the build. The installer can install only the runtime libraries: in the custom
   install choose the CUDA runtime, cuBLAS, cuFFT and cuRAND, or run it silently with
   the subpackages, for example
   `setup.exe -s cudart_13.4 cublas_13.4 cufft_13.4 curand_13.4`. The installer sets
   `CUDA_PATH` and adds its `bin` folder to `Path`. CUDA 12 keeps its DLLs in `bin`;
   CUDA 13 in `bin\x64`.
3. Install cuDNN 9 for the same CUDA major version, with NVIDIA's graphical
   installer or the zip. Pick a release in the range
   [tested on Linux](#what-the-cuda-execution-provider-needs), such as 9.10.2 for the
   CUDA 12 build (no release has been tested on Windows yet). Add the folder that
   holds `cudnn64_9.dll` to `Path`. With the graphical installer it has the form
   `C:\Program Files\NVIDIA\CUDNN\v9.<minor>\bin\<CUDA version>\x64`.
4. Sign out and back in, then check from a new PowerShell window:

   ```powershell
   nvidia-smi
   where.exe cudart64_12.dll cublas64_12.dll cufft64_11.dll curand64_10.dll cudnn64_9.dll   # CUDA 12
   where.exe cudart64_13.dll cublas64_13.dll cufft64_12.dll curand64_10.dll cudnn64_9.dll   # CUDA 13
   ```

## Check the installation

`nvidia-smi` lists each GPU, the driver version, and the newest CUDA version that
driver supports (it doesn't show which CUDA libraries are installed):

```bash
nvidia-smi
nvidia-smi --query-gpu=name,driver_version,compute_cap --format=csv
```

If you run ONNX Runtime's GPU build yourself on Linux, check that its CUDA execution
provider finds every library it links (no output means all are found):

```bash
curl -fsSL https://github.com/microsoft/onnxruntime/releases/download/v1.24.4/onnxruntime-linux-x64-gpu-1.24.4.tgz \
  | tar -xz -C /tmp
ldd /tmp/onnxruntime-linux-x64-gpu-1.24.4/lib/libonnxruntime_providers_cuda.so | grep 'not found'
```

Both GPU archives unpack into a folder of the same name, so check one at a time.

## Sources

- ONNX Runtime, [CUDA execution provider requirements](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#requirements):
  1.21.x to 1.26.x built with CUDA 12.8 and cuDNN 9.x, needing CUDA 12.8 or newer;
  CUDA 13 builds need CUDA 13.0 or newer; cuDNN 8 and 9 builds don't mix.
- The [ONNX Runtime v1.24.4 release](https://github.com/microsoft/onnxruntime/releases/tag/v1.24.4)
  assets; the library names above are read from their `libonnxruntime_providers_cuda.so`.
- NVIDIA, [CUDA minor version compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)
  (CUDA 12.x: driver 525 or newer; CUDA 13.x: 580 or newer) and the
  [CUDA Toolkit release notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)
  (driver per toolkit release; Maxwell, Pascal and Volta removed in CUDA 13.0).
- NVIDIA, [driver installation guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
  (Ubuntu and Debian), and Ubuntu's
  [NVIDIA driver guide](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/).
- NVIDIA, cuDNN installation on [Linux](https://docs.nvidia.com/deeplearning/cudnn/installation/latest/linux.html)
  and [Windows](https://docs.nvidia.com/deeplearning/cudnn/installation/latest/windows.html).
- NVIDIA, [CUDA installation guide for Windows](https://docs.nvidia.com/cuda/cuda-installation-guide-microsoft-windows/index.html).
- NVIDIA Container Toolkit [installation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html),
  [CDI](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html)
  and [sample workload](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html);
  Docker's [GPU support in Compose](https://docs.docker.com/compose/how-tos/gpu-support/).
