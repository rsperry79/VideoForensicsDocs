# Live view: up to 4 streams at once

Status: draft, not started. Written by hand after the plan helper returned no steps (twice).

## Decisions (user)
- Cap: 4 streams at once.
- A 5th Start is disabled, with a reason, until one stream stops.
- Telemetry panel shows data for the focused stream only.

## Facts from the code
- `LiveView.razor`: one stream. `ToggleLiveAsync` stops the current stream before starting another. Single `_liveDeviceId`, `_selectedSessionId`, `_pendingStreamSessionId`, one `MediaPlayer` with fixed id `liveViewPlayer`, one telemetry subscription.
- `liveview.ts` and `wwwroot/scripts/liveview.js` (both must match): `startLiveView` stores one global `window._liveViewPeerConnection`; `addIceCandidate` uses it. Each start registers a new `hubConnection.on("ReceiveIceCandidate")` that is never removed. The server offer is non-trickle, so that handler appears to be dead.
- `stopLiveView` is defined but never called, so the browser peer connection is not closed on Stop (existing leak).
- Server session limit is per device (`LiveViewSessionOrchestrator.StartAsync` returns the active session for that device). Different devices can stream together at the server.
- Not verified: provider API budget (`IProviderApiBudgetGuard`) and Ring's own concurrent live-view limit.

## Units (tests first; lite gate per unit; commit on wip, pull then push wip)
1. Browser module keeps a peer connection per session. Replace the window global in both `liveview.ts` and `liveview.js`; `addIceCandidate` takes the session id; remove the per-start `ReceiveIceCandidate` registration. Tests: JS unit test if the project has a runner, otherwise a bUnit JS-interop check that the module is called with the session id.
2. `stopLiveView` closes the peer connection; the page calls it on Stop and when a start fails. Test: stop closes the connection for that session only.
3. `LiveView.razor` holds a set of live streams (max 4) instead of one. Per-device player element ids (`liveViewPlayer-{deviceId}`). Stopping one stream leaves the others running.
4. Start on other tiles is disabled with a localized reason when 4 streams are live (new keys in `LiveView.resx`, all locales present).
5. Telemetry subscribes to the focused stream only; focus moves when a stream is started or clicked.
6. bUnit tests: cap enforced at 4, per-tile players, disable reason shown at the cap, stop closes only that stream.
7. Lite gate for Ui.Shared, WebApp, Hosting; manual check with two cameras on the deployed build.

## Open risks (verify before declaring the cap safe)
- Provider API budget guard may refuse a fourth concurrent session.
- Ring may limit concurrent live-view connections per account.
- Garden and Front Door must keep working on the single-stream path.
