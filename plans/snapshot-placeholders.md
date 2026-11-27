# Stop storing placeholder snapshots from dead cameras

Branch: `feat/snapshot-placeholders` (worktree `F:/vf-worktrees/snapshot-placeholders`, from `wip`; docs submodule on the same branch).

## Problem
Garden had dead batteries. Its snapshots kept "succeeding": the newest 8 stored JPEGs (10:35 to 11:45) have the same MD5 and size (`2653d118053d`, 5442 bytes), the gray placeholder Ring returns for an offline camera. `SnapshotRollupService` built hourly time-lapses from 6 identical frames. Nothing checks the camera's health or the frame before storing it.

## Decisions (user)
- Detection: skip a frame when the provider reports the camera not connected, OR when the image hash equals the device's previous snapshot.
- Existing placeholder snapshots: mark them (typed column plus a one-off backfill); files stay, no rows are deleted.
- A skipped capture's record: see "Open decision" below.

## Review findings (Sonnet, read-only), folded in
- Ring snapshot classification (`RingMediaDownloadService.ClassifySnapshotResponse`, ~L292) only checks HTTP 200 and JPEG magic bytes, so the 5442-byte gray JPEG passes. `attempt.Connected` is `DeviceHealth.IsOnline` (Health.Connected, falling back to Subscribed); it may be true for a dead-battery camera or null when the health read fails. The hash rule is the certain one; the connected rule is a bonus.
- Bytes are the camera's raw JPEG (`WriteImage` = `File.WriteAllBytes`); `RecordCaptureMediaAsync` and the pull path both store lowercase-hex SHA-256 of the written file. Compare hex strings, not byte[] against hex.
- The hash must be computed from the in-memory bytes BEFORE `WriteImage`; today `RecordCaptureMediaAsync` hashes after writing and is best-effort. The check moves above the write.
- `SnapshotRollupService.LoadStillsAsync` reads `SnapshotCapture` attempts (`_captures` + `SnapshotStillMapper`) when registered; the Media-row query is only the fallback. `SnapshotTimelineService` also reads `GetAttemptsForDeviceAsync`. Both paths need the skip.
- `SnapshotCapture.Create` allows trigger fresh/latest and status 100-599 only; there is no skipped state. The timeline mapper renders failed attempts as gray gaps.
- Scheduled pull: `EventMediaPuller` calls `IMediaDownloadService.DownloadSnapshotsAsync`; Ring's implementation (`RingMediaDownloadService.DownloadSnapshotsAsync` ~L1202-1380) writes Media rows and a DownloadEvent, hashes at ~L1310, has no connected value and no previous-hash lookup. Uniview has its own implementation (~L492/634/797); out of scope for this plan (Ring placeholder only), noted for later.
- `MediaItemDto` is a positional record with ~15 construction sites: the new field gets a default value.
- The filter belongs in the SQL query (`MediaItemQueries.SnapshotsByDeviceAndDateRange`), server side, so every caller benefits (client/split and providers rules).
- Backfill needs a window function (SQLite LAG) or per-device paging as a batched, resumable single UPDATE; `MediaFlagBackfill` only covers a 7-day window.
- Marking is metadata only; confirm purge and export flows are unaffected.

## Units (failing test first for each; one commit per unit; lite gate per unit)
1. [ ] `SnapshotPlaceholderDetector.IsPlaceholder(bool? connected, string sha256Hex, string? previousSha256Hex)`: true when `connected == false` or hashes match (case-insensitive). Tests: each branch, `connected == null`, no previous hash.
2. [ ] Previous-hash query: `MediaItemQueries.LatestSnapshotHash(deviceId)` (single column, server side) exposed through the existing repository interface and its remote counterpart (`RemoteMediaItemRepository`, `MediaApiEndpoints` ~L465, `UnitOfWork` forward ~L612).
3. [ ] Capture path (`SnapshotCycle.CaptureDeviceAsync`): hash the response bytes, ask the detector, and on a placeholder skip `WriteImage` and the Media row; record the attempt per the open decision; log at Information with device name and reason (offline or identical frame).
4. [ ] Scheduled-pull path (`RingMediaDownloadService.DownloadSnapshotsAsync`): same rule before the file and Media row are written.
5. [ ] Schema: `MediaItem.IsPlaceholder` (bool, default false), `MediaItemConfiguration`, EF migration under `src/data/database/sqlite/src/Migrations/` plus regenerated `VideoForensicsDbContextModelSnapshot.cs` (confirm the migration is not empty; `ModelMatchesMigrationsTests`), `MediaItemDto` (+ default value) and `MediaItemDto.Mapping.cs` (`ToDto`/`ToDomain`).
6. [ ] Query filter: `SnapshotsByDeviceAndDateRange` (and `LatestDeviceMedia` if it returns snapshots) excludes `IsPlaceholder`; covers the live view still, the Devices tab and the rollup fallback with no UI change.
7. [ ] Rollup and timeline: `SnapshotRollupService.LoadStillsAsync` and `SnapshotTimelineService` skip placeholder/skipped attempts; no time-lapse when fewer than 2 real stills remain.
8. [ ] Backfill: `SnapshotPlaceholderBackfill` (Data.Core interface + DI in `ServiceCollectionExtensions` + hosted service modelled on `MediaFlagBackfillService`): marks snapshot rows whose hash equals the device's immediately preceding snapshot (repeats only; the first of a run stays). Batched, resumable, idempotent, logs counts.
9. [ ] Lite gate (Data.Core, Data.Database, Hosting, Ring provider, Core.Providers.Common, Ui.Shared, WebApp and callers), commit, push `feat/snapshot-placeholders` (docs first).

Order: 1, 2, 5, 3, 4, 6, 7, 8, 9 (schema and query before consumers).

## Decision recorded: skipped capture
A skipped capture is recorded as a `SnapshotCapture` attempt with a new typed `SkipReason` column (values: Offline, IdenticalFrame; null for a normal attempt), no media path, and a distinct gap on the timeline. This adds to unit 5 (schema): `SnapshotCapture.SkipReason` + `SnapshotCapture.Create` accepting a skip (status check relaxed only for a skip), entity configuration, migration, DTO/mapping, and `SnapshotStillMapper`/`SnapshotTimelineService` treating it as "skipped" not "failed". Unit 3 writes it; unit 7 reads it. Existing rows have null.
