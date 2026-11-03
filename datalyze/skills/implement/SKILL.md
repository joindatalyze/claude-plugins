---
name: implement
description: Build analytics tracking changes a person approved, with Datalyze. Applies an approved tracking proposal (.datalyze/proposals/pr-<n>.json) or audit findings a person picked, changes the tracking plan and the track calls in the team's own style, runs the rules checks, and has an independent verifier check the result. Use for "implement the approved proposal", "@datalyze apply", "fix these audit findings", or when Datalyze dispatches an implement run.
---

# Implement

A person has already decided what changes. Build exactly that: the plan entries and the track calls, nothing else, written the way this codebase already writes tracking.

## 1. Get the knowledge and the decision

Call the `knowledge` tool and read all five files. If the licence is inactive, say so in one line and stop.

Read `.datalyze/context.md`, then what was approved:

- **A proposal:** the pull request's proposal file on this branch (`.datalyze/proposals/pr-<n>.json`, or `branch-<branch>.json` on a laptop run), in the format in `../review/proposal-format.md` (from this skill's folder). Build the items marked approved. When the run comes from an `@datalyze apply` comment, every item not marked rejected counts as approved. The reviewer's edits to `proposed`, their notes and their answers to questions are binding. Rejected and undecided items are left alone.
- **Audit findings:** the finding ids you were given, all from one product area, in the audit file (`audit.json` in the `.datalyze/` folder beside the registry). When it isn't on this branch, read it from the audit branch: `git show origin/datalyze/audit:analytics/.datalyze/audit.json`. For each finding, find why in the code and the data, and decide the smallest change that makes the plan, the code and the data agree: fix the call when the code is wrong, update the plan when the data shows the plan is stale. Don't add events beyond what the fix needs. A fix to an existing event follows the breaking-change protocol.

## 2. Change the plan and the calls

- **The plan:** write each item into its area file under the registry, with a changelog line (date, the pull request or the branch, what changed). A new property name goes into `_properties.yaml`.
- **A modify keeps everything it doesn't change.** Every description and property description the item doesn't touch stays word for word.
- **The calls:** through the wrapper the tracking profile names for that side, copying its example call: same import, same argument order, same style. Send exactly the plan's properties, with real values the code has at that moment. Fire at the moment the description says.
- **A deprecation with a `dual_fire_until` date** keeps the old call next to the new one until that date. Without one, the old call goes.
- **Nothing outside the approved items.** No refactors, renames, restyling or unrelated fixes.
- **Never edit** `_conventions.yaml`, the `synonyms` in `_properties.yaml`, `_baseline.json`, `.datalyze/context.md`, `.datalyze/config.yaml`, `.claude/` or `.github/`.

If an item can't be built as written (the value doesn't exist at that moment, the event would fire twice), don't improvise. Leave it out and raise it as an open question with the options you see.

## 3. Run the checks

Run `validate` and `scan`, each with `base` set to the branch's base (the pull request's base, or the default branch for audit fixes), so they judge only what this change did, by the rules already merged. Fix until both pass. Never change the rules or `_conventions.yaml` to make them pass. If a rule is genuinely wrong, say so in the report and leave the failure for a person.

## 4. Have the verifier check it

Give the `verifier` agent the code diff, the registry diff (`git diff <base> -- analytics/`), and the decisions already made: the approved targets with the reviewer's notes and answers (for audit findings, the finding ids and titles). Not your reasoning, not the proposal's summaries, not this conversation: its value is that it comes to the change cold. It reads the knowledge itself.

Apply its blockers and fixes, and run the checks again. When you disagree with one, keep the finding and write why in the report. At most two verifier rounds. A blocker or fix still open after that, disputed or not, becomes an open question on its item.

## 5. Keep every verified item

Keep every item that passed the checks and the verifier. Hold back only the items with open questions: undo their edits in the plan and the code, and list them in the report with the question. One bad item never loses the rest.

Mark each built item `"implemented": true` in the proposal file, so a later review knows it is done. Leave all edits in the working tree; the Datalyze action commits them with its own token, so the team's CI runs on them.

## 6. Write the report

`.datalyze/report.md` (the action posts it and doesn't commit it). Its first line is exactly one of:

- `Implemented and verified` (every approved item built, checks and verifier passed)
- `Implemented, <n> held back` (the rest are listed under Held back)
- `Needs a person` (nothing could be kept, or the checks still fail)

Then:

1. **What changed.** One line per item, in plain words, with the files.
2. **Held back.** Each item with its open question.
3. **Checks.** `validate` and `scan` results.
4. **Verification.** Rounds run, findings applied, findings disputed and why.

Questions offer options the person can pick in the Datalyze UI, with your recommendation. Never ask them to send you something, run something, or change a setting.
