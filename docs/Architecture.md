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
- Manifest `<meta-data>` on the Service carries display name, icon, and contract version, so the
  host can render its integrations list without waking the companion process.

## Payload: versioned JSON over a thin AIDL surface

- The AIDL surface is deliberately thin: `String`/`byte[]` parameters plus callback interfaces,
  `oneway` for long-running calls.
- No custom Parcelables cross the boundary — two independently shipped apps cannot share a
  Parcelable safely.
- Respect the ~1 MB Binder transaction limit: results are paged; artwork travels as URLs, never
  bytes.
- Transport objects live in the `contracts` module (pure Kotlin/JVM, kotlinx.serialization).

## Media identity

Every payload addresses media as **media type + TMDB id (+ season/episode)**. The companion app
owns translation into any other id space (its server's ids, IMDb, TVDB, …).

## Security: mutual verification

- **Host side** — per-companion consent (package name + signing-cert hash) before first use; every
  companion-supplied URL or Intent is validated before the host touches it; the host's TMDB session
  never crosses the boundary.
- **Companion side** — the companion must verify the caller's signing certificate before serving a
  request. Its exported Service fronts the user's provider session; without the check, any app on
  the device could drive it. The SDK ships this verification helper.

## Play stance

- No bundled providers, no in-app plugin directory, no promotion of infringing companions.
- STREAM and PLAYER default to hand-off over in-app playback; render-surface decisions are made
  per-contract.

## Versioning

Contracts version independently of each other and of the apps. The client and service handshake at
bind time; AIDL evolves append-only; JSON payloads carry a schema version.
