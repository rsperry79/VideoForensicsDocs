| Path | Kind | Title | Summary | Code | Semantic |
| --- | --- | --- | --- | --- | --- |
| .claude/rules/admin-setup.md | file | First-run admin setup | - Empty database: `AuthGate.razor` redirects to `/setup` when `IOperatorRepository.IsEmptyAsync()`. The installer pic... | code:.claude/rules/admin-setup.md | sem:pending |
| .claude/rules/ci.md | file | CI, build, test, and release (required) | Before editing any workflow, read `.claude/skills/ci-release/SKILL.md`. Its rules are required: | code:.claude/rules/ci.md | sem:pending |
| .claude/rules/client-libraries.md | file | Client-side libraries | - Before writing any Syncfusion code, query the Syncfusion MCP (`sf-blazor-mcp` for Blazor, `sf-maui-mcp` for MAUI) f... | code:.claude/rules/client-libraries.md | sem:pending |
| .claude/rules/client-live-data.md | file | Live data in client apps | - Data a client app displays must come from an observable update source: an `IObservable<T>` contract in `Contracts/`... | code:.claude/rules/client-live-data.md | sem:pending |
| .claude/rules/client-split.md | file | Client/server split | Applies to MauiApp, Ui.Shared, Client.MCP, and any MCP bridge. Not WebApp (it is the server). | code:.claude/rules/client-split.md | sem:pending |
| .claude/rules/csharp.md | file | C# conventions | - `.csproj`: `Microsoft.CodeAnalysis` 5.9.0. Signed projects use `VideoForensics.snk` (root). | code:.claude/rules/csharp.md | sem:pending |
| .claude/rules/providers.md | file | Providers and data | - Before adding a provider, use the `provider-onboarding` skill. | code:.claude/rules/providers.md | sem:pending |
| .claude/rules/syncfusion.md | file | Syncfusion license | - One resolver: `SyncfusionLicenseRegistrar` (`VideoForensics.Client.Core`). No second license reader. | code:.claude/rules/syncfusion.md | sem:pending |
| .claude/rules/tests.md | file | Tests | - Location: `<Project>/tests/<Feature>Tests.cs`, under the implementation's solution folder. | code:.claude/rules/tests.md | sem:pending |
| .claude/rules/ui-shared.md | file | Ui.Shared | - Every user-visible string (labels, buttons, headings, placeholders, tooltips, errors, aria labels, validation/alert... | code:.claude/rules/ui-shared.md | sem:pending |
| CLAUDE.md | file | VideoForensics | - Subagents are `model: "haiku"`, except the standing Sonnet plan review (global CLAUDE.md) and a model the user appr... | code:CLAUDE.md | sem:pending |
| artifacts/ | folder | artifacts |  | code:artifacts | sem:pending |
| deploy/ | folder | deploy |  | code:deploy | sem:pending |
| docs/ | folder | docs |  | code:docs | sem:pending |
| docs/JAMMING_ANALYSIS_GUIDE.md | file | Jamming Analysis Guide | VideoForensics MCP server includes RF jamming detection tools for identifying signal interference in Ring device reco... | code:docs/JAMMING_ANALYSIS_GUIDE.md | sem:pending |
| docs/README.md | file | VideoForensicsDocs | Documentation for VideoForensics | code:docs/README.md | sem:pending |
| plans/ | folder | plans |  | code:plans | sem:pending |
| src/ | folder | src |  | code:src | sem:pending |
| tools/ | folder | tools |  | code:tools | sem:pending |
