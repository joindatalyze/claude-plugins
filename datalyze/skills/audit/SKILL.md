---
name: audit
description: Audit a product's analytics tracking with Datalyze. Compares the tracking plan, the code and what Mixpanel, PostHog or Amplitude receives, explains every problem with evidence, and writes the audit the Datalyze UI reads. Use for "audit our tracking", "what's broken in our analytics", "audit the checkout events", "is signup_completed firing", "what changed in tracking since 1 October", and the first audit after Datalyze setup, which also writes the first tracking plan. Works on the whole repo, one product area, a list of events, or what changed since a date.
---

# Audit

An audit compares three sources, the plan (the registry, `analytics/registry/` unless `.datalyze/config.yaml` says otherwise), the code and the data, and says what is wrong and why. It never changes product code. Fixes go through `implement` afterwards, from the findings a person picks.

The formats this skill writes are in the files next to it: `profile-format.md`, `snapshot-format.md` and `audit-format.md`.

## 1. Get the knowledge

Call the `knowledge` tool. It checks the team's Datalyze licence and returns the five knowledge files. Read all of them before anything else. If the licence is inactive, say so in one line and stop.

## 2. Read what the team has

Read `.datalyze/context.md` (the team's own words about their product), `.datalyze/config.yaml`, and the plan with the `registry` tool. Note the scope you were asked for: the whole repo, an area, a list of events, or since a date.

The **first audit** is the one where the plan has no events yet (the setup skeleton). It also writes the first plan, in step 5.

## 3. Work out everything the code sends

Find every track call, identify, reset and alias call, and user-property write, through any wrapper, on the client and the server, in every language in the repo. Read the code around each call, not only the call site: what triggers it, whether it fires before or after the work succeeds, under which conditions, how old the call is (git history), and what each property's value really is. For a large repo, split it by product area across subagents.

Run `scan` and compare: every call you found must be one `scan` sees. A call it misses means the tracking profile (`_tracking_profile.yaml`) doesn't describe that way of sending yet.

## 4. Get what the analytics tool receives

Call `snapshot`. It collects everything the tool received in the window; the scope applies when you compare. Without keys, build the snapshot from the session's connection to the analytics tool if there is one. With neither, carry on with the plan and the code alone, and say so at the top of the report.

## 5. On the first audit, write the first plan

Do this before comparing, so `match` compares against a real plan. On later audits, skip to step 6.

- **The tracking profile** (`profile-format.md`). Run `scan` until it finds every call you found in step 3. Anything left it can't read goes in the report as a blind spot, with why.
- **One area file per product area.** Areas are what the product does (Audits, Billing, Setups, Checkout), never folder names (CONTEXTS, SERVICES, ROUTES, components).
- **Every event entry says when it fires,** when it doesn't, and the boundary, from the code you read. Example values come from the code's literals and the snapshot's top values. `UNCONFIRMED` only where the code really has no literal and the data shows none, and each becomes an open question. A gap ("there is no event for X") is a finding, never a description.
- **Names stay exactly what the code sends,** because that's what the data holds. A name that breaks the convention is a finding, fixed later through the breaking-change protocol.
- **`_user_properties.yaml`, `_super_properties.yaml` and `_identity.yaml`** from what the code actually does. `_identity.yaml` comes from the code's identify, reset and alias calls, never from the template.
- **Events the data has that no code here sends.** Use the platform breakdown to decide: another service sends it (adopt it with `external: true`, naming the sender); it duplicates a plan event (adopt it as deprecated, with `replaced_by`); it is a rename of a plan event (don't adopt it; a finding); it still arrives from old app versions or old code (a finding: "still arriving from old clients"); nobody needs it (don't adopt it; a finding).
- **The team's naming.** Compare the code's main naming style and the "Naming and existing events" section of `context.md` with `_conventions.yaml`. If they differ, don't edit `_conventions.yaml` (only the team changes it). Raise an open question with the pattern you recommend.

## 6. Compare

Call `match` with the snapshot and the scope you were asked for (events, area, or since). It puts every event and user property in one bucket. For each item that isn't healthy, open the code and the data and explain why, with evidence. Then run the data checks in `datalyze-method.md`: volume breaks against the git history, funnel sanity, double-firing, near-duplicate names, fill rate, identity health. Where `match`'s bucket disagrees with the call's age from git, use the age. Group items that share one cause into one finding.

Then look for what isn't tracked at all. List the product's core actions (sign-up, login, each onboarding screen, the first value moment, creating or changing the main thing the product is for, money) and its server outcomes with no person present. Each core action with no event is a finding with a proposed event. Other untracked actions get a proposed event only within the budget per area; the rest are open questions.

## 7. Have the verifier try to disprove every finding

Give the `verifier` agent the findings, numbered, with the code and data each one cites, and nothing about how you reached them. Keep what it confirms. Turn what it can't decide into an open question. Drop what it disproves, and say so in the report.

## 8. Write the audit

**`audit.json`** (`audit-format.md`) is what the Datalyze UI reads. Check it with `check_audit`. If the check fails, fix the file from the error message and check again. Never discard it.

**`audit.md`,** next to it, for people:

1. **Summary.** Health, counts by bucket, the three things to fix first.
2. **How this product tracks today.** SDKs, wrappers, client and server, identity calls, and anything the scan can't see, with why.
3. **Findings,** most important first, each with its evidence and fix.
4. **Fix plan by product area.** Money and sign-up first, then core actions, then the rest. Each area becomes one fix pull request when its findings are picked.
5. **Open questions,** each with options the person can pick in the Datalyze UI and your recommendation. Never ask them to send you something, run something, or change a setting.

On the first audit, finish the plan last: fix what can be fixed in the plan, run `vocabulary` to write `_properties.yaml` from the plan, add a `synonyms:` entry for every property spelled two ways (`kb_id: knowledge_base_id`), run `vocabulary` again (it keeps synonyms), then `baseline` to accept the remaining problems, so later changes are judged only on what they add. Then `validate` must pass.

Then stop. Tell the person what you wrote and the three things to fix first.
