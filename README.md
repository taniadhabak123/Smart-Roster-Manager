# Smart Roster Manager

A browser-based tool that lets an operations manager plan, generate, validate and change monthly rosters for many brands, processes and pods. It is a single static page (`index.html`, no build step, no backend, no environment variables, so no `.env.example` is needed).

**Live app:** `(https://taniadhabak123.github.io/Smart-Roster-Manager/)`
**Built by:** Tania Alimuddin Dhabak. The data model, rules and trade-offs are documented below.

---

## Try it in two minutes

1. Open the app. A roster for **Nov 2026** is already generated for 35 employees. The KPI strip shows 0 mandatory conflicts and 100% coverage.
2. **Roster tab:** switch between Company / Pod / Brand / Process views. Under the employee grid, the coverage rows show staff covering each process each day (red = short, `b` = a backup was deployed). Click any cell to edit it. Edits are pinned and survive regeneration.
3. **Employee tab:** one person's calendar, assignments and issues.
4. **Changes tab → "Demo" button:** adds a leave for Aarav Mehta, the only person on Kaya Naturals, on 17 Nov. The brand loses coverage, and the KPI and Conflicts count update.
5. **Conflicts tab → Suggest fix:** options such as adding a free floater as backup, or reconsidering the leave. Apply one, and the gap closes.
6. **Publish** is blocked while mandatory conflicts remain.
7. **Setup tab:** add or edit employees, pods, brands, processes, calendars, minimum staffing, assignments and holidays. No code changes are needed.

---

## Sample data

| | |
|---|---|
| Employees | 35 (including 3 floaters who act as backups) |
| Pods | 4 (each with a lead) |
| Brands | 24: 19 have one dedicated employee (about 79%), 5 have two or three |
| Processes | 46 (CS, ACC, RET, NDR) |
| Calendars | `365 days`, `Daily, PH off`, `Sun & PH off` |

- The multi-employee brands cover both cases: employees on separate processes, and overlapping or shared processes.
- Examples: Shoptide has two CS employees (CS needs 2 people on weekends) plus an ACC/NDR owner. Velvetique and Heliox deliberately overlap staff.
- Single-resource brands on 365-day or daily calendars have floater backups. Single-resource brands that are closed on Sundays use Sunday as the weekly off.
- Generating Nov 2026 to Mar 2027 with the sample data gives 0 mandatory conflicts. A few preference warnings remain, for example a preferred weekly-off day that couldn't be honoured.

---

## Assumptions (all configurable in Setup unless noted)

1. The roster is per day, not per shift. Coverage means "enough people working on that day".
2. Each employee gets one weekly off in every Monday–Sunday week that has at least 4 days in the month and ends on a Sunday. Nobody works more than 6 days in a row. Month-end partial weeks are not forced. The stretch rule catches them instead.
3. For a Sunday-closed process, Sunday counts as the weekly off. A public holiday on a Sunday also counts as that week's weekly off. Other public holidays are separate from weekly offs.
4. An employee is off on a day automatically if every one of their primary processes is closed that day.
5. If a 365-day process requires work on a public holiday, the person works it. The system raises an info item ("comp-off due"). Comp-offs are recorded manually and not auto-credited.
6. Each process has a minimum staff count and a separate minimum for Sat/Sun, which allows different requirements on specific days.
7. Preferences are soft. Each employee can have one preferred weekly-off weekday.
8. Leaves and comp-offs entered by the manager are treated as approved.
9. Pod leads are normal employees who also have a lead role in their pod.

---

## How it works

**Data model.** Everything is data, not code.
- `pods` → `employees` (pod, preferred weekly off)
- `brands` → `processes` (calendar, min staff, weekend min)
- `assignments`: a many-to-many link between employees and processes, with a role of `primary` or `backup`
- `leaves`: leave or comp-off by date
- `holidays`: a list of dates
- `roster[month][employee][day]`: a code of WO / SO / PH / L / C. A working day is the absence of a code.
- `manual`: pinned overrides

**Generation** (deterministic, no randomness):
1. Mark closure days (SO/PH), then leaves and comp-offs, then manual pins.
2. Process employees tightest-first. The tightest are those whose process has the fewest co-workers, so single-resource brands are placed before anyone else.
3. For each week, score every candidate weekly-off day. The cost is 1000 per new coverage gap (counting backup deployment), plus a penalty for missing the preferred day, plus spread across the pod, plus a penalty for a run of more than 6 days.
4. Pick the cheapest day.
5. A final pass inserts a rest day wherever a 7-day run remains.

**Validation** is a separate function (`an()`) that recomputes everything from the roster. Generation and validation share the same coverage logic, so the generator can't "agree" with itself while the validator disagrees.
- Mandatory: coverage gaps, a process with no primary employee, 7+ days in a row, a week with no off.
- Preference: weekly off not on the preferred day.
- Info: working on a public holiday.

**Backups.** A backup covers a shortfall only if they are working, have spare capacity in their own primary processes, and are serving at most one brand that day. This stops the system from counting one person as covering several brands at once.

---

## Known limitations

- Day-level scheduling only. There are no shifts or time slots, so "incompatible shifts" is only approximated by the one-brand-per-day backup rule.
- The generator is greedy, not globally optimal. It is verified on the sample months but can leave gaps on very tight data, and those gaps are flagged.
- State is stored in the browser (`localStorage`), so there is no multi-user sync.
- CSV export shows text to copy, not a downloaded file.
- Each month starts fresh: the previous month's trailing stretch is not carried over.
