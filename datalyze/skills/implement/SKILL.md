---
name: implement
description: Build analytics tracking changes a person approved, with Datalyze. Applies an approved tracking proposal (.datalyze/proposals/pr-<n>.json), or a person's decisions on an audit (their answers to its questions, the findings they picked, no-data items to look into and their note), changes the tracking plan and the track calls in the team's own style, runs the rules checks, and has an independent verifier check the result. Use for "implement the approved proposal", "@datalyze apply", "fix these audit findings", or when Datalyze dispatches an implement run.
---

# Implement

A person has already decided what changes. Build exactly that: the plan entries and the track calls, nothing else, written the way this codebase already writes tracking.

## 1. Get the knowledge and the decision

Call the `knowledge` tool and read all six files. If the licence is inactive, say so in one line and stop.

A `_map/` folder in the registry is a tracking map an older Datalyze version wrote. It's out of date: never read it, search it or edit it, and leave it out of everything you write.

Read `.datalyze/context.md` and `.datalyze/decisions.yaml` if it's there (the team's answers to earlier audit questions, binding), then what was approved:

- **A proposal:** the pull request's proposal file on this branch (`.datalyze/proposals/pr-<n>.json`, or `branch-<branch>.json` on a laptop run), in the format in `../review/proposal-format.md` (from this skill's folder). Build the items marked approved. When the run comes from an `@datalyze apply` comment, every item not marked rejected counts as approved, except `existing` ones (problems that were there before the change): those are built only when a person marked them approved (Fix in this PR). The reviewer's edits to `proposed`, their notes and their answers to questions are binding. So is the proposal's top-level `note` ("Anything else"): do what it asks within tracking, and put what you can't do, or what reaches outside tracking, under Held back with why. Rejected and undecided items are left alone.
- **A person's decisions on an audit:** `answers.json` next to the audit file (`audit.json` in the `.datalyze/` folder beside the registry), both on the audit branch: `git show origin/datalyze/audit:analytics/.datalyze/answers.json`. It holds, all for this one change, across every product area:
  - **`answers`:** the option the person picked for each of the audit's `questions`. The picked option's text is binding: build what it says, in the plan, the code, `_conventions.yaml` or `.datalyze/context.md`, wherever it lands. `option: null` is "Decide later": leave that question alone. Apply the answers first, since they can change the rules the fixes are written against.
  - **`findings`:** the finding ids to fix (below).
  - **`look_into`:** no-data items from the audit's `no_data`. For each, trace the code path that should send it. If it's broken, fix it like a finding. If it's sound and the event simply hasn't happened, change nothing and say what you checked.
  - **`tracking`:** the person's choice for events the audit's `drop_candidates` listed as probably not needed. `keep`: change nothing (the Datalyze action records it in `decisions.yaml`). `drop`: mark the event `deprecated` in the plan with the candidate's reason as `deprecated_reason`, and remove its call. `fold`: add the property in the candidate's `fold_into` to that event (a new event when it isn't in the plan yet), send it from the call sites, and deprecate the old event with `replaced_by`; a `dual_fire_until` window only when a report or another event's check uses the old one, else the old call goes now. An event left out of `tracking` is undecided: leave it alone.
  - **`conditions`:** the person's choice on each of the audit's `property_conditions` (`W1`...). `keep` on one with `in_plan: false`: write its `only_when` on the property's plan entry, remove `required: false` or the `null` or `""` from its example, and add a changelog line; one already `in_plan` is only recorded (the Datalyze action writes `decisions.yaml`). `always`: the property applies on every path. Send it, with a real value, from every call site of the event that leaves it out, remove its `only_when`, `required: false` or blank example from the plan, and add a changelog line. If a path has no value to send, don't invent one: hold the item back with a question. The action checks afterwards that every call sends it.
  - **`suggestions`:** the property suggestions the person picked to add (ids from the audit's `property_suggestions`). Build each as an extend: the property on the event's plan entry (type, an example from the code, a changelog line; a new name goes into `_properties.yaml`) and sent from every call site the event's card lists, with the value the suggestion's evidence shows is there. A suggestion the person skipped is recorded in `decisions.yaml` by the Datalyze action; leave it alone.
  - **`note`:** anything else the person wants done, in their words. Binding, like a reviewer's note. Do what it asks within tracking; what you can't do, or what reaches outside tracking, goes under Held back with why.
- **Audit findings:** the finding ids you were given, in the audit file. When it isn't on this branch, read it from the audit branch: `git show origin/datalyze/audit:analytics/.datalyze/audit.json`. For each finding, find why in the code and the data, and decide the smallest change that makes the plan, the code and the data agree: fix the call when the code is wrong, update the plan when the data shows the plan is stale. Don't add events beyond what the fix needs. A fix to an existing event follows the breaking-change protocol. Every picked finding goes into this one change, whatever its area.

**The audit fix's work list:** `.datalyze/fixes/audit-<date>.json` on the fix branch (the prompt names it). The Datalyze action writes it before you start, one item per thing the person asked for: `Q1` (an answered question), `F3` (a finding), `S2` (a property suggestion to add), `W1` (a property condition kept or made every path), `look:<name>`, `tracking:<event>`, `note`. Build only the items whose `state` is `todo`; one that's `done` was built by an earlier run on this branch and is already in the code. As you finish each item, set its `state` to `done` with a one-line `summary` of what changed, or to `held_back` with a `question` (`{ "text", "options": [two or more], "recommended": <index> }`) the person answers on the audit screen. A held-back item whose `question` has an `answer` is the person's decision: build it. Change nothing else in the file: the items, their titles and what was asked are the person's. Update it as you go, not at the end, so a run that stops part-way still says what it finished.

**Carrying on:** when the prompt says Datalyze already built part of this on the branch, the earlier run's work is in the code and the plan. Read the branch's diff against the base first (`git diff <base>`), don't redo what's there, and build what's left. The same goes for a second pass in the same run: the first pass's work is in the working tree.

## 2. Change the plan and the calls

- **The plan:** write each item into its area file under the registry, with a changelog line (date, the pull request or the branch, what changed). A new property name goes into `_properties.yaml`.
- **A property sent only sometimes** says when with `only_when`, from the code (`event-check.md`, "Only when"). Never write `required: false`, or `null` or `""` in an example, for it.
- **A modify keeps everything it doesn't change.** Every description and property description the item doesn't touch stays word for word.
- **The calls:** through the wrapper the tracking profile names for that side, copying its example call: same import, same argument order, same style. Send exactly the plan's properties, with real values the code has at that moment. Fire at the moment the description says.
- **A deprecation with a `dual_fire_until` date** keeps the old call next to the new one until that date. Without one, the old call goes.
- **Nothing outside the approved items.** No refactors, renames, restyling or unrelated fixes.
- **Never edit** `_conventions.yaml`, the `synonyms` in `_properties.yaml`, `_baseline.json`, `.datalyze/context.md`, `.datalyze/decisions.yaml`, `.datalyze/config.yaml`, `.claude/` or `.github/`. The one exception: a person's answer to an audit question that says to change `_conventions.yaml` or `context.md`. Make exactly the change the answer says, nothing more. The Datalyze action records the answers in `decisions.yaml` itself.

If an item can't be built as written (the value doesn't exist at that moment, the event would fire twice), don't improvise. Leave it out and raise it as an open question with the options you see.

## 3. Run the checks

Run `validate` and `scan`, each with `base` set to the branch's base (the pull request's base, or the default branch for audit fixes), so they judge only what this change did, by the rules already merged. Fix until both pass. Never change the rules or `_conventions.yaml` to make them pass. If a rule is genuinely wrong, say so in the report and leave the failure for a person.

## 4. Have the verifier check it

Give the `verifier` agent the code diff, the registry diff (`git diff <base> -- analytics/`), the proposal file's path when there is one (it checks the `checked` cards), and the decisions already made: the approved targets with the reviewer's notes and answers and the proposal's top-level note (for an audit, the finding ids and titles, each answered question with the picked option's text, the property suggestions added, the no-data items looked into, each event's tracking choice, and the person's note). Not your reasoning, not the proposal's summaries, not this conversation: its value is that it comes to the change cold. It reads the knowledge itself. Wait for its answer before you go on: never start the verifier, or any subagent, in the background. Your run ends when you stop, so a round still running when you stop is lost, and so is the report.

The verifier records its own result with its `verdict` tool; never call that tool yourself. Without it the build isn't counted as verified. Apply its blockers and fixes, and run the checks again. When you disagree with one, keep the finding and write why in the report. At most two verifier rounds. A blocker or fix still open after that, disputed or not, becomes an open question on its item.

## 5. Keep every verified item

Only mark an item built (`implemented: true`, or `done` in a work list) when the code sends it: after the run, the action scans the code, and an event that isn't sent, or is sent without its planned properties, goes back to not built with the reason. Keep every item that passed the checks and the verifier. Hold back only the items with open questions: undo their edits in the plan and the code, and list them in the report with the question. One bad item never loses the rest.

When the run built from a proposal, also add each held-back item's question to the proposal file's `questions`, with `item` set to its id, the options and your `recommended` (format in `../review/proposal-format.md`). That is where the Datalyze UI shows it and the person answers; the next run builds from the answer.

Mark each built item `"implemented": true` in the proposal file as soon as it's built and checked, so a later review, or a run that carries on after this one stops, knows it is done. Leave all edits in the working tree; the Datalyze action commits them with its own token, so the team's CI runs on them.

## 6. Write the report

`.datalyze/report.md` (the action posts it and doesn't commit it). Write it before you stop, every time: a run that stops without it keeps nothing. Its first line is exactly one of:

- `Implemented and verified` (every approved item built, checks and verifier passed)
- `Implemented, <n> held back` (the rest are listed under Held back)
- `Needs a person` (nothing could be kept, or the checks still fail)

Then:

1. **What changed.** One line per item, in plain words, with the files.
2. **For a person's decisions on an audit:** under **Answers**, each answered question and what it changed; under **Added**, each property suggestion and where it's sent from; under **Looked into**, each no-data item and what you found; under **Tracking**, each event stopped or folded and what changed; under **When properties apply**, each condition kept or made every path, and what changed; under **The note**, what you did about it. For a proposal with a top-level `note`, the same **The note** line.
3. **Held back.** Each item with its open question.
4. **Checks.** `validate` and `scan` results.
5. **Verification.** Rounds run, findings applied, findings disputed and why.

Questions offer options the person can pick in the Datalyze UI, with your recommendation. Never ask them to send you something, run something, or change a setting.
