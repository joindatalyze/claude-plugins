# The proposal file

`.datalyze/proposals/pr-<n>.json` (or `branch-<branch>.json` on a laptop run with no pull request, slashes in the branch name becoming `-`) is what `review` writes, what a person edits and approves in the Datalyze UI (or with `@datalyze apply`), and what `implement` builds. `check_proposal` checks it; when the check and this page disagree, the check is right.

```json
{
  "version": 1,
  "pr": 412,
  "base": "origin/main",
  "head": "4f2c9a1d7e0b3c55a1f2e9d8c7b6a5f4e3d2c1b0",
  "run": "11893312077-1",
  "verdict": "changes_proposed",
  "summary": "Adds an export button to the reports page.",
  "items": [
    {
      "id": "p1",
      "kind": "event",
      "action": "extend",
      "target": "report_exported",
      "summary": "Reports can now be exported from the new toolbar button; the existing event gets entry_point = toolbar.",
      "question": "Which way into export do people use, and does the toolbar button change how often they export?",
      "breaking": false,
      "proposed": {
        "section": "Reports",
        "description": "The person exports a report and the file download starts. Fires once per export, in the browser, after the export request returns. Not on opening the export menu.",
        "track_via": "client",
        "properties": [
          { "name": "report_id", "type": "string", "example": "rpt_8f2k1x" },
          { "name": "entry_point", "type": "string", "example": "report_header | toolbar" },
          { "name": "file_type", "type": "string", "example": "csv | pdf" }
        ]
      },
      "changelog": "Added entry_point value toolbar",
      "evidence": [{ "code": "app/reports/toolbar.tsx:41", "quote": "<Button onClick={onExport}>Export</Button>" }]
    }
  ],
  "checked": [
    {
      "name": "report_viewed",
      "why": "Still fires on the report page load; the toolbar doesn't change it.",
      "tier": "core",
      "calls": [{ "code": "app/reports/page.tsx:22", "quote": "useEffect(() => track('report_viewed', { report_id }), [report_id])" }],
      "checks": {
        "fires":           { "result": "ok", "data": { "query": "counts_1" }, "why": "310 in 30 days." },
        "moment":          { "result": "ok", "why": "In an effect keyed on report_id, after the report loads." },
        "paths":           { "result": "ok", "why": "The one report page; the toolbar opens no other view." },
        "once":            { "result": "ok", "why": "useEffect [report_id], a string: once per report." },
        "name_accurate":   { "result": "ok", "why": "The report is on screen when it fires." },
        "name_clear":      { "result": "ok" },
        "properties":      { "result": "ok", "why": "report_id, as planned." },
        "values_filled":   { "result": "problem", "item": "p2", "data": { "query": "sample" }, "why": "report_id is null on 12% of rows: the shared-link route renders before params resolve." },
        "values_sensible": { "result": "ok", "why": "rpt_ ids, as the API returns." },
        "could_carry":     { "result": "ok", "why": "No value at the call site answers a new question." }
      }
    }
  ],
  "questions": [
    {
      "id": "q1",
      "item": null,
      "question": "Track the new column-picker in the export dialog?",
      "options": ["No: nobody decides anything from it", "Yes, as file_columns_count on report_exported"],
      "recommended": 0
    }
  ]
}
```

- **`pr`:** the pull request number, or null on a laptop run.
- **`head` and `run`:** the commit and the run the prompt names ("for head … and run …"), copied exactly. Write them on every run, even when the rest of the file is unchanged: a proposal with another run's `run` or another commit's `head` doesn't count, and the pull request stays red. On a laptop run, `head` is the branch's head commit and `run` is left out.
- **`verdict`:** `no_change` only when there are no items and no questions. Anything else is `changes_proposed`, so a question is never hidden behind "no change needed".
- **`kind`:** `event`, `event_property`, `user_property`, `super_property` or `identity`.
- **`action`:** `add`, `extend`, `modify` or `deprecate`.
- **`target`:** the name as it is in the plan at the base: an event (`checkout_started`), an event property (`checkout_started.coupon_code`), a user or super property (`plan_tier`), or `identify`, `alias` or `reset`. An item the change deleted from the plan is named as it was on the base. A new event names its `area` (the registry file, without `.yaml`).
- **`proposed`:** the complete entry after the change, in registry format, with real example values. A new event carries its `question`. Every field the item doesn't change is copied from the plan word for word.
  - Event property: `{ "type", "example", "required", "description" }`.
  - User property: `{ "type", "example", "operation", "set_via", "set_when", "source", "question", "pii" }`.
  - Super property: `{ "type", "example", "set_when", "set_via" }`.
  - `set_via` is one side (`"server"`), or every side that writes it (`["client", "server"]`), as the plan has it.
  - Identity: `{ "when", "id_source" }` (`id_source` for identify only).
  - Deprecating an event or user property: `{ "deprecated_reason", "replaced_by", "dual_fire_until" }`, the last two when something replaces it. The replacement is an `add` item in the same file, or an event already in the plan (merging a duplicate into it). Deprecating an event property: `{ "deprecated_reason" }`.
- **The plan is right and the code broke it** (an outcome moved before the outcome is known): still an item, a `modify` whose `proposed` is the entry unchanged, with a summary saying what the call must go back to.
- **`breaking: true`** for anything that changes what past data means. A changed firing moment's `changelog` starts with `SEMANTICS:`.
- **`evidence`:** at least one `{ "code", "quote" }` with the line quoted exactly, or `{ "data": { "tool", "query", "result" } }`.
- **`budget_reason`:** on each new event past the budget.
- **`questions`:** each offers options the person can pick in the UI, with `recommended` the index of yours. `item` links it to an item, or is null. A C10 property suggestion ("Add `coupon_applied` to `checkout_completed`?") has `"suggestion": true`: at most two per event, never one `decisions.yaml` skipped.
- **`checked`:** every event `reach` lists, and any other the change reaches. An event the change leaves alone (a shared helper changed, its call untouched) can be `{ "name", "why", "unaffected": true }`, with `why` saying what you checked. Otherwise, a card for every event the change reaches, problems or not, in the audit's card format (`../audit/audit-format.md`, "Cards"), with `name` and `why` (one line, as before) on top. Every `problem` names the item that fixes it (`item`) or says why there is none (`not_an_item`).
- **`existing: true`:** an item for a problem that was there before this change (true on the merge-base, or seen in today's data). Written in full like any item, so it can be built. It never holds the status, `@datalyze apply` doesn't approve it, and the reviewer picks Fix in this PR or Leave for the audit. Never write `decision` on it yourself.

## What Datalyze adds

On a pull request, the action writes `from_audit`: what the latest audit on `datalyze/audit` found on the events the change reaches (`{ audited_at, events: [{ name, bucket, warnings, findings }], clean, not_in_audit }`). The review screen and the pull request comment show it. Never write it yourself: one you wrote is replaced.

## What the reviewer adds

The UI or the action writes `approved_by` (the approver's GitHub login) when the proposal is approved. The person reviewing may, per item, set `"decision": "approved" | "rejected"`, edit `proposed`, and add a `"note"`. Per question they set `"answer"` (an option's index or their own text). At the top level they may add a `"note"`: anything else they want done that no item or question covers, in their words. All of it is binding on `implement`. Never write either `note` yourself.

`implement` builds items marked approved. When the run comes from an `@datalyze apply` comment, every item not marked rejected counts as approved, except `existing` items, and every answered question is binding. An `existing` item is approved only by Fix in this PR in the UI, which writes `decision: approved` with `decided_by`.
