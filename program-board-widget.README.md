# Program Board widget

`program-board-widget.html` is a single-file Rally Custom HTML Widget (no build step) that draws a SAFe Program Board for the Release selected in a **Release-scoped custom view**.

## Deploy

1. Create (or open) a custom view whose view filter is **Release**.
2. Add a **Custom HTML Widget** and paste the full contents of `program-board-widget.html` into its *HTML Source* field.
3. Pick the PI Release in the view filter. The project scope of the page (including *scope down* to child projects) is used for Features, stories, iterations and teams.
4. In Edit Mode, adjust the settings if needed (below).

## What it shows

- **Columns:** the iterations that fall inside the Release's start/end dates, de-duplicated across projects by start date (so each team's copy of "Sprint 1" is one column), plus a **Not yet scheduled** column.
- **Rows:** the page's project (ART-level Features), then each child project. Projects outside the page's project scope tree get **no row** unless one of their Features is needed to show an open dependency (it is the end of a dependency string, or carries a dependency chip); then that project gets a labelled row showing only those Features. The notes under the board say how many outside-scope Features were left off.
- **Placement:** a Feature is placed in the iteration of its **last scheduled user story**. Dashed cards still have unscheduled stories. Features with no scheduled story go in *Not yet scheduled*.
- **Milestones & Events row:** Milestones whose target date falls in the Release window, in the column containing that date.
- **Dependencies:** both direct Feature predecessors and *implied* ones (a story whose predecessor story is in a different Feature). Only **open** predecessors are drawn, and the arrowhead is always on the successor. A predecessor that is on the board gets a string (and its card turns red); one that is not on the board shows as a red chip on the successor card.
- **Links:** every formatted ID uses `applyFormattedIDTemplate` (the repo's standard Rally link).

## Settings (Edit Mode)

| Setting | Default | Meaning |
|---|---|---|
| IP iteration | Last iteration | Which column is labelled IP (or none). |
| Team rows with no Features | Show (dimmed) | Hide to shorten the board. |
| Feature State values that count as done | `Done` | Comma-separated, case-insensitive. |
| Story ScheduleState values that count as done | `Accepted` | Comma-separated, case-insensitive. |

## Decisions and deviations to review

- **"Open" means not done.** The original request was to highlight predecessors that are "not done or not yet scheduled". Here a predecessor is open when its State (Feature) or ScheduleState (story) is not in the done list. A *done* predecessor that has no story scheduled in this PI is **not** highlighted (it was finished earlier). Unscheduled, not-done predecessors are highlighted, and the tooltip says "has no scheduled stories". Tell me if you want done-but-unscheduled predecessors highlighted too.
- **Archived predecessors are open** (never completed), unless you add `Archived` to the done list.
- **Red cards** now mean "source of an open dependency string", not just "has successors".
- **Milestones are loaded for the whole workspace** by target date; they are not filtered to the page's project (Milestones can target several projects).
- **No Rally SDK, no Highcharts, no Ext JS**; only functions copied from `examples/utility.js` (unchanged apart from removing `export`, and the logging line in `loadWsapiData`) plus a few lines of vanilla JS.
- **Printing:** `window.print()` is silently ignored inside the widget's sandboxed iframe, so the button is **Open printable view**: it opens a static copy of the board in a new tab (landscape, scaled to fit) and then tries to open the print dialog there. If the dialog does not open, press Ctrl/Cmd+P in that tab. The copy is a snapshot; it does not refresh. Settings use plain `<select>`/`<input>` controls instead of `createStyledDropdown`, to avoid pulling in `ui-utilities.css`.
- **Iteration columns come from dates, not names.** If a project has iterations that start on different days than the rest, they become their own column.

## What was tested, and what was not

Tested offline, in a JavaScript engine with stubbed DOM and synthetic data: column de-duplication, IP labelling, last-scheduled-story placement, partial-scheduling counts, done/open filtering for Feature-level and story-level dependencies, chips for off-board predecessors, same-Feature story links being ignored, team rows and milestone placement.

**Not tested against live Rally** (no widget runtime available while building). Check these first if something looks wrong:

1. **Collection hydration.** The widget fetches `Predecessors` with `collectionsize` and expects the items inline (`Results`). If they are not, it falls back to calling each collection's `_ref`. If dependencies look missing, inspect the network requests for the Feature and story queries.
2. **`loadWsapiData` swallows API errors.** In the repo's utility, the error branch calls `Promise.reject(...)` without `return`, so a failed query resolves to `undefined`. The widget treats that as "no rows" instead of showing an error, so an empty board can mean a failed call. Check the browser console / network tab.
3. **Query syntax:** `Feature.Release.Name = "…"` on stories, date comparisons on `StartDate` / `EndDate` / `TargetDate`, and `ObjectID = n` OR-chains (40 per request). All are valid WSAPI patterns but were not run from the widget.
4. **Project list.** The widget loads all projects in the workspace with their `Parent` and works out the page project's subtree itself (descendants only when the page's *scope down* is on), because a `project` query does not reliably honor the scope flags. Confirm the team lanes match the project's real children, including nested teams.
5. **Performance.** Everything loads in the browser: Features, all stories in the Release, then dependency lookups. A Release with thousands of stories may take several seconds; a status line shows progress.
6. **Layout inside the widget iframe:** sticky headers, the SVG string overlay (redrawn on window resize only), and Print.
7. **Detail links** open in a new tab as `<page URL>?detail=<ref>` (the pattern the endorsed *recent-activity* widget uses). Click one Feature, a chip and a milestone to confirm each opens its detail view.
