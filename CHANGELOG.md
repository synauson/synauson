# Changelog

All notable changes to the Synauson engine are documented here: the server image, the
`jsyn-natives` jars, the gRPC API they serve and the models they use. The engine
follows [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html), and this file
follows [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/), except that it
has no Unreleased section: the engine's source is private, so each version's entry is
written when it is released. The artifacts and how to verify them are in each
version's [GitHub Release](https://github.com/synauson/synauson/releases).

## [1.8.0] - 2026-10-14

_Before upgrading, read the [1.8.0 upgrade guide](https://github.com/synauson/synauson/blob/v1.8.0/docs/upgrading/1.8.0.md): gRPC clients built from the 1.7 protos must move to API 1.0.0._

### Added

- **API:** Add the public, stable gRPC API `synauson.v1` (services `Syn` and `SynAdmin`), published as synauson/api 1.0.0 and `synauson-api` 1.0.0 on PyPI
- **License:** Raise the Free plan to 4 AI sessions and 10 conferences at once, and add a conference pool: paid licenses add the sessions and conferences they buy to that base
- **License:** Add Developer licenses: free, for development and testing only, with 4 AI sessions and 10 conferences, for one year
- **API:** Add the conference pool, this instance's share of it and the license's plan (`LicensePlan`) to `GetCapabilities`
- **API:** Add `server_version` and `api_version` to `GetCapabilities`, and document every proto element with the errors each request can return

### Changed

- **API (breaking):** Split the `Synauson` service into `Syn` and `SynAdmin`, and rename, retype and renumber fields ([upgrade guide](https://github.com/synauson/synauson/blob/v1.8.0/docs/upgrading/1.8.0.md#grpc-clients))
- **API (breaking):** Restrict ids to 1 to 128 characters from `A-Z`, `a-z`, `0-9`, `.`, `_` and `-`; other ids fail with `INVALID_ID`
- **API (breaking):** Limit `SetAudioRoutes` to 4096 routes and `AddPriorityAudioFiles` to 1 to 64 files, and check VAD ranges (`INVALID_ARGUMENT`)
- **API (breaking):** Default an unset VAD or turn-detection threshold to 0.5, not 0; set `confidence_threshold` to 0 to end turns at every pause
- **API (breaking):** Change status codes: a license that starts nothing new is `FAILED_PRECONDITION`, a loading model or an instance without a pool share `UNAVAILABLE`, and a speaker with text-to-speech off `FAILED_PRECONDITION` ([upgrade guide](https://github.com/synauson/synauson/blob/v1.8.0/docs/upgrading/1.8.0.md#errors-and-ids))
- **Server (breaking):** Rename the readiness health service to `synauson.v1.Syn`; liveness stays on `""`
- **Server (breaking):** Refuse to start with `--offline` when the state directory holds no usage receipt and the license isn't air-gapped
- **Server (breaking):** Serve TLS in `--auth=token` mode when `--tls-cert` and `--tls-key` are set (1.7 ignored them), and refuse to start with only one
- **Natives (breaking):** Start every exception message with its reason code, such as `CONFERENCE_NOT_FOUND: ...`
- **Natives (breaking):** Throw `InvalidArgumentException` for ids, route and file counts and VAD values outside the new API limits, and `FailedPreconditionException` for a speaker with text-to-speech off or no license pool share yet
- **Natives (breaking):** Rename diagnostic fields: `ModelInfo.variant`, `WebRtcStats.maxJitterNs` and `connectionState`, `RuntimeStats.workerThreads` and `blockingThreads`
- **Natives (breaking):** Report `license.plan` as `free`, `paid`, `developer` or `unlicensed` instead of `all` or `none`
- **Server:** Attach a `google.rpc.ErrorInfo` with a stable reason to every error the server returns
- **Server:** Return `LICENSE_SHARE_PENDING` (`UNAVAILABLE`, retryable) while an instance has no share of a license pool yet, and report readiness `NOT_SERVING` while it has no conference share
- **Server:** Return `LICENSE_EXPIRED` once a Developer license expires; running calls continue
- **Server:** Close HTTP/2 connections whose client stops answering keepalive pings, so a vanished client's streams end
- **Server:** Report `NOT_SERVING` on every health service and refuse new conferences as soon as a shutdown starts
- **License:** Count conferences in usage reports beside AI sessions; reports still carry no call content
- **License:** Keep the public IP address usage reports come from (IPv6 as its /64) and its country for 90 days after last seen, only to detect misuse of Free and Developer licenses
- **License:** Update the License Agreement to 1.13 ([summary of changes](https://github.com/synauson/synauson/blob/v1.8.0/docs/upgrading/1.8.0.md#license-agreement-113), [full text](https://synauson.com/legal/eula/1.13/))

### Removed

- **API (breaking):** Remove fields that were always 0, ignored or retired ([upgrade guide](https://github.com/synauson/synauson/blob/v1.8.0/docs/upgrading/1.8.0.md#removed-fields))
- **Natives (breaking):** Remove `SipStats.rttMs`, which was always 0

### Fixed

- **Server:** Fix a hang when removing a file participant, or ending its conference, after its file ended under a priority file
- **License:** Fix license files that could last less than the 30 days Section 9.5 of the License Agreement promises

## 1.7.1 - 2026-10-11

### Added

- **Server:** Sign the server image and the natives jars with Synauson's key, attach a signed SPDX SBOM to the image, and publish the source of its Debian packages as `synauson-server:1.7.1-oss-source`

### Changed

- **License (breaking):** Require a license key to start, and share each license's pool of concurrent AI sessions across all its instances, which report to `usage.synauson.com` every 30 seconds
- **Server (breaking):** Use no STUN server unless `--webrtc-stun-server` or a participant's `stun_server` sets one
- **License:** Give every license every AI feature; the Free plan has 2 concurrent AI sessions, for production use, with no expiry
- **Models:** Replace the text-to-speech model `lettura-1` with `lettura-2`, with the same voices
- **Server:** Run the image as user 10001 instead of root, with only the GStreamer plugins Synauson uses
- **License:** Update the License Agreement to 1.11 ([full text](https://synauson.com/legal/eula/1.11/))

### Fixed

- **Server:** Fix speech-to-text on phone calls that join a running conference, turns on looping and priority files, turn text on Windows, audio timing when participants or files join a running conference, two crashes and several error statuses

### Security

- **Server:** Update h2, rustls and rustls-webpki, which had published security advisories

[1.8.0]: https://github.com/synauson/synauson/releases/tag/v1.8.0
