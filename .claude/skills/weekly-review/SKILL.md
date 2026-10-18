---
name: weekly-review
description: "Weekly review of Rocco's lift-log workouts: read the week's sessions from the private data repo, evaluate every lift, adjust next week's plan.json and push it."
---

# Weekly review

Rocco logs workouts in **Lift Log**, a single-file PWA in `Rocco-garr/lift-log` (GitHub Pages serves `main`). The program is `plan.json`; every finished workout is backed up to the private repo `Rocco-garr/lift-log-data` as `sessions/<id>.json`. This skill reads the week that just ended, applies the rules below, edits `plan.json` and pushes it to `main`. The app fetches `plan.json` network-first, so the new plan shows up the next time the phone is online.

Runs every Sunday evening (America/Chicago) from a scheduled task in a fresh cloud session, or by hand with `/weekly-review` (`/weekly-review 3` reviews week 3). Paths below assume the cloud layout: `/home/user/lift-log` and `/home/user/lift-log-data`.

## Vocabulary

| Term | Meaning |
|---|---|
| Plan week | Monday–Sunday blocks counted from `plan.startDate` (2026-10-19, a Monday). Week 1 = Oct 19–25. Sessions logged before the start carry `week: 0`. |
| `benchWeeks[]` | The bench progression, one entry per plan week: `{week, label, day1:{sets,reps,weight,note?}, day3:{…}, day5:{…}}`. Template exercises with `"bench":"day1"` etc. take their sets/reps/weight from here. |
| `templates[]` | 5 workouts, ids `day1`…`day5`, `weekday` 1=Mon, 2=Tue, 4=Thu, 5=Fri, 6=Sat. Each exercise: `{name, sets, reps, weight, rest, effort, note?}`. `reps` is `"8-12"`, `"10"`, `"8-12/leg"` or `"10/leg"`. `weight: null` = "find your weight". |
| `library{}` | One entry per exercise, keyed by exact name: `{m, e, inc, how, swaps[], main?}`. `inc` = lb to add when progressing (dumbbells: per hand). `swaps` = acceptable replacements. Powers the app's How and Swap buttons. |
| Double progression | Work inside the rep range; when every set hits the top, add `inc`; when set 1 can't reach the bottom, take `inc` off. Big lifts stop 1–2 reps short of failure; machines/cables take the last set to failure. Bench never goes to failure and follows `benchWeeks` exactly. |
| e1RM | Estimated 1-rep max, Epley: `w × (1 + r/30)`, only for 1–15 reps. |
| Session | `{id, templateId, date, week, feel?, notes?, prs?[], exercises[]}`; each exercise `{name, swappedFrom?, target{sets,reps,weight,rest}, sets[{w,r}]}`. `w: 0` = bodyweight. `feel` is Easy / Good / Hard / Beat up. `notes` is text the lifter typed for Claude. `prs` = `{name, v, prev, set}`, new estimated maxes computed at finish. |

## 1. Get the data

1. Data repo (private). In a cloud session call the `add_repo` tool (owner `Rocco-garr`, repo `lift-log-data`, access `read`), then:
   `[ -d /home/user/lift-log-data ] && git -C /home/user/lift-log-data pull || git clone --depth 1 https://github.com/Rocco-garr/lift-log-data /home/user/lift-log-data`
   If cloning fails, fall back to the GitHub MCP tool `get_file_contents` (owner `Rocco-garr`, repo `lift-log-data`, path `sessions/` to list, then each file). By hand on a machine that already has GitHub access, just clone or pull.
2. App repo, on the branch Pages serves:
   `git -C /home/user/lift-log fetch origin main && git -C /home/user/lift-log checkout main && git -C /home/user/lift-log pull`
3. Read `/home/user/lift-log/plan.json` in full. Never edit or delete anything in the data repo.

## 2. Which week

Review date = today in **America/Chicago** (the task fires Sunday evening, which is already Monday in UTC; never use UTC). `SUN` = that date if it is a Sunday, otherwise the most recent Sunday before it. Then:

`week = floor((SUN − startDate) / 7 days) + 1`, covering Monday `SUN − 6` through Sunday `SUN`.

Worked example: start 2026-10-19, review on Sunday 2026-10-25 → 6 days / 7 = 0, + 1 = **week 1** (Oct 19–25). Review on 2026-11-01 → 13 / 7 = 1, + 1 = week 2. This prints it (pass a date to override):

```bash
node -e '
const p=require("/home/user/lift-log/plan.json");
const today=process.argv[1]||new Date().toLocaleDateString("en-CA",{timeZone:"America/Chicago"});
const d=new Date(today+"T00:00:00Z"), sun=new Date(d); sun.setUTCDate(d.getUTCDate()-d.getUTCDay());
const mon=new Date(sun); mon.setUTCDate(sun.getUTCDate()-6);
const week=Math.floor((sun-new Date(p.startDate+"T00:00:00Z"))/6048e5)+1;
console.log(JSON.stringify({today,week,from:mon.toISOString().slice(0,10),to:sun.toISOString().slice(0,10)}));'
```

If the skill was invoked with a week number, review that week instead (from = startDate + 7 × (week − 1)). The week's sessions are the files whose `date` falls in from…to (or whose `week` field equals it). If `plan.review.week` already equals this week, it was reviewed; stop unless asked to redo it. If `week ≤ 0` (plan not started) or the week has no sessions: set `review` to `{week, date, summary: two sentences saying so, changes: [], next: one sentence}`, bump `updated`, change nothing else, and skip to step 6.

## 3. Read the week

Print every session compactly (widen the range to see earlier weeks for the stall, swap and "beat up" checks):

```bash
node -e '
const fs=require("fs"),dir="/home/user/lift-log-data/sessions",from=process.argv[1],to=process.argv[2];
const e1=(w,r)=>r>=1&&r<=15?Math.round(w*(1+r/30)):null;
for(const f of fs.readdirSync(dir).sort()){const s=JSON.parse(fs.readFileSync(dir+"/"+f,"utf8"));
 if(s.date<from||s.date>to)continue;
 console.log(`\n# ${s.date} ${s.templateId} week=${s.week??"?"} feel=${s.feel||"-"} notes=${JSON.stringify(s.notes||"")}`);
 if(s.prs&&s.prs.length)console.log("  PRs: "+s.prs.map(p=>`${p.name} ${p.prev}→${p.v}`).join("; "));
 for(const x of s.exercises){const t=x.target||{};
  console.log(`  ${x.name}${x.swappedFrom?" (swapped from "+x.swappedFrom+")":""} | target ${t.sets}×${t.reps}@${t.weight??"?"} | did ${x.sets.map(z=>z.w+"×"+z.r).join(", ")} | best e1RM ${Math.max(...x.sets.map(z=>e1(z.w,z.r)||0))}`);}}' 2026-10-19 2026-10-25
```

For the reviewed week, write down: attendance (distinct dates), the bench sets on day1/day3/day5 next to `benchWeeks[week]`, every accessory's sets against its template range, `feel`/`notes` per day, `prs`, and any `swappedFrom`. If a day was logged twice, use the most recent session.

## 4. Decide

Work through the tables in order. Every edit you decide on becomes one string in `review.changes`.

### 4a. Attendance (5 planned: Mon, Tue, Thu, Fri, Sat)

| Sessions | Do |
|---|---|
| 4–5 | Normal progression (4b and 4c). |
| 3 | Progress the bench (4b). Accessories trained this week progress normally; anything not trained holds. |
| ≤ 2 | **Repeat the week.** Insert a copy of `benchWeeks[week]` right after it and renumber so `week` stays 1…N consecutive (the test week slides out one week). Hold every accessory weight (still fill in `null` weights that were found). Say so in the summary, and note that workouts not yet synced from the phone aren't counted. |

### 4b. Bench (`Bench Press (Barbell)`, the goal lift)

Compare each logged set on day1/day3/day5 with `benchWeeks[week]`'s target for that day. Judge reps at the weight actually lifted; the hard cap below handles a lighter weight. One set 1 rep short still counts as a hit.

| Verdict | Day 1 evidence | Next week's day1 |
|---|---|---|
| Hit | every set at target reps | as planned |
| Crushed | Hit, plus `feel` Easy or notes saying it was easy, **or** any set's e1RM ≥ next week's day1 weight + 10 | as planned, never above the planned step. You may add 5 lb to next week's day3 and day5 (do it when `feel` is Easy/Good, skip it when Hard). |
| Small miss | one set ≥ 2 reps short, or two sets 1 short | repeat this week's day1 weight (keep next week's sets × reps) |
| Big miss | two or more sets ≥ 2 short, ≥ 4 reps short in total, or a set not done | this week's day1 weight − 5 lb |
| No day1 session | — | as planned (4a decides) |

Then:

- **Hard cap, always:** next week's day1 weight ≤ this week's logged day1 weight + 5 lb (use the last non-deload week when this week was a deload). The test week (`label` starts with "Test") is exempt.
- Later weeks stay as planned; the cap is re-applied at every review, so a miss slows the rest of the plan by itself. If the week before the test ends up more than 10 lb below the test weight, insert one more week like it (+5 lb) before the test instead of changing the goal.
- Day3/day5: two or more sets ≥ 2 short → hold that day's weight next week instead of the planned step. Otherwise as planned.
- **Deload:** `feel` "Beat up" this week *and* last week, or notes mentioning joint pain (shoulder, elbow, wrist, knee) → insert after this week `{label: "Deload 3×5", day1: {sets: 3, reps: 5, weight: 80% of this week's day1 rounded to 5, note: "Deload week. Should feel easy."}, day3: {sets: 3, reps: 6, weight: 70%}, day5: {sets: 3, reps: 8, weight: 70%}}`, renumber, so the rest of the plan shifts out one week. Never two deloads in a row.
- Reviewing a deload week: no bench change unless it still felt Hard or Beat up (then next week's day1 − 5). Reviewing the test week: report the result, don't edit `benchWeeks`, and let `next` say a new plan is needed (that's a conversation with Rocco, not this skill).

### 4c. Accessories (every non-bench template exercise, double progression)

Use the most recent session this week that contains the exercise (by its own name, or as the `swappedFrom` of a replacement). Range `"8-12"` → bottom 8, top 12; `"10"` or `"10/leg"` → bottom = top = 10.

| Observation | Action |
|---|---|
| every set ≥ top of the range, at ≥ the template weight | `weight += library[name].inc` |
| set 1 < bottom of the range | `weight −= inc` |
| anything else, or not trained this week | hold |
| template `weight` is `null` | set it to the weight logged on the last set |
| bodyweight (library `e` "Bodyweight", or logged `w` 0) and every set at top | leave `weight`; in `next` suggest added load (dumbbell between the feet, plate on the back), or more reps if `inc` is 0 |
| assisted (the note says to log the assist as the weight) | inverted: every set at top → assist − inc; set 1 below bottom → assist + inc; at 0 it's bodyweight |
| same weight, no increase possible, 3 consecutive weeks | **stalled.** Main lifts (`library[name].main`): narrow the range one notch (8-12 → 6-10, 10-15 → 8-12). Isolation/machine work: swap for the first `library[name].swaps` entry not already in any template; keep sets/reps/rest/effort, `weight` null unless it was logged before. Say why in `changes`. |
| the lifter swapped it (`swappedFrom`) to the same replacement 2+ weeks in a row | make it permanent: template `name` = replacement, keep sets/reps/rest/effort, `weight` = last logged weight, drop a `note` that only made sense for the old exercise, and replace the name in `graphs` if it was there |

Add nothing to `library` unless a template now names an exercise missing from it; then add `{m, e, inc, how, swaps}` in the same style (2–3 plain form cues, 2–3 swaps that already exist).

### 4d. Never

Change `goal`, `startDate`, `startMax`, template `id`s or `weekday`s, or any `benchWeeks` entry for a week that is already finished. Every template exercise name and every swap target must be an exact `library` key. Only touch `note` if the overall approach changes.

## 5. Write plan.json

1. `updated` = the review date (YYYY-MM-DD).
2. `review` = `{"week": N, "date": "YYYY-MM-DD", "summary": "…", "changes": [...], "next": "…"}`
   - `summary`: 2–3 plain sentences to the lifter. What went well, what changed, honest but encouraging, no jargon. Mention PRs from `prs`.
   - `changes`: one string per plan edit, e.g. `"Lat Pulldown (Cable): 140 → 150 lb"`, `"Week 5 day1: 165 → 160 lb (missed reps on 4×5)"`, `"Pec Fly (Machine): weight set to 90 lb"`, `"Week 3 repeated (only 2 sessions)"`. Empty array if nothing changed.
   - `next`: one sentence on what to focus on next week.
3. Apply the template and `benchWeeks` edits. Small edits: the Edit tool. Structural edits (insert a week, renumber): a node script, which also keeps the 2-space format:

```bash
node -e '
const fs=require("fs"),f="/home/user/lift-log/plan.json",p=JSON.parse(fs.readFileSync(f,"utf8"));
const N=+process.argv[1], i=p.benchWeeks.findIndex(b=>b.week===N);
p.benchWeeks.splice(i+1,0,JSON.parse(JSON.stringify(p.benchWeeks[i])));   // or a deload entry (4b)
p.benchWeeks.forEach((b,k)=>b.week=k+1);
fs.writeFileSync(f,JSON.stringify(p,null,2)+"\n");' 3
```

Leave every other field exactly as it was, 2-space indentation, trailing newline. Keep `×`, `–` and other non-ASCII characters literal (node does; Python's json needs `ensure_ascii=False`).

## 6. Verify

```bash
node -e "JSON.parse(require('fs').readFileSync('/home/user/lift-log/plan.json','utf8'))" && echo valid JSON
```

```bash
node - <<'EOF'
const p=require("/home/user/lift-log/plan.json"), L=p.library, bad=[];
const isInt=n=>Number.isInteger(n), isDate=s=>/^\d{4}-\d{2}-\d{2}$/.test(s||"");
if(!isDate(p.updated)) bad.push("updated is not YYYY-MM-DD");
if(p.goal!==185||p.startDate!=="2026-10-19") bad.push("goal/startDate changed");
p.benchWeeks.forEach((b,i)=>{ if(b.week!==i+1) bad.push(`benchWeeks[${i}].week=${b.week}, expected ${i+1}`);
  for(const d of ["day1","day3","day5"]){ const x=b[d]||{}; if(!isInt(x.sets)||!isInt(x.reps)||typeof x.weight!=="number") bad.push(`week ${b.week} ${d}: sets/reps/weight must be numbers`); }});
const ids=p.templates.map(t=>t.id).join(","), wd=p.templates.map(t=>t.weekday).join(",");
if(ids!=="day1,day2,day3,day4,day5"||wd!=="1,2,4,5,6") bad.push(`template ids/weekdays changed: ${ids} / ${wd}`);
for(const t of p.templates) for(const x of t.exercises){
  if(!L[x.name]) bad.push(`${t.id}: "${x.name}" is not in library`);
  if(x.bench){ if(!["day1","day3","day5"].includes(x.bench)) bad.push(`${t.id}: bad bench ref ${x.bench}`); continue; }
  if(!isInt(x.sets)||typeof x.reps!=="string"||!(x.weight===null||typeof x.weight==="number")||!isInt(x.rest)) bad.push(`${t.id}: "${x.name}" sets/reps/weight/rest types wrong`);
  if(!/^\d+(-\d+)?(\/leg)?$/.test(x.reps)) bad.push(`${t.id}: "${x.name}" reps "${x.reps}" is not N, N-M, N/leg or N-M/leg`); }
for(const [n,e] of Object.entries(L)){ if(!e.m||!e.e||typeof e.inc!=="number"||!e.how||!Array.isArray(e.swaps)) bad.push(`library "${n}" missing m/e/inc/how/swaps`);
  for(const s of e.swaps||[]) if(!L[s]) bad.push(`library "${n}" swap "${s}" is not in library`); }
for(const g of p.graphs) if(!L[g]) bad.push(`graphs: "${g}" is not in library`);
if(p.review!==null){ const r=p.review; if(!isInt(r.week)||!isDate(r.date)||typeof r.summary!=="string"||!Array.isArray(r.changes)||typeof r.next!=="string") bad.push("review shape wrong"); }
console.log(bad.length?"PROBLEMS:\n- "+bad.join("\n- "):"plan.json OK: "+p.benchWeeks.length+" bench weeks, "+Object.keys(L).length+" library entries");
process.exit(bad.length?1:0);
EOF
```

Fix every problem it prints and re-run. `git -C /home/user/lift-log diff --stat` must show only `plan.json`; `git -C /home/user/lift-log diff` must match `review.changes`.

## 7. Ship

```bash
cd /home/user/lift-log && git add plan.json && git commit -m "Week N review: <one line>" && git push origin main
```

(If git asks for an identity: `git config user.name "Claude" && git config user.email "noreply@anthropic.com"`. Add any commit trailer the session's instructions require.) If the push to main is refused: `git checkout -b claude/weekly-review-wN && git push -u origin claude/weekly-review-wN`, open a pull request with the GitHub MCP tool `create_pull_request` (owner `Rocco-garr`, repo `lift-log`, head `claude/weekly-review-wN`, base `main`, body = the summary and changes), and say clearly in the report that the plan is waiting on a PR. Don't bump `VERSION` in `sw.js` (plan.json is fetched network-first). Never commit to the data repo.

## 8. Report

End with exactly five lines:

```
Attendance: 4/5 (missed Thu)
Bench: hit 4×5 @ 145, week 2 stays as planned (150)
Accessories: 3 changes (2 up, 1 weight found)
Next: <the review.next sentence>
Pushed: main @ <short sha>   (or: PR #n from claude/weekly-review-w1)
```
