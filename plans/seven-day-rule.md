# Seven-day rule for every sync method

Status: proposed (not started)

## Decision (from the user)
- Every sync method follows the rule: one run pulls at most seven days. Strict: no run pulls a year of history at once.

## Current state
- `AccountSyncRunner.SyncAsync` (~307–361): full history walks `WeeklyWindows(now - 1y, now)` in one run (~342). A custom range walks weekly windows clamped to one year (~348–356). Any other range is clamped by `OneWeekFloor` (~360, 399).
- Sync Now (`AccountEndpoints.cs` ~372–425) passes the range straight through, so All and custom ranges reach the weekly loop.
- Scheduler backfill (`ScheduledSyncService.cs` ~575–590) already pulls one week per tick and advances `BackfillCursorUtc`. It already follows the rule.
- Scheduler catch-up (`ScheduledSyncService.cs` ~645–670, `PullCatchUpAsync`) walks every weekly window after a gap in one run. It does not follow the rule.
- The first-sync coordinator (`InitialSyncCoordinator.cs`) validates only and pulls nothing.

## Units
1. **Sync Now and custom ranges: one seven-day pull.** In `AccountSyncRunner`, replace the full-history and custom weekly loops with one `OneWeekFloor` pull. Test first: an All request and a custom request each produce exactly one pull, starting no earlier than now minus seven days. Log that the range was limited.
2. **Scheduler catch-up: one window per run, with a cursor.** Replace `PullCatchUpAsync` with one window. Store the cursor (reuse `BackfillCursorUtc`, or a separate catch-up cursor) so the next run continues where this one stopped. Test first: a run after a 3-week gap pulls one week and advances the cursor by one week.
3. **Backfill stays one week per tick** (already compliant). Add a test that it does not exceed one week per run.
4. **UI.** Sync Now's All and custom choices must match what runs. Open item: keep the choices and clamp them, or remove them. The user's last answer did not settle this.

## Open items
- Confirm the UI choice above before unit 4.
- Whether Sync Now should say when a range was limited (proposed: yes, in the sync log).

## Done when
- No sync path pulls more than seven days in one run.
- A year of history fills in over successive scheduler runs, one week each.
