# The audit file

`audit.json` sits in a `.datalyze/` folder beside the registry folder (`analytics/.datalyze/audit.json` for the default `analytics/registry`), with `audit.md` next to it. The Datalyze UI's audit screen reads it; `check_audit` checks it, and when the check and this page disagree, the check is right.

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
      "fix": "Send checkout_completed from the order handler after the provider confirms, with status and failure_reason. Breaking: SEMANTICS changelog line."
    }
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
- **Findings:** the title starts with the event or property name. Every finding has evidence. Its `fix` says what changes, and for an existing event, how it goes through deprecate, `replaced_by` and a dual-fire window.
