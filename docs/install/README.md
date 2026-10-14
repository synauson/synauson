# Installing Synauson

Synauson runs in one of three ways. Pick the one that fits, then follow its page.

| You want to | Install | Platforms | Page |
|---|---|---|---|
| Run a gRPC media server in Docker, Podman or Kubernetes | The `synauson-server` container image | Linux x86_64 | [Linux: container](linux.md#container) |
| Run the gRPC server directly on a Linux host under systemd | The `synauson` binary, GStreamer and ONNX Runtime | Ubuntu 24.04, Debian 13 | [Linux: native server](linux.md#native-server-systemd) |
| Embed the engine in a Java application | The jsyn library from Maven, plus GStreamer | Linux x86_64, Windows x86_64 | [jsyn's requirements](https://github.com/synauson/jsyn#requirements), then [Linux](linux.md#embedded-jsyn) or [Windows](windows.md) |

The server is supported in production on Linux only. On Windows, embed the engine with
jsyn, or call a Linux server over gRPC.

Every build is x86_64 today. Native ARM builds are planned but not released. There is
no macOS build: on a Mac, run the Linux container in Docker Desktop. On Apple silicon
it runs under emulation, which is slower. Pass `--platform linux/amd64` to `docker
run` so Docker pulls the x86_64 image, and use it for development, not to size a
production host. Docker Desktop has no full host networking, so SIP and WebRTC media
in a container on a Mac are untested.

Configuration, flags, authentication and licensing behavior are in
[`deployment.md`](../deployment.md); these pages cover what to install and how to
check that it works.

## NVIDIA GPUs

Synauson does not use the GPU yet. All inference runs on the CPU through ONNX
Runtime's CPU execution provider, so a machine with an NVIDIA GPU runs Synauson on its
CPU, and needs no NVIDIA driver, CUDA or cuDNN for it. Install Synauson exactly as the
pages above describe.

GPU inference through ONNX Runtime's CUDA execution provider is planned but not yet
released. [`nvidia.md`](nvidia.md) lists the NVIDIA software it will need, for anyone
who wants to prepare a host in advance.

## System requirements

| | Requirement |
|---|---|
| CPU | x86_64 (ARM builds are planned; macOS only through the container, see above). Every measurement Synauson publishes was on CPUs with AVX2 or newer; ONNX Runtime picks AVX2, AVX-512 or VNNI kernels at run time, and older CPUs are untested. |
| Operating system | Container: any Linux host that runs Docker or Podman. Native server: Ubuntu 24.04 or Debian 13 (the GStreamer versions below decide this). jsyn: Linux with glibc 2.34 or newer, or Windows 10 or 11. |
| GStreamer | 1.24 or 1.26. **1.28 is not supported**: it changed the `webrtcbin` pad API, which breaks WebRTC. 1.22 and older are too old. The container image carries its own. |
| ONNX Runtime | The release Synauson ships (1.24.4 today; `GetCapabilities` reports it as `onnxRuntime.shippedVersion`). The container image and the jsyn natives jars bundle it; a native server install downloads it. |
| Memory | About 1.2 GB per speech-to-text (STT) worker for the default STT model, measured, plus a few hundred MB for the engine. Synauson sizes the STT pool to the memory available, so a small host runs fewer STT streams rather than failing. VAD and turn detection need little (about 30 MB per turn detection worker). |
| Disk | About 700 MB for the models a license can include today (STT is most of it), plus about 25 MB for ONNX Runtime and, for a native install, up to about 700 MB for GStreamer and its dependencies (measured on minimal Ubuntu 24.04 and Debian 13 images; `gstreamer1.0-plugins-bad` pulls in most of it). Allow twice the model size so an upgrade can download a new model version beside the old one. |
| License key | Required, free tier included. See [Licensing](../deployment.md#licensing). |

How many calls a CPU carries depends on its generation more than its core count.
In Synauson's voice-agent benchmarks, STT ran about one call per vCPU on current Intel
Xeons (Emerald Rapids, with AVX-512, VNNI and AMX) and about half that on older Intel (n2) and AMD Genoa. On one thread, a
Broadwell core (AVX2, no VNNI) decoded STT about five times slower than a current
Xeon core (real-time factor 2.1 against 0.43). Synauson measures the host when STT starts and reports the result
in `GetCapabilities` (see [STT capacity](../deployment.md#stt-capacity)).

## Network

| Direction | Endpoint | Why |
|---|---|---|
| Outbound HTTPS (TCP 443) | `usage.synauson.com` | Required on every instance: session-count reports every 30 seconds (the license's shared pool) and license renewal at startup and about once a day. Up to 7 days without it are fine; an air-gapped license needs none |
| Outbound HTTPS (TCP 443) | `license.synauson.com` | Model downloads, and the first start without a cached license |
| Outbound HTTPS (TCP 443) | `status.synauson.com` | Synauson's outage notices: while one is up, an instance whose reports fail keeps its 7 days paused. Blocked, an instance gets no pause |
| Outbound HTTPS (TCP 443) | `*.r2.cloudflarestorage.com` (Cloudflare R2) | Downloads the models the license includes, once per model version: `license.synauson.com` redirects each file there |
| Outbound UDP 123 (NTP) | your time source | Keep the clock synchronized: licensing answers count only within 5 minutes of it, outage notices within 10 |
| Outbound HTTPS (TCP 443) | `cr.synauson.com` (container), `maven.synauson.com` (jsyn) | Where the image and the jsyn jars come from, at install time only |
| Outbound UDP (the STUN server's port) | the STUN server you set | Only with `--webrtc-stun-server` (jsyn: `webrtcStunServer`); Synauson uses no STUN server unless you set one |
| Inbound TCP | 50051 by default (`--listen`) | gRPC clients (server only) |
| Inbound and outbound UDP | the SIP RTP port range and the WebRTC ICE port range | Media; see the [firewall section](linux.md#firewall) |

A running server keeps working when the licensing server can't be reached: it uses
its cached license file. A host with no internet access at all needs a license file
from Synauson and the models copied in by hand: see
[Offline hosts](linux.md#offline-hosts).
