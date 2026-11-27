# Live view: up to 4 streams at once

Status: draft, reviewed by Sonnet (read-only), findings folded in. Not started.

## Decisions (user)
- Cap: 4 streams at once, enforced on the server as well as the page.
- A 5th Start is disabled, with a localized reason, until one stream stops.
- Telemetry panel shows data for the focused stream only.

## Facts (verified in code)
- `src/client/ui/VideoForensics.Ui.Shared/src/Pages/LiveView.razor`: one stream. `ToggleLiveAsync` stops the current stream before starting another. Single `_liveDeviceId`, `_selectedSessionId`, `_pendingStreamSessionId`; one `MediaPlayer` (`ElementId` is a parameter, so per-device ids work); one telemetry subscription. `_isToggling` disables every Start during any toggle. The catch calls `StopLiveAsync()`, which would stop every stream.
- Browser module, two files that must match: `src/client/ui/VideoForensics.Ui.Shared/src/ts/liveview.ts` and `src/client/ui/VideoForensics.Ui.Shared/src/wwwroot/scripts/liveview.js`. The window global is also declared in `src/ts/globals.d.ts` (line 34). `startLiveView` sets one global `window._liveViewPeerConnection` (liveview.ts:105); `addIceCandidate` uses it and is never called from the page. Each start registers `hubConnection.on("ReceiveIceCandidate")` (line 95) that is never removed. The server offer is non-trickle, and `LiveViewSignalingHubTests.cs` (~line 327) asserts no `ReceiveIceCandidate` is pushed, so that handler is dead.
- `startLiveView` already closes its peer connection on a negotiation error (line 125). `stopLiveView` is never called from the page. `CleanupAsync` (razor ~407-459) stops only one session and does not close peer connections.
- JS tests exist: `src/client/web/VideoForensics.WebApp/tests/js/liveview.test.ts` (uses the global, the `ReceiveIceCandidate` handler and `stopLiveView(pc)`). They must be rewritten in the same unit as the JS change.
- Server session limit is per device (`LiveViewSessionOrchestrator.StartAsync`, lines 374-379 return the active session for that device). There is no global cap. `IProviderApiBudgetGuard` throws `InvalidOperationException` (line 414) and the page shows the raw message.
- `LiveViewIdleTimeoutService` ends every active session on its own timer; the page never extends them (no `ExtendAsync` caller). A tile can show live after the server has ended the stream.
- Not verified: Ring's own concurrent live-view limit per account.

## Units (tests first; lite gate per unit; commit on wip, pull then push wip)
1. JS: one peer connection per session. Keep a `Map` keyed by session id in both `.ts` and `.js`. `startLiveView` returns and records the pc; `stopLiveView(sessionId)` closes and removes it; delete the window global, the per-start `ReceiveIceCandidate` registration and `addIceCandidate`. Rewrite `liveview.test.ts` in this unit. Merged with the old unit 2 because a stop-only change cannot be shipped alone.
2. Page: `CleanupAsync` loops over every live session, each in its own try/catch, and closes every peer connection. Stop closes only that session.
3. Page: a set of live streams (max 4) replacing the single state. Per-device player ids (`liveViewPlayer-{deviceId}`). A failed start stops only that device, not every stream. A pending-start set or queue replaces `_pendingStreamSessionId`. Starts are serialized by a decision, not by the shared `_isToggling` flag, so pick one: keep serialization or allow concurrent starts with a per-device busy flag.
4. Page: at 4 live, Start is disabled on the other tiles with a localized reason (new keys in `LiveView.resx`, all locales present). Map the budget-guard error to a localized message instead of the raw text.
5. Telemetry follows the focused stream. Cache the last sample per session so focus changes do not show "no data" until the next sample. Check how `LiveViewActivationWaiter` subscribes to the session push group before relying on it.
6. Server cap: `StartAsync` refuses a start when the active session count reaches the configured cap (default 4, configurable). The page cap stays as the courtesy.
7. Idle timeout and reconnect: the page handles a session-ended push (flip the tile back to its still) and keeps streams alive while visible (`ExtendAsync` on a timer). Handle a SignalR reconnect: surface the failure, do not silently keep a dead stream.
8. Tests (bUnit): cap enforced at 4, per-tile players, disable reason at the cap, Stop closes only that stream, failed start stops only that device, cleanup closes all.
9. Lite gate (Ui.Shared, WebApp, Hosting, Server.Core) and a manual check with two cameras on the deployed build. Garden and Front Door single-stream behaviour must still work (`LiveViewPageTests.cs` uses `liveViewPlayer`).

## Open risks (verify before declaring the cap safe)
- Ring may limit concurrent live-view connections per account. Keep the cap configurable and test with two cameras.
- The provider budget guard may refuse a fourth session. Map the error and test.
- Simultaneous starts overwrite the single pending id today; unit 3 must cover this.
