# Live view: up to 4 streams at once (implementation plan)

Status: approved decisions, not started. Sonnet review rounds 1 and 2 folded in (see "Review history").
Branch: work directly on `wip` (pull, commit, push `wip` at the end of each unit; no merges or PRs).

## Decisions (user)
- Cap: 4 streams at once, enforced on the server and on the page.
- A 5th Start is disabled, with a localized reason, until one stream stops.
- Telemetry shows the focused stream only.
- Starts are serialized: the clicked tile's Start is disabled during its own start; stops stay possible.

## Facts (verified in code)
- `src/client/ui/VideoForensics.Ui.Shared/src/Pages/LiveView.razor`: one stream. `ToggleLiveAsync` stops the current stream before starting another. Single `_liveDeviceId`, `_selectedSessionId`, `_pendingStreamSessionId`, `_latestTelemetry`, `_telemetrySubscription`, `_error`. The catch calls `StopLiveAsync()`, which would stop every stream.
- Browser module, both files must match: `src/ts/liveview.ts` and `src/wwwroot/scripts/liveview.js` (both under `Ui.Shared/src`). Window global declared in `src/ts/globals.d.ts` (line 34). `startLiveView` sets `window._liveViewPeerConnection` (liveview.ts:105); `addIceCandidate` uses it and the page never calls it. Each start registers `hubConnection.on("ReceiveIceCandidate")` (line 95), never removed. Server offer is non-trickle; `LiveViewSignalingHubTests.cs` (~line 327) asserts no `ReceiveIceCandidate` is pushed, so the handler is dead.
- `startLiveView` creates the pc before the video-element check, so a missing element leaks it. It closes the pc on negotiation error (line 125).
- JS tests: `src/client/web/VideoForensics.WebApp/tests/js/liveview.test.ts`. It imports `src/wwwroot/scripts/liveview.js` (not the `.ts`), so `.js` changes first. Run with `npm run test:js` from `src/client/web/VideoForensics.WebApp` (check `package.json` for the exact script name before the red run).
- Existing page tests (`src/client/ui/VideoForensics.Ui.Shared/tests/LiveViewPageTests.cs`) cover stills, camera selection, module import and hub init. They do NOT start a stream. Unit 0 builds the helpers the stream tests need.
- Server limit is per device: `LiveViewSessionOrchestrator.StartAsync` (lines 374-379) returns the active session for that device. No global cap. Budget guard throws `InvalidOperationException` (line 414).
- Web path: `RemoteLiveViewSessionService` uses `EnsureSuccessStatusCode`, so a 409 arrives as a generic `HttpRequestException`. `LiveViewEndpoints.TryMapFailure` (line 235) maps every `InvalidOperationException` to a bare 409. So a localized cap or budget message needs a typed error and a typed 409 body.
- Config: `IForensicsConfiguration` (lines 134-142, defaults at 253), `ConfigurationLoader.cs` (81-85), `ConfigDtos.cs` (67-71), `ConfigDtos.Mapping.cs`, and `RemoteForensicsConfigurationServiceTests.cs`. Orchestrator tests mock config (`OrchestratorTests.cs:43`), so an unset value reads as 0 and must mean "use 4".
- `LiveViewIdleTimeoutService`: idle timeout is 5 minutes (`IForensicsConfiguration.cs:254`). The page never calls `ExtendAsync`.
- Not verified: Ring's concurrent live-view limit per account.

## Units
Each unit: failing test first (record the red run), then the change, then the lite gate, then commit and push `wip`.

### Unit 0: test helpers for stream tests (done)
- Files: `src/client/ui/VideoForensics.Ui.Shared/tests/LiveViewPageTests.cs` (and a shared helper if the file grows).
- Change: make the `ILiveViewSessionService` mock a field with `StartAsync`, `GetActiveSessionAsync`, `StopAsync` and `ExtendAsync` set up; add a fake `liveview.js` module (`JSInterop.SetupModule(ExpectedModulePath)`) with `initializeHubConnection` returning a reference and `startLiveView` and `stopLiveView` recorded; expose a `Subject<LiveViewSessionDto>` for `SessionChanged`.
- Test: a helper-only smoke test starts one stream and sees `startLiveView` called once.
- Done: smoke test green; existing LiveViewPageTests still green.
- Status: [x] (red: compile failure, green 11/11)

### Unit 1: one peer connection per session (JS)
- Files: `src/wwwroot/scripts/liveview.js`, `src/ts/liveview.ts`, `src/ts/globals.d.ts`, `src/client/web/VideoForensics.WebApp/tests/js/liveview.test.ts`.
- Change: `Map<sessionId, pc>`, recorded only after negotiation succeeds (fixes the leak on a missing element); `stopLiveView(sessionId)` closes and removes it; delete the window global, the per-start `ReceiveIceCandidate` registration, `addIceCandidate`, and the `globals.d.ts` field.
- Failing tests: two sessions each keep their pc; stopping one closes only its pc; a missing video element leaves no pc in the map; the global is gone. Rewrite the "feeds server ICE candidates", both `addIceCandidate` tests, and the `stopLiveView(pc)` test.
- Done: JS tests pass; `.ts` and `.js` match.
- Status: [x] (red: 11 failed; green 42/42)

### Unit 2: sessions in the page (set, max 4, teardown)
- Files: `src/Pages/LiveView.razor`. (Merged with the old teardown unit: it cannot be tested alone.)
- Change: replace the single state with a set of live streams; per-device players `liveViewPlayer-{deviceId}`; a failed start stops only its device; `CleanupAsync` stops and closes every live session, each in its own try/catch; starts serialized: the clicked tile's button is disabled during its start, other tiles stay enabled for stops; a queue replaces `_pendingStreamSessionId`.
- Failing tests: two streams live at once; a failed start leaves the other live; stopping one keeps the other; teardown with two live sessions calls `stopLiveView` for both; a second start while one is starting still gets its own player.
- Done: tests pass; Garden and Front Door single-stream behaviour covered by the new tests.
- Status: [x] (red: 4 failed; green LiveViewPageTests 15/15, Ui.Shared 2373/2373)

### Unit 3: disable Start at the cap, with a typed error
- Files: `src/server/core/src/Tools/LiveViewCapReachedException.cs` (new, derives `InvalidOperationException`), `src/client/web/VideoForensics.WebApp/src/Api/LiveViewEndpoints.cs` (409 body carries a `code`), `src/client/host/VideoForensics.Hosting/src/Remote/RemoteLiveViewSessionService.cs` (rethrows the typed error), `src/Pages/LiveView.razor`, `src/Pages/LiveView.resx`.
- Change: at 4 live, Start on other tiles is disabled with a localized reason; the cap and budget errors map to localized messages by `code`, not by message text.
- Failing tests: fifth Start disabled with the reason (bUnit); 409 body includes the code (`LiveViewEndpointsTests.cs`); the Remote service rethrows the typed error (`RemoteLiveViewSessionServiceTests.cs`); the page shows the localized budget message.
- Done: tests pass; the new key exists in `LiveView.resx` (only resx in the repo for this page).
- Status: [ ]

### Unit 4: telemetry follows the focused stream
- Files: `src/Pages/LiveView.razor`, `src/Services/LiveViewActivationWaiter.cs` only if its subscription changes.
- Change: telemetry subscribes to the focused stream; last sample cached per session; focus changes do not blank the panel.
- Failing test: switching focus shows the other stream's cached sample; a sample for an unfocused stream is ignored.
- Done: test passes.
- Status: [ ]

### Unit 5: server cap
- Files: `src/server/core/src/Tools/LiveViewSessionOrchestrator.cs`, `src/core/api` config (`IForensicsConfiguration.cs`, `ConfigurationLoader.cs`, `ConfigDtos.cs`, `ConfigDtos.Mapping.cs`), tests in `src/server/core/tests/Tools/LiveViewSessionOrchestratorTests.cs` and `RemoteForensicsConfigurationServiceTests.cs`.
- Change: config `LiveViewMaxConcurrentSessions` (0 or unset means 4); `StartAsync` refuses a new device when the active count reaches the cap, throwing `LiveViewCapReachedException`; the check and the insert run under one lock (two fifth starts cannot both pass). Decide the count source: the in-memory `_activeSessions` map (use it, it is what the orchestrator owns).
- Failing tests: fifth distinct-device start refused; re-starting a device that already has a session returns that session (idempotent, not refused); unset config means 4; two concurrent starts at 4 do not exceed the cap.
- Done: tests pass; config key documented.
- Status: [ ]

### Unit 6: idle timeout and reconnect
- Files: `src/Pages/LiveView.razor`, `src/ts/liveview.ts` and `.js` (connection state callbacks through a `DotNetObjectReference`).
- Change: session-ended push flips the tile back to its still; while visible, each live session is extended on a timer well below 5 minutes (injectable interval); SignalR `onclose` or reconnect failure stops the affected streams and shows the failure.
- Failing tests: session-ended push shows the still; extend is called on the timer (fake clock); connection loss shows the failure. Mock `ExtendAsync` in the sessionService setup from unit 0.
- Done: tests pass; manual check that a stream left running is not cut by the idle timeout.
- Status: [ ]

### Unit 7: lite gate and two-camera check
- Lite gate projects: Ui.Shared, WebApp, Hosting, Server.Core, Client.Common, Core.Api, Client.Core (config).
- Deploy with `Debug-WebAppService.ps1` (elevated path), start two cameras, watch both for 2 minutes.
- Done: gate green; both cameras play; Ring's concurrent limit recorded here.
- Status: [ ]

## Open risks
- Ring may limit concurrent live-view connections per account. The cap stays configurable; unit 7 settles it.
- Budget guard may refuse a fourth session. Unit 3 maps the error; unit 7 tests it.

## Review history
- Round 1 (Sonnet, read-only, on the design): corrected JS paths; merged the JS change with stop-close; added server cap, idle timeout, reconnect, JS test rewrite, teardown of all sessions.
- Round 2 (Sonnet, read-only, on the implementation plan): added unit 0 for the missing test helpers; merged teardown into the page-set unit; added the typed cap error and its 409 body; added config plumbing and the race lock; corrected the resx claim (one file, no locales); fixed unit 1's test list and the leak; decided serialized starts.
