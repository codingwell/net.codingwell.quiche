# Quiche.NET

A C# wrapper for [Cloudflare's quiche](https://github.com/cloudflare/quiche)
(raw QUIC streams), shipped as a NuGet package with prebuilt native binaries
for the platforms it supports.

## The consumer promise

Using Quiche.NET should require as little as practical from the consumer:

- **One package reference.** `dotnet add package net.codingwell.quiche` — that's the
  whole install.
- **No native setup, on supported platforms.** For a supported
  platform, the right quiche binary rides inside the package and is
  resolved automatically. No manual downloads, no `DllImport` path
  wrangling, no native library placement. See Included runtimes
  for which platforms a given package version actually includes.
- **No build-time toolchain.** Consumers never need Rust, cargo, or a C
  compiler. Native binaries are built in CI and published with the
  package.
- **Managed API, sane defaults.** Connections, streams, and the packet
  pump are exposed as idiomatic .NET types. The UDP socket pump,
  timers, and connection-ID demultiplexing are handled inside the
  library — including the case other stacks refuse: **one UDP socket
  shared between a listener and outbound connections**, which is what
  makes NAT hole punching possible.

This wrapper exposes quiche's raw QUIC connection/stream API directly.
It does **not** wrap quiche's HTTP/3 layer — HTTP/3 is future work, not
part of this package.

## Status

Pre-release. The managed wrapper, native build matrix, and NuGet
packaging are implemented; see Limitations and the roadmap below.
Latest published version: `0.1.0-alpha.1` (unlisted on NuGet.org,
linux-x64 and linux-arm64 only). Release history:
[RELEASE_NOTES.md](RELEASE_NOTES.md).

## Included runtimes

Which native libraries a package actually contains, version by
version. This table is the record — update it whenever a release
changes the set, rather than describing the set in prose elsewhere.

| RID | Native library | Built in CI | In 0.1.0-alpha.1 | In publish workflow |
| --- | --- | --- | --- | --- |
| win-x64 | quiche.dll | Yes | No | Yes |
| win-arm64 | quiche.dll | Yes | No | Yes |
| linux-x64 | libquiche.so | Yes | Yes | Yes |
| linux-arm64 | libquiche.so | Yes | Yes | Yes |
| linux-musl-x64 | libquiche.so | Yes | No | Yes |
| osx-x64 | libquiche.dylib | Yes | No | Yes |
| osx-arm64 | libquiche.dylib | Yes | No | Yes |
| linux-musl-arm64 | — | No (deferred) | No | No |

"In publish workflow" means a package built by
[.github/workflows/publish.yml](.github/workflows/publish.yml) on a
version tag includes that RID. The alpha predates the workflow and
was hand-built, which is why its contents differ from the workflow
column. On any platform not listed as included for the version you
installed, the native loader will fail to find a library — that is
expected, not a broken install.

## Roadmap

1. **Native build matrix (CI):** ✅ quiche 0.30.0 built as a shared
   library for `win-x64`, `win-arm64`, `linux-x64`, `linux-arm64`,
   `linux-musl-x64`, `osx-x64`, `osx-arm64` -- all 7 cells green in
   CI (2026-10-28), each smoke-checked for the core C API symbols
   and staged with quiche + BoringSSL license texts. `linux-musl-arm64`
   is deferred: rustc drops `cdylib` for that target (see the design
   doc). Design: [docs/native-build-matrix.md](docs/native-build-matrix.md);
   build scripts: `native/build.sh` (Unix) and `native/build.ps1`
   (Windows); CI: `.github/workflows/native.yml`. Known
   per-cell notes: `win-arm64` builds with `OPENSSL_NO_ASM` (C crypto,
   same API) because BoringSSL's Windows ARM64 assembly path is broken
   in this toolchain combination -- see the design doc.
2. **P/Invoke layer:** ✅ 146 `DllImport` signatures over quiche
   0.30.0's C API, generated from quiche's own Rust FFI source with
   csbindgen (regenerable: `tools/bindgen`, see its README), plus
   hand-written companions for types csbindgen cannot see (opaque
   `Config`/`Connection`/iterators, libc `sockaddr` structs, error
   codes) and a native library resolver (NuGet `runtimes/<rid>/native/`
   layout, `$QUICHE_NATIVE_PATH` override). A loader smoke test
   asserts the pinned version string. CI: `.github/workflows/tests.yml`
   builds the linux-x64 library and runs the suite.
3. **Managed API:** ✅ `QuicSocket` (one UDP socket that is
   simultaneously listener and dialer, pump thread, connection-ID
   demultiplexing), `QuicConnection`, `QuicStream`, and TLS with safe
   defaults (verification on, system roots unless a CA file is given,
   ALPN `quiche-dotnet`, 30 s idle timeout, transport parameters set
   explicitly since quiche's own defaults allow no streams). Proven
   by a shared-socket echo test: two sockets each dial and accept
   with echo both ways, green on Forge (2026-10-28).
4. **Tests:** unit tests for the managed layer and interop tests
   against a real quiche peer (the port-sharing proof landed with
   step 3's echo test).
5. **NuGet publishing:** `0.1.0-alpha.1` published unlisted
   2026-10-30 (linux-x64 and linux-arm64 only). Versioned releases
   and release notes: see [RELEASE_NOTES.md](RELEASE_NOTES.md).
   Trusted publishing workflow:
   [.github/workflows/publish.yml](.github/workflows/publish.yml)
   builds all 7 RIDs and publishes on version tags via NuGet/login
   OIDC — the nuget.org policy and `NUGET_USER` secret for it are
   still to configure (see the workflow header). Symbol packages:
   not yet done.

## Limitations

These are current, documented limits — not bugs to be surprised by:

- **IPv4 only.** The managed layer implements only the IPv4
  `sockaddr_in` layout; `QuicSocket` creates an `InterNetwork`
  socket and throws `NotSupportedException` on IPv6 endpoints
  rather than mis-parsing them. quiche's C API itself is
  address-family agnostic, so IPv6 is a contained follow-up, not a
  redesign.
- **RSA-only server certificates.** quiche/BoringSSL rejects
  .NET-generated ECDSA server certificates in every variant tested;
  RSA-2048 server certificates work. Use an RSA server key.
- **No server-side client-certificate verification option.**
  The client has `VerifyPeer`; `QuicServerOptions` has none.
- **No HTTP/3.** Only the raw QUIC connection/stream API is
  wrapped (see above).
- **Packaged runtimes lag the CI matrix for 0.1.0-alpha.1.**
  See Included runtimes above: the other CI-built RIDs are built,
  not yet packaged, and arrive with the first workflow-built
  release.
- **Tests run on linux-x64 in CI only.** Other RIDs get a native
  build and symbol smoke check, not the managed test suite.
- **Pre-release API.** The public surface (`QuicSocket`,
  `QuicConnection`, `QuicStream`, options) may still break before
  1.0.

## Why quiche, and why not msquic

Microsoft's msquic — the engine behind `System.Net.Quic` — refuses to
share one UDP port between a listener and an outbound connection in the
same process, in both its managed API (OS bind failure) and its raw C
API (the transport aborts the connection internally before any packet
leaves the machine). That limitation is in msquic's binding model, not
in QUIC itself: QUIC demultiplexes connections by connection ID, and a
single socket is the protocol's natural shape.

Quiche inverts the model: the *application* owns the UDP socket and
feeds packets in and out. Port sharing then falls out by construction —
the library binds one socket and routes inbound packets to connections
by connection ID. This project exists to bring that model to .NET.

## Licensing

- This project's own code (the C# wrapper, build scripts, and docs) is
  MIT-licensed — see [LICENSE](LICENSE).
- quiche itself is BSD-2-Clause; its TLS stack is BoringSSL. Their
  license texts ship inside the native runtime packages.
