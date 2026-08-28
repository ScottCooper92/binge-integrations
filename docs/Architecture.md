# Architecture

The platform decisions below are locked at the direction level. Contract shapes are design until
the first stub companion app proves them with real traffic.

## Transport: companion APKs over Binder IPC

A companion app exports one bound Service per capability, discovered by intent action:

| Action | Capability |
| --- | --- |
| `com.binge.integration.REQUEST` | Media-request servers (request, track, manage) |
| `com.binge.integration.STREAM` | Resolve a title to playable sources |
| `com.binge.integration.TRACKING` | Sync watch state with an external tracker |
| `com.binge.integration.PLAYER` | External playback with a progress callback |

- Binge declares matching `<queries>` entries (Android 11+ package visibility).
- Manifest `<meta-data>` on the Service carries the display name, icon, and the contract majors it
  serves, so the host can render its integrations list — and spot a version mismatch — without
  waking the companion process.

## RPC layer: gRPC over Binder, protobuf payloads

Calls cross the app boundary as gRPC, over the official Android Binder transport
(`io.grpc:grpc-binder`), with protocol-buffer messages (`protobuf-javalite` at runtime). The
`.proto` files in `contracts/` are the normative contract; stubs, docs, and the conformance
harness generate from them.

Why gRPC instead of a hand-rolled thin-AIDL + JSON surface (the earlier design):

- The service definition is the documentation — typed rpcs instead of AIDL methods that marshal
  opaque JSON strings.
- An integration author implements a generated service base (suspend functions and `Flow`s via
  grpc-kotlin). No marshalling code, no bespoke callback-registration protocol.
- A gRPC service runs over any channel, so an implementation is unit-testable on the JVM with no
  device, and the conformance harness needs no emulator.
- Streaming rpcs replace hand-rolled callbacks; deadlines, cancellation, and a standard error
  model come with the framework.
- `grpc-binder`'s `SecurityPolicy` implements the mutual signing-cert verification the contract
  requires — configuration instead of hand-written security code.

Practical rules:

- Errors travel as gRPC status codes; response messages never carry error fields. Each contract
  documents its code mapping in the `.proto`.
- Respect the ~1 MB Binder transaction limit: results are paged; artwork travels as URLs, never
  bytes.
- The REQUEST stub spike validates the transport on real hardware (phone and a SHIELD-class TV
  device) and measures the APK cost after R8, before the contract is declared stable.

## Media identity

Every payload addresses media as **media type + TMDB id (+ season/episode)**. The companion app
owns translation into any other id space (its server's ids, IMDb, TVDB, …).

## Capabilities

The intent action is the discovery unit; the capability set is the feature-detection unit inside
it. Each contract keeps a small mandatory core (for REQUEST: handshake, submit, status). Every
other rpc is gated by a declared capability.

- **Static capabilities** — declared once in the handshake response: everything this connection can
  ever do, for this provider and this signed-in user. The host hides UI for undeclared
  capabilities and never calls a gated rpc without one.
- **Dynamic per-item actions** — each status response lists the subset that applies to that title
  right now (for example, approve appears only on a pending request the user may moderate).

The boundary rule: a behavioural variation over the same data model is a capability; a new data
model with its own lifecycle is a new contract. Capability enums are append-only, and peers ignore
values they do not recognise. Feature detection never rides on version numbers.

## Security: mutual verification

- **Host side** — per-companion consent (package name + signing-cert hash) before first use; every
  companion-supplied URL or Intent is validated before the host touches it; the host's TMDB session
  never crosses the boundary.
- **Companion side** — the companion verifies the caller's signing certificate before serving a
  request. Its exported Service fronts the user's provider session; without the check, any app on
  the device could drive it.
- Both checks are enforced with `grpc-binder` `SecurityPolicy` instances, wired by the SDK on each
  side.

## Play stance

- No bundled providers, no in-app plugin directory, no promotion of infringing companions.
- STREAM and PLAYER default to hand-off over in-app playback; render-surface decisions are made
  per-contract.

## Versioning

Capabilities answer "what can you do?". Versions answer only "can we parse each other?".

- The proto package version (`binge.integration.request.v1`) is the contract **major**. Within a
  package, evolution is strictly additive — new fields, new enum values, new rpcs — and
  `buf breaking` fails CI on anything else. Additive changes need no negotiation: protobuf field
  numbering plus unknown-field preservation keeps old and new peers compatible in both directions.
- A breaking change is a new package (`v2`) with a new service, which a companion serves
  side-by-side with `v1` from the same exported Service. The manifest `<meta-data>` lists the
  majors a companion serves, so the host picks the highest common major before binding — and shows
  a directional "update Binge" / "update the companion app" when there is none.
- A major bump is an escape hatch, not a tool. The append-only discipline is the compatibility
  story.
