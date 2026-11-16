# Changelog

## 0.3.0

### Minor Changes

- Test the telemetry pipeline end to end against the backend contract: routing, tokens, collector answers, retries, holds and the write-ahead cache.
- Add the telemetry pipeline: per-room scopes, typed spans, and an exporter that caches and uploads OTLP batches under a device-aware cadence and hold policy.
- Add `global`, a process-wide telemetry pipeline with platform instruments and a synchronous opt-out that retries incomplete erasures.
- Clarify exclusive ownership of the configured telemetry cache directory and the conservative opt-out result when a final per-cache clear fails.
- Recover cached batches with streaming splits above an 8 MiB threshold for decoded input. Retain unsupported batches and remove stale metadata after parent eviction.
- Rename telemetry stats report input arguments to `track_id` alongside the breaking `RtcStatsSample` field migration while preserving track values and existing report admission semantics.

### Patch Changes

- Clearing the telemetry disk cache now syncs the directory once for its deletion sweep instead of once per batch, and still syncs it when a deletion fails.
- Clearing the telemetry disk cache, as the opt-out does, now also deletes stray files in the storage directory and reports an entry it cannot delete, so the opt-out no longer reports success while a stray file stays.
- A Room that connects while the exporter retires unused credentials keeps its token, so its telemetry no longer waits forever without uploading.
- A telemetry pipeline that `install` refuses after the opt-out is now cancelled, so its exporter stops by itself (the caller still joins the task), and it stays known to later `disable` calls, which retry a failed delete of its caller-provided cache; installing the installed pipeline again no longer hands it back as the outgoing one.
- Clarify that a failed batch split can report an already evicted parent without resurrecting records or counting the loss twice.
- Keep telemetry correlation consistent: a room identity change closes the open RTC windows and applies all four fields together, scoped records and spans no longer link to a span of another session, and `max_cache_bytes` gets its documented one-byte floor.
- A telemetry shutdown now gives the records and spans that a retained scope captures during its final drain further bounded encoding turns, which start only before the original export deadline; a started turn may finish after that deadline, and what is still queued when no further turn may start is not cached.

## 0.2.0 (2026-11-13)

### Breaking Changes

- Add an opt-in decoded HTTP response limit, used by telemetry export, that reports an oversized response as the new `TransportError::ResponseTooLarge` variant. Code that matches `TransportError` exhaustively, in Rust or in the generated Swift and Kotlin bindings, needs a case for it.

### Features

- Add the write-ahead batch cache (`BatchCache`, `MemoryCache`, `FileCache`) that holds encoded batches in memory or on disk until they upload.
- Add device state and device events (`DeviceState`, `DeviceEvent`) and RTC stats samples (`RtcStatsSample`, `RtcStat`) with their on-device windowing.
- Bound telemetry collector responses to 65,536 decoded bytes and retain batches with Retry-After on overflow. Document in `SPEC.md` that the exporter, not the transport, enforces the per-attempt export timeout and the 24-hour cap on server-directed pauses.
- Rename the public `RtcStatsSample.track_sid` field and constructor parameter to `track_id`; callers and generated bindings must migrate to the new name while keeping the same LiveKit track identifier values.
- Add the `TelemetryTransport` seam, a `net` HTTP transport over `livekit-net`, and LiveKit Cloud destination routing from a Room's server URL and token.

### Fixes

- Serialize cache writes and clears across process instances and reject replacements whose parent is already missing without resurrecting evicted batches.
- Compact unowned recomputable dead destinations while preserving all-dead routing, disabled projects and remembered consent.
- Speed up deduplication of large telemetry attribute lists while preserving last-value precedence and attribute order.

## 0.1.1 (2026-10-20)

### Features

- Add `livekit-telemetry`, the internal client telemetry core shared by the LiveKit client SDKs.
