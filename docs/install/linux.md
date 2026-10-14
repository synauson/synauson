# Installing Synauson on Linux

Requirements, network endpoints and the choice between these paths are in
[the install overview](README.md). Flags, authentication and licensing behavior are
in [`deployment.md`](../deployment.md).

## Contents

- [Container](#container)
- [Native server (systemd)](#native-server-systemd)
- [Embedded (jsyn)](#embedded-jsyn)
- [Firewall](#firewall)
- [Offline hosts](#offline-hosts)
- [Troubleshooting](#troubleshooting)

## Container

The image holds the server, ONNX Runtime and the GStreamer plugins it needs, so the
host needs only a container runtime. It is the recommended way to run the server, and
the only one on distributions whose GStreamer is unsupported (see
[Native server](#native-server-systemd)).

1. Install Docker Engine ([docs.docker.com/engine/install](https://docs.docker.com/engine/install/))
   or Podman. On a host with an NVIDIA GPU, nothing NVIDIA is needed: Synauson runs
   on the CPU (see [`nvidia.md`](nvidia.md)).

2. Pull the image. Images are public, need no login, and are published for every
   release, with a rolling `edge` tag of unreleased development builds:

   ```
   cr.synauson.com/synauson/synauson-server:<version>   # e.g. the tag v1.5.0 publishes 1.5.0
   cr.synauson.com/synauson/synauson-server:latest      # newest non-prerelease tag
   cr.synauson.com/synauson/synauson-server:edge        # development builds
   ```

   Pin a version in production.

3. Run it with a license key and two volumes. The state volume keeps the cached
   license file and this instance's shares of the license's pools, so a
   restart works while `usage.synauson.com` is unreachable (for 7 days after the
   last report that got through), and the
   calibration cache, so a restart doesn't time the models again; the models
   volume keeps the downloaded models, so they download once.

   ```bash
   export SYNAUSON_LICENSE_KEY=<your license key>
   docker run -d --name synauson --restart unless-stopped \
     -p 50051:50051 \
     -e SYNAUSON_LICENSE_KEY \
     -v synauson-state:/var/lib/synauson/state \
     -v synauson-models:/var/lib/synauson/models \
     cr.synauson.com/synauson/synauson-server:<version> \
     --listen 0.0.0.0:50051
   ```

   With Docker Compose:

   ```yaml
   services:
     synauson:
       image: cr.synauson.com/synauson/synauson-server:<version>
       command: ["--listen", "0.0.0.0:50051"]
       environment:
         SYNAUSON_LICENSE_KEY: ${SYNAUSON_LICENSE_KEY:?set SYNAUSON_LICENSE_KEY}
       ports:
         - "50051:50051"
       volumes:
         - synauson-state:/var/lib/synauson/state
         - synauson-models:/var/lib/synauson/models
       restart: unless-stopped
   volumes:
     synauson-state:
     synauson-models:
   ```

   At startup the server downloads the models the license includes; features whose
   model is still downloading fail with a precise error until it arrives (see
   [Models](../deployment.md#models)).

   The server in the image runs as the unprivileged user `synauson`, uid and gid
   10001. Named volumes, as above, take that ownership from the image. A host
   directory you bind-mount instead must be writable by it
   (`sudo chown -R 10001:10001 <dir>`), and files you mount read-only, such as TLS
   keys, readable by it.

4. Media ports. SIP and WebRTC participants send and receive RTP on UDP ports from
   the ranges in [Firewall](#firewall). With bridge networking each of those ports
   must be published (`-p 40000-40999:40000-40999/udp`) and the server sees only the
   container's own address, so for SIP or WebRTC media run the container with
   `--network host` (Compose: `network_mode: host`) and drop the `-p` options.

5. [Verify it works](#verify-a-server).

## Native server (systemd)

For hosts that run the binary directly, for example for tight CPU pinning or direct
access to RTP sockets.

### 1. Check the distribution

The server needs GStreamer 1.24 or 1.26 from the distribution's packages:

| Distribution | GStreamer | Native install |
|---|---|---|
| Ubuntu 24.04 LTS | 1.24.2 | Supported; Linux CI runs on it |
| Debian 13 (trixie) | 1.26.2 | Supported; the container image is built on it |
| Ubuntu 26.04 LTS | 1.28 | Not supported: 1.28 breaks WebRTC. Use the container. |
| Debian 12, Ubuntu 22.04 | 1.22, 1.20 | Too old. Use the container. |
| Fedora, RHEL and derivatives | 1.26 (RHEL 10), 1.28 (Fedora 44) | Not supported: `gstreamer1-plugins-bad-free` lacks `dtmfdetect` (spandsp), which every SIP participant needs, and Fedora's GStreamer is 1.28. Use the container. |

### 2. Install GStreamer

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
    libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad gstreamer1.0-nice gstreamer1.0-tools \
    ca-certificates curl
```

`gstreamer1.0-nice` is webrtcbin's ICE backend and is easy to miss.
`gstreamer1.0-tools` provides `gst-inspect-1.0` for the check below; the server
itself doesn't need it. `ca-certificates` is the trust store Synauson's HTTPS
client uses for `license.synauson.com` and the model downloads from Cloudflare R2. The
container image installs the same packages.

Check that every element Synauson creates is present; the command prints `ok`:

```bash
gst-inspect-1.0 --version
for e in webrtcbin nicesrc nicesink dtlssrtpenc srtpenc srtpdec rtpbin \
         rtpjitterbuffer udpsrc udpsink multiudpsink audiomixer audioconvert \
         audioresample opusenc opusdec rtpopuspay rtpopusdepay mulawenc mulawdec \
         alawenc alawdec rtppcmupay rtppcmapay dtmfsrc rtpdtmfsrc rtpdtmfdepay \
         rtpdtmfmux dtmfdetect errorignore decodebin3 urisourcebin wavenc \
         appsrc appsink tee queue capsfilter; do
  gst-inspect-1.0 --exists "$e" || echo "missing: $e"
done; echo ok
```

### 3. Install ONNX Runtime

ONNX Runtime isn't packaged by the distributions. Install Microsoft's CPU release at
the version Synauson ships, which `GetCapabilities` reports as
`onnxRuntime.shippedVersion`. Another version runs, but model speed and outputs can
differ:

```bash
ORT_VERSION=1.24.4   # the release synauson ships
curl -fsSL "https://github.com/microsoft/onnxruntime/releases/download/v${ORT_VERSION}/onnxruntime-linux-x64-${ORT_VERSION}.tgz" \
  | sudo tar -xz -C /opt/
sudo ln -sfn "/opt/onnxruntime-linux-x64-${ORT_VERSION}" /opt/onnxruntime
ls /opt/onnxruntime/lib/libonnxruntime.so
```

This is the CPU build. Don't use the `-gpu` archives: Synauson doesn't use them yet
(see [`nvidia.md`](nvidia.md)).

Synauson finds the library through `ORT_DYLIB_PATH`, set in the systemd unit below.
Without it, the system library path is searched, which can find another release.

### 4. Install the binary

Synauson publishes the server as a container image, not as a separate download.
Copy the binary out of the image for the release you want; it needs glibc 2.34 or
newer and runs on both distributions above:

```bash
docker create --name synauson-extract cr.synauson.com/synauson/synauson-server:<version>
docker cp synauson-extract:/usr/local/bin/synauson ./synauson
docker rm synauson-extract
```

Create the service account, then install the binary and its directories. The server
creates `state/` and `models/` under `/var/lib/synauson` itself, so the service user
must own it:

```bash
sudo groupadd -r synauson
sudo useradd  -r -g synauson -s /usr/sbin/nologin \
    -d /var/lib/synauson -c "synauson media server" synauson
sudo install -m 0755 -o root -g root -t /usr/local/bin/ ./synauson
sudo install -d -m 0750 -o synauson -g synauson /var/lib/synauson /var/log/synauson
sudo install -d -m 0750 -o root -g synauson /etc/synauson
synauson --version
```

### 5. Add the systemd unit

`/etc/systemd/system/synauson.service`:

```ini
[Unit]
Description=synauson audio media server
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
User=synauson
Group=synauson
WorkingDirectory=/var/lib/synauson

# The ONNX Runtime library installed in step 3.
Environment=ORT_DYLIB_PATH=/opt/onnxruntime/lib/libonnxruntime.so

# Server config; override via EnvironmentFile if you prefer
ExecStart=/usr/local/bin/synauson \
    --listen 0.0.0.0:50051 \
    --model-store /var/lib/synauson/models \
    --log-format json

# Per-deployment settings, including the license key
EnvironmentFile=-/etc/synauson/synauson.env

# SIGTERM ends every conference, waiting at most 5 s, then exits.
# When conferences need longer to end, call SynAdmin.ShutdownServer
# before stopping the unit (see Signals in deployment.md).
TimeoutStopSec=30
KillSignal=SIGTERM

# Logging, captured by journald
StandardOutput=journal
StandardError=journal

# Hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/synauson /var/log/synauson
PrivateTmp=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictRealtime=true
LockPersonality=true
MemoryDenyWriteExecute=false   # GStreamer JIT plugins require W^X be disabled
SystemCallArchitectures=native

# Auto-restart on crash
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

`/etc/synauson/synauson.env` (owner root, group `synauson`, mode 0640) holds the
license key and any deployment-specific settings from
[Configuration](../deployment.md#configuration):

```sh
SYNAUSON_LICENSE_KEY=replace-with-your-license-key
SYNAUSON_AUTH=token
SYNAUSON_TOKEN=replace-with-a-32-byte-secret
SYNAUSON_MAX_CONFERENCES=200
SYNAUSON_LOG_LEVEL=info,synauson_server=debug
```

```bash
sudo install -m 0640 -o root -g synauson /dev/null /etc/synauson/synauson.env
sudoedit /etc/synauson/synauson.env
```

### 6. Fetch the models (optional)

The service downloads the licensed models into `/var/lib/synauson/models` when it
first starts. To fetch them ahead of time:

```bash
sudo -u synauson env SYNAUSON_LICENSE_KEY=<your license key> \
    synauson --model-store /var/lib/synauson/models models pull
```

On a host without internet access, see [Offline hosts](#offline-hosts).

### 7. Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now synauson.service
sudo systemctl status synauson.service
journalctl -u synauson.service -f
```

The startup log names the GStreamer version (`synauson starting`), the ONNX Runtime
it loaded (`onnx runtime loaded`, with version and path), the license state and each
model's state. Then [verify it works](#verify-a-server).

### Updating

```bash
sudo systemctl stop synauson
sudo install -m 0755 -o root -g root -t /usr/local/bin/ ./synauson
sudo systemctl start synauson
```

`systemctl stop` sends SIGTERM, which ends every conference within 5 s; call
`SynAdmin.ShutdownServer` first when they need longer (see
[Signals](../deployment.md#signals)). When a release moves
to a new ONNX Runtime version, repeat step 3 with the new `ORT_VERSION`.

## Verify a server

The server doesn't offer gRPC reflection, so `grpcurl` needs the `.proto` files.

1. Health. [`grpc_health_probe`](https://github.com/grpc-ecosystem/grpc-health-probe)
   needs no proto files and exits 0 when the server is serving:

   ```bash
   grpc_health_probe -addr=localhost:50051
   ```

   Kubernetes can probe the same service natively with a `grpc:` liveness or readiness
   probe on port 50051.

2. Capabilities, with the proto files from [synauson/api](https://github.com/synauson/api)
   (`git clone https://github.com/synauson/api`):

   ```bash
   grpcurl -plaintext -import-path api -proto synauson/v1/synauson.proto \
       -d '{}' localhost:50051 synauson.v1.Syn/GetCapabilities
   ```

   Add `-H 'authorization: Bearer <token>'` with `--auth=token`. Check that:

   - `license.state` is `LICENSE_STATE_VALID` (or `LICENSE_STATE_EXPIRING`), `license.plan` is the plan you
     have (`LICENSE_PLAN_FREE`, `_PAID` or `_DEVELOPER`), `sessions.limit` and
     `conferences.limit` are the pools you bought (4 and 10 on a Free key), and
     `capabilities` lists the capabilities you expect;
   - `onnxRuntime.state` is `ONNX_RUNTIME_STATE_LOADED` and `onnxRuntime.version` equals
     `onnxRuntime.shippedVersion`;
   - each model's `state` is `MODEL_STATE_READY` (`MODEL_STATE_DOWNLOADING` right after
     the first start; a `MODEL_STATE_MISSING` with a download error in `detail` is retried
     on its own);
   - with STT in the license, `stt.state` is `POOL_STATE_READY` and `stt.streams.limit`
     shows how many STT streams this host transcribes in real time.

The full RPC catalog is in [`api.md`](../api.md).

## Embedded (jsyn)

jsyn loads the engine into a JVM. Its README lists the requirements and the
install steps: [jsyn requirements](https://github.com/synauson/jsyn#requirements).
On Linux that is Java, the GStreamer packages from step 2 of the native install
(the same list, `gstreamer1.0-tools` optional), and a license key. ONNX Runtime
ships inside the `jsyn-natives-linux` jar, so don't install it or set
`ORT_DYLIB_PATH`. The distribution table above applies to jsyn too.

The JVM process receives RTP and WebRTC media itself, so open its port ranges as in
[Firewall](#firewall). jsyn's RTP range is `JSynConfig.Builder.rtpPortMin` and
`rtpPortMax` (defaults in its javadoc), and its WebRTC range `webrtcIcePortMin` and
`webrtcIcePortMax`.

## Firewall

| Traffic | Ports | Set by |
|---|---|---|
| gRPC (server only) | TCP 50051 | `--listen` |
| SIP media (RTP and RTCP) | UDP 49152 to 65534 by default | `--rtp-port-min`, `--rtp-port-max` |
| WebRTC media and ICE | Any free port unless a range is set | `--webrtc-ice-port-min`, `--webrtc-ice-port-max` (UDP, plus TCP for ICE-TCP) |

Set a WebRTC range for any host behind a firewall, keep it clear of the RTP range, and
size it at one UDP and one TCP port per concurrent WebRTC participant per local
address ([WebRTC ports](../deployment.md#webrtc-ports)). For example, with ufw:

```bash
sudo ufw allow 50051/tcp
sudo ufw allow 49152:65534/udp                 # SIP RTP
sudo ufw allow 40000:40999/udp                 # with --webrtc-ice-port-min 40000 --webrtc-ice-port-max 40999
sudo ufw allow 40000:40999/tcp
```

With firewalld (container hosts on the Red Hat family):

```bash
sudo firewall-cmd --permanent --add-port=50051/tcp \
    --add-port=49152-65534/udp --add-port=40000-40999/udp --add-port=40000-40999/tcp
sudo firewall-cmd --reload
```

Outbound, allow HTTPS to `usage.synauson.com` (required: usage reports every 30
seconds and license renewal; see [the shared pools](../deployment.md#pools-shared-across-all-your-instances)),
`license.synauson.com` and `*.r2.cloudflarestorage.com` (model downloads, and a start
without a cached license file), `status.synauson.com` (Synauson's outage notices),
NTP to your time source (keep the clock within a few minutes), and UDP to the STUN
server if you set one (`--webrtc-stun-server`). Synauson's HTTPS client honors `HTTPS_PROXY` and `NO_PROXY`.

## Offline hosts

A host that can't reach Synauson (`usage.synauson.com`, `license.synauson.com`)
and Cloudflare R2:

1. Arrange an air-gapped license with Synauson and get its license file. Put it
   on the host (for example in `/etc/synauson/`), and start the server with
   `--license-file <path> --offline`. An air-gapped license sends no usage
   reports, downloads no models and makes no other request to Synauson, even
   without `--offline`, and works until its file expires.
2. Get the model files from Synauson as one flat folder: the files sit directly in
   it under their catalog names (`sentito-1.onnx`, `fermata-1.onnx`, the STT model's
   `spartito-1-*` files and so on, each model with its `<id>-NOTICE.txt`), with no
   subfolders. Copy it to the host and import it into the
   store, which lays the files out as `<model-id>/<variant>/<file>`. Import takes
   every model whose files are all there, refuses a model with only some of them,
   and checks each file's size and SHA-256 against the catalog built into the
   binary:

   ```bash
   sudo -u synauson synauson --model-store /var/lib/synauson/models models import /path/to/models
   sudo -u synauson synauson --model-store /var/lib/synauson/models models verify
   ```

   In a container, mount the folder and run the same subcommands with the image
   (`docker run --rm -v synauson-models:/var/lib/synauson/models -v /path/to/models:/import:ro
   cr.synauson.com/synauson/synauson-server:<version> models import /import`).

## Troubleshooting

Start with the startup log and [`GetCapabilities`](#verify-a-server); the
[server troubleshooting list](../deployment.md#troubleshooting) covers licensing,
model and limit errors once the server runs.

| Symptom | Cause and fix |
|---|---|
| `error while loading shared libraries: libgstwebrtc-1.0.so.0` (or `libgstanalytics`, `libgstsdp`, `libgstapp`) | GStreamer's libraries are missing. Install the packages in [step 2](#2-install-gstreamer); the first two come with `gstreamer1.0-plugins-bad`. |
| Exits with `cannot load ONNX Runtime from 'libonnxruntime.so'` or another path | `ORT_DYLIB_PATH` is unset or points at a file that doesn't exist. Set it to the library from [step 3](#3-install-onnx-runtime). The license includes AI features, so the server refuses to start without the runtime. |
| `does not provide the API version ort needs` | The runtime is older than 1.23. Install the release in [step 3](#3-install-onnx-runtime). |
| Warning `onnx runtime is not the shipped release` | Another ONNX Runtime version loaded, for example one found on the library path. It runs, but speed and model outputs can differ; point `ORT_DYLIB_PATH` at the shipped release. |
| A missing element in the check of step 2, or `INTERNAL` errors naming `ElementFactory::make(<element>) failed` when a participant is added | A plugin package is missing: `nicesrc` and `nicesink` come from `gstreamer1.0-nice`; `webrtcbin`, `srtpenc`, `dtlssrtpenc`, `dtmfdetect` and `errorignore` from `gstreamer1.0-plugins-bad`; the RTP, µ-law and A-law elements from `-good`; Opus and the mixer from `-base`. |
| WebRTC participants never connect while SIP works | `gstreamer1.0-nice` is missing, GStreamer is 1.28, or the ICE port range is blocked or used up (the log says `webrtc ice gathering produced no candidates`). |
| SIP calls connect but carry no audio | The RTP range is blocked by a firewall, or not published from the container. |
| Exits with `synauson cannot start without a valid license` | No key, a malformed key, a key the licensing server refused, or no working license file (the log says which). An unreachable `usage.synauson.com` is fatal only without one: the server starts on its cached license file for 7 days after the last usage report that got through, logs warnings, and renews when the service answers; with no file that still works ("synauson needs a valid license to start"), it can't start. Check outbound HTTPS to `usage.synauson.com` and `license.synauson.com` (`curl -i https://usage.synauson.com/v1/ping` answers 204), the proxy settings, and that `ca-certificates` is installed. If the message says the license "is not a valid synauson license", the license itself is illegal for this build: send the listed problems to Synauson. |
| `Permission denied` under `/var/lib/synauson` | The service user doesn't own the directory. Re-run the `install -d` commands in [step 4](#4-install-the-binary). |
