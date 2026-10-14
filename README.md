# Synauson

Synauson is a self-hosted voice AI engine for teams building voice agents. It combines
a real-time media engine with local speech AI: voice activity detection, turn
detection, streaming speech-to-text and text-to-speech. It runs as a gRPC server or
inside a Java application, on your machines, under a license instead of per-minute
billing.

This repository holds the engine's changelog, its release notes and the documentation
for running it. The engine's source isn't public: each release is published here as
one commit, tagged `vX.Y.Z`, with a GitHub Release, so this repository takes no pull
requests.

## Get it

| You want to | Use | Start with |
|---|---|---|
| Run the gRPC server | The container image `cr.synauson.com/synauson/synauson-server`, on Linux x86_64 | [Install guide](docs/install/README.md) |
| Call the server from your code | The protos and the `synauson-api` Python package | [synauson/api](https://github.com/synauson/api) |
| Embed the engine in a Java application | jsyn, the Java SDK, from https://maven.synauson.com/releases | [synauson/jsyn](https://github.com/synauson/jsyn) |
| See complete applications | Quickstarts, a voice agent and a WebRTC testbed | [synauson/examples](https://github.com/synauson/examples) |

Every instance needs a license key. The Free plan's key is free, for production use
too: [get a key](https://synauson.com/pricing/#get-a-key).

## Documentation

- [Install guide](docs/install/README.md): system requirements, network access,
  and the pages for [Linux](docs/install/linux.md), [Windows](docs/install/windows.md)
  and [NVIDIA GPUs](docs/install/nvidia.md)
- [Deployment guide](docs/deployment.md): configuration, licensing, sizing the speech
  pools, health checks and troubleshooting
- [gRPC API guide](docs/api.md): every call, its events and its errors
- [Upgrade guides](docs/upgrading/): what to change when you move to a new version
- [CHANGELOG.md](CHANGELOG.md): what changed in each version; each version's
  [GitHub Release](https://github.com/synauson/synauson/releases) also lists its
  artifacts and how to verify them

## Report a problem

Report a bug as an [issue](https://github.com/synauson/synauson/issues/new/choose): say
which version and platform you run, and attach the log of a run with
`SYNAUSON_LOG_LEVEL=debug`. Report a security problem privately instead, as
[SECURITY.md](SECURITY.md) describes, which also says how to verify a release.

## License

Synauson is licensed under the Synauson End User License Agreement, in
[LICENSE](LICENSE) and at https://synauson.com/legal/eula/.
