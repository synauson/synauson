# Security

## Report a vulnerability

Report a security problem privately, never in a public issue:

- through GitHub's private vulnerability reporting on this repository: the
  **Security** tab, then **Report a vulnerability**; or
- by email, to the contact in Synauson's
  [security.txt](https://synauson.com/.well-known/security.txt).

Say which version you run and how to reproduce the problem. On every plan, including
the Free plan, Synauson publishes security fixes for the current minor release and the
one before it ([security](https://synauson.com/security/)). A fix is listed under
Security in [CHANGELOG.md](CHANGELOG.md), with a link to its advisory on this
repository when there is one.

## Verify a release

Synauson signs the server image and the `jsyn-natives` jars with its own key. The
public key is at https://synauson.com/cosign.pub; fetch it from synauson.com over
HTTPS. The signatures are kept out of Sigstore's public transparency log, so each
[cosign](https://docs.sigstore.dev/cosign/system_config/installation/) command needs
`--insecure-ignore-tlog=true`: the key is what proves them.

The server image, and the SPDX SBOM attested on it:

```sh
cosign verify --key https://synauson.com/cosign.pub --insecure-ignore-tlog=true \
  cr.synauson.com/synauson/synauson-server:<version>
cosign verify-attestation --key https://synauson.com/cosign.pub --insecure-ignore-tlog=true \
  --type spdxjson cr.synauson.com/synauson/synauson-server:<version>
```

A natives jar, with the `.sigstore.json` bundle beside it in
https://maven.synauson.com/releases (Gradle and Maven ignore the bundle):

```sh
cosign verify-blob --key https://synauson.com/cosign.pub --insecure-ignore-tlog=true \
  --bundle jsyn-natives-linux-<version>.jar.sigstore.json jsyn-natives-linux-<version>.jar
```

Each GitHub Release here is immutable: its tag can't move, and GitHub signs an
attestation of the tag and its commit. With the [GitHub CLI](https://cli.github.com/):

```sh
gh release verify v<version> -R synauson/synauson
```

jsyn's own jars carry build provenance instead; its README says how to check them
([synauson/jsyn](https://github.com/synauson/jsyn#versions)).
