# Integrate open branches into one branch

Status: reviewed by Sonnet (read-only); findings folded in. Target: one integration branch `chore/integration-all` from origin/wip. Nothing touches `wip` until the user approves.

Base note: origin/wip is 8c87ce61. The main checkout's local `wip` (01e22268) is not equal to origin/wip. Branch from origin/wip, never from local wip.

## Branches in scope
| Branch | Head (origin) | New vs origin/wip | Content |
|---|---|---|---|
| fix/session-renewal | 7bd125b6 | 1 commit | Expired-session notice, token-swap tests |
| chore/open-items-plan | 2e91c8df | 3 commits incl. this plan's branch history | Docs pointer to plan results, Ring failure log entry |
| chore/gitignore-agent-tooling | e27e6180 | 1 commit | .gitignore entries for .agents/, .codex/, AGENTS.md |
| test/mtp-reporting | 01e22268 | 1 commit | .runsettings TRX logger and ResultsDirectory; Directory.Build.props TrxReport 2.5.2 |
| chore/test-project-settings | 87a611e9 | 1 commit on top of test/mtp-reporting | nuget.config, Directory.Build.props (Testing.Platform and TrxReport to 2.5.3, VSTestBridge), 39 test .csproj line-ending/BOM churn |

None of these five is in origin/wip. All five heads exist on origin.

Local-only, not in scope: `test/run-wip` (local branch and worktree F:\vf-worktrees\run-tests, same commit as test/mtp-reporting). It is not on origin.
Out of scope: dev, main, wip, and the dependabot branches on origin.

## Units

### U0 Review
- Done: Sonnet review (read-only). Findings folded in.

### U1 Integration worktree
- Check submodules first: `git submodule status` in the superproject. The `docs` submodule is in scope if its pointer changes. For each submodule whose commit this integration changes, branch it with the same name from the same base, and push the submodule branch before the superproject.
- `git fetch origin --prune`; create worktree `F:\vf-worktrees\integration` on new branch `chore/integration-all` from origin/wip.
- Done when: branch exists, clean, at origin/wip.

### U3 Decision (before U2)
- Ask the user, with AskUserQuestion, which of these to take for chore/test-project-settings:
  (a) skip the whole branch; (b) take the whole branch (includes the 2.5.3 version bump and the 39-file line-ending/BOM churn); (c) take only the Directory.Build.props version change.
- Option (c) is a partial-file change, not a plain merge. If chosen, apply it by hand as its own commit and say so in the commit body.
- No default. U2 does not start until the user answers.

### U2 Merge in order (with --no-ff)
1. origin/fix/session-renewal
2. origin/chore/open-items-plan
3. origin/chore/gitignore-agent-tooling
4. origin/test/mtp-reporting
5. origin/chore/test-project-settings, only if U3 chose (b). Under (b), merge it after step 4; it fast-merges on top of test/mtp-reporting with no conflict. Under (c), skip this step and apply the props change by hand.
- Done when: merges complete with no conflict markers. Four merge commits under (a) or (c); five under (b).

### U4 Verify
- Full gate, via the `full-gate` skill: build the solution (`VideoForensics.sln`), run the full `dotnet test` (note the TRX output directory `TestResults` at the solution level, from the .runsettings change), and `npm run check` in `src/client/web/VideoForensics.WebApp`.
- Run after the final merge set, since the props and runsettings changes affect test discovery.
- Done when: build has 0 errors; test totals are recorded; npm check passes. Failures are logged, not hidden.

### U5 Push
- `git push -u origin chore/integration-all`. Do not push wip. Report the commits, test totals, and anything skipped.
- Done when: branch on origin, results reported.

### U6 Merge to wip (needs approval)
- Only after the user reviews U5 and approves. Done in a worktree from origin/wip, not in the main checkout (where local wip is checked out). Fast-forward wip from the integration branch, then one push question.

## Commit rules
- Merge and commit messages carry no Claude attribution. Write messages to a file and use `git commit -F <file>`, or pass them with `-m`.

## Risks
- Git commit and push from PowerShell have been denied by the auto-mode classifier. Haiku agents may hit the same denial. If a commit is denied, the agent stops and reports; it does not work around it.
- The main checkout (F:\VideoForensics) stays untouched.

## Results
(filled as units finish)
