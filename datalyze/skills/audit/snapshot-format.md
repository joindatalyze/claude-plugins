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
        { "name": "entry_point", "fill_rate": 0.97, "types": ["string"], "top_values": ["editor", "quick_post"], "source": "q2" }
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
3. Per event, each property seen: the share of rows carrying it (`fill_rate`), the JSON types seen (`types`; a date-shaped string goes in `shapes: ["datetime"]`), and top values.
4. User properties on profiles active in the window, the same way.
5. Volume by platform or library per event.
6. From a raw sample, where the tool allows: `duplicate_rate` (repeats for one person within a second) and `pii_values` (names of properties whose values looked like emails or phone numbers; never the values).
7. Identity: `identified_share`, and `duplicate_profiles` when the tool can tell.

## Rules

- **Every figure has a `source`,** the `id` of an entry in `queries` holding the exact query, request or tool call with its arguments. A description of a query isn't enough.
- **`sample_n`:** the raw rows behind an event's property figures. Under 30, its property checks are skipped.
- **`properties_complete: true`** only when the event's `properties` list every key it carries. Otherwise leave it out, and properties you didn't query aren't reported missing.
- **User properties:** `user_properties_complete: true` only when the list holds every property on the sampled profiles. Leave `user_properties` out entirely if you didn't collect them; they are then "not measured", not broken.
- **Vendor properties are left out:** names starting with `$` or `mp_`, plus `time`, `distinct_id` and `token`.
- **Production only** when the team names an environment property: record it in `environment` (`property`, `values`, `method`: `filtered` or `scaled`) with `excluded_share`.
