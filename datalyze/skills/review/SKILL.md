---
name: review
description: Decide what analytics tracking a code change needs, with Datalyze, and write the proposal a person approves. Use on a pull request, a branch or the working tree in any repo with a Datalyze tracking plan (a `.datalyze/` folder), for "what does this branch need", "what does this branch need for tracking", "is this PR ready", "review this PR's tracking", "did this break any events", "what events does this change need", or when Datalyze runs on a pull request. In such a repo, a question about what a branch or PR needs includes its tracking, so run this skill even when the question doesn't say "tracking", alongside any code review. Most changes need none. It checks every existing event the change can reach before proposing anything, and it doesn't edit code or the plan.
---

# Review

The job is to say what a change needs for tracking, and nothing more. Most changes need nothing, and "no tracking change needed" is a good result. Breaking an existing event is worse than missing a new one, so the existing events come first.

Review doesn't edit code or the plan. It writes a proposal; `implement` builds it once a person approves.

## 1. Get the knowledge and the change

Call the `knowledge` tool and read all six files. If the licence is inactive, say so in one line and stop.

Read `.datalyze/context.md`, `.datalyze/decisions.yaml` if it's there (the team's answers to audit questions and skipped suggestions: binding, and never asked again), the plan (the `registry` tool), and the change: `git diff <base>...HEAD`, where the base is the one you were given, else the default branch. On a working tree with no branch, use `git diff HEAD` plus untracked files. Leave `.datalyze/` out of the change. A `_map/` folder in the registry is a tracking map an older Datalyze version wrote. It's out of date: never read it, search it or edit it, and leave it out of everything you write.

The proposal file is `.datalyze/proposals/pr-<n>.json` for pull request n, or `.datalyze/proposals/branch-<branch>.json` when there is no pull request yet (a laptop run; slashes in the branch name become `-`). If it already exists on the branch (the pull request was pushed again), read it first. Items `implement` already built are done; they aren't part of what you decide.

If the change is empty, or touches nothing but `.datalyze/`, write a `no_change` proposal saying so, and stop.

## 2. Map and check before deciding

Write nothing yet. Go end to end, as `event-check.md` says:

- **Every tracking call** the change can touch: run `scan` (each JS or TS call comes with its `within`, `component`, `trigger` and `after`), then read the code around each call.
- **What the change does for users,** in product terms: "reports can be exported from a toolbar", "coupon checks moved to the server". Skip refactors, styling, copy, tests and tooling, or say "no user-facing change" when that's the truth.
- **Every existing event, user property and identify or reset call it can reach,** by the five steps in `datalyze-method.md`. Start from the `reach` tool (with `base`): a short summary by area, then call it with `area` for one area's events at a time. Its list is the least you account for: add what else you find, never drop one of its events. `check_proposal` refuses a proposal that leaves one out. For each event, map it: its call sites, what triggers it, its side, where each property's value comes from, its funnel, its tier, and the places the action happens that don't send it.
- **The event check on every reached event, C1 to C10, from the code.** One card each (`proposal-format.md`, `checked`), problems or not. Assume it misfires until the code shows it doesn't. Walk `good-vs-bad.md`'s checklist for how the change affects it.

**A review doesn't ask the analytics tool.** Don't call `snapshot` or `match`, or query the tool through a connection, even when keys or a connection exist: the data side of each card is `not_measured`. What the data showed is the latest audit's to say: Datalyze adds the audit's findings and data warnings for the events the change reaches to the proposal itself (`from_audit`), from the audit stored on `datalyze/audit`, and shows them beside your cards. Don't write `from_audit`, and don't repeat the audit's findings as items unless this change makes one worse.

**Caused by the change, or already there.** Split each card's problems in two:

- **Caused by this change** (it moved the call, added a path without it, made a value nullable, changed a type): these become proposal items as below, and they decide the status. A change that makes a value nullable (a new optional chain, a value now read before it loads) is caught on the code side of C8 and is a change item, even though the data can't show it yet.
- **Already there** (true on the base too): `existing` items, written in full like any item so they can be built, with `"existing": true`. They never fail the status; the reviewer picks Fix in this PR or Leave for the audit. To tell the two apart, run the code checks on the base's version of each call (`git show $(git merge-base <base> HEAD):<file>`).

**Could carry more:** on reached core, money, onboarding, identity and funnel events, at most two C10 suggestions each, as proposal questions ("Add `coupon_applied` to `checkout_completed`?"), each with its question and the value at the call site. Skip any `decisions.yaml` already skipped.

## 3. Decide each item, in this order

No change, extend, modify, deprecate, new. Stop at the first that fits. A new event must pass the question test, have no near-duplicate in the plan, come with its pair if the action can fail, and fit the budget, with a written reason for each event past it.

- **A new thing a person can do** (an export button, a new way to share) gets an item or a visible question with your recommendation. Never a buried note.
- **A rename or deprecation** comes with its replacement event fully written in the same proposal, and a `dual_fire_until` date. A `replaced_by` pointing at an event nobody proposed can't be built.
- **Legacy problems** outside the reached events aren't yours. Leave them to the audit. Inside the reached events, they are `existing` items (step 2).
- **If the change already edits the plan correctly,** there is nothing to propose for that item. List it as checked.
- **On a second run,** keep the reviewer's decisions, edits, notes and answers for every item whose target and code didn't change, and keep the proposal's top-level `note` as it is. Don't propose a rejected item again unless the change altered the reason it was rejected.

## 4. Write the proposal and the report

**The proposal file** in the format in `proposal-format.md` (next to this file). Check it with `check_proposal`. If the check fails, fix the file from the error message and check again. Never discard it.

**`.datalyze/report.md`,** short, for the pull request comment. The Datalyze action posts it and doesn't commit it.

1. **Verdict.** One line: `No tracking change needed`, or what the proposal changes and asks.
2. **Changes.** Per item: the event, the kind of change, breaking or not, the question it answers.
3. **Existing events checked.** A short table of the events the change reaches: the event, then ✓ (ok), ✗ (problem), ◦ (suggestion) or – (not measured) for each of the ten checks, and one line under it for each ✗. Every event with a ✗ in full; past 20 events, count the rest ("41 more events checked, no problems: see Datalyze"). No unrelated events.
4. **Already wrong in events this change touches.** Each `existing` item: what's wrong, the evidence, and that the reviewer can fix it in this pull request or leave it for the audit. It doesn't block the merge.
5. **Breaking changes.** What each affects and the migration (`replaced_by`, dual-fire date).
6. **Open questions.** Each with options the person can pick in the Datalyze UI and your recommendation. Never ask them to send you something, run something, or change a setting ("send that excerpt", "raise the budget").

Then stop.
