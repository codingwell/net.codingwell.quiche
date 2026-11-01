# Release notes — Quiche.NET (`net.codingwell.quiche`)

## Unreleased

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

Still pending before the first public (listed) release — not done by
this file:

- Merge the draft PR stack (#1–#4) to main and re-run CI on the
  merged result; the rename, readme, and publish-workflow commits
  are currently local only, and the workflow cannot run until it
  is pushed to GitHub.
- Configure the nuget.org Trusted Publishing policy (Package
  Owner = the org that owns net.codingwell.quiche; Repository
  Owner = codingwell, repo net.codingwell.quiche, workflow file
  `publish.yml`, no environment) and the `NUGET_USER` repo secret
  (= the org name) — see the workflow header for the exact values.
- Decide the packaged runtime scope for the first public release
  (all 7 CI cells, or Linux only) and make the README match it.
  Note: the publish workflow as written ships all 7 RIDs, so
  choosing Linux-only would mean narrowing its matrix, not just
  the README.
- Verify a QUIC handshake through the published package itself
  (the ProjectReference path is verified; a package-consumer
  handshake is not yet), on a host that allows UDP.

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
