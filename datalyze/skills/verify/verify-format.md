# The verification report and tested paths

## `analytics/.datalyze/verify.json`

`assemble_verify` writes it. The run and the results come from the capture service, as `check_captured` and `lookup_landed` recorded them; you never write a result. You give five things.

**`selection`:** what the person picked: `[{ "event": "checkout_completed" }, { "event": "signup_completed", "properties": ["plan"] }]`.

**`findings`:** one for every problem, the same shape as an audit's findings, with ids `V1`, `V2`, …:

```json
{
  "id": "V1",
  "severity": "high",
  "title": "checkout_completed fires before the order is created",
  "items": ["checkout_completed"],
  "checks": ["checkout_completed.moment"],
  "explanation": "The call runs on the Place order click; createOrder resolves 180 ms later.",
  "evidence": [{ "code": "app/checkout/page.tsx:88", "quote": "track('checkout_completed', { cart_value })" }],
  "fix": "Move the call into the .then after createOrder resolves.",
  "verifier": "V1 confirmed"
}
```

- `checks` names the problems it explains: `event.check` covers every path of the event, `event#1.check` one path. The check names are `fires`, `moment`, `paths`, `once`, `name_accurate`, `properties`, `values_filled`, `values_sensible`, `project` and `identity`. Every problem in the results must be named by a finding.
- `evidence` is at least one item: `{ "code": "file:line", "quote": "the line" }` (the quote must be at that line), or `{ "log": [14, 22] }`, the capture-log records the check cited.
- `severity`: `high` when the event's numbers are wrong (fires early, twice, to the wrong project, never); `medium` when a property is missing, empty or wrong; `low` for the rest.

**`not_done`:** every pick with no result, and why: `{ "event": "otp_verified", "status": "waiting_on_person", "why": "Needs the code sent to the test number" }`. `status` is `couldnt_reach` (with the step it stopped at), `waiting_on_person` or `not_checked`.

**`journeys`:** `{ "checkout_completed#0": "checkout" }`, so the screen can group results, and so a refusal for missing paths names the journey to save.

**`unsaved`:** each event you drove that no tested path from this run reaches, and why a path can't replay it: `{ "event": "invite_accepted#0", "why": "Needs a fresh invite link from the inbox each run" }`. With `final: true`, every result you drove (`reached_by: agent`, reached) is in a path saved or replayed whole in this run, or here.

Each result keeps its scripted `status` (`passed`, `problem`, `couldnt_reach`, `waiting_on_person`, `not_checked`), `checks`, `why`, `reached_by`, and `landed` (`ok`, `pending`, `not_found`, `not_measured`). Captured property values stay only on results with a problem, and never a property the plan's PII patterns flag. A result a tested path reaches has `tested_path`, its name, and the report's `paths` says which paths this run saved, which it replayed whole, and the `unsaved` events. `verify.md` is the readable copy.

With `final: true`, `assemble_verify` also pushes `datalyze/verify` and syncs the Google Sheet: Tracking Plan's Verification Result tells every path of each event, and the Verification tab lists the whole run, one row per event and path.

## `analytics/.datalyze/paths.json`

Tested paths, replayed by `replay_path` without the model. `save_path` writes them, and every run saves each journey it drove again under the same name, so the file holds the way that works now; a replay that runs every step marks its path tested today. Never edit the file yourself.

`save_path { name, events, values }` with no `steps` drafts them from this run's log: from the last page opened before the first event to the step that fired the last one, each step as Playwright printed it (locators, never what was typed), replayed steps as they were saved. A failed step, a snapshot or a hover is left out. `values` are the fills that aren't secrets, in order; a secret is filled by its name.

| `do` | Fields |
|---|---|
| `goto` | `url`, relative to the environment (`/cart`), or absolute on the environment's own site (`https://app.example.com/x` for `www.example.com`) |
| `back` | nothing: the browser's back button |
| `click` | a locator: `role` and `name`, or `label`, `testid` or `text` (or `css`, only as drafted from a step you took by selector); `nth` when several match (`-1` the last); `exact: false` when the name is only its start (Playwright shortens long names) |
| `fill` | the locator and `value`: a literal, `$secret:NAME`, `$unique_email`, `$unique_text` or `$test_card:success.card` |
| `select`, `check` | the locator, and `option` for select |
| `press` | `key`, with a locator to press it in that element, or without one for the keyboard (Enter in the focused field) |
| `wait_for` | `url`, a locator to be visible, or `time` in seconds |
| `totp` | the locator and `account`: the runner fills the code |

`consent` holds the steps that accept the site's cookie banner, saved with `save_path { consent: true, steps }` (by hand, never drafted): clicks on the banner, no fills and no labelled events. `check_environment` runs it before its probe and `replay_path` after a path's first page opens, only when the banner's first element shows within 4 seconds; otherwise they skip it, since this browser already gave consent.

`label_event` on a step names the event it should fire (`checkout_completed#0`, or a list). Never a snapshot ref (`e12`) or a CSS selector: they don't survive a reload.
