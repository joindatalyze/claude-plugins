---
name: review
description: Decide what analytics tracking a code change needs, with Datalyze, and write the proposal a person approves. Use on a pull request, a branch or the working tree in any repo with a Datalyze tracking plan (a `.datalyze/` folder), for "what does this branch need", "what does this branch need for tracking", "is this PR ready", "review this PR's tracking", "did this break any events", "what events does this change need", or when Datalyze runs on a pull request. In such a repo, a question about what a branch or PR needs includes its tracking, so run this skill even when the question doesn't say "tracking", alongside any code review. Most changes need none. It checks every existing event the change can reach before proposing anything, and it doesn't edit code or the plan.
---

# Review

The job is to say what a change needs for tracking, and nothing more. Most changes need nothing, and "no tracking change needed" is a good result. Breaking an existing event is worse than missing a new one, so the existing events come first.

Review doesn't edit code or the plan. It writes a proposal; `implement` builds it once a person approves.

## 1. Get the knowledge and the change

Call the `knowledge` tool and read all five files. If the licence is inactive, say so in one line and stop.

Read `.datalyze/context.md`, the plan (the `registry` tool), and the change: `git diff <base>...HEAD`, where the base is the one you were given, else the default branch. On a working tree with no branch, use `git diff HEAD` plus untracked files. Leave `.datalyze/` out of the change.

The proposal file is `.datalyze/proposals/pr-<n>.json` for pull request n, or `.datalyze/proposals/branch-<branch>.json` when there is no pull request yet (a laptop run; slashes in the branch name become `-`). If it already exists on the branch (the pull request was pushed again), read it first. Items `implement` already built are done; they aren't part of what you decide.

If the change is empty, or touches nothing but `.datalyze/`, write a `no_change` proposal saying so, and stop.

## 2. Map before deciding

Write nothing yet. First work out:

- **What the change does for users,** in product terms: "reports can be exported from a toolbar", "coupon checks moved to the server". Skip refactors, styling, copy, tests and tooling, or say "no user-facing change" when that's the truth.
- **Every existing event, user property and identify or reset call it can reach,** by the five steps in `datalyze-method.md`, and for each one, how the change affects it, walking the checklist in `good-vs-bad.md`. Use `scan` and read the code around each call.
- **What the data says,** when keys exist: before deprecating an event, check with `snapshot` that it doesn't still arrive from another platform; before adding an enum value, check the format the data already uses.

## 3. Decide each item, in this order

No change, extend, modify, deprecate, new. Stop at the first that fits. A new event must pass the question test, have no near-duplicate in the plan, come with its pair if the action can fail, and fit the budget, with a written reason for each event past it.

- **A new thing a person can do** (an export button, a new way to share) gets an item or a visible question with your recommendation. Never a buried note.
- **A rename or deprecation** comes with its replacement event fully written in the same proposal, and a `dual_fire_until` date. A `replaced_by` pointing at an event nobody proposed can't be built.
- **Legacy problems** the change doesn't touch aren't yours. Leave them to the audit.
- **If the change already edits the plan correctly,** there is nothing to propose for that item. List it as checked.
- **On a second run,** keep the reviewer's decisions, edits, notes and answers for every item whose target and code didn't change. Don't propose a rejected item again unless the change altered the reason it was rejected.

## 4. Write the proposal and the report

**The proposal file** in the format in `proposal-format.md` (next to this file). Check it with `check_proposal`. If the check fails, fix the file from the error message and check again. Never discard it.

**`.datalyze/report.md`,** short, for the pull request comment. The Datalyze action posts it and doesn't commit it.

1. **Verdict.** One line: `No tracking change needed`, or what the proposal changes and asks.
2. **Changes.** Per item: the event, the kind of change, breaking or not, the question it answers.
3. **Existing events checked.** Only events the change reaches, each with why it is unaffected. No unrelated events.
4. **Breaking changes.** What each affects and the migration (`replaced_by`, dual-fire date).
5. **Open questions.** Each with options the person can pick in the Datalyze UI and your recommendation. Never ask them to send you something, run something, or change a setting ("send that excerpt", "raise the budget").

Then stop.
