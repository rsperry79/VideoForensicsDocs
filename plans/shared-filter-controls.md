# Shared filter controls

Status: in progress

## Context
Filters on several pages (Accounts, Manual Sync, Notices, Live View, Review) are copies: each page keeps its own provider, account and camera state and its own storage key. A choice on one page does not reach another. Rule: a filter that appears on more than one page is one reusable control with one shared backing, and the control is listed in `.claude/reference/ui-shared-controls.md`.

## Decisions (from the user)
- Each sub-filter becomes a reusable control. All pages showing that control use the same control and the same backing.
- Controls are listed in the reusable-controls index.

## Units
1. **Provider and account state.** Scoped `ProviderAccountFilterState` (Providers: string[]; Account: string?; OnChange) with one storage key, loaded and saved through `RightPanelFilterStore`. Test first: load restores providers and account; set saves them; a second page reading the state sees the same choice.
2. **Provider and account control.** `Components/Filters/ProviderAccountFilter.razor`: provider multi-select and account dropdown over the shared state. Options passed in as parameters. Test first: changing the provider clears an account that no longer matches.
3. **Adopt on Accounts, Manual Sync, Notices.** Remove each page's own provider and account state. Notices' provider filter is single-select today; the shared control is multi-select, so Notices filtering must accept a set. Test first per page.
4. **Camera control.** Keep `ReviewCameraSelector` as the camera control on the shared `ReviewPageState` backing (already shared by Review, Timeline, Events and Live View). Index it.
5. **Index.** Add both controls to `.claude/reference/ui-shared-controls.md`.

## Open items
- Notices provider filter changes from single to multi-select with this plan. Confirm this is acceptable in unit 3 if it changes what Notices shows.
- Shared state means a choice on Accounts also applies on Notices. This is the intent of "same backing".

## Done when
- No page keeps its own provider, account or camera filter state.
- Every page showing a filter uses the shared control and backing.
- The index lists both controls.
