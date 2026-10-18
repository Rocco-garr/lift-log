# Lift Log

Rocco's workout logger: a single-file PWA served by GitHub Pages from `main`, installable to the iPhone home screen, works offline. One program, one goal: a 185 lb bench. Each finished workout is backed up to a private GitHub repo, where Claude reads it for the weekly review.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app: workout templates, set logging, rest timer, How/Swap buttons, Progress tab, GitHub backup. |
| `plan.json` | The program (below). Fetched network-first, so edits show up the next time the phone is online. |
| `sw.js` | Service worker; caches the app shell for offline use. |
| `manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` | PWA install metadata and icons. |
| `.claude/skills/weekly-review/SKILL.md` | The Sunday review procedure Claude follows (`/weekly-review`). |

**Bump `VERSION` in `sw.js` whenever `index.html` or `sw.js` changes**, or phones never get the "New version ready" banner. `plan.json` edits need no bump.

## plan.json

| Field | What it holds |
|---|---|
| `updated` | Date of the last edit (YYYY-MM-DD). |
| `goal`, `startMax` | Target bench (185) and the estimated max the plan started from (170). |
| `startDate` | A Monday. Plan week N = Monday–Sunday blocks counted from here, 1-based; before it is week 0. |
| `note` | Standing coach note shown in the app. |
| `review` | `null`, or the last weekly review: `{week, date, summary, changes[], next}`, written for the lifter. |
| `graphs` | Exercise names charted on the Progress tab. |
| `benchWeeks` | One entry per plan week: `{week, label, day1:{sets,reps,weight,note?}, day3:{…}, day5:{…}}`. Template exercises with `"bench": "day1"` etc. take their sets/reps/weight from here. |
| `templates` | 5 workouts (`day1`…`day5`; `weekday` 1=Mon, 2=Tue, 4=Thu, 5=Fri, 6=Sat), each with exercises `{name, sets, reps, weight, rest, effort, note?}`. `reps` is a range (`"8-12"`) or fixed (`"10"`, `"10/leg"`); `weight: null` means find it. |
| `library` | One entry per exercise, keyed by exact name: `{m` muscle, `e` equipment, `inc` lb to add when every set hits the top of the range (dumbbells: per hand), `how` form cues, `swaps` replacement names, `main?}`. Powers the in-app How and Swap buttons. Every template exercise and swap target must be a key here. |

Progression: accessories use double progression (every set at the top of the range → add `inc`; set 1 below the bottom → take `inc` off). Big lifts stop 1–2 reps short; machines and cables take the last set to failure. Bench never goes to failure and follows `benchWeeks` exactly. Estimated 1RM everywhere is Epley, `w × (1 + r/30)`, for 1–15 reps.

## Data repo (private): `Rocco-garr/lift-log-data`

The app writes one file per finished workout to `sessions/<id>.json` using a fine-grained GitHub token (Settings tab; scoped to that repo only; stored on the phone). Workouts also stay on the phone, so a missing token just means "not backed up". Session format:

```
id, templateId (day1–day5), templateName, status, date (YYYY-MM-DD), startedAt/finishedAt (ISO),
week (plan week when logged, 0 = before the plan), feel ("Easy"|"Good"|"Hard"|"Beat up", optional),
notes (free text for Claude, optional), prs ([{name, v, prev, set}] new estimated maxes, optional),
exercises[]: {name, swappedFrom? (template exercise it replaced), note, target {sets, reps, weight, rest}, sets [{w, r}]}   (w 0 = bodyweight)
```

## Weekly review

Every Sunday evening (America/Chicago) a scheduled task in Claude Code runs `/weekly-review` in a fresh cloud session; it can also be run by hand. The skill pulls the week's sessions from the data repo, judges attendance, the bench against `benchWeeks`, and every accessory against its rep range, then edits `plan.json` on `main` (next week's bench weights, accessory weights, stalled-exercise swaps, `review`, `updated`) and pushes. The app picks up the new plan the next time it is online. Rules live in `.claude/skills/weekly-review/SKILL.md`.

## Swaps

The Swap button replaces an exercise with one of its `library[...].swaps`. The choice is stored on the phone (localStorage), so it survives app updates but not a device change; Settings has an Undo button. Each logged exercise records `swappedFrom` so the review can see it, and a swap used two weeks in a row gets written into the template by the review.
