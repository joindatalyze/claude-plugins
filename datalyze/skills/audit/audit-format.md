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
  ]
}
```

- **`kind`:** `install` for the first audit, `audit` for every later one.
- **`snapshot_source`:** null when there was no data. `health` is then null too.
- **`scope`:** null for the whole repo, else what it was limited to (`area checkout`, `events checkout_*`, `changed since 2026-10-01`).
- **`items`:** `match`'s items, unchanged.
- **Findings:** the title starts with the event or property name. Every finding has evidence. Its `fix` says what changes, and for an existing event, how it goes through deprecate, `replaced_by` and a dual-fire window.
