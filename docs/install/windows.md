# Installing Synauson on Windows

On Windows, Synauson runs inside a Java application through
[jsyn](https://github.com/synauson/jsyn). The gRPC server is supported on Linux only:
run it there ([Linux](linux.md)) and call it from Windows over the network.

jsyn's README is the reference for the Java side: its
[requirements](https://github.com/synauson/jsyn#requirements), the Maven
coordinates, and a [Windows quickstart](https://github.com/synauson/examples/tree/main/java/jsyn-windows-quickstart).
This page collects everything a Windows machine needs, including what that README
leaves out.

## Contents

- [What to install](#what-to-install)
- [1. Java](#1-java)
- [2. Visual C++ runtime](#2-visual-c-runtime)
- [3. GStreamer runtime](#3-gstreamer-runtime)
- [4. Environment variables](#4-environment-variables)
- [5. Plugin registry](#5-plugin-registry)
- [6. Firewall](#6-firewall)
- [7. Verify it works](#7-verify-it-works)
- [CPU and memory limits](#cpu-and-memory-limits)
- [Troubleshooting](#troubleshooting)

## What to install

| | What | Notes |
|---|---|---|
| Windows | 10 or 11, x86_64 | No Windows on ARM build |
| Java | 11 or newer | |
| Visual C++ runtime | The latest Microsoft Visual C++ v14 Redistributable, x64 | `synauson_jni.dll` and ONNX Runtime link against it |
| GStreamer | The 1.26.7 MSVC x86_64 runtime installer | The version Synauson is tested with on Windows. 1.28 is not supported. You don't need the devel installer. |
| ONNX Runtime | Nothing to install | Ships inside the `jsyn-natives-windows` jar |
| NVIDIA software | Nothing to install | Synauson runs on the CPU on every machine today; see [`nvidia.md`](nvidia.md) |
| License key | Required, free tier included | See the [install overview](README.md#network) for the hosts it contacts |

Run the PowerShell commands below in an elevated (Administrator) PowerShell.

## 1. Java

Install a JDK or JRE 11 or newer, for example Eclipse Temurin from
[adoptium.net](https://adoptium.net/), or with winget:

```powershell
winget install --id EclipseAdoptium.Temurin.21.JDK -e
java -version
```

## 2. Visual C++ runtime

`synauson_jni.dll` imports `VCRUNTIME140.dll`, and ONNX Runtime also imports
`VCRUNTIME140_1.dll`, `MSVCP140.dll` and `MSVCP140_1.dll`. Many machines already
have them, but a clean Windows install doesn't, and GStreamer's `bin` folder doesn't
carry them. Install Microsoft's
[latest supported Visual C++ v14 Redistributable](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist)
(x64):

```powershell
Start-BitsTransfer -Source 'https://aka.ms/vc14/vc_redist.x64.exe' -Destination "$env:TEMP\vc_redist.x64.exe"
Start-Process "$env:TEMP\vc_redist.x64.exe" -Wait -ArgumentList '/install', '/quiet', '/norestart'
Test-Path "$env:WINDIR\System32\VCRUNTIME140_1.dll"   # True
```

## 3. GStreamer runtime

Install the MSVC x86_64 runtime MSI system-wide with every feature, into
`C:\gstreamer`. Use 1.26.7, the version Synauson is tested with on Windows; it is also
the last release with an MSI that installs silently.

```powershell
$gst = '1.26.7'
Start-BitsTransfer -Source "https://gstreamer.freedesktop.org/data/pkg/windows/$gst/msvc/gstreamer-1.0-msvc-x86_64-$gst.msi" -Destination "$env:TEMP\gst-runtime.msi"
Start-Process msiexec.exe -Wait -ArgumentList '/i', "$env:TEMP\gst-runtime.msi", '/quiet', '/norestart', 'ADDLOCAL=ALL', 'INSTALLDIR=C:\gstreamer\'
Test-Path 'C:\gstreamer\1.0\msvc_x86_64\bin\gstreamer-1.0-0.dll'   # True
```

In the graphical installer, choose **Complete**, the equivalent of `ADDLOCAL=ALL`.
CI tests only the complete install; a smaller one can leave out plugins Synauson
needs.

## 4. Environment variables

On the machines we checked, the installer set `GSTREAMER_1_0_ROOT_MSVC_X86_64` to
`C:\gstreamer\1.0\msvc_x86_64\` but did not add GStreamer's `bin` folder to `Path`,
and Windows loads `synauson_jni.dll`'s GStreamer dependencies from `Path`. Set both
machine-wide:

```powershell
[Environment]::SetEnvironmentVariable('GSTREAMER_1_0_ROOT_MSVC_X86_64', 'C:\gstreamer\1.0\msvc_x86_64\', 'Machine')
$p = [Environment]::GetEnvironmentVariable('Path', 'Machine')
if ($p -notlike '*C:\gstreamer\1.0\msvc_x86_64\bin*') {
    [Environment]::SetEnvironmentVariable('Path', "$p;C:\gstreamer\1.0\msvc_x86_64\bin", 'Machine')
}
```

Sign out and back in (or restart the service that runs your application) so new
processes see the change. A process started before it keeps the old `Path`.

Don't set `ORT_DYLIB_PATH` for jsyn: the natives jar loads its own ONNX Runtime.

## 5. Plugin registry

Once per Windows user account that runs the application, build GStreamer's plugin
registry:

```powershell
gst-inspect-1.0.exe coreelements
```

Without it, the first `new JSyn(...)` performs the scan, which took 7 to 44 seconds
on fresh CI machines. For a Windows service, run it as the service's account.

## 6. Firewall

The JVM receives SIP and WebRTC media itself, on the UDP ports jsyn's
`JSynConfig.Builder.rtpPortMin`/`rtpPortMax` (SIP) and
`webrtcIcePortMin`/`webrtcIcePortMax` (WebRTC) set; see their javadoc for the
defaults. The first time `java.exe` listens, Windows Defender Firewall may ask
whether to allow it; on a server nobody answers, so inbound media is blocked. Add
rules for the Java executable that runs your application, with your ranges:

```powershell
$java = (Get-Command java.exe).Source   # or the full path your service uses
New-NetFirewallRule -DisplayName 'synauson SIP RTP' -Direction Inbound -Action Allow `
    -Program $java -Protocol UDP -LocalPort 10000-20000
New-NetFirewallRule -DisplayName 'synauson WebRTC ICE (UDP)' -Direction Inbound -Action Allow `
    -Program $java -Protocol UDP -LocalPort 40000-40999
New-NetFirewallRule -DisplayName 'synauson WebRTC ICE (TCP)' -Direction Inbound -Action Allow `
    -Program $java -Protocol TCP -LocalPort 40000-40999
```

Replace `10000-20000` with your RTP range and `40000-40999` with the WebRTC range you
configure. Set a WebRTC range on any machine behind a firewall: unset, ICE uses any
free port. Outbound, allow HTTPS to `usage.synauson.com` (required: usage reports and
license renewal), `license.synauson.com` and `*.r2.cloudflarestorage.com` (model
downloads, and a start without a cached license file), `status.synauson.com`
(Synauson's outage notices), NTP to your time source (Windows Time; keep the clock
within a few minutes), and UDP to the STUN server if you set one.

## 7. Verify it works

From a new PowerShell window:

```powershell
$env:GSTREAMER_1_0_ROOT_MSVC_X86_64
where.exe gstreamer-1.0-0.dll        # C:\gstreamer\1.0\msvc_x86_64\bin\gstreamer-1.0-0.dll
gst-inspect-1.0.exe --version        # GStreamer 1.26.7
foreach ($e in 'webrtcbin','nicesrc','srtpenc','dtmfdetect','errorignore','audiomixer','opusenc','rtpbin','decodebin3','wavenc') {
    gst-inspect-1.0.exe --exists $e; if ($LASTEXITCODE -ne 0) { "missing: $e" }
}
```

Nothing printed after the version means every element is present. Then run jsyn's
[Quickstart](https://github.com/synauson/jsyn#quickstart) with
`SYNAUSON_LICENSE_KEY` set, and check `jsyn.capabilities()`: the license state, each
model's state (`ready` once downloaded), and with STT in the license, `stt.state`
`ready`. The ONNX Runtime it loaded, and from which file, is in the startup log; jsyn's
`Capabilities` doesn't expose it.

## CPU and memory limits

Synauson sizes its STT and turn detection pools from the CPUs the JVM may use: the
process affinity mask (`start /affinity`, or a job's affinity limit) and, when the
JVM runs in a job object with a hard CPU cap (a Windows container started with
`--cpus`, or a service host that sets one), that cap. A job's memory limit bounds
how many STT workers load. The `resources` section of `jsyn.capabilities()` shows
what was detected and what set it (`limitedBy`: `affinity`, `job_cpu_rate` or
`std_parallelism`). To set the budget yourself, for example when other services share
the machine, give jsyn's config `cpuBudget` (cores) and `memoryBudget` (bytes).

## Troubleshooting

jsyn's README has a [troubleshooting table](https://github.com/synauson/jsyn#troubleshooting)
for its own errors. On Windows also check:

| Symptom | Cause and fix |
|---|---|
| `UnsatisfiedLinkError: ... Can't find dependent libraries` loading `synauson_jni.dll` | GStreamer's `bin` folder isn't on the process's `Path`, or the Visual C++ runtime is missing. Run the checks in [step 7](#7-verify-it-works) as the account that runs the application, and install [step 2](#2-visual-c-runtime). |
| The same error only for `onnxruntime.dll` | The Visual C++ runtime is missing or older than ONNX Runtime needs. Install the latest one ([step 2](#2-visual-c-runtime)). |
| Works in a terminal, fails as a service or in the IDE | That process started before `Path` changed, or runs as another user. Restart it; for a service, sign the service account in once or restart the machine. |
| `GStreamer sanity check failed: required GStreamer element '…' not found` | The typical GStreamer install left plugins out. Reinstall with **Complete** or `ADDLOCAL=ALL` ([step 3](#3-gstreamer-runtime)). |
| WebRTC participants never connect | The ICE ports are blocked ([step 6](#6-firewall)), or GStreamer is 1.28. |
| SIP calls connect but carry no audio | Inbound UDP to the RTP range is blocked for `java.exe` ([step 6](#6-firewall)). |
| First `new JSyn(...)` takes tens of seconds | The plugin registry isn't built for this user ([step 5](#5-plugin-registry)). |
| A warning that the ONNX Runtime isn't the shipped release, or that another ONNX Runtime library is loaded | Another `onnxruntime.dll` was loaded first, for example an old copy in `C:\Windows\System32` or one that another library in the same JVM brought along. Remove it from the process's `Path`. |
