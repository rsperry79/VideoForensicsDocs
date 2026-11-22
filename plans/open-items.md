# Open items

Status: draft, pending Sonnet review. Base: origin/wip with origin/dev merged (plan branch `chore/open-items-plan`).

## Units

### U1 Session token: sliding renewal and expiry path
- Decision (user): sliding renewal. A session renews while the operator is active, once past half its lifetime.
- Today: `SessionTokenService` (src/client/host/VideoForensics.Hosting/src/SessionTokenService.cs) issues a fixed 12-hour token; `Validate` returns null after expiry; nothing renews.
- Work: renew on use past half-life and return the new token to the client; on expiry, send the operator to sign-in with a clear message, not a blank page.
- Start: failing tests first (token renews past half-life; expired token is rejected; expiry redirects to sign-in).
- Files: SessionTokenService.cs, PairedDeviceAuthenticationHandler, the client session state (PairedSessionState), the sign-in redirect.
- Done when: tests red then green, lite gate passes, branch pushed.

### U2 Ring test failures
- 4 failures in src/providers/ring/provider/tests (RingMediaDownloadServiceTests). Reproduces on the plan branch.
- Work: check whether they fail on origin/wip before the plan branch. If pre-existing, fix or log with a reason. If caused by our changes, fix.
- Done when: the Ring test project passes, or the failure is logged with the cause.

### U3 Full gate on wip
- Run after U2 and the merges: build (only NU1902 notices allowed), dotnet test (full), npm run check.
- Done when: all three pass, results recorded.

### U4 Package report
- Complete `dotnet list VideoForensics.sln package --outdated` (the earlier output was cut off).
- NU1902 for Microsoft.AspNetCore.WebUtilities 2.3.10 stays reported only (pin decision).
- Done when: the full list is in the plan's results section.

### U5 Live idle-resume check
- Blocked: needs an operator sign-in on the live site and a deployed build with U1.
- Check: after the idle timeout, returning keeps Local-only pages available.

### U6 Live browser checks (blocked on sign-in)
- Grid previews, pending-tags text, keyboard, and the /media content layout after the latest deploy.
- Blocked: needs the operator to sign in (passkey) in Edge.

### U7 Debug attach
- Verify Debug-WebAppService.ps1 attaches to the running service in Visual Studio.
- Blocked: needs a user-run elevated attach.

### U8 WebView2 range playback
- Blocked: needs the MAUI head.

### U9 Archive the media-review plan
- Move plans/media-review-implementation.md to plans/archived/ after U6 passes.

## Results
(filled as units finish)
