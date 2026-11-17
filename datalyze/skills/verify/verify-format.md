# The verification report and tested paths

## `analytics/.datalyze/verify.json`

`assemble_verify` writes it. The run and the results come from the capture service, as `check_captured` and `lookup_landed` recorded them; you never write a result. You give four things.

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

**`journeys`:** `{ "checkout_completed#0": "checkout" }`, so the screen can group results.

Each result keeps its scripted `status` (`passed`, `problem`, `couldnt_reach`, `waiting_on_person`, `not_checked`), `checks`, `why`, `reached_by`, and `landed` (`ok`, `pending`, `not_found`, `not_measured`). Captured property values stay only on results with a problem, and never a property the plan's PII patterns flag. `verify.md` is the readable copy.

## `analytics/.datalyze/paths.json`

Tested paths, replayed by `replay_path` without the model. `save_path` writes them; never edit the file yourself.

| `do` | Fields |
|---|---|
| `goto` | `url`, relative to the environment (`/cart`) |
| `click` | a locator: `role` and `name`, or `label`, `testid` or `text`; `nth` when several match |
| `fill` | the locator and `value`: a literal, `$secret:NAME`, `$unique_email`, `$unique_text` or `$test_card:success.card` |
| `select`, `check`, `press` | the locator, and `option` or `key` |
| `wait_for` | `url`, a locator to be visible, or `time` in seconds |
| `totp` | the locator and `account`: the runner fills the code |

`label_event` on a step names the event it should fire (`checkout_completed#0`, or a list). Never a snapshot ref (`e12`) or a CSS selector: they don't survive a reload.
