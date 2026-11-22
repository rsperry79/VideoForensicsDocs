# Open items

Status: reviewed by Sonnet (read-only); findings folded in. Base: origin/wip with origin/dev merged (plan branch `chore/open-items-plan`).

## Gates
- Lite gate (per unit): `dotnet build` on the touched .csproj, `dotnet test --filter` on their tests plus direct callers; `npm run check` when JS/TS changes.
- Full gate (U3): `full-gate` skill, which runs build, `dotnet test` (whole solution), and `npm run check`.

## Units

### U1 Session token: sliding renewal and expiry path
- Decisions: sliding renewal once past half the lifetime; absolute cap of 24 hours from the original sign-in, after which the operator signs in again.
- Today: `SessionTokenService` (src/client/host/VideoForensics.Hosting/src/SessionTokenService.cs) issues a fixed 12-hour token; `Validate` returns null after expiry; nothing renews.
- Renewal rules (tests for each):
  - Keeps the same OperatorId, CredentialId, CredentialKind and SecurityStampAtIssuance.
  - Never renews a PasskeySetupOnly session (15 minutes, enforced from IssuedAtUtc in the handler).
  - Never renews after the 24-hour absolute cap.
  - A revoked paired device or a changed security stamp does not renew.
- Transport: the renewed token goes back on the same channel the current token uses. Confirm that channel at the start of U1 by reading the handler, then name it in the commit message.
- Redirect on expiry: the sign-in page (`/signin`). Message text is proposed as "Your session ended. Sign in again." Confirm the wording before merge.
- Files: SessionTokenService.cs, ISessionTokenService (add a renew method), src/client/web/VideoForensics.WebApp/src/Auth/PairedDeviceAuthenticationHandler.cs, PairedSessionState (src/client/ui/VideoForensics.Ui.Shared/src/Services), the sign-in redirect.
- Start: failing tests first.
- Done when: the new tests are red then green, the lite gate passes, and the branch is pushed.

### U2 Ring test failures
- Failing tests (src/providers/ring/provider/tests, Services/RingMediaDownloadServiceTests.cs):
  - DownloadVideosAsync_MatchedEventLinkedToUnmatchedRow_RekeysThatRowInsteadOfAddingOne
  - DownloadVideosAsync_MotionEventWithPersonDetection_NamesFileWithDetectionType
  - DownloadVideosAsync_UnmatchedEvent_StoresEventRowKeyedByDoorbotId
  - DownloadVideosAsync_NewVideo_ReleasesFileHandle
- Command: `dotnet test <path to the Ring provider tests csproj>`.
- Work: check whether the same tests fail on origin/wip before the plan branch. If pre-existing and not fixed here, log them through the `failure-log` skill with the cause. If caused by our changes, fix them.
- Done when: the Ring test project passes, or the failure is logged with the cause.

### U3 Full gate on wip
- Run after U1 and U2 land. Re-run if U1 lands after this unit.
- Build: only NU1902 notices allowed. Tests: full solution. Also `npm run check`.
- Done when: all three pass, and results are recorded in Results.

### U4 Package report
- Run `dotnet list VideoForensics.sln package --outdated`, complete (the earlier output was cut off).
- NU1902 for Microsoft.AspNetCore.WebUtilities 2.3.10 stays reported only (pin decision).
- Done when: the full list is in Results.

### U0 Deploy (user-run, elevated)
- Build and stage the merged branch with `Deploy-WebAppToProd.ps1` or the debug script, run by you in an elevated shell.
- Needed before U5 and U6 because they test live behaviour.
- Done when: `status.json` shows the new version and the site returns 200.

### U5 Live idle-resume check
- Blocked: needs U0 and an operator sign-in on the live site.
- Check: after the idle timeout, returning keeps Local-only pages available. Also check the expired-session redirect from U1.

### U6 Live browser checks
- Blocked: needs U0 and an operator passkey sign-in in Edge.
- Checks: grid previews, pending-tags text, keyboard, and the /media content layout.
- Done when: each check is recorded in Results with what was seen.

### U7 Debug attach
- Blocked: needs a user-run elevated attach.
- Done when: the debugger attaches to the running service and a breakpoint is hit.

### U8 WebView2 range playback
- Blocked: needs the MAUI head.
- Done when: range requests return 206 in WebView2.

### U9 Archive the media-review plan
- Move the media-review plan to the `plans/archived/` folder in the same `docs` repo, after U6 passes. Confirm the exact current path first.
- Depends on U6; it may not run until U6 is unblocked.

### Excluded until confirmed
- docs/plans/todo.md was not found in either checkout. Its items (backfill build deploy, test flakes, F:\VF-strip removal) are not in this plan until the file or its items are provided.

## Results
- U2 (Ring tests): closed. Passes on origin/wip cb14696a and on this branch. Earlier 4 failures not reproduced; logged in .claude/task_failure_log.md (2026-11-22).
- U1, U3, U4: not started.
- U0, U5–U8: blocked on user actions.
- U9: depends on U6.
- todo.md items: not included; the file was not found in either checkout.
