---
name: verify
description: Verify analytics events in a real browser with Datalyze. It opens the product, triggers the events a person picked, checks every request the app sends against the tracking plan and looks each event up in Mixpanel or PostHog. Use for "verify the checkout events", "check signup_completed fires", "test our tracking in the browser", "does plan_selected send the right plan", "set up verification", in a repo with a Datalyze tracking plan (a `.datalyze/` folder).
---

# Verify

You reach the moment each event should fire; scripts judge it. Whether an event fired, when, once, with which properties and values is decided by `check_captured` from what the capture service saw leave the browser. Your only judgement is C5: did what the event's name says actually happen on the screen. Never pass payloads along, place marks, or report what you typed: the hooks and the capture service do.

Every failure is "couldn't verify, and why", never a guess. `passed` is the one result that must never be wrong.

## 1. Knowledge and setup

Call `knowledge` first. Then call `setup_verify` with no arguments. No `verify:` block, or it breaks a rule: follow [setup.md](setup.md) and come back. Setup not complete (a secret empty, the file not ignored or not denied): tell the person exactly what's left, and stop.

## 2. What to verify

The person names events (`checkout_completed`) or single properties (`signup_completed.plan`). If they say "everything" or an area, list what that is and confirm. Call `registry` for the plan entries. For each event, read its `fires_at`: each entry is one path to check (`event#0`, `event#1`, …). An event with no `fires_at` gets its places from `scan` and the code, as you would for the plan.

**Group them into journeys:** one walk through the product that reaches several events (one checkout walk covers `checkout_started`, `payment_info_entered`, `checkout_completed`). Group by shared `page`, `from` and `needs`. Journeys that need the same account go one after another. With more than 25 events, say so, and verify them a journey at a time.

## 3. Start the run

1. Pick the environment: the one in `verify.web.environments`, or ask which. Production only if `allow_production` is true.
2. Start capture, which opens a visible Chrome with its own profile for this repo:
   `node "${CLAUDE_PLUGIN_ROOT}/bin/datalyze.js" capture start --environment <name>`
3. Call `check_environment`. If the URL serves a different commit, say so in one line: what's verified is what's deployed. If it doesn't answer, or refuses (401, 403), stop and say why.

The browser tools are `mcp__plugin_datalyze_playwright__*`. Screenshots are off: read the page from its snapshot. `browser_evaluate`, routes, cookie and storage setters are refused during a run. Reach events through the page, or ask the person.

## 4. Each journey

1. **Meet `needs`.** Signed in as the role's account (`needs: [role=member]`): if `paths.json` has a sign-in for it, replay a path with `sign_in: true`; otherwise sign in through the page. Type a password by its secret name: put `MEMBER_PASSWORD` in the password field and the browser fills the value. Google or SSO sign-in: `ask_person` to sign in once in the verification browser. The profile keeps the sign-in for later runs.
2. **A tested path first.** If `paths.json` has a path covering the journey's events, call `replay_path`, one at a time (replays share the browser: a second call waits for the first). It runs without you and checks each labelled step itself. When it stops at a step, take over from the page it returns. When it finishes, judge C5 for each event from the page it shows after that event's step, and call `check_captured` for each with `reached_by: replay` and your `screen`.
3. **Otherwise drive it,** guided by `fires_at` (`page`, `element`, `trigger`, `from`). For each event's step:
   - call `capture_label` with the event and path, right before the action that should fire it. When one action should fire several events (one Create account click sends `signup_started` and, after the API answers, `signup_completed`), label each of them before that one action, then check each. A step taken without its label can't be checked again by repeating it on the same page;
   - take the action (click, type, select, navigate);
   - call `check_captured` right after, before the next action. Give `screen` with your verdict (`ok` or `problem`) and one sentence on what the page showed. `problem` means the page shows the thing the name says didn't happen (`checkout_completed` with an error on screen), never "the event looks wrong": that's the script's to say.
   A `page_load` event's step is the navigation. An `after_api` event's step is the action that sends the request; `check_captured` waits for the response itself.
4. **Stuck?** Ask only when a step needs something you can't do: a one-time code the account sets to `otp: ask`, a real payment, a captcha, an external inbox. Or ask when two different tries at the same step failed. Call `ask_person` with the event and what to do, tell the person in one or two sentences, and end your turn. Their reply ends the step. On "done", call `check_captured` with `reached_by: person`. On "skip", or if they say they can't, it's `waiting_on_person`: put it in `not_done` and go on. Never ask a person whether an event is right.
5. **Save what worked.** When a driven journey reached its events, call `save_path`. Use the steps you took, in the step vocabulary (`verify-format.md`): role and name, label, testid or text, never a snapshot ref; secrets as `$secret:NAME`, values you invented as `$unique_email` / `$unique_text`, test payment values as `$test_card:success.card`. It refuses a path the run didn't prove. A sign-in you did goes in with `account: <role>`.

An event you can't reach after two tries and an ask is `couldnt_reach`: name the step it stopped at in `not_done`.

## 5. Landing, the report, the verifier

1. **Landing:** call `lookup_landed` once, after every journey. `pending` is never a problem: some tools can't confirm yet. Recheck later.
2. **Findings:** for every check that's a `problem`, find why in the code: the call site, what it sends and when. Write a finding (`verify-format.md`) with the code line quoted. Problems with one cause share one finding.
3. Call `assemble_verify` with the selection, the findings, `not_done` and the journeys. Fix whatever `check_verify` reports.
4. **The verifier:** if there are findings, give the `verifier` agent the path to `verify.json` and the finding ids, nothing else. Ask for verify mode. Wait for it. Write its line for each finding into the finding's `verifier` field; correct a cause or a fix it showed was wrong.
5. Call `assemble_verify` again with `final: true`. It commits the report and `paths.json` to `datalyze/verify`, locally, without touching the checkout.
6. Stop capture: `node "${CLAUDE_PLUGIN_ROOT}/bin/datalyze.js" capture stop`. That deletes the run's log.

## 6. Tell the person

Lead with the counts: passed, problems, not checked, not reached, waiting on them. Then each finding in one line, with its id, and what's not done, with what it needs. Then say where the report is (`verify.md`) and that pushing `datalyze/verify` (`git push origin datalyze/verify`) puts it on the Verification screen, where picking findings and pressing Fix starts `implement` with their ids. Don't push it yourself.
