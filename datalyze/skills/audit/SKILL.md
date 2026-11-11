---
name: audit
description: Audit a product's analytics tracking with Datalyze. Compares the tracking plan, the code and what Mixpanel, PostHog or Amplitude receives, explains every problem with evidence, and writes the audit the Datalyze UI reads. Use for "audit our tracking", "what's broken in our analytics", "audit the checkout events", "is signup_completed firing", "what changed in tracking since 1 October", and the first audit after Datalyze setup, which also writes the first tracking plan. Works on the whole repo, one product area, a list of events, or what changed since a date.
---

# Audit

An audit compares three sources, the plan (the registry, `analytics/registry/` unless `.datalyze/config.yaml` says otherwise), the code and the data, and says what is wrong and why. It goes end to end, the way a person audits by hand: every event in scope gets the event check (`event-check.md`), from the code first, then from the analytics tool. It never changes product code. Fixes go through `implement` afterwards, from the findings a person picks.

The formats this skill writes are in the files next to it: `profile-format.md`, `snapshot-format.md` and `audit-format.md`.

## 1. Get the knowledge

Call the `knowledge` tool. It checks the team's Datalyze licence and returns the six knowledge files. Read all of them before anything else. If the licence is inactive, say so in one line and stop.

## 2. Read what the team has

Read `.datalyze/context.md` (the team's own words about their product), `.datalyze/decisions.yaml` if it's there (the team's answers to earlier audit questions and skipped suggestions: binding, and never asked again), `.datalyze/config.yaml`, and the plan with the `registry` tool. Note the scope you were asked for: the whole repo, an area, a list of events, or since a date. A `_map/` folder in the registry is a tracking map an older Datalyze version wrote. It's out of date: never read it, search it or edit it, and leave it out of everything you write.

The **first audit** is the one where the plan has no events yet (the setup skeleton). It also writes the first plan, in step 5.

**Carrying on:** when the prompt says some areas already have cards (`.datalyze/cache/cards/`), don't redo them or clear the folder: do only the areas it lists, then steps 8 to 10.

## 3. Find every tracking call

Track, identify, reset and alias calls, and user-property writes, through any wrapper, on every side: browser, server, workers and jobs, mobile, other services in the repo. Run `scan` first, then search by hand. Every call you find must be one `scan` sees. A call it misses means the tracking profile (`_tracking_profile.yaml`) doesn't describe that way of sending yet.

## 4. Understand the codebase and map every event

Work out the product areas (what the product does: Audits, Billing, Checkout; never folder names), the services and how they talk (client to API to handler to job), and every place a user action can start (buttons, routes, deep links, APIs, jobs). Then map each event, as `event-check.md` says: every call site, what triggers it (`scan` gives each JS or TS call's `within`, `component`, `trigger` and `after`; read one level up and down from there), which side sends it, where each property's value comes from, its funnel, its tier (core, money, onboarding, identity, funnel or other), and every place the same action happens that doesn't send it. `match` dates each call from git for the seven-day rule.

Keep the map in `.datalyze/cache/event-map.json`, for this run only. It is never committed.

## 5. Collect the data, and on the first audit write the first plan

Call `snapshot` with `audit: true` once, here, for the whole run. It collects everything the tool received in the window; the scope applies when you compare. On Mixpanel and PostHog it reads every record of the last 30 days into a DuckDB file, so every figure is exact; on a big project it can take up to 3 hours: wait for it, and never start it in the background. Its answer says which days weren't measured (a slice that failed after its retries: unknown, never zero) and which events were capped per day to fit the 3 hours or the disk; put both at the top of the report. When the tool refused the key or DuckDB couldn't be installed (the answer says why), every data result is `not_measured`; say why at the top of the report. Without keys, build the snapshot from the session's connection to the analytics tool if there is one. With neither, every data result is `not_measured`; say so at the top of the report. Don't read the snapshot to judge events yet: the code comes first (step 6).

**On the first audit, write the first plan now,** so `match` compares against a real plan and every area has its file:

- **The tracking profile** (`profile-format.md`). Run `scan` until it finds every call you found in step 3. Anything left it can't read goes in the report as a blind spot, with why.
- **One area file per product area,** from the map.
- **Every event entry says when it fires,** when it doesn't, and the boundary, from the code you read. Example values come from the code's literals and the snapshot's top values. `UNCONFIRMED` only where the code really has no literal and the data shows none, and each becomes an open question. A gap ("there is no event for X") is a finding, never a description.
- **Names stay exactly what the code sends,** because that's what the data holds. A name that breaks the convention is a finding, fixed later through the breaking-change protocol.
- **`_user_properties.yaml`, `_super_properties.yaml` and `_identity.yaml`** from what the code actually does. `_identity.yaml` comes from the code's identify, reset and alias calls, never from the template.
- **Events the data has that no code here sends.** Use the platform breakdown to decide: another service sends it (adopt it with `external: true`, naming the sender); it duplicates a plan event (adopt it as deprecated, with `replaced_by`); it is a rename of a plan event (don't adopt it; a finding); it still arrives from old app versions or old code (a finding: "still arriving from old clients"); nobody needs it (don't adopt it; a finding).
- **The team's naming.** Compare the code's main naming style and the "Naming and existing events" section of `context.md` with `_conventions.yaml`. If they differ, don't edit `_conventions.yaml` (only the team changes it). Raise an open question with the pattern you recommend.

## 6. The event check, area by area

Clear `.datalyze/cache/cards/` first, unless the prompt says you're carrying on (step 2). Then write a card for every event in scope (`audit-format.md`, "Cards"): every live event, every planned event (only `fires`: is it implemented?), and every event the data or the code has that the plan doesn't. One card file per area, `.datalyze/cache/cards/<area>.json` named like the area's registry file, at most 25 events each (split a bigger area: `checkout-1`, `checkout-2`). Events still not in the plan after step 5 go in `unplanned.json`.

For each area, in this order:

1. **The code side.** Assume each event misfires until the code shows it doesn't, and fill each check from the code and the map.
2. **The data side.** Call `match` with the snapshot and `area` set to the area (for `unplanned`, the event names). Fill each card's data side from `match`'s checks and the snapshot's figures: null, empty and placeholder rates, values, double-firing, platforms. Confirm or overturn each code verdict: a null rate of 0 overturns "may be null"; a 30% null rate on a property the code always sets means another sender, an old app version or a path the map missed, so find which before you write it. Take `not_yet_arriving` and `never_arrived` as `match` gives them: they already use the call's first commit and the plan entry's first change (`since`). Every item that isn't healthy gets an explanation; a healthy item keeps its code checks. A check `match` warned about stays a problem unless you can show the warning is expected; then keep it `ok` and say why in the card's `warning_notes` (`audit-format.md`). The warning is shown either way. An event that arrives with no code that sends it (`unknown_source`, `firing_from_elsewhere`) may carry `sent_by` once assembled, from the data's library field: say the same on its card ("sent by the Python library, and we found no code that sends it"), and where its code probably lives when the audit's "not read" list has that language.
3. **Write the area's file** as soon as it's done, so a run that stops halfway keeps it. Leave `finding` off: the findings come in step 9.

On a repo with more than one area, give each area to a subagent: the area's name, its events from the map, the snapshot's path, and this step's instructions. Each subagent calls `knowledge` itself, never calls `snapshot`, writes only its own card file, and answers with one line per problem (`<event>.<check>: why, file:line`), one per suggestion, and one per `ok` on C2, C3, C7 and C8 of a core or money event, as a bare claim (`checkout_completed.moment: fires only after createOrder resolves`). Run them in the foreground, never in the background. Don't read the card files back yourself: work from the lines they return. That keeps your context small enough for a large product.

### On a full pull

On a snapshot from the full pull (it has `pull`), `match` judges every day of the window against the days before it, and its checks say when a change began: `volume_drop` and `volume_jump` (`began`, `change` in percent), `stopped` (`began`, the first empty day), `property_blank`, `blank_rise` and `values_shifted`, plus `one_value`, `dominant_value`, `double_firing` (critical from 5%, `exact_rate` and `rapid_rate` apart; Mixpanel's times are whole seconds, `precision: second`) and `type_split` with its `detail`. For a finding about one of these, set its `began` in `_audit.json` to the check's day and say it in the title ("sign_up dropped 70% from 4 Nov"); find what changed in the code around that day in git. The snapshot's `identity` and `pull.sessions` figures aren't judged by `match`: judge them with these thresholds, and only on the figures given.

- **Bad ids** (`identity.bad_ids`): a finding over 1% of records, high from there.
- **Shared ids** (`identity.shared_ids`): a finding for each distinct_id carrying more than 50 `$user_id`s.
- **Nothing before login** (`identity.before_login`): with 100 or more identified, a finding over 40% without prior anonymous activity, high over 70%. Mixpanel's is from `$device_id` (`device-id-approx`): say so.
- **Ids per person** (`identity.ids_per_person`, PostHog only): a finding for persons on more than 5 distinct_ids, high from 50.
- **Page views** (`pull.sessions`, web only): with 500 or more sessions, page views under 0.1 a session is a finding; with at least 50 sessions of 10+ records, over 30% of them on a single page view is a finding (a single-page app not sending its route changes).

**Follow-ups on the records.** On a snapshot from the full pull, `data_query` runs one read-only SQL query (DuckDB) on every record of the window: use it whenever the snapshot's figures leave the question open, and never guess a figure it can give. Its description lists the tables (`events`, `prop_values`, `day_status`, `profiles`, ...). The usual follow-ups:

- **Fires at the right moment (C2):** the order of two events per user. For an outcome event, how many users have it before (or without) the step it follows: `WITH f AS (SELECT distinct_id, min(time_ms) FILTER (event = 'payment_succeeded') AS paid, min(time_ms) FILTER (event = 'checkout_completed') AS done FROM events GROUP BY distinct_id) SELECT count(*) FILTER (done < paid OR (done IS NOT NULL AND paid IS NULL)) AS early, count(*) FILTER (done IS NOT NULL) AS users FROM f`.
- **Fires from every path (C3):** which platforms, libraries or app versions never send it, against those that send the events around it: `SELECT platform, lib, count(*) FILTER (event = 'signup_completed') AS signups, count(*) AS records FROM events GROUP BY ALL ORDER BY records DESC`; an app version is a property: `properties->>'app_version'`.
- **A drop or a blank spike:** the day `began`, split by platform, library, version or a property's value, to find what changed with it: `SELECT day, platform, count(*) AS n, count(*) FILTER (coalesce(properties->>'method', '') = '') AS blank FROM events WHERE event = 'sign_up' AND day BETWEEN '2026-10-28' AND '2026-11-09' GROUP BY ALL ORDER BY day`.

One question a query; aggregate in SQL rather than reading rows (at most 1,000 come back, and 30 seconds unless you ask for up to 120). Personal values come back `[withheld]`. Every query and its answer is logged in the snapshot as `dq1`, `dq2`, ...: cite the one a finding rests on as `{ "query": "dq3" }`, and take its numbers from that answer exactly.

Area subagents may call `data_query` too (never `snapshot`); give them this section with the rest of the step.

## 7. What isn't tracked, and what needn't be

Look for what isn't tracked at all. List the product's core actions (sign-up, login, each onboarding screen, the first value moment, creating or changing the main thing the product is for, money) and its server outcomes with no person present. Each core action with no event is a finding with a proposed event. Other untracked actions get a proposed event only within the budget per area; the rest are open questions.

Then look the other way: at what is tracked but probably not needed. Check every live event against the "What not to track" section of `what-to-track.md`, reading the code around its call: what the person did to send it, and whether anything decides on it. Each one that breaks a rule is a drop candidate, not a finding: the team decides, in the Datalyze UI. For each, give the rule in a few words, one sentence on why this event breaks it, the call as evidence, its volume when there's data, and your recommendation:

- **`drop`:** it carries no decision. Stop tracking it.
- **`fold`:** it is one of several events for one action (a tab, a step, an entry point). Name the event and property it becomes in `fold_into` (`settings_changed.setting`), an existing event when there is one.
- **`keep`:** it breaks the rule's letter but the team relies on it (`context.md` says so, or it is the one signal for something). Say so in the reason.

Never list a core action, an onboarding screen, a money event, an identity call, or an event a funnel or another event's check depends on. Never list an event `decisions.yaml` already keeps (a decision whose `event` is its name). When unsure, leave it out: a candidate costs the team a decision.

## 8. Have the verifier try to break the work

Draft the findings first: problems that share one cause, across cards and areas, become one finding. Then give the `verifier` agent, one area at a time, run one after another in the foreground: that area's draft findings, numbered, with the code and data each cites, and the area's `ok` claims from step 6. Nothing about how you reached any of it. **Every area with a card file gets its own round, `unplanned` included, even an area with no draft findings: its `ok` claims still need breaking.** Name the area in the round's description (`Verify connections`). A finding that spans areas goes in the round of each area it touches; a cross-area round can follow those, never replace one. Before step 9, check that every card file had its round, and run any that's missing. Keep what it confirms. Turn what it can't decide into an open question. Drop what it disproves from the findings, list it in `_audit.json`'s `dismissed` with the verifier's reason, and say so in the report. A problem it disproves isn't one: set that card check to `ok`, with the verifier's reason as its `why`, so the grid never shows a problem nobody can choose to fix. An `ok` it breaks becomes a problem: edit that one check in the area's card file, and add it to a finding. Wait for each answer before you go on: never start the verifier, or any subagent, in the background. Your run ends when you stop, so a round still running when you stop is lost, and so is the report.

## 9. The findings and the rest of the audit

Write `.datalyze/cache/cards/_audit.json` (`audit-format.md`): the findings, each listing in `checks` the card checks it covers (`checkout_completed.moment`); in `not_findings`, every card problem no finding covers, with the reason: an open question now (name the event in the question), or a drop candidate. Every other problem is a finding: the Findings tab is where the person chooses what to fix, and it shows only these; the questions, the no-data items and the drop candidates.

## 10. Write the audit

**`audit.json`** (`audit-format.md`) is what the Datalyze UI reads. Never write it by hand: call `assemble_audit` (with the same scope you gave `match`). It merges the card files, `match`'s items and `_audit.json` into `audit.json`, links each card check to its finding, and checks the result. Besides the shape and the code quotes, it holds every data number in a finding's title and explanation, and in a card check's `why`, to the answer behind it: a `data_query` answer it cites, or `match`'s checks and the snapshot's figures for its items. Write each number as the answer gives it, or rounded to the digits you show ("3.4%" for 0.0342, "about 4,100" for 4,120); a figure you worked out yourself (a sum, a ratio, a difference) gets its own query first. A number that isn't data (a count of call sites, a line) goes in words or backticks; dates, durations, versions and the thresholds above after "over" or "at least" are fine as they are. If it reports problems, fix the files it names and call it again. Never discard them. An area still without cards when you have to stop goes in `_audit.json`'s `incomplete_areas` with why: the audit is published with it, never thrown away.

The person answers the questions, picks findings to fix, adds or skips the property suggestions, picks no-data items to look into and keeps, stops or folds each drop candidate in the Datalyze UI, and one fix run builds all of it.

**`audit.md`,** next to it, for people:

1. **Summary.** Health, counts by bucket, the three things to fix first.
2. **How this product tracks today.** SDKs, wrappers, client and server, identity calls, and anything the scan can't see, with why.
3. **Event by event.** One row per event, the ten checks as ✓ (ok), ✗ (problem), ◦ (suggestion) or – (not measured), and one line under the table for each ✗. This is the part a person would have written by hand. On a large product, list every event with a ✗ in full and count the rest.
4. **Findings,** most important first, each with its evidence and fix.
5. **Could carry more.** Each property suggestion with the question it answers.
6. **Fix plan by product area.** Money and sign-up first, then core actions, then the rest.
7. **Open questions,** each with options the person can pick in the Datalyze UI and your recommendation, the same ones as in `audit.json`. Never ask them to send you something, run something, or change a setting. Never ask again what `decisions.yaml` already answers.
8. **Items with no data that aren't findings,** each with why no data is expected, the same ones as `no_data` in `audit.json`.
9. **Tracked but probably not needed,** each drop candidate with its rule, reason and recommendation, the same ones as `drop_candidates`.

On the first audit, finish the plan last: fix what can be fixed in the plan, run `vocabulary` to write `_properties.yaml` from the plan, add a `synonyms:` entry for every property spelled two ways (`kb_id: knowledge_base_id`), run `vocabulary` again (it keeps synonyms), then `baseline` to accept the remaining problems, so later changes are judged only on what they add. Then `validate` must pass.

Then stop. Tell the person what you wrote and the three things to fix first.
