---
title: "Gamification"
summary: "The XP formula, the quadratic level curve, two streak systems that only break on a real miss, decayed late check-ins, and the signed daily ledger that makes every number auditable."
---

Everything in Beyou's gamification serves one behavior: showing up daily. This document explains the exact mechanics, formula by formula, including where the numbers come from, what deliberately does not exist, and the three real inconsistencies found while verifying the code for this page.

## The check-in formula

One static calculator produces every earned XP amount:

```
xp = round( 5 × (difficulty + importance) × (1 + min(streak × 1%, 50%)) )
```

Difficulty and importance are clamped to 1 through 5, so the base range is 10 to 50, and the streak bonus (one percent per consecutive day, capped at fifty percent after day 50) stretches the ceiling to 75. The streak used is the one standing before today's check, so a check never feeds its own multiplier. An earlier formula multiplied difficulty by importance and swung 10 to 250; the additive version keeps a hard habit worth five times a trivial one instead of twenty-five.

The amount fans out whole, not split:

```mermaid
flowchart LR
  CHK["✅ Check a habit"] --> CALC["🎮 5 × (diff + imp) × streak bonus"]
  CALC --> U["👤 User XP"]
  CALC --> R["📋 Routine XP"]
  CALC --> H["💪 Habit XP"]
  CALC --> CATS["📂 Every linked category"]
  CHK --> LEDGER["📅 Signed daily ledger<br/>EntityXpDay + EntityCheckDay"]
```

Tasks play by slightly different rules: a task carries no XP of its own, and a task with no categories earns nothing at all, since there would be nobody to pay besides the user and routine and the design routes task credit through life areas. One-time tasks never build streaks and never write history rows; they are checked once and scheduled for cleanup.

Unchecking removes exactly the stored generated amount rather than recomputing, so tuning the constants later can never desync a removal from its original award.

## Levels

The curve is one line: reaching level L costs round(50 × L²) XP, levels 0 through 100. Level 2 costs 200, level 10 costs 5,000, level 100 sits at 500,000. Early levels are hours apart, late ones are seasons apart. Four things level independently on this curve: the user, each category, each habit, and each routine.

New habits and categories can start with a head start: the experience question at creation seeds BEGINNER at level 0, INTERMEDIARY at level 5, ADVANCED at level 8, so someone who already runs marathons does not grind a running habit from zero.

## Streaks

Streak logic follows one philosophy: **only a real miss breaks a streak**. Every day gets an outcome, and the outcomes are not symmetric:

| Outcome | Effect on a streak |
|---------|--------------------|
| DONE | Counts and continues |
| SKIPPED | Continues without counting: an honest "not today" is not a failure |
| NOT_SCHEDULED / NOT_IN_ROUTINE | Continues without counting |
| MISSED | Breaks |

Nothing increments a counter. Streaks are re-derived by walking the daily outcome rows backwards from the owner's own today, which is what made the old fragile counters retireable: a derived streak cannot drift.

Two parallel systems share that walk. Each habit, task, and routine has its own streak (current, best, total check-ins) stored as scalars and recomputed on every change, with the best-streak as a ratchet that only rises. The account streak, the "constance" on the dashboard, walks the user's completed days against what was scheduled, and the user chooses what "completed" means: ANY (checking anything today counts) or COMPLETE (every item in every section checked or skipped).

**The day-close job** is what turns absence into outcomes. An hourly scheduler, working per timezone, closes yesterday during a grace window in the small hours (a window rather than an exact hour, because daylight-saving jumps once skipped the closing hour entirely and left a day permanently open). It writes one row for each owner that has none, insert-only, so a real check landing during the race can never be overwritten by an absence. Routines are deliberately excluded from day-close: no presence writer exists for them, so every routine row would be a false miss.

**Dormancy** softens long gaps: a streak with nothing scheduled or completed for 14 days reports as dormant rather than broken, and the UI shows a pause instead of a zero. Checking anything clears it instantly.

## Late check-ins and decay

Past days live in routine snapshots, and checking one still pays, but through a decay chosen by the user:

| Strategy | Multiplier |
|----------|-----------|
| GRADUAL | 0.8 one day late, then 0.6, 0.4, 0.2 from day four on |
| FLAT | 0.5 regardless of delay |
| TIME_WINDOW | Full XP up to two days late, nothing after |

Late checks never receive a streak bonus (the streak was already broken by the miss), and difficulty and importance come frozen from the snapshot, not from the habit as it is today. What a late check does do is repair history: the missed day flips to DONE and the streak walk reconnects across it, so two five-day runs separated by one repaired miss become an eleven-day streak, a behavior pinned by tests. Unchecking flips the day back and leaves the best-streak record standing. When the original habit or routine has since been deleted, the XP still pays out in shrinking tiers: full distribution, then user plus routine, then user alone.

## Goals

Goals pay once, through the explicit complete endpoint, calculated from target size, difficulty, urgency, and finishing before the deadline (the full factor table lives in the domain model topic). The reward goes to the user and the goal's categories; goals themselves have no level. Completing is a toggle: un-completing takes the XP back and returns the goal to in-progress. Progress updates pay nothing, which is what makes a fake pre-completed goal worthless.

## The ledger

Every XP movement, in both directions, writes a signed delta into a per-owner per-day ledger, upserted with in-database addition so concurrent check-ins queue instead of overwriting. Summing an owner's rows reproduces its total exactly; unchecking makes the day give the XP back rather than remembering a high-water mark. The ledger feeds the XP history endpoint, which returns dense series (a value for every day of the window, zeros included) for the dashboard's charts. The parallel outcome table does the same for checks. Both tables use owner references without foreign keys on purpose: history must outlive whatever produced it.

## The celebration layer

The frontends detect moments by comparison, not by backend flags: a check response carries fresh totals, the client compares against the previous level and streak, and a crossing pushes a celebration into a queue. Level-ups celebrate any rise; streak milestones celebrate crossing 7, 14, 21, 30, 60, 90, or 100 days, lowest crossed first. The "+N XP" float over a checked item shows the real generated amount, decay and all.

## What deliberately does not exist

No daily completion bonus, no routine-completion payout, no weekly recap XP. The four XP paths (habit check, task check, snapshot check, goal completion) and their four reversals are the whole economy. Skips cannot be placed in the future, since an unbounded forward skip would make a streak unbreakable.

## Honest findings

Writing this page against the code surfaced real inconsistencies, listed here until fixed:

| Finding | Consequence |
|---------|-------------|
| The experience head-start seeds predate the current level curve: INTERMEDIARY starts at level 5 with 750 XP against a 1,250 threshold, ADVANCED at level 8 with 1,800 against 3,200 | Both start below their own level's floor, so the first uncheck collapses them (5 → 3, 8 → 6) |
| Un-completing a goal recomputes the reward instead of reading back the stored one, and the on-time factor depends on when the recomputation runs | Complete before the deadline, un-complete after it, and about a quarter of the payout stays behind permanently |
| The XP ledger writes late check-ins under today's date while the outcome table writes them under the snapshot's date | The two histories disagree for any late check, and unchecking an old day can render a negative bar on today |
| Back-dating a skip over a day that closed as MISSED rewrites it to SKIPPED | A streak broken weeks ago can be retroactively repaired without doing anything; the code's own comments name this hole |
