# Release notes — Quiche.NET (`net.codingwell.quiche`)

## Unreleased

- Fixed `QuicStream.SendAsync` treating quiche's Done (-1) result
  as a fatal error: a send larger than the current congestion
  window (e.g. >= the ~12 KB initial window) now waits for the
  pump to report the stream writable again and retries the same
  chunk, instead of throwing "Stream send failed (quiche error
  -1)". The same handling applies to the FIN send. Other
  negative results remain fatal. Covered by the new
  `LargeSendBeyondCongestionWindowEchoes` test (64 KiB echo).
  This is the fix intended for 0.1.0-alpha2; 0.1.0-alpha1 is
  locked and cannot be re-pushed with it.
- Trusted publishing workflow: `.github/workflows/publish.yml`
  builds quiche for all 7 RIDs (same matrix as `native.yml`),
  stages the natives into the package via a new csproj packaging
  item (`native/pack/runtimes/<rid>/native/`, gitignored), and
  publishes on version tags via `NuGet/login@v1` OIDC — no
  long-lived API key. The tag (minus `v`) is the version of
  record. Verified locally: staged natives land at
  `runtimes/<rid>/native/` in the nupkg, a pack with no staging
  dir is managed-only, and `-p:Version` overrides the nuspec
  version as the workflow relies on.
- Fixed the package description and README: the package is a raw QUIC
  wrapper, not a QUIC + HTTP/3 wrapper — it does not wrap quiche's
  HTTP/3 layer. The "every major runtime identifier" claim is removed;
  packaged runtimes are now stated per version (see below).
- Added the Limitations section to the README (IPv4 only, RSA-only
  server certificates, no server-side client-certificate verification
  option, no HTTP/3, packaged runtimes lagging the CI matrix,
  pre-release API).

Still pending — not done by this file:

- Decide the first non-alpha release number (0.1.0 vs 1.0) and
  whether the libp2p transport switches from ProjectReference
  to this package (a separate decision from this release).
- The README's "Latest published version" line still says
  0.1.0-alpha.1 (unlisted); it was not updated when alpha1
  shipped listed, and is not updated by this change — it
  should be refreshed at release time.

Completed since this file was last written (no longer pending):

- The squashed import merged as PR #1, the Trusted Publishing
  policy and `NUGET_USER` secret were configured for this repo,
  and the publish workflow ran successfully (see 0.1.0-alpha1
  below).
- A QUIC handshake through the published package itself was
  verified from NuGet.org on a host that allows UDP, including
  a shared-socket echo through the package.

## 0.1.0-alpha1 — 2026-11-01 (listed on NuGet.org)

First workflow-built release, published by trusted publishing
from tag `v0.1.0-alpha1` (the tag minus `v` is the version).
Install with the exact version, since it is a prerelease:

```
dotnet add package net.codingwell.quiche --version 0.1.0-alpha1
```

### Included

- Everything from 0.1.0-alpha.1, plus native quiche 0.30.0
  binaries for all 7 CI RIDs (win-x64, win-arm64, linux-x64,
  linux-arm64, linux-musl-x64, osx-x64, osx-arm64) with
  quiche + BoringSSL license texts, and the repo README as
  the package readme.
- Verified from NuGet.org (cache cleared first): handshake
  with verification on, peer certificate byte-identical,
  4,096-byte echo both ways, and a shared-socket echo both
  directions through the package.

### Known bug (fixed in Unreleased, above)

- `QuicStream.SendAsync` fails for sends at/above the current
  congestion window with "Stream send failed (quiche error
  -1)". Single-send probes through this version: 18, 4,096,
  and 12,000 bytes pass; 16,384 and 32,768 bytes fail.

## 0.1.0-alpha.1 — 2026-10-30 (unlisted on NuGet.org)

First published package. Install with the exact version, since it is
unlisted and will not appear in search or browse:

```
dotnet add package net.codingwell.quiche --version 0.1.0-alpha.1
```

### Included

- Managed API: `QuicSocket` (one UDP socket that is simultaneously
  listener and dialer, pump thread, connection-ID demultiplexing),
  `QuicConnection`, `QuicStream`, with TLS safe defaults
  (verification on, system roots unless a CA file is given, ALPN
  `quiche-dotnet`, 30 s idle timeout, explicit transport parameters).
- `QuicConnection.PeerCertificate` (peer leaf certificate as DER,
  null when none or closed) and `QuicClientOptions.CertificateFile` /
  `PrivateKeyFile` (both-or-neither).
- Managed assemblies for `net8.0` and `netstandard2.0`.
- Native quiche 0.30.0 binaries plus quiche + BoringSSL license
  texts for `linux-x64` and `linux-arm64` only.
- Repo README as the package readme.

### Limitations (same as the README)

- IPv4 only; IPv6 endpoints throw `NotSupportedException`.
- RSA-only server certificates; .NET-generated ECDSA server
  certificates are rejected by quiche/BoringSSL.
- No server-side client-certificate verification option.
- No HTTP/3 layer.
- Windows, macOS, and linux-musl natives are built in CI but not
  included in this package.
- A QUIC handshake through this package (as opposed to the
  ProjectReference path) has not been re-verified.
- Pre-release API: may break before 1.0.

### Version note

This version is locked on NuGet.org. Any further change to the
package contents needs a new version — it cannot be re-pushed as
`0.1.0-alpha.1`.
