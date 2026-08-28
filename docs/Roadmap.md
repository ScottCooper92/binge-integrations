# Roadmap

Build order. Each stage proves the previous one before the surface grows.

1. **Platform record** — this repository, the architecture pages, CI. ✅
2. **Transport objects** — the versioned JSON payload types in `contracts` (pure Kotlin/JVM).
3. **REQUEST contract v1** — the AIDL surface, bind-time handshake, and discovery/consent flow in
   the host, proven against a stub companion app.
4. **Reference companion: Binge Seerr** — extract Binge's in-tree Seerr integration into a real
   companion app that serves REQUEST v1 with production traffic.
5. **SDK + conformance harness** — publish `binge-integration-sdk` (service scaffolding, caller
   verification) and a conformance suite companion authors can run in CI.
6. **STREAM contract** — source resolution, hand-off-first.
7. **PLAYER contract** — external-player hand-off with a progress callback so watch tracking
   survives.

Artifacts publish to Maven Central under `io.github.scottcooper92` once the REQUEST contract is
stable.
