# Events tab merge, Detail tab removal

Branch: `feat/events-merge` (worktree `F:/vf-worktrees/events-merge`, from `wip`; docs submodule on the same branch).

## Decisions (user)
- The Events tab lists all system events: camera events plus the types the Detail tab showed (jamming incidents, signal anomalies, bit-rate flags).
- Clicking a row sets the camera and cursor to the event time, switches to Timeline and plays.
- Hovering a row shows that event's data (the fields Detail showed: description, detection type, confidence, anomaly, similarity, model version, zones, people, alerts; window start/end and level for signal rows).
- The Detail tab is removed. Desktop and mobile share `ReviewPage`, so one change covers both.

## Units
1. [x] (done; red run: CS0246/CS0103 missing types; green 8/8, verified by rerun) Row model: one `ReviewEventRow` type (kind, device, start, optional end, label, data lines) built from `EventDto`+`EventDetailDto`, `JammingIncidentDto`, `ReviewAnomaly`, `ReviewBitRateFlag`; sort newest first. Failing unit tests first.
2. [x] `ReviewEventsTab`: load all four sources, merge via the row model, honour the event-type filter (`ReviewEventFilters.Shows`), keep paging for camera events and live refresh. bUnit tests first.
3. [x] Row interaction: click = SetCameras (if needed), SelectEvent (camera events), SetCursor, SetTab(Timeline), start playback; hover shows the per-row data (title/popover, localized). Check whether Timeline auto-plays on cursor change; if not, add the play trigger. Tests first.
4. [ ] Remove Detail: `ReviewTab.Detail`, `ReviewDetailTab.razor`, its tests, tab button in `ReviewSidebarControls`, branch in `ReviewPage`, `ReviewUrl` handling of `tab=detail` (decision needed: map to Events; ask user, no silent shim), unused resource keys.
5. [ ] Localization: every new string in resx (`rules/ui-shared.md`).
6. [ ] Lite gate (build + tests for Ui.Shared and tests of direct callers), commit, push `feat/events-merge` (docs submodule first).

## Review findings (Sonnet, read-only)
- Playback never auto-starts: `SetCursor` only seeks (`ReviewPlayer.OnParametersSet`); play is only `TogglePlayAsync`. `ReviewTimelineTab` mounts only on the Timeline tab. Unit 3 adds a "play on arrival" flag in `ReviewPageState`, consumed by `ReviewTimelineTab` after first render and clip load, never toggling when already playing. Test in `ReviewPlaybackCursor_Tests.cs`.
- `ReviewUrl.TryParseTab` loops `Enum.GetValues<ReviewTab>()`; once `Detail` goes, `?tab=detail` silently parses to Timeline. `ReviewUrlTests.cs` lines 119-122 must change. Decision asked of the user (no silent shim).
- Event-type filter: gate each system source with `ShowJamming/ShowAnomalies/ShowBitRate` before loading (as Detail does), not after merging. A system-only selection gives an empty provider list; that must not mean "no events".
- Paging: camera events page server-side; system sources are unpaged. Decision asked of the user.
- Live refresh: add refresh for system rows (`IDeviceHealthChangedSource`, as Detail does).
- Hover has no touch equivalent; mobile gets tap-to-navigate only unless the user says otherwise.
- Keep `IReviewDetailLoader` methods (jamming, anomalies, bit rate); only the tab goes.
- Extra files: `ReviewTab.cs`, `ReviewPage.resx` (TabDetail, Detail* keys; some feed row labels), `MauiProgram.cs`, `WebAppUiStateRegistration.cs`, `plans/media-page-rework.md`; tests `ReviewPage_Tests`, `ReviewPersistence_Tests`, `ReviewPageStateTests`, `ReviewSidebarControls_Tests`, `ReviewFilterPanel_Tests`, `ReviewTimelineFeeder_Tests`, `ReviewTestContext`, `ReviewFakes`, `ReviewEventsTab_Tests`, delete `ReviewDetailTab_Tests`.

## Answers recorded
- `tab=detail` opens Timeline (unknown tab falls to default; no mapping code). Update `ReviewUrlTests` accordingly.
- Merged list loads every row in the range, no paging: drop Load more and the Total line for this tab.

## Notes
- Failing test first for every unit; record the red run here.
- Subagents: Haiku only, edit inside the worktree only.
