# Roadmap

Build order. Each stage proves the previous one before the surface grows.

1. **Platform record** — this repository, the architecture pages, CI. ✅
2. **REQUEST contract v1 (draft)** — the `.proto` messages and `RequestService` definition in
   `contracts`, with `buf lint` + `buf breaking` gating CI. ✅ (draft — the stub spike below can
   still reshape it)
3. **Host client + stub companion** — Binge's discovery/consent flow and grpc-binder client, proven
   against a stub companion app on real hardware (phone and a SHIELD-class TV device). This spike
   validates the transport choice and measures APK cost after R8.
4. **Reference companion: Binge Seerr** — extract Binge's in-tree Seerr integration into a real
   companion app that serves REQUEST v1 with production traffic.
5. **SDK + conformance harness** — publish `binge-integration-sdk` (binder server bootstrap,
   `SecurityPolicy` wiring, handshake scaffold) and a conformance suite companion authors run in
   their own CI, no emulator required.
6. **STREAM contract** — source resolution, hand-off-first.
7. **PLAYER contract** — external-player hand-off with a progress callback so watch tracking
   survives.

Artifacts publish to Maven Central under `io.github.scottcooper92` once the REQUEST contract is
stable.
