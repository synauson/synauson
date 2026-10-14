# Synauson Deployment Guide

This guide covers configuring and operating Synauson as a **gRPC server**: a separate
process that gRPC clients connect to over the network or via loopback. To install it
(container image, or the native binary under systemd), see [`install/`](install/README.md).
For the **in-process** model, which loads the engine into your JVM, see
[synauson/jsyn](https://github.com/synauson/jsyn). Moving from 1.7:
[Upgrading to 1.8.0](upgrading/1.8.0.md).

## Deployment modes at a glance

| Mode | What it is | Install |
|---|---|---|
| **Container (recommended)** | OCI image with Synauson, GStreamer and ONNX Runtime | [Linux: container](install/linux.md#container) |
| **systemd service (binary)** | Native Linux binary managed by systemd | [Linux: native server](install/linux.md#native-server-systemd) |

The server is supported on Linux only. Windows callers reach a Linux-hosted server
over gRPC, or embed the engine with jsyn ([Windows install](install/windows.md)).

## Models

At startup Synauson downloads the AI models your license includes into its model store (`/var/lib/synauson/models`; mount a volume there too, so they're downloaded once). Downloads go through `license.synauson.com`, which redirects each file to a short-lived signed URL on Cloudflare R2 storage (`*.r2.cloudflarestorage.com`), and every file is checked against the size, SHA-256 and Synauson signature pinned in the binary before it's used. The server doesn't wait for them: a feature whose model is still downloading fails with a precise error until it arrives. A failed download is retried with backoff (30 s doubling to 30 min), and a license change (an upgrade) downloads the newly licensed models without a restart. Instances that share a store download each model once: the others wait on its `<variant>.download.lock` and report it as `downloading`. `synauson models pull` downloads them ahead of time. To use a different store, mount it at that path or pass `--model-store` / `SYNAUSON_MODEL_STORE`. A store is laid out as `<model-id>/<variant>/<file>`; it can also be filled with `synauson models import <folder>` from a flat folder of model files ([offline hosts](install/linux.md#offline-hosts)), and checked with `synauson models verify`.

The first STT start on a machine also writes a derived copy of the STT encoder into the store, under `<model-id>/<variant>/derived/` (about 1.2 GB per ONNX Runtime version and CPU type), holding its weights in the layout this CPU uses. Every STT worker in every Synauson process that uses the same store maps that one file, so the weights take about 600 MB of shared memory once instead of about 800 MB per worker. Building it takes about 20 seconds on an 8-core machine and happens while the STT pool loads, so STT stays `loading` in `GetCapabilities` that much longer on the first start, and an STT participant added meanwhile fails with `MODEL_LOADING` (`UNAVAILABLE`). Later starts reuse it. Several processes starting together build it once (the others wait on a lock file beside it). Instances on one host should therefore share one writable store: in Kubernetes, a node-local volume mounted into each pod. With a read-only store (an image volume) or no free disk, STT still starts, with a private copy of the weights per worker, and logs `shared model weights unavailable, loading in-process` at WARN with the reason. Hosts with different CPUs can share a store: each CPU type keeps its own copy. A copy left by an older ONNX Runtime or Synauson version is removed by a later build once no process uses it, so a rolling upgrade on a shared store (NFS included) never pulls files from under a running instance. On NFS the locks only work across hosts when the share is not mounted with `local_lock=flock` (or `all`). The derived directory can be deleted while Synauson is stopped; it is rebuilt on the next start.

## Licensing

Every Synauson install runs under a license key from Synauson. Every license has every
AI feature (VAD, turn detection, STT, TTS) on every session; licenses differ in how many
concurrent AI sessions and concurrent conferences they buy. A Free key buys none: the
Free base of 4 concurrent AI sessions and 10 concurrent conferences, no burst, no expiry,
production use included. A developer key, which Synauson issues for development and
testing only (not for production use), has the same pools for one year; Synauson logs
"developer license: for development and testing only, not for production use" at WARN when it starts on one, and once it
expires it starts nothing new (`LICENSE_EXPIRED`) until Synauson renews it.
The key is set with `--license-key` / `SYNAUSON_LICENSE_KEY`; in Kubernetes, put it
in a Secret and expose it as that environment variable.

At startup Synauson renews its license through `usage.synauson.com`, which fetches a
signed license file from `license.synauson.com` on its behalf; Synauson caches it at
`<state dir>/license.lic` (owner-only permissions; it contains the key) and renews it
about once a day. It is verified offline with a key built into Synauson, and says how
large the license's two pools are. They are independent, and both apply:

- **Concurrent AI sessions.** A session is one participant using any AI feature,
  whatever it uses; participants without AI take none. The pool is the Free base of 4
  plus the sessions the license buys (`purchasedSessions`).
- **Concurrent conferences.** Every live conference, with AI or without, takes one
  from when it starts until it ends. The pool is the Free base of 10 plus the
  conferences the license buys (`purchasedConferences`).

An order-form license may make either pool unlimited instead. So on a Free key, a
caller talking to an AI agent uses 1 of 10 conferences and 1 of 4 sessions, and a 3-way
conference with one AI participant uses the same. From 80% of a pool each new session
or conference logs a warning; above it the license's burst (`overdraft`, 25% of the
purchased units unless the license says otherwise, one figure for both pools; none on
the base) still admits them, each logged as overage; past the burst a new one is
refused with `RESOURCE_EXHAUSTED`: `SESSION_LIMIT_REACHED` for an AI participant,
`CONFERENCE_POOL_LIMIT_REACHED` for a conference. So 10 purchased sessions allow 14, up
to 17 as a burst (4 + 12.5, rounded up), and 10 purchased conferences allow 20, up to 23.
The burst belongs to the license as a whole, so a single instance gets all of it and
several instances' shares of it add up to it exactly. Running sessions and conferences
are never stopped, and a full conference pool never refuses a participant joining a
running conference. Every license downloads every model, the TTS model (about 347 MB
with its voices and pronunciation data) with the STT model.

`--max-conferences` is your own cap on one instance, beside the license's conference
pool: both apply, and the smaller wins. It is checked first (`CONFERENCE_LIMIT_REACHED`),
so a refusal by your cap never takes a pool unit and never counts as admitted, refused
or overage.

### Pools shared across all your instances

The pools belong to the license, not to one server: every server replica and every
JVM running jsyn under the same key, at any site, shares them. Each instance reports
its AI session and conference counts to `usage.synauson.com` every 30 seconds and gets
back its share of each pool, sized by what each instance is using, with the burst on
its purchased part. Calls are admitted locally against that share, so no call waits
on the network, and traffic that moves to another site gets the capacity within
about 10 minutes. A single instance gets the whole of each pool. `GetCapabilities`
shows the pools (`sessions`, `conferences`), this instance's shares and the
license-wide counts (`usage_service`), the license's plan (`LicenseInfo.plan` over
gRPC, `license.plan` in jsyn: Free, paid, Developer, or unlicensed when no license
is in force), and `limits_scope` is `LIMITS_SCOPE_LICENSE` (jsyn: "all
instances of this license").

A report holds only counts: the license and a random id per process, sessions and
conferences in use and their per-minute peaks, admitted, refused and overage counts,
the version and adapter. No call content, audio, transcripts, conference or
participant ids, or hostnames. Each report is logged at DEBUG, so you can see exactly
what is sent. The service also records, per license, the public IP address each report
comes from (an IPv6 /64), with its country and when it was first and last seen, kept 90
days after last seen and used only to detect misuse of Free and developer keys.

**Every instance must be able to reach these over HTTPS (port 443)**; allow them
in your firewall or egress proxy:

- `usage.synauson.com`: usage reports and license renewal. Test egress with
  `curl -i https://usage.synauson.com/v1/ping` (it answers 204).
- `license.synauson.com`: model downloads (which redirect to
  `*.r2.cloudflarestorage.com`, so allow that too), and a start without a cached
  license file.
- `status.synauson.com`: Synauson's outage notices. Without it an instance can't see
  that an outage is Synauson's, and its 7 days keep counting.

Keep each host's clock synchronized (NTP): the outage notice counts only within 10
minutes of the instance's clock, and the licensing server's answers only within 5.

**Persist the state directory** (`--state-dir`, `/var/lib/synauson/state` in the
image). It holds the cached license file, the last share and the outage notice. A new
instance, or one restarted without its state directory, has no license file of its
own, so it must reach `license.synauson.com` and `usage.synauson.com` before it can
start, **including during an outage on Synauson's side**: the outage pause protects
only instances that are already running or can restart from their state directory.
Mount a volume there, especially where instances scale out or are replaced
automatically.

If an instance can't reach the service, nothing stops: it keeps admitting against
the last share it held, with its burst, and calls in progress always finish. You
have **7 days** from the last report that got through to restore the connection.
If the outage is on Synauson's side, Synauson posts a signed notice on
`status.synauson.com` and the 7 days stop counting until the service is back; they
then run again from when it recovered. The pause reaches only instances that can
read `status.synauson.com` and whose clock is within 10 minutes of the notice's time;
any other instance recovers in full at its next report that gets through. An
instance that never held a share (one started from a `--license-file` with no state
directory, say) starts no AI session and no conference, with or without AI, until it
reaches the service; the same holds for a pool an older receipt carries no share of
(a receipt kept by Synauson 1.7.1 has no conference share). While it has no
conference share, `synauson.v1.Syn` health is NOT_SERVING, so a load balancer sends
calls elsewhere until it gets one. With `--offline` it never reaches the service, so an instance not
air-gapped needs the receipt an earlier online run kept in its state directory:
without one it doesn't start.

| Situation | What happens |
|---|---|
| No key, or a malformed one | Synauson exits with an error naming the setting |
| No license file that still works: `usage.synauson.com` can't be reached (or `--offline`) and the cached or `--license-file` file is missing, for another key, or more than 7 days past the last report that got through | Synauson exits with "synauson needs a valid license to start" |
| `--offline` on a license that isn't air-gapped, and the state directory holds no usage receipt for it (`usage/receipt.json`, kept by an earlier run whose report got through) | Synauson exits with "offline mode needs the usage receipt an earlier online run keeps in the state directory". Run it once without `--offline` on that state directory, or arrange an air-gapped license |
| The licensing server refuses the key (unknown, suspended, banned) | Synauson exits with an error; contact Synauson |
| The license is illegal (see below) | Synauson exits naming every problem; contact Synauson |
| `usage.synauson.com` can't be reached | Synauson starts, or keeps running, on the cached license file and its last share at full function for 7 days after the last report that got through. After an hour of failed reports the state is `offline`, logged at WARN, and at ERROR daily from day 3; `GetCapabilities` shows when the 7 days end (`usage_service.grace_end_time`; jsyn `usageService.graceEnds`). After the 7 days, on every plan, it is `disconnected`, logged at ERROR: no new conference or participant starts, `synauson.v1.Syn` health is NOT_SERVING, and calls in progress continue until the next report that gets through restores everything |
| `usage.synauson.com` is down on Synauson's side | as above, but the 7 days are paused while Synauson's outage notice is up (`usage_service.pause_time`; jsyn `usageService.pausedSince`) |
| The service refuses the license (the licensing server refused it) | the 7 days run from the refusal at the latest, and an outage notice doesn't extend them; contact Synauson |
| The license expires within 14 days | `GetCapabilities` reports `expiring` and the days left; a warning is logged daily |
| The license has expired | it runs on the Free plan (`expired`, logged daily): every AI feature with 4 concurrent AI sessions and 10 concurrent conferences and no burst, so a new AI session past four or conference past ten is refused with `RESOURCE_EXHAUSTED`; calls in progress continue, and a restart starts on the Free plan too. Renewing it restores the full plan without a restart |
| A developer key has expired | nothing new (conferences, participants) starts (`LICENSE_EXPIRED`, health NOT_SERVING); calls in progress continue. Ask Synauson to renew it; the renewal applies at the instance's next daily license renewal, up to a day later, or at once on a restart |
| A renewal brings an illegal or unverifiable license | the last valid license stays in force (`invalid-kept-last-valid`), and an error is logged at every renewal until a good one arrives |
| The license is suspended while running | nothing new (conferences, participants) starts; calls in progress continue |

A license is **illegal** when its content doesn't fit this design: entitlements that
don't cover every feature; neither `purchasedSessions` (a whole number from 0) nor
`unlimitedSessions: true`, or both; both `purchasedConferences` and
`unlimitedConferences`, or either malformed; an `overdraft` outside 0.25 to 2 (EULA 3.2
promises at least 25%), or with both pools unlimited;
an `airGapped` that isn't `true` or `false`; a `developer` that isn't `true` or
`false`, or a developer key that buys anything, is unlimited, has an `overdraft` or is
air-gapped; a retired limit key (`maxSessions`, `limitConferences`, `limitAiConferences`,
`limitStreams`) or any other `limit*` / `max*` key Synauson doesn't know (such as
`maxConferences`). A license with neither conference key (one checked out before
Synauson 1.8.0, say) gets the Free plan's 10 conferences, and Synauson logs a warning
once. `synauson license lint` checks a license
with the same code (see [Subcommands](#subcommands)).

Mount a volume at `--state-dir` (default `/var/lib/synauson/state`). It keeps the
cached license file, the calibration cache ([STT capacity](#stt-capacity)) and, under
`usage/`, the last share and receipt, unsent peaks and the outage status, so a
restart during an outage keeps its share and its place in the 7 days. Without it, a
restart while `usage.synauson.com` can't be reached has no license and doesn't start.

**Air-gapped deployments.** A host that can never reach Synauson needs an
air-gapped license, arranged with Synauson on the order form: Synauson marks the
license air-gapped and issues a long-lived license file. Mount it and pass
`--license-file`. An air-gapped license sends no reports, has no 7-day window (its
file's own expiry is the limit) and counts its pools on each instance. It makes no
request to any Synauson service, model downloads included, with or without
`--offline`: fill the model store with `synauson models import <folder>` from the
model files Synauson provides; a missing model's error says so. `--offline`
with any other license sends no reports either, so it needs a state directory from an
earlier online run: it runs on the shares that run's last receipt leased, for 7 days
after the last report that got through, and without a kept receipt it doesn't start.
It starts conferences only against a kept conference share: a receipt kept by
Synauson 1.7.1 has none, so no conference starts offline until a run online keeps a
new one.

## Configuration

Synauson reads CLI flags or env vars. CLI takes precedence.

| Flag | Env var | Default | Description |
|---|---|---|---|
| `--listen` | `SYNAUSON_LISTEN` | `0.0.0.0:50051` | gRPC server bind address |
| `--auth` | `SYNAUSON_AUTH` | `none` | Auth mode: `none`, `token`, or `mtls` |
| `--token` | `SYNAUSON_TOKEN` | (none) | Static bearer token (required for `--auth=token`) |
| `--tls-cert` | `SYNAUSON_TLS_CERT` | (none) | Server TLS cert path (required for `--auth=mtls`; with `--tls-key`, serves TLS in `--auth=token`) |
| `--tls-key` | `SYNAUSON_TLS_KEY` | (none) | Server TLS key path (required for `--auth=mtls`; optional with `--auth=token`, as `--tls-cert`) |
| `--client-ca` | `SYNAUSON_CLIENT_CA` | (none) | Client CA bundle path (required for `--auth=mtls`) |
| `--log-level` | `SYNAUSON_LOG_LEVEL` | `info` | Tracing-subscriber EnvFilter expression |
| `--log-format` | `SYNAUSON_LOG_FORMAT` | `json` | Log format: `json` or `pretty` |
| `--model-store` | `SYNAUSON_MODEL_STORE` | `/var/lib/synauson/models` | Model store root (`<model-id>/<variant>/<file>`). A missing model disables only the features that need it. Keep it writable and give instances on one host the same store: the STT encoder's shared weights are built into it on first use (see [STT capacity](#stt-capacity)). |
| `--license-key` | `SYNAUSON_LICENSE_KEY` | (none, required) | License key from Synauson. See [Licensing](#licensing). |
| `--license-file` | `SYNAUSON_LICENSE_FILE` | (none) | A license file from Synauson: an air-gapped license's, or one to use when `usage.synauson.com` can't be reached, which like the cache works for 7 days after the last report that got through. |
| `--state-dir` | `SYNAUSON_STATE_DIR` | `/var/lib/synauson/state` | Persistent state: the cached license file, the calibration cache (`calibration.json`, model timings on this host; see [STT capacity](#stt-capacity)) and, under `usage/`, the last share and receipt, unsent peaks and the outage status. Mount a volume here. |
| `--recalibrate` | `SYNAUSON_RECALIBRATE` | `false` | Ignore the model timings cached in `--state-dir`, time the models again and replace them. |
| `--offline` | `SYNAUSON_OFFLINE` | `false` | Never contact Synauson (no usage reports, no renewal, no outage status checks, no model downloads): run on `--license-file`, else the cached file, and on the shares of the usage receipt an earlier online run kept in `--state-dir` (conferences start only against a kept conference share), until 7 days after the last report that got through (an air-gapped license needs no receipt and runs until its file expires); without a license file, or without that receipt, Synauson doesn't start. |
| `--model` | | (build defaults) | `ROLE=MODEL`, repeatable: use catalog model MODEL for ROLE (`vad`, `turn-detection`, `stt`, `tts`) instead of the build default, e.g. `--model stt=spartito-1`. Model ids are codenames with a generation (`sentito-1`, `fermata-1`, `spartito-1`, `lettura-2`). Only selected models are downloaded. |
| `--max-conferences` | `SYNAUSON_MAX_CONFERENCES` | (unlimited) | Process-wide cap on live conferences: the operator's own, beside the license's conference pool. Both apply and the smaller wins; this one is checked first (`CONFERENCE_LIMIT_REACHED`) |
| `--max-participants-per-conference` | `SYNAUSON_MAX_PARTICIPANTS_PER_CONF` | (unlimited) | Per-conference participant cap |
| `--turn-detection-workers` | `SYNAUSON_TURN_DETECTION_WORKERS` | cores / (4 x threads), at least 1 | Turn detection decisions that can run at once, one model session each. A decision that waits over 100 ms for a worker logs a warning (at most every 10 s): raise this. |
| `--turn-detection-threads` | `SYNAUSON_TURN_DETECTION_THREADS` | 2 on 8+ cores, else 1 | ONNX Runtime threads per turn detection decision |
| `--turn-detection-trailing-ms` | `SYNAUSON_TURN_DETECTION_TRAILING_MS` | `100` | Audio after VAD's speech end that each turn detection decision waits for and includes, 0 to 1000 ms. More catches more turn ends and delays every decision by as much; anything outside the range refuses to start. |
| `--stt-workers` | `SYNAUSON_STT_WORKERS` | calibrated | STT decoding workers, each with its own model session: about 75 MB each for the default STT model when the encoder's weights are shared through the model store, about 800 MB when they can't be. See [STT capacity](#stt-capacity). |
| `--stt-threads` | `SYNAUSON_STT_THREADS` | calibrated | ONNX Runtime threads per STT worker's decode |
| `--max-stt-streams` | `SYNAUSON_MAX_STT_STREAMS` | calibrated | Concurrent STT streams this instance admits (each STT participant also takes a license session); `0` turns STT off. A participant over it is refused with `RESOURCE_EXHAUSTED` (`STT_CAPACITY_REACHED`). |
| `--cpu-budget` | `SYNAUSON_CPU_BUDGET` | detected | Cores this instance may use, fractional allowed. Detected as the smallest of the cgroup's `cpu.max` quota (own cgroup and its ancestors), its cpuset, the affinity mask and the CPU count; never the CPU weight. STT and turn detection size their pools from it. An override below the detected budget wins; one above it is used and warned about. Startup logs one `resource decision` per value, then `resources ready` |
| `--memory-budget` | `SYNAUSON_MEMORY_BUDGET` | the cgroup's limit | Bytes of memory this instance may use; STT sizes its workers within it |
| `--stt-turn-flush` | `SYNAUSON_STT_TURN_FLUSH` | `false` | Close each turn's transcript as soon as turn detection ends the turn, on a forecast of its last words decoded ahead of real time on the STT workers (see `StreamTranscriptEvents` in [the API reference](api.md)). On a Ryzen 7 3700X it closed long turns about 130 ms sooner (p50 820 ms against 950 ms after turn detection's decision) for 26% more CPU, and its forecast reserve lowers the stream cap (see [STT capacity](#stt-capacity)). `false` waits for the transcription to reach the turn's end. |
| `--stt-eager` | `SYNAUSON_STT_EAGER` | `false` | Allow eager end of turn (`TurnConfig.eager`, see `StreamAgentEvents` in [the API reference](api.md)) on participants with STT: their pauses forecast the last words ahead of real time on the STT workers, so the capacity reserves decoding for one forecast per pause, as the turn flush does, and the stream cap drops the same way (see [STT capacity](#stt-capacity)). `false` refuses eager end of turn with STT unless `--stt-turn-flush` is on, whose reserve covers it; without STT it is always allowed. |
| `--tts-workers` | `SYNAUSON_TTS_WORKERS` | from the cores | TTS synthesis workers, each with its own TTS model session (about 330 MB). See [TTS capacity](#tts-capacity). |
| `--tts-threads` | `SYNAUSON_TTS_THREADS` | from the cores | ONNX Runtime threads per TTS worker's synthesis. |
| `--max-tts-streams` | `SYNAUSON_MAX_TTS_STREAMS` | 2 per worker | Utterances this instance synthesises at once (a speaker counts only while it speaks); `0` turns TTS off. An utterance that would pass it fails with `utterance_failed` (`TTS_CAPACITY`); the participant stays. |
| `--rtp-port-min` | `SYNAUSON_RTP_PORT_MIN` | `49152` | Lower bound of RTP port allocation pool (must be even). Each SIP participant binds its RTP and RTCP pair exclusively, so instances on one host can share a range: each skips a pair another holds. |
| `--rtp-port-max` | `SYNAUSON_RTP_PORT_MAX` | `65534` | Upper bound of RTP port allocation pool. |
| `--rtp-jitter-buffer-ms` | `SYNAUSON_RTP_JITTER_BUFFER_MS` | `200` | rtpbin jitter buffer window in milliseconds (applies to SIP participants). |
| `--webrtc-stun-server` | `SYNAUSON_WEBRTC_STUN_SERVER` | (none) | STUN server for WebRTC ICE gathering, `stun://host[:port]`. Without one, Synauson offers only its host candidates, which is enough when browsers reach it directly (same network, or a public address); set one (for example `stun://stun.l.google.com:19302`) when peers reach it through NAT. A participant's `stun_server` overrides it. |
| `--webrtc-jitter-buffer-ms` | `SYNAUSON_WEBRTC_JITTER_BUFFER_MS` | `200` | rtpbin jitter buffer (inside webrtcbin) latency in milliseconds, 0 to 2000. |
| `--webrtc-ice-port-min` | `SYNAUSON_WEBRTC_ICE_PORT_MIN` | (any port) | Lowest local port for WebRTC ICE candidates, and so for WebRTC media. Set with `--webrtc-ice-port-max`. |
| `--webrtc-ice-port-max` | `SYNAUSON_WEBRTC_ICE_PORT_MAX` | (any port) | Highest local port for WebRTC ICE candidates. |

### STT capacity

How many STT streams a machine transcribes in real time depends on its CPU far more
than its core count, so Synauson measures it. STT's decoding pool starts in the
background: at startup when the license includes STT and the STT model is in the
store, else once the model downloads, or when an STT participant is first asked for.
Starting times a short decode (1.3 s of audio after a warm-up chunk, at most two model
loads) and sets:

- threads per worker: 1 if one thread decodes at under the target load, else the
  fewest threads expected to get there (decode speed grows about as threads^0.6),
  timed again at that count. The target load is 0.85 of real time divided by
  1 + the forecast reserve (below). On a CPU with AMX (Sapphire Rapids and later
  Xeons) it is never 2: ONNX Runtime 1.24's AMX kernels decode the default STT model wrongly with
  exactly two threads (WER 8.66% against 6.19% at 1, 3 or 4), so 2 becomes 3, or 1 when
  STT's share has fewer than 3 cores; `--stt-threads 2` is changed the same way, with a
  WARN;
- workers: half the CPU budget's whole cores (below) divided by the threads, at least 1, and no more than
  memory holds. The first load and its warm-up decode measure what a worker costs in
  two parts: the weights mapped from the store's shared derived model (`shared_model_bytes`
  in the `stt ready` log and `GetCapabilities`, about 600 MB for the default STT model), counted once, and the
  private memory each worker adds (`model_bytes`: 45 to 75 MB per extra session with
  shared weights, though the first load also counts ONNX Runtime's own setup and
  whatever else the process allocates meanwhile, so the measure errs high; about
  800 MB to 1.2 GB when the weights load in-process, with no shared part). The workers may use the available memory
  (`MemAvailable`, within the process's cgroup limit: a container's, or a systemd unit's or slice's `MemoryMax=`; or `--memory-budget`) less a quarter of it or 1 GiB,
  whichever is more, less the shared part. When memory, not the cores,
  sets the count, the log says `stt workers limited by memory` and `GetCapabilities`
  reports `limited_by: memory`. In jsyn on Windows the memory is the free physical
  memory within the JVM's job object limit, if it has one;
- streams: workers x floor(target load / the measured real-time factor), at least
  one per worker.

The forecast reserve is the decoding the turn flush (`--stt-turn-flush`) or eager end
of turn (`--stt-eager`) adds per stream, as a share of the stream's own: pauses per
second x the most audio one forecast decodes. A pause runs at most one forecast (an
eager end of turn's, reused by the turn end that follows it, or the turn flush's), and a forecast decodes at most the
model's commit delay in silence, rounded up to chunks, plus two chunks, plus one
chunk of waiting audio decoded again: 8 chunks or 1.28 s for the default STT model. The turn
rate is 0.3 a second, measured on the speech fixture with every pause a turn (4 turns
in 14 s); calls at the default turn detection threshold end fewer. So the default STT model reserves
0.3 x 1.28 = 0.384, and the target load is 0.85 / 1.384 = 0.61. Forecasts in that
measurement averaged 2.4 chunks of silence, well under the cap, so the reserve errs
safe; an eager end of turn at every speech end forecasts more often than turns end at
the default threshold, which the every-pause measurement already counts. With both
off, or a model whose streams can't fork, the reserve is 0. `GetCapabilities` reports
it (`stt.forecast_reserve`) with `stt.turn_flush` and `stt.eager`.

Then the workers load their models, in parallel. The result is logged (`stt ready`)
and reported by `GetCapabilities` (`stt`). Until the pool is ready (seconds; on a slow
host up to about 20), adding a participant with STT fails with `MODEL_LOADING`
and `GetCapabilities` says `loading`. The measurement is of one stream alone, so on a
shared or busy host it can be optimistic: a Broadwell Xeon (32 vCPUs) calibrates to 3
workers of 5 threads and 3 streams with the turn flush off, which load tests found to
be its limit, and to 2 workers of 8 threads and 2 streams with it on. Any of the
three flags above overrides its part; setting all three skips the measurement, and
`--max-stt-streams 0` turns STT off without loading anything.

#### The calibration cache

Each timing is kept in `<state-dir>/calibration.json`, and a later start reuses it
instead of loading the model to time it again: STT's decode timings (and the memory
measured with them), and the cost of one VAD chunk and one turn detection decision, which
are timed in the background at startup once their models are in the store. An entry is
reused only when everything it depends on matches: the model's SHA-256, the execution
provider, the CPU model and its ISA flags, the ONNX Runtime version, the CPU budget and
the thread count. A different machine, a new CPU budget or an ONNX Runtime upgrade
measures again and replaces the entry. `stt ready` and `GetCapabilities` say where each
number came from: `source` is `auto` (measured on this start), `cached` or `override`
(STT set entirely by flags); the detector timings log `resource decision` lines
(`vad_chunk`, `turn_detection_decision`) and appear in `GetCapabilities.calibration`.

A timing taken while the CPU is contended would stay wrong for as long as it is
cached, so where the kernel reports CPU pressure (PSI, Linux) a timing taken while
some task waited for a CPU more than 10% of the last 10 s is used for that run but not
cached, with a WARN (`timing not cached: the cpu was under pressure`). A file that
can't be read or comes from another Synauson format is logged once and replaced. After
changing hardware in place, or to measure again on an idle host, start once with
`--recalibrate`. Two instances sharing a state directory both write it; the last write
wins.

### TTS capacity

TTS has no calibration yet; the pool follows a fixed rule from the logical CPUs. A
quarter of them are TTS's (at least 1); each worker gets 2 threads when there are two
TTS cores or more, else 1; workers are TTS cores over threads, 1 to 4; and the cap is
2 speaking utterances a worker. The TTS model synthesises about 0.38 s per second of audio on
two threads of an i7-9700, so a worker keeps two
utterances ahead of playback; on a slower CPU playback has gaps (reported as
`underrun_ms` on `utterance_done`) rather than refusals. The cap counts utterances
while they are synthesised and played, not speakers, because an agent talks for a
fraction of a call. The engine loads in the background like STT's, and
`GetCapabilities.tts` reports it. `--tts-workers`, `--tts-threads` and
`--max-tts-streams` override each part; `--max-tts-streams 0` loads nothing.

### Subcommands

Maintenance commands touch only the model store or the license and exit. They read the same flags and environment as the server: `--model-store` may go
anywhere on the command line, other flags go before the subcommand
(`synauson --license-key ... models pull`), and the environment variables work either way.

| Command | What it does |
|---|---|
| `synauson models pull` | Download every licensed model that isn't installed yet |
| `synauson models import <folder>` | Fill the store from a flat folder of model files (offline hosts); verifies SHA-256 |
| `synauson models verify` | Check every catalog model in the store; exits non-zero unless all are installed and valid |
| `synauson models notices [--model <id>]` | Print the license notice of each model in the store (what it derives from, under which license) |
| `synauson license lint [<input>...]` | Check licenses with the rules the server starts on, print every problem and warning, and exit non-zero if any is illegal. An input is a license file, a terms document (JSON: `name`, `entitlements`, `metadata`), or a license key, whose file is checked out from `license.synauson.com`; with none, the configured `--license-key`, which keeps the key off the command line |

These WebRTC flags are defaults. A participant can override each of them, and set TURN servers, a relay-only policy and Opus encoder options, through `AddWebRtcParticipantRequest.options` (see [the API reference](api.md#addwebrtcparticipant)).

### WebRTC ports

With `--webrtc-ice-port-min/max` set, every WebRTC participant binds its ICE candidates, and therefore its RTP, RTCP and DTLS traffic, to UDP ports in that range. Publish the range (UDP, plus TCP for ICE-TCP) from the container. Each concurrent participant takes one UDP port and one TCP port per local address. When the range runs out, a new participant gathers no candidates and never connects; Synauson logs `webrtc ice gathering produced no candidates` at WARN ten seconds after adding it. Keep the range clear of `--rtp-port-min/max`: both bind UDP ports on the same host, and Synauson warns at startup if they overlap. Unset, the ICE agent uses any free port. Firewall rules for both ranges: [Linux install, Firewall](install/linux.md#firewall).

## Auth modes

### `none` (default)

No middleware. Server accepts any client. Suitable for development or trusted private networks.

### `token`

Per-request bearer token check on `Syn` and `SynAdmin` (health stays open). Clients must send the metadata `authorization: Bearer <token>` matching `--token`; a refusal is `UNAUTHENTICATED` with reason `INVALID_TOKEN`. Give `--tls-cert` and `--tls-key` too and the server speaks TLS, so the token never crosses the network in clear; without them, keep token mode on networks you trust.

```bash
docker run --rm -p 50051:50051 -e SYNAUSON_LICENSE_KEY \
  -v /path/to/certs:/certs:ro \
  cr.synauson.com/synauson/synauson-server:latest \
  --auth=token --token=mysecret \
  --tls-cert=/certs/server.crt --tls-key=/certs/server.key
```

### `mtls`

Mutual TLS at the transport layer. The server presents `--tls-cert` + `--tls-key` and validates client certs against `--client-ca`.
The container runs as uid 10001, so the mounted key must be readable by it.

```bash
docker run --rm -p 50051:50051 -e SYNAUSON_LICENSE_KEY \
  -v /path/to/certs:/certs:ro \
  cr.synauson.com/synauson/synauson-server:latest \
  --auth=mtls \
  --tls-cert=/certs/server.crt \
  --tls-key=/certs/server.key \
  --client-ca=/certs/clients-ca.crt
```

## Signals

- `SIGTERM` / `SIGINT` (Ctrl-C or Ctrl-Break on Windows): terminates every conference (recordings finalized), waiting at most 5 s, then the gRPC server stops accepting requests and gives open connections 2 s to close before the process exits. Health reports NOT_SERVING and new conferences are refused from the signal on. `docker stop` allows 10 s before SIGKILL; Kubernetes, 30 s.
- `SynAdmin.ShutdownServer` RPC: terminates every conference (recordings finalized, participants removed), waiting at most the request's grace period, then stops the server. Use it when conferences may need longer than 5 s to end.

## Health

The standard `grpc.health.v1.Health` service is registered alongside the Synauson services, under the empty name, `synauson.v1.Syn` and `synauson.v1.SynAdmin`. The server doesn't offer gRPC reflection, so probe it with [`grpc_health_probe`](https://github.com/grpc-ecosystem/grpc-health-probe), which needs no proto files, or a Kubernetes `grpc:` probe:

```bash
grpc_health_probe -addr=localhost:50051
```

The names answer different questions. `""` is the process: SERVING while it runs, so use it for liveness, and a probe never restarts a server whose calls are still running. `synauson.v1.Syn` is whether it takes new work, so use it for readiness (`grpc_health_probe -addr=localhost:50051 -service=synauson.v1.Syn`, or `grpc.service` in a Kubernetes probe) to route new calls elsewhere. It is NOT_SERVING whenever the server won't start new work, for example while the license starts nothing new (`rejected`, `disconnected`: 7 days without a report reaching usage.synauson.com, or an expired developer key; it returns to SERVING when the license does), while the conference pool is limited and this instance holds no share of it (none leased yet, as after a restart while usage.synauson.com can't be reached and no share was kept; a full pool still serves), or once a shutdown starts; treat any NOT_SERVING the same way, as later releases may add causes. The Free plan (`expired`) still serves. `synauson.v1.SynAdmin` stays SERVING until a shutdown starts, which turns all three NOT_SERVING. gRPC health has no warning status: an `expiring` or `offline` license stays SERVING, and its warning is in the logs and `GetCapabilities`.

`grpcurl` needs the `.proto` files for every call; the commands below use a clone of [synauson/api](https://github.com/synauson/api) in `./api` (`git clone https://github.com/synauson/api`).

## Resource introspection

Capacity-planning metrics via the operator service, `synauson.v1.SynAdmin.GetResourceSnapshot`:

```bash
grpcurl -plaintext -import-path api -proto synauson/v1/admin.proto \
    -d '{}' localhost:50051 synauson.v1.SynAdmin/GetResourceSnapshot
```

Returns process memory (VmRSS and threads, on Linux only), the server's thread pools, cgroup memory usage, and each conference's participant count.

## Troubleshooting

- **Start with `GetCapabilities`.** It reports the license standing and plan, which capabilities are entitled, the AI session and conference pools, and each model's state, which explains most of the errors below:
  `grpcurl -plaintext -import-path api -proto synauson/v1/synauson.proto -d '{}' localhost:50051 synauson.v1.Syn/GetCapabilities`
- **The server doesn't start, or ONNX Runtime, GStreamer or a port range is the problem**: see the [Linux install troubleshooting](install/linux.md#troubleshooting).
- **Adding a participant with VAD or turn detection fails with `MODEL_UNAVAILABLE` "model '...' is not installed" or "failed verification"**: the model store at `--model-store` is missing that model, or its file doesn't match the SHA-256 this binary pins. Right after startup the model may still be downloading, which fails with `MODEL_LOADING` instead. The startup log lists every model's state ("model ready", "model not installed", "model failed verification"). Run `synauson models verify` against the store, then `synauson models pull` (or `models import <folder>` offline) to repair it.
- **`FAILED_PRECONDITION` on StartConference or when adding a participant, reason `LICENSE_REJECTED`, `LICENSE_DISCONNECTED` or `LICENSE_EXPIRED`**: the license was rejected (suspended, banned, unknown), no report has reached usage.synauson.com for 7 days (`disconnected`), or a developer key expired (ask Synauson to renew it); the message names the license state. Nothing already running is stopped. Any other expired license is the Free plan, which refuses AI past its 4 sessions with `SESSION_LIMIT_REACHED` and conferences past its 10 with `CONFERENCE_POOL_LIMIT_REACHED`.
- **`RESOURCE_EXHAUSTED`**: the error's reason says which limit: `CONFERENCE_LIMIT_REACHED` or `PARTICIPANT_LIMIT_REACHED` (`--max-conferences`, `--max-participants-per-conference`), `RTP_PORTS_EXHAUSTED` (the SIP RTP port pool), `SESSION_LIMIT_REACHED` (the license's concurrent AI sessions, beyond its burst), `CONFERENCE_POOL_LIMIT_REACHED` (the license's concurrent conferences, beyond its burst) or `STT_CAPACITY_REACHED`. An instance that holds no share of a pool yet is not a full pool: it fails with `UNAVAILABLE`, `LICENSE_SHARE_PENDING`, so retry later or use another instance.
- **The server exits with "is not a valid synauson license"**: the license content is illegal for this build (the message lists every problem). Nothing you configure fixes it; send the message to Synauson for a corrected license. `synauson license lint` prints the same list.
- **gRPC client gets `UNAUTHENTICATED`** (`INVALID_TOKEN`): token mismatch or missing `authorization` metadata. Verify the client sends `authorization: Bearer <token>` exactly matching the server's `--token`.
- **mTLS handshake fails**: Verify the server's `--client-ca` includes the issuer of the client cert; verify the client's chain includes the server cert's issuer.
