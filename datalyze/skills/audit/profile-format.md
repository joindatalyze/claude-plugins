# The tracking profile

`_tracking_profile.yaml`, in the registry folder, says how this codebase sends analytics. `scan` only counts calls the profile describes, and `implement` writes new calls by copying its examples. Nothing is generated: the team keeps its own wrappers.

```yaml
version: 1
tool: mixpanel                    # mixpanel | posthog | amplitude, from .datalyze/config.yaml
stack:
  - { path: ., language: typescript, framework: nextjs, sides: [client, server] }
sdks:
  - { package: mixpanel-browser, side: client }
  - { package: mixpanel, side: server }
wrappers:                         # the only files allowed to call the SDK; not scanned themselves
  - { path: lib/analytics.ts, side: client, import: "import { track, identify } from '@/lib/analytics'" }
  - { path: lib/server-analytics.ts, side: server }
calls:                            # every way the code sends, each with a real example
  - { callee: track, from: '@/lib/analytics', side: client, adds: [platform],
      example: "track('post_created', { post_id: post.id })", ref: app/posts/new.tsx:41 }
  - { kind: identify, callee: identify, from: '@/lib/analytics', example: "identify(user.id)", ref: lib/auth.ts:20 }
  - { kind: user_set_once, callee: setOnce, from: '@/lib/analytics', example: "setOnce({ signup_date })", ref: lib/auth.ts:24 }
direct_sdk: fail                  # fail: an SDK call outside a wrapper fails the scan. allow: no wrapper; the team calls the SDK directly
paths: { server: [worker/], client: [web/] }
sdk_exceptions:                   # reviewed SDK calls that aren't product analytics
  - { file: app/booked/page.tsx, pattern: "gtag\\('event', 'conversion'", reason: Google Ads conversion tag }
dynamic_names:                    # every name a runtime-built call can send, when the code shows them all
  - { file: lib/steps.ts, names: [checkout_started, checkout_completed], why: picks the name from STEP_EVENT by step }
extensions: [.py]                 # optional: more file types to open, beyond _conventions.yaml's scan.extensions
notes: ["lib/analytics.ts registers utm_source and utm_campaign on load"]
```

## A call entry

- **`callee`:** how the call reads in code (`track`, `mixpanel.track`, `analytics.capture`). With **`from`** (the module the code imports it from, `@/lib/analytics` or `mixpanel-browser`), the scanner follows renamed imports, default and namespace objects and local aliases, so one entry covers every way the code reaches that function. A global SDK object needs no `from` (`{ callee: mixpanel.track }` also matches `window.mixpanel.track`).
- **`kind`:** `event` (the default), `user_set`, `user_set_once`, `user_increment`, `user_append`, `user_unset`, `group_set` (with `group`, a key in `_groups.yaml`), `identify`, `reset` or `alias`.
- **`name_arg`, `props_arg`:** argument positions when they aren't 0 and 1 (user properties default to 0).
- **`name_key`:** for a call that takes one object and reads the name from a key in it (`dataLayer.push({ event: 'sign_up', plan })` is `{ callee: dataLayer.push, name_key: event }`).
- **`name_template`:** for a wrapper that builds the name from its argument (`trackDocs('opened')` sends `docs_opened`): `docs_{name}`.
- **`adds`:** properties the wrapper always adds itself, so they count as sent on every call. Read the wrapper's body for them.
- **`side`:** `client` or `server`, for a function that only runs on one side, when the calling file doesn't say.
- **`example`, `ref`:** a real call from the code and its `file:line`.
- **Vue, Svelte, Astro, HTML and Liquid files:** their `<script>` sections (and an Astro page's front matter) are read as JavaScript, at their own lines, so the same `callee` entries cover them. A handler written in the markup itself (`@click="track(...)"`, `onclick`) isn't read: report it as a blind spot.
- **Swift, Kotlin and Dart are parsed too:** use `callee`, the call's dotted path with the calls in the chain left out (`Mixpanel.mainInstance().track(...)` is `Mixpanel.mainInstance.track`; `*` stands for any one name: `AnalyticsManager.*.track`). `name_label` and `props_label` name the argument labels when they aren't `event` and `properties`.
- **Python is parsed too:** use `callee` for the team's own functions (`{ callee: track_event, extensions: [.py] }`, or `{ callee: analytics.track, from: app.analytics }` with the dotted module it's imported from). Arguments are read by position (`name_arg`, `props_arg`) or by keyword (`name_label`, `props_label` name the keyword). A `callee` that is a vendor's own call (`mp.track`, `posthog.capture`) is read the way that library takes its arguments, unless you give positions. With `direct_sdk: allow`, Mixpanel's `track` and `people_*`, PostHog's `capture` (old and new forms), `set` and `set_once`, Segment's `track` and `identify` and Amplitude's `BaseEvent` are read without an entry. Python runs on the server unless `paths.client` says otherwise. Amplitude's `Identify()` user property writes aren't read yet.
- **Other languages** (Ruby, Go, Java): `pattern`, a regular expression whose group 1 is the event name, with the file `extensions` it applies to. It runs over the whole file, so `\s` crosses line breaks and a call split over lines is found at its first line; `^` and `$` still mean a line's start and end. The scan opens every file type a call's `extensions` names, and the profile's top-level `extensions` list, so you don't need to touch `_conventions.yaml` to read them.

## Checking it

Run `scan` after writing the profile. It must find every call you found by reading the code. Look at what it lists as event names built at runtime and as SDK calls outside a wrapper. For each one, either make it readable (a second-level wrapper whose callers pass literal names becomes a call entry of its own; a file that is really a wrapper goes in `wrappers`; a name map goes in `dynamic_names`), or explain it in the audit report as a blind spot.
