# Live view: up to 4 streams at once (implementation plan)

Status: approved decisions, not started. Sonnet review round 1 folded in (see "Review history").
Branch: work directly on `wip` (pull, commit, push `wip` at the end of each unit; no merges or PRs).

## Decisions (user)
- Cap: 4 streams at once, enforced on the server and on the page.
- A 5th Start is disabled, with a localized reason, until one stream stops.
- Telemetry shows the focused stream only.

## Facts (verified in code)
- `src/client/ui/VideoForensics.Ui.Shared/src/Pages/LiveView.razor`: one stream. `ToggleLiveAsync` stops the current stream before starting another. Single `_liveDeviceId`, `_selectedSessionId`, `_pendingStreamSessionId`. `_isToggling` disables all Start buttons during a toggle. The catch calls `StopLiveAsync()`, which would stop every stream.
- Browser module, both files must match: `src/client/ui/VideoForensics.Ui.Shared/src/ts/liveview.ts` and `src/client/ui/VideoForensics.Ui.Shared/src/wwwroot/scripts/liveview.js`. Window global declared in `src/ts/globals.d.ts` (line 34). `startLiveView` sets global `window._liveViewPeerConnection` (liveview.ts:105); `addIceCandidate` uses it and the page never calls it. Each start registers `hubConnection.on("ReceiveIceCandidate")` (line 95), never removed. Server offer is non-trickle; `LiveViewSignalingHubTests.cs` (~line 327) asserts no `ReceiveIceCandidate` is pushed, so the handler is dead.
- `startLiveView` closes its pc on negotiation error (line 125). `stopLiveView` is never called from the page. `CleanupAsync` (razor ~407-459) stops one session and closes no peer connections.
- JS tests: `src/client/web/VideoForensics.WebApp/tests/js/liveview.test.ts` uses the global (line 65), the `ReceiveIceCandidate` handler (line 83) and `stopLiveView(pc)` (line 119). Rewrite in unit 1.
- Server limit is per device: `LiveViewSessionOrchestrator.StartAsync` (lines 374-379) returns the active session for that device. No global cap. `IProviderApiBudgetGuard` throws `InvalidOperationException` (line 414); the page shows the raw text.
- `LiveViewIdleTimeoutService` ends sessions on its own timer; the page never calls `ExtendAsync`.
- Not verified: Ring's concurrent live-view limit per account.

## Units
Each unit: failing test first (record the red run), then the change, then the lite gate, then commit and push `wip`.

### Unit 1: one peer connection per session (JS)
- Files: `src/ts/liveview.ts`, `src/wwwroot/scripts/liveview.js`, `src/ts/globals.d.ts`, `src/client/web/VideoForensics.WebApp/tests/js/liveview.test.ts`.
- Change: `Map<sessionId, pc>`; `startLiveView` records its pc; `stopLiveView(sessionId)` closes and removes it; delete the window global, the per-start `ReceiveIceCandidate` registration and `addIceCandidate`.
- Failing test: two sessions start, each pc is kept; stopping one closes only its pc; the global is gone.
- Done: JS tests pass; both `.ts` and `.js` match.
- Status: [ ]

### Unit 2: close every connection on page teardown
- Files: `src/Pages/LiveView.razor` (`CleanupAsync`).
- Change: loop over all live sessions, each in its own try/catch; close every peer connection and stop every session.
- Failing test (bUnit, JS interop): two live sessions, dispose, both `stopLiveView` calls happen.
- Done: test passes.
- Status: [ ]

### Unit 3: set of live streams (max 4) in the page
- Files: `src/Pages/LiveView.razor`, `src/Pages/LiveView.resx` only if a new key is needed here (unit 4 owns the reason string).
- Change: replace the single state with a set; per-device player ids `liveViewPlayer-{deviceId}`; a failed start stops only that device; a pending-start set replaces `_pendingStreamSessionId`. Decision to make before coding: serialize starts (keep `_isToggling`) or allow concurrent starts with a per-device busy flag.
- Failing tests: two streams live at once; a failed start leaves the other live; stopping one keeps the other; two starts in quick succession both get a player.
- Done: tests pass; Garden and Front Door single-stream tests still pass (`LiveViewPageTests.cs`).
- Status: [ ]

### Unit 4: disable Start at the cap, with a reason
- Files: `src/Pages/LiveView.razor`, `src/Pages/LiveView.resx` (all locales present), budget error mapping.
- Change: at 4 live, Start is disabled on the other tiles with a localized reason; the budget-guard error shows a localized message, not raw text.
- Failing tests: fifth Start disabled with the reason; budget error shown localized.
- Done: tests pass; resx keys in every locale file.
- Status: [ ]

### Unit 5: telemetry follows the focused stream
- Files: `src/Pages/LiveView.razor`, `src/Services/LiveViewActivationWaiter.cs` only if its subscription needs a change.
- Change: telemetry subscribes to the focused stream; last sample cached per session; focus changes do not blank the panel.
- Failing test: focus switch shows the other stream's cached sample; a sample for an unfocused stream is ignored.
- Done: test passes.
- Status: [ ]

### Unit 6: server cap
- Files: `src/server/core/src/Tools/LiveViewSessionOrchestrator.cs`, its tests in `src/server/core/tests/Tools/`.
- Change: `StartAsync` refuses when the active session count reaches the configured cap (default 4, from config); the refusal is a typed error the endpoint maps to 409.
- Failing test: fifth distinct-device start refused; re-starting an existing device still returns its session (idempotent).
- Done: tests pass; config key documented.
- Status: [ ]

### Unit 7: idle timeout and reconnect
- Files: `src/Pages/LiveView.razor`, a SignalR connection-state handler in the page or the JS module.
- Change: on a session-ended push, the tile returns to its still; while visible, keep each live session alive with `ExtendAsync` on a timer; on SignalR reconnect, show the failure and stop the affected streams.
- Failing tests: session-ended flips the tile to its still; reconnect surfaces the failure.
- Done: tests pass; manual check: a stream left running is not cut by the idle timeout.
- Status: [ ]

### Unit 8: lite gate and two-camera check
- Run the lite gate for Ui.Shared, WebApp, Hosting and Server.Core.
- Deploy with `Debug-WebAppService.ps1` (elevated path), then start two cameras and watch both for 2 minutes.
- Done: gate green; both cameras play; Ring's limit recorded in the plan.
- Status: [ ]

## Open risks
- Ring may limit concurrent live-view connections per account. Keep the cap configurable; unit 8 settles it.
- Provider budget guard may refuse a fourth session. Unit 4 maps the error; unit 8 tests it.

## Review history
- Round 1 (Sonnet, read-only): corrected JS paths; merged the JS change with stop-close; added server cap, idle timeout, reconnect, JS test rewrite, cleanup of all sessions.
