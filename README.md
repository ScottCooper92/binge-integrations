# binge-integrations

The companion-app integration platform for [Binge](https://github.com/ScottCooper92), an Android
client for movie and TV tracking built on TMDB.

Binge ships as a simple, policy-compliant TMDB client with zero bundled providers. Companion apps —
separate APKs, installed from anywhere — add capability by exporting bound Android Services that
implement the contracts published here. Binge discovers them on the device, the user consents per
app, and Binge renders their data through its own UI.

This is the Kodi position (clean host, ecosystem outside it), not the Stremio position (addon
catalog inside the app). No dynamic code loading: each companion app runs its own code in its own
process, and only data crosses the IPC boundary.

> **Status: pre-alpha.** The contracts are under active design. Nothing here is stable yet, and no
> artifacts are published. See [docs/Architecture.md](docs/Architecture.md) for the platform
> decisions and [docs/Roadmap.md](docs/Roadmap.md) for the build order.

## How it works

- **Discovery** — a companion app exports one bound Service per capability, matched by intent
  action (`com.binge.integration.REQUEST`, `.STREAM`, `.TRACKING`, `.PLAYER`). Manifest
  `<meta-data>` carries the display name, icon, and contract version, so Binge can list installed
  companions without waking their processes.
- **Transport** — gRPC over the official Android Binder transport (`io.grpc:grpc-binder`), with
  protocol-buffer messages. The `.proto` files in `contracts/` are the normative contract; a
  companion implements a generated service base (suspend functions and `Flow`s via grpc-kotlin)
  and is unit-testable on the JVM without a device. Results are paged and artwork travels as URLs,
  never bytes, to respect the ~1 MB Binder transaction limit.
- **Media identity** — every payload addresses media as media type + TMDB id (+ season/episode).
  The companion app owns translation into any other id space.
- **Versioning** — the proto package version (`binge.integration.request.v1`) is the contract
  major; within a package, evolution is strictly additive and `buf breaking` gates it in CI. A
  handshake rpc opens every connection and declares the companion's capabilities.

## Security model

Verification is mutual and part of the contract:

- **Host side** — Binge asks the user to consent per companion app (package name + signing-cert
  hash), validates every companion-supplied URL and Intent before use, and never sends its TMDB
  session across the boundary.
- **Companion side** — a companion app must verify the caller's signing certificate before serving
  a request (the SDK ships this helper). Its exported Service guards the user's provider session
  from every other app on the device.

## Capabilities

| Action | Purpose | Status |
| --- | --- | --- |
| `com.binge.integration.REQUEST` | Request movies/shows on a media-request server, track and manage request status | In design |
| `com.binge.integration.STREAM` | Resolve a title to playable sources (hand-off first) | Planned |
| `com.binge.integration.TRACKING` | Sync watch state with an external tracker | Planned |
| `com.binge.integration.PLAYER` | Hand off playback to an external player with a progress callback | Planned |

## Planned artifacts

| Coordinates | Contents |
| --- | --- |
| `io.github.scottcooper92:binge-integration-contracts` | Protobuf messages + gRPC service stubs (protobuf-javalite, grpc-kotlin; Kotlin/JVM) |
| `io.github.scottcooper92:binge-integration-sdk` | Android library for companion apps: binder server bootstrap, `SecurityPolicy` caller verification, handshake scaffold |
| `io.github.scottcooper92:binge-integration-conformance` | Test harness that validates a companion app against the contract |

## Repository layout

```
contracts/   The .proto contracts + generated Kotlin/JVM stubs (messages, gRPC services)
```

The SDK, conformance harness, and the reference companion app will join as they are built.

## Building

```sh
./gradlew build
```

Requires JDK 17+.

## License

[Apache-2.0](LICENSE)
