# Hudson dashboard: review findings and fix list

Handoff for Claude Code (PyCharm, sandbox repo). Drawn from the BCH site and dashboard review of 19 November 2026. The full review is the Claude Doc "BCH site and dashboard review, November 2026": https://claude.ai/artifact/JFchZfN87XdyMQnjZtjY2A

## Why this comes first

The website's central promise is "Every report is checked against the books. If the check fails, the report waits until it's fixed." The dashboard currently undercuts that promise in ways a careful prospect, such as a construction controller, would catch in a demo. These fixes matter more than any website wording.

## Scope and ground rules

- **What was tested:** the published demo artifact "Hudson dashboard demo (sample)" (https://claude.ai/artifact/4ouQtiLg1K99sGsLpiY2th). It's a self-contained export of the sandbox's new dashboard (`estimating_app.py` + `view_static/estimating/`) and Hudson's weekly report. Fixes belong in the sandbox. Re-export the demo afterwards.
- **How it was tested:** an agent playing Hudson's operations manager worked through six Monday-morning tasks using the demo's code and data. It did not watch the page render. Findings marked **Confirmed** were checked against the demo's code by hand. Those marked **Reported** come from the tester's reading of the code: reproduce them before fixing.
- **Follow the repo's `CLAUDE.md`:** Learning style; a short-lived branch per change (`fix/…` or `learn/…`), never on main; Dexter commits, pushes and merges in GitHub Desktop; check `git status` and the diff first; fake data only; services bound to 127.0.0.1.
- **Design decisions are Dexter's.** Items marked **Decision** have more than one reasonable answer. Lay out the options in terms of his pipeline and let him choose before writing code.
- **House style** (`platform/bch_core/brand.py`): citron #C5D63A for current, chosen or positive (never negatives); mauve #9A7590 (light #D3B9CC, tint #EFE6EC) for what needs attention; deep red #741626 only on negative figures; slightly rounded corners in the client dashboard only. Mauve for "needs attention" is therefore correct. Where mauve is used, the problem is size, not colour.

## Priority 1: fix before the demo is shown to anyone

### 1.1 Internal wording leaked into the client's weekly report. Confirmed
- The report contains "the sandbox's sample data ends on 1 November" and "Targets are agreed with the client and set in its client.yaml."
- **Find it:** search the report templates and generator for `sandbox` and `client.yaml`.
- **Done when:** no client-facing text mentions the sandbox, file names, config keys or internal tools. Consider a test that scans rendered reports for a list of banned internal words.

### 1.2 "Reconciled to the books" shows whether or not anything reconciled. Confirmed
- In the Home page template the badge is a fixed string (`${ic('check', 14)} Reconciled to the books`) with no condition. Meanwhile the data marks costs as not taken from the books, and 41 records are open in "Records to tidy".
- **Decision:** what does "reconciled" mean for a weekly figure? Which checks must pass (for example billings against QuickBooks, costs against the ledger, record issues below a threshold), and what shows when one fails? Options: a pass/fail badge driven by the run's check results; a badge naming what was checked ("Billings checked against QuickBooks · costs not yet"); or no badge, with a link to the Data page's check list.
- **Done when:** the badge reflects real check results from the run, and clicking it shows what was checked and when.

### 1.3 Three dates disagree. Confirmed
- Figures run to 1 Nov (`O.info.as_of`; Home says "Week to 1 Nov 2026"). The report says "issued 16 Nov". The note card says "Read from your systems Mon 16 Nov, 06:00". Per the tester, the QuickBooks card on the Data page says "Last read Tue 17 Nov, 10:20", a day after the report was issued.
- The read time comes from `READ = CONF.read || 'Mon 16 Nov, 06:00'`: a configured or default string, not each system's real read time. Per the tester, every source card shows the same "Last read" value.
- **Decision:** one date model. For example, "Figures to [period end] · read [actual run time]" on every screen, with each source's own last successful read on the Data page.
- **Done when:** every date shown comes from run metadata, the header states it once on every screen, and the report's issue date can't come before the read it's built on.

### 1.4 The same figure differs between report and dashboard. Reported (reproduce first)
| Figure | Weekly report | Dashboard |
| --- | --- | --- |
| Murals overrun on completed work | 24.1% (also in "This week's note") | 21.9% on Man-day estimates |
| Overrun, all completed work | +12.4% (targets table) | +6.6% or +6.9% |
| Problem count | 16 problems | 41 records to tidy |
| Union Hotel advice | likely a CRM typo | "check for a missing change order" (Billing) |
| Bid shortfall, Liberty Bank | 48.1 man-days / $32,700, against the expected value | "Try another figure" box: 30.2 man-days / $20,536, against the low end of the range |

- Likely causes to check: different denominators (hours against man-days, a different job set), report data frozen at issue while the dashboard recomputes, and different reference points (expected against low).
- **Decision:** whether report and dashboard must always show identical figures for the same period (recommended for trust), or may differ when each screen labels what it measures.
- **Done when:** each shared metric is computed once (for example one dbt model or one Python function) and both surfaces read it, or every difference is labelled on screen. A test asserting report and dashboard values match for the same run would hold this in place.

### 1.5 No navigation on phones. Confirmed
- `@media (max-width:860px){aside.side{display:none}}` hides the sidebar and nothing replaces it. A prospect opening the demo link on a phone can't leave Home.
- **Done when:** a menu (top bar or menu button) works below 860 px, and the comparison layouts stack instead of overflowing. Per the tester, they have fixed minimum widths of 390 px and 230 + 170 px.

## Priority 2: make it usable without Dexter in the room

### 2.1 Attention items are spread across five places. Reported
- They are Home "Money to act on", Home "Records to tidy", "This week's note", the weekly report's "For attention" (8 items), and each job page's "N things to look at". Only the report says what to do. Home links open Billing or Data unfiltered. Nothing shows an owner or lets an item be marked done.
- "Money to act on" totals "$2,867,769 across 4 kinds of action", which adds contract value to unpaid invoices: not a meaningful sum.
- **Decision:** one "This week" list on Home (the report's For attention items, plus record issues), each with why it's flagged, what to do, who owns it, and a link filtered to that item? Whether to add "mark done" depends on whether the dashboard should hold any state.
- Working guide standard: "readable without Dexter present: numbers first, plain wording, why something is flagged and what to do."

### 2.2 Active jobs can't answer "are we over our man-days?" Reported
- "Man-day estimates" covers only finished jobs and open bids. Active jobs show "Crew hours used" (for example "168 of 201 h") and "Labor against estimate" (for example "84% used"). Units switch from hours to man-days between screens.
- There's no percent complete or planned finish next to hours used, so a job heading over can't be seen until it's past 100%.
- Clicking the "Labor against estimate" header doesn't reorder active jobs, because the sort uses finished-job overruns only.
- Home's "Active jobs" is sorted by "Needs attention", which actually ranks by amount unbilled.
- **Decision:** one unit across the dashboard (man-days, matching how decorative arts firms estimate, or hours), and how to flag jobs heading over: pace against percent complete, or the existing "At the last four weeks' pace … run out around [date]" line set against a planned finish.

### 2.3 Unpaid invoices. Reported
- Ages stop at 1 Nov, and there's no due date or days past due. The download has blank "Due" and "Days past due" columns. There's no total per customer: four customers have two invoices each.
- Home's line "Majestic Church Foundation $262k, oldest 67 days; Capitol Museum Trust $151k, oldest 64 days" reads as the oldest debts but lists the largest balances.

### 2.4 One name for each thing. Reported (partly confirmed)
- Job, project and library are used for the same thing. The report sends readers to "the Projects tab" (in the report data); the dashboard calls them Jobs. Report tables put the number under "Job" and the name under "Project".
- "Done, not billed", "Done but not billed", "Ready to bill" and "Behind progress" are used for one idea.
- "operations CRM", "operations list" and "Operations job list" name one source.
- "Labor" and "Crew labour" both appear ("Crew labour" 235 times in the export). **Decision:** American or British spelling. The website uses "labor".
- Plain words are needed for "Middle half", "Per 100", "Feeds", "Next read", "Library range", "facts compared", "parties" and "close-out lines".
- **Done when:** a short glossary (a dict in the shared code, or a doc in `docs/`) fixes one term per idea, and the dashboard and report both use it.

### 2.5 Numbers. Reported
- The same figure is rounded differently on different screens ($207,827 against $208k; $1,518,217 against $1.52m). Pick one rounding rule per figure type.
- "▼ 409 on the week before" (crew hours) doesn't say whether that's good or bad.
- A bid marked "Within range" still shows a value in the "Short" column (Riverside Courthouse, 3.6).
- The Union Hotel $348,120 amount appears three times in "Records to tidy". Group duplicate issues by record.
- "0 of 8 targets met" has no context (what the targets are, or the trend).
- Bad tiles are marked only by a 7 px mauve dot. Keep mauve, but make it larger or add a word.

### 2.6 Navigation and traceability. Reported
- "Data" sits under the "Estimating" heading in the sidebar but has nothing to do with estimating. "Estimating" looks clickable but isn't.
- Headline tiles don't link anywhere. Each should open its detail and source. Working guide standard: "Every figure traceable to its source."
- "Files matched to a project 100%" is a fixed value in the code (`kpi('check', 'Files matched to a project', '100%', …)`, **Confirmed**). Compute it.
- Same-name jobs can only be told apart by number: St. Mary Courthouse P-2424/P-2418, Liberty Hotel P-2427/P-2417, Riverside Courthouse P-2439/P-2425. Consider showing the year or location.
- The weekly report sits in a fixed 1,100 px-tall frame, so it scrolls inside a scrolling page.

## Priority 3: accessibility. Reported, from code
- The citron focus outline on inputs is about 1.6:1 against white, which is hard to see. Use ink or a darker outline on light grounds.
- The light bars (#C9CBCC, #D3B9CC) fall below 3:1 against white.
- Sortable column headers aren't reachable by keyboard (no tabindex, not buttons). Make them buttons.
- Chart labels are 10.5 px inside scaling SVGs, so they shrink to about 6 px on a phone. Charts are labelled only "Chart" for screen readers.
- Much of the text is 12–12.5 px.

## What works, and should stay
- Billing and cash: "Unpaid invoices · Oldest first" answers "what are we owed" in two steps.
- Man-day estimates: "Closest finished work" for an open bid ("Man-days: est. → actual", "Per 100") answers "how did similar jobs go" in two steps.
- The weekly report's For attention items are plain and mostly end with an action ("Confirm with the project manager and raise the invoice").
- The Data page lists each source, each line records its source, and "Records to tidy" exists at all.

## Suggested order of work (for Dexter to confirm)
1. `fix/report-internal-text`: item 1.1. Small, no decision needed, a good warm-up.
2. Decide the date model and what "reconciled" means (1.2, 1.3), then one branch to drive both from run metadata and check results.
3. Trace the five mismatched figures to their sources (1.4). This is a good "trace a figure" exercise from the learning plan, and leads into one shared metric per figure.
4. Phone menu and stacking (1.5).
5. Priority 2, starting with the glossary (2.4), since it touches everything else.
6. Re-export the Hudson demo artifact and refresh the EverGreene dossier screenshots, which are already on the "offered" list.
