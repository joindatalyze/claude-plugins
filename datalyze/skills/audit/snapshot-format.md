# The data snapshot

What the analytics tool received, in one JSON file whatever the tool. The `snapshot` tool writes it from the team's keys. When you build it yourself from a connection to the analytics tool, write the same shape and pass the file to `match`. The rules in `datalyze-method.md` ("Collecting the data") apply.

```json
{
  "tool": "mixpanel",
  "project": "2317312",
  "generated_at": "2026-10-20T05:30:00Z",
  "window_days": 30,
  "window_end": "2026-10-19",
  "queries": [
    { "id": "q1", "tool": "mixpanel", "query": "Insights: $all_events, total, 30d, breakdown $event_name", "ran_at": "2026-10-20T05:30:00Z" }
  ],
  "events": [
    {
      "name": "post_created",
      "volume": 1240,
      "daily": [40, 38, 45],
      "platforms": { "web": 1100, "ios": 140 },
      "sample_n": 500,
      "properties_complete": true,
      "duplicate_rate": 0.01,
      "source": "q1",
      "properties": [
        {
          "name": "entry_point", "fill_rate": 0.91, "null_rate": 0.06, "empty_rate": 0, "types": ["string"],
          "placeholders": [{ "value": "undefined", "share": 0.03 }],
          "distinct": 3, "top_values": ["editor", "quick_post", "undefined"],
          "top_value_shares": [{ "value": "editor", "share": 0.81 }, { "value": "quick_post", "share": 0.16 }, { "value": "undefined", "share": 0.03 }],
          "by_platform": { "web": { "n": 440, "fill_rate": 0.97, "null_rate": 0 }, "ios": { "n": 60, "fill_rate": 0.5, "null_rate": 0.5 } },
          "source": "q2"
        },
        {
          "name": "word_count", "fill_rate": 1, "null_rate": 0, "empty_rate": 0, "types": ["numeric"],
          "numbers": { "n": 500, "min": 0, "max": 4210, "p05": 12, "p50": 180, "p95": 1400, "zero_share": 0.02, "negative_share": 0 },
          "source": "q2"
        }
      ]
    }
  ],
  "user_properties_complete": true,
  "user_properties": [
    { "name": "plan_tier", "fill_rate": 0.92, "types": ["string"], "top_values": ["free", "pro"], "stale_share": 0.04, "source": "q3" }
  ],
  "identity": { "identified_share": 0.81, "source": "q4" }
}
```

## What to collect

1. Every event that arrived in the window, with its total volume.
2. Daily counts for each event in the plan, so drops show up.
3. Per event, each property seen, from the raw rows:
   - `fill_rate`: the share of rows with a value. `null_rate`: the key is there and the value is null. `empty_rate`: an empty string (or only spaces) or an empty list. Rows without the key are the rest.
   - `placeholders`: strings standing in for no value, as seen, with their share of rows: `undefined`, `null`, `NaN`, `[object Object]`, `N/A`, `unknown`, `none` (compared trimmed, any case).
   - `types`: the JSON types seen (a date-shaped string goes in `shapes: ["datetime"]`).
   - `distinct` (`distinct_capped: true` past 500), `top_values`, and `top_value_shares`: each top value's share of the values.
   - `numbers` for numeric values: `n`, `min`, `max`, `p05`, `p50`, `p95`, `zero_share`, `negative_share`.
   - `dates` for date-shaped values: `earliest` and `latest`.
   - `by_platform`, when the event arrives from two or more: per platform, its rows (`n`), `fill_rate` and `null_rate`.
4. User properties on profiles active in the window, the same way.
5. Volume by platform or library per event.
6. From a raw sample, where the tool allows: `duplicate_rate` (repeats for one person within a second) and `pii_values` (names of properties whose values looked like emails or phone numbers; never the values).
7. Identity: `identified_share`, and `duplicate_profiles` when the tool can tell.

## Rules

- **Every figure has a `source`,** the `id` of an entry in `queries` holding the exact query, request or tool call with its arguments. A description of a query isn't enough.
- **`sample_n`:** the raw rows behind an event's property figures. Under 30, its property checks are skipped.
- **`properties_complete: true`** only when the event's `properties` list every key it carries. Otherwise leave it out, and properties you didn't query aren't reported missing.
- **A figure you can't get is left out,** never guessed. Its check is then not measured.
- **Personal properties** (named like personal data, or `pii: true` in the plan) get rates only: no `top_values`, `top_value_shares`, `numbers` or `dates`. Their `placeholders` are kept: the words are Datalyze's list, never a person's value.
- **Scoped:** a pull request's snapshot asks for the reached events only (`events` on the `snapshot` tool). The same shape, with only those events.
- **User properties:** `user_properties_complete: true` only when the list holds every property on the sampled profiles. Leave `user_properties` out entirely if you didn't collect them; they are then "not measured", not broken.
- **Vendor properties are left out:** names starting with `$` or `mp_`, plus `time`, `distinct_id` and `token`.
- **Production only** when the team names an environment property: record it in `environment` (`property`, `values`, `method`: `filtered` or `scaled`) with `excluded_share`.
