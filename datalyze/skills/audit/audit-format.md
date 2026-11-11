# The audit file

`audit.json` sits in a `.datalyze/` folder beside the registry folder (`analytics/.datalyze/audit.json` for the default `analytics/registry`), with `audit.md` next to it. The Datalyze UI's audit screen reads it; `check_audit` checks it, and when the check and this page disagree, the check is right.

**You never write `audit.json` by hand.** You write two kinds of working file in `.datalyze/cache/cards/` (never committed), and `assemble_audit` builds `audit.json` from them, `match`'s items and the snapshot:

- **`<area>.json`, one per area** (at most 25 events; `checkout-1.json`, `checkout-2.json` for a bigger one; `unplanned.json` for events not in the plan): `{ "area": "checkout", "events": [ cards ], "property_suggestions": [ suggestions ], "property_conditions": [ conditions ] }`. Suggestion and condition ids only need to be unique in their file: `assemble_audit` numbers them S1, S2... and W1, W2... across the audit.
- **`_audit.json`, one:** everything else in the example below except `items`, `health`, `snapshot_source`, `events`, `property_suggestions` and `property_conditions`, which the tool fills in. It adds `not_findings` (below), `dismissed` (the findings the verifier disproved), `incomplete_areas` when the run must stop with areas left, and on the first audit `baselined` (below).

`_verified.json` beside them is the tools' own record of what each verifier round checked: never write or edit it.

```json
{
  "kind": "install",
  "generated_at": "2026-11-03T09:12:00Z",
  "snapshot_source": { "tool": "mixpanel", "project": "2317312", "window_days": 30, "generated_at": "2026-11-03T09:00:00Z" },
  "scope": null,
  "health": 72,
  "items": [ "every item from match, as match returns it: name, type, bucket, status, checks" ],
  "findings": [
    {
      "id": "F1",
      "title": "checkout_completed fires before the payment succeeds",
      "severity": "high",
      "items": ["checkout_completed"],
      "explanation": "The call runs on the Pay click, before the payment request returns, so every attempt counts as a sale.",
      "evidence": [
        { "code": "app/checkout/pay-button.tsx:58", "quote": "track('checkout_completed', { order_id })" },
        { "query": "q1" }
      ],
      "fix": "Send checkout_completed from the order handler after the provider confirms, with status and failure_reason. Breaking: SEMANTICS changelog line.",
      "checks": ["checkout_completed.moment", "checkout_completed.name_accurate"]
    }
  ],
  "not_findings": [
    { "check": "refund_issued.values_filled", "reason": "The verifier found the null rows are test refunds from the admin tool, which the plan excludes." }
  ],
  "events": [
    {
      "name": "checkout_completed",
      "tier": "money",
      "area": "checkout",
      "calls": [{ "code": "app/checkout/pay-button.tsx:58", "quote": "track('checkout_completed', { order_id })" }],
      "checks": {
        "fires":           { "result": "ok", "data": { "query": "counts_1" }, "why": "Arrives daily; 1,240 in the window." },
        "moment":          { "result": "problem", "finding": "F1", "why": "Sent on the Pay click, before createOrder resolves.", "evidence": [{ "code": "app/checkout/pay-button.tsx:58", "quote": "track('checkout_completed', { order_id })" }] },
        "paths":           { "result": "ok", "why": "Only the pay button places orders; the API has no other caller." },
        "once":            { "result": "ok", "why": "A single onClick call; duplicate_rate 0.002." },
        "name_accurate":   { "result": "problem", "finding": "F1", "why": "Nothing is completed when it fires.", "evidence": [{ "code": "app/checkout/pay-button.tsx:58", "quote": "track('checkout_completed', { order_id })" }] },
        "name_clear":      { "result": "ok" },
        "properties":      { "result": "ok", "why": "order_id on the one call." },
        "values_filled":   { "result": "ok", "data": { "query": "sample" }, "why": "order_id comes from the cart; null rate 0." },
        "values_sensible": { "result": "ok", "why": "order_id matches the ord_ shape the cart makes." },
        "could_carry":     { "result": "suggestion", "why": "coupon is in scope at the call.", "suggestions": ["S1"] }
      },
      "properties": [{ "name": "order_id", "fill_rate": 1, "null_rate": 0, "empty_rate": 0, "top_values": ["ord_k29x8w1m"] }]
    }
  ],
  "property_suggestions": [
    { "id": "S1", "event": "checkout_completed", "property": "coupon_applied", "type": "boolean",
      "question": "Do coupons change how many checkouts complete?",
      "evidence": [{ "code": "app/checkout/pay-button.tsx:41", "quote": "const { coupon } = useCart()" }] }
  ],
  "property_conditions": [
    { "id": "W1", "event": "checkout_completed", "property": "coupon", "only_when": "a coupon is applied",
      "why": "Only the coupon branch of applyDiscount has a code; the saved-card and plain checkouts have none to send.",
      "in_plan": true,
      "evidence": [{ "code": "app/checkout/pay-button.tsx:41", "quote": "const { coupon } = useCart()" }, { "code": "app/checkout/one-click.tsx:22", "quote": "track('checkout_completed', { cart_id, amount })" }] }
  ],
  "incomplete_areas": [],
  "dismissed": [
    { "title": "signup_completed stopped arriving on 28 Oct", "reason": "The verifier found it arrives from the auth service since the 28 Oct release; the query read only the web project.", "items": ["signup_completed"] }
  ],
  "questions": [
    {
      "id": "Q1",
      "question": "Event naming: the code uses Title Case, and _conventions.yaml says snake_case. Which should the plan follow?",
      "options": [
        { "id": "a", "text": "Change the convention to Title Case events and camelCase properties, matching the existing events" },
        { "id": "b", "text": "Keep snake_case for new events only" },
        { "id": "c", "text": "Move every event to snake_case through dual-fire" }
      ],
      "recommended": "a",
      "why": "context.md says to keep existing names, and one style is better than two.",
      "findings": ["F15"]
    }
  ],
  "no_data": [
    { "name": "refund_issued", "reason": "A rare money path: no refunds in the window, and the call is wired after the refund succeeds." }
  ],
  "drop_candidates": [
    {
      "name": "billing_tab_clicked",
      "rule": "one event per tab",
      "reason": "Sent on each click of the Billing tab in Settings; profile_tab_clicked and team_tab_clicked are the same action on the other tabs.",
      "recommended": "fold",
      "fold_into": "settings_tab_opened.tab",
      "volume": 4120,
      "evidence": [{ "code": "app/settings/tabs.tsx:31", "quote": "track('billing_tab_clicked')" }]
    },
    {
      "name": "tooltip_shown",
      "rule": "tooltips",
      "reason": "Sent each time a help tooltip opens on hover; no funnel or check uses it.",
      "recommended": "drop",
      "volume": 88310,
      "evidence": [{ "code": "components/help-tip.tsx:12", "quote": "track('tooltip_shown', { id })" }]
    }
  ]
}
```

- **`kind`:** `install` for the first audit, `audit` for every later one.
- **`snapshot_source`:** null when there was no data. `health` is then null too.
- **`scope`:** null for the whole repo, else what it was limited to (`area checkout`, `events checkout_*`, `changed since 2026-10-01`).
- **`items`:** `match`'s items, unchanged.
- **`questions`:** every open question in `audit.md`, empty when there are none. Option ids are short (`a`, `b`, `c`) and each option's `text` reads on its own, because it is what's recorded when the person picks it. `recommended` is an option id, or null when you don't recommend one. `findings` (optional) names the findings it bears on.
- **`no_data`:** every item with no data in the window that isn't a finding, with the reason, empty when there are none. The person can ask the fix run to look into any of them.
- **`drop_candidates`:** every live event that breaks a "what not to track" rule, empty when there are none. `name` is an event in `items`, spelled as `match` spells it. `recommended` is `keep`, `drop` or `fold`; `fold` needs `fold_into` (`<event>.<property>`), and the person can only pick Fold when it's there. `volume` is left out when there was no data. `evidence` has the track call. A Keep the person picks goes into `decisions.yaml` with the event's name, so it isn't listed again.
- **Cards (`events`):** one per event in scope: every live event, every planned event (`not_implemented`: only `fires`), and every event the data or the code has that the plan doesn't. `name` as `match` spells it. `tier`: `core`, `money`, `onboarding`, `identity`, `funnel` or `other`. `calls`: every call site, quoted. `checks`: the ten, keyed `fires`, `moment`, `paths`, `once`, `name_accurate`, `name_clear`, `properties`, `values_filled`, `values_sensible`, `could_carry` (`knowledge/event-check.md`), each with `result` `ok`, `problem`, `suggestion` (could_carry only) or `not_measured`:
  - `problem`: `evidence` (`{ code, quote }` or `{ query }`) or `data` (`{ query }`), and covered by a finding's `checks` or listed in `not_findings` (`assemble_audit` writes `finding` or `not_a_finding` onto the check from those).
  - `ok`: a `why` saying what was checked, on every tier but `other` (`name_clear` may go without).
  - `not_measured`: a `why` (no data, too few rows, no keys).
  - Data-based results cite a query id from the snapshot's `queries` (a `data_query` answer is `dq1`, `dq2`, ...); with no snapshot, every result is code-only or `not_measured`. On a full pull's snapshot, every data number in a check's `why` must be in the answer it cites or in the snapshot's and `match`'s figures for its event; `assemble_audit` says which isn't.
  - `could_carry` with `suggestion` lists at most two `suggestions` ids, only on tiers other than `other`.
  - **`match`'s warnings stand.** The screen shows each item's `checks` on its own row under the event, as `match` wrote them. A check whose warning `match` raised (`property_null` on `values_filled`, `double_firing` on `once`: the table in `event-check.md`) can't be `ok` unless the card explains the warning in `warning_notes`: `[{ "warning": "property_null.coupon", "note": "Expected: blank means no coupon." }]`, the id being the warning's `kind`, then `.` and its `property` when it has one. The note shows beside the warning; it never hides it. `assemble_audit` refuses an `ok` without one, and a note on a warning `match` didn't raise.
  - `properties` is filled in by `assemble_audit` from the snapshot. Evidence whose quote isn't in the code drops that one check to `not_measured`, with a `note` saying so. Evidence in the wrong shape (`{ "query": { "query": "pull_properties" } }`, a `code` without `:line`) is refused, printed as written: the query id is always a plain string. A `fires` check's own count must be in something it cites: the snapshot's counts (`pull_counts`), or a `data_query` answer that counts its event (`count(*) FILTER (event = 'Payment Initiated')` answers 0 for an event with no records). An answer that only counts the events around it can stand beside one of those, never alone.
- **`property_conditions`:** every property this audit decided applies only sometimes (`knowledge` "Only when"): the card's `event`, the `property`, `only_when` (when it applies, `<property>=<value>` where another property decides it), `why` (why the paths that don't send it don't need it, in one line), `in_plan` and `evidence` (a call that sends it and one that doesn't). On the first audit, every `only_when` you wrote into the plan, with `in_plan: true` (`assemble_audit` refuses one left out). On a later audit, every `required: false` or blank example you decided is a real exception (an `unexplained_exception` on its card), with `in_plan: false`: the plan doesn't change in an audit, so the fix writes it once the person keeps it. One the team already decided (`decisions.yaml`, `property: <event>.<name>`, `decision: only_when` or `always`) stands: never list it again. The person picks Keep (recorded; the fix writes it into the plan when `in_plan` is false) or Every path (the fix sends the property on every path and removes the exception).
- **`property_suggestions`:** C10. Each names its card's `event`, the `property`, its `type`, the `question` it answers and `evidence`: the value already at the call site. Never one `decisions.yaml` skipped (`property: <event>.<name>`, `decision: skip`). The person picks Add (built by the fix run as an extend) or Skip (recorded in `decisions.yaml`).
- **`not_read`:** written by `assemble_audit`, never by you: the code files in languages the scan doesn't open, by language and top folder. Say in `audit.md`'s "what couldn't be seen" that events sent from there aren't checked.
- **`dismissed`:** every draft finding the verifier disproved: its `title`, the verifier's `reason`, and the `items` it was about. The screen lists them, folded, under "Dismissed by the verifier", so nothing is dropped without a trace.
- **`incomplete_areas`:** `{ area, reason }` for each area with no cards when the run had to stop. The audit is published without them, and the screen says which are missing. An event with no card fails the check unless its area is listed.
- **`rules`** (first audit, on a finding or a question): the plan's rule errors it covers, as `validate` names them (`"R013 analyzers"`, `"R007 Audit Completed.analyzerCount"`), or a rule alone (`"R007"`) for all of its errors. R004, R005, R006, R012, R013 and R025 are named one by one.
- **`baselined`** (first audit): the rule errors accepted into the baseline as debt, `{ "rule": "R023", "where": "*" or as validate names it, "why": "the reason it's accepted for now" }`. `baseline` refuses while an error is neither fixed, nor covered by a finding's or question's `rules`, nor listed here; `"*"` doesn't accept the rules named one by one. `audit.json` keeps the list.
- **`began`** (optional, yyyy-mm-dd): on a finding about a change in the data, the day it began, copied from the `began` of a check on one of its items. `check_audit` refuses any other day. The audit screen shows it after the title ("from 4 Nov").
- **Findings:** the title starts with the event or property name. `items` names the existing events or properties it is about (an identity call by its name: `identify`, `reset`, `alias`). A finding about something nothing tracks yet ("no event when a subscription starts", "report downloads have no event") leaves `items` empty and names the events its fix adds in `proposed_events` (`["Subscription Started"]`), with no `checks` unless a card check really is about it: never put it under an unrelated existing event to pass the check. The screen groups those under "Not tracked yet" and marks each new event. Every finding has evidence: `{ code, quote }` for the code, `{ query }` for the data (a snapshot query id, or a `data_query` answer, `dq1`). Every data number in its title and explanation must be in an answer it cites or in its items' figures; `check_audit` says which number isn't, and which `dq` answer holds it when the finding doesn't cite it. Its `fix` says what changes, and for an existing event, how it goes through deprecate, `replaced_by` and a dual-fire window.
