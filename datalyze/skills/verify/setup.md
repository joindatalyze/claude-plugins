# Verification setup (web)

Asked once per repo, the first time someone verifies and `.datalyze/config.yaml` has no `verify:` block. `setup_verify` writes everything and checks it. Never write the config, the secrets file or the settings yourself.

## Secrets never pass through you

Passwords, fixed one-time codes, TOTP secrets and bypass tokens are named, never given. You write `password: MEMBER_PASSWORD`; the person puts the value in `.datalyze/verify.local.yaml` themselves. Claude is denied reading that file, and the browser fills a secret by its name. Never ask for a secret's value. If the person pastes one into the chat anyway, don't repeat it or put it anywhere. Tell them it's now in this conversation, so it should be changed, and that the new value goes in the file.

Test accounts only, never a person's real login. A real card is never asked for.

## 1. Where things stand

Call `setup_verify` with no arguments. If the block is already there and passes, skip to step 4.

## 2. The questions

Ask them a few at a time, in plain words. Offer what you can read from the repo (deploy config, `.env.example`, the analytics init) as a suggestion to confirm, never as an answer. If a field is left empty, the run asks the person when it gets there.

1. **Which URL should verification open?** A staging or preview deploy is best. For each: a short name, the URL, and whether it's staging, preview or production.
2. **Which analytics project does that deploy send to?** The tool, the project id and the public token or key the page sends with (it's in the page source). Staging usually sends to a different project from production. Ask whether the read-only key from onboarding can query that project. If it can't, which saved key can: its environment-variable prefix (`read_key: STAGING` means `STAGING_MIXPANEL_SA_USERNAME` and the like). The read key is a secret like a password: `setup_verify` names it in `.datalyze/verify.local.yaml` and the person copies the value in (from their `.env.local`, for example). Without it no event can be confirmed as landed, and server events can't be verified at all.
3. **Is it production?** Only with the person's clear yes after the warning: test traffic lands in real analytics, and on production a run uses test accounts only, leaves the account as it found it, asks before anything that can't be undone, and never pays or messages a real person (SKILL.md, "On production"). The yes goes on that environment: `allow_production: true` in its entry. Production then needs `allow_production: true` and the rule the team already uses to filter test users out (`exclude_test_users: { property, op, value }`).
4. **Can the deploy tell which commit it serves?** For example a response header or a meta tag. Optional.
5. **Is there protection in front of it?** For example Vercel deployment protection or basic auth. Its token or password is a secret: name it (`VERCEL_BYPASS`).
6. **Test accounts, one per role the plan needs.** `setup_verify` says which roles `fires_at.needs` names. For each: the role, how it signs in (`email_password`, `magic_link`, `phone_otp`, `google`, `sso`, `other`), the username, the password's secret name, and the one-time code: `fixed` (a test number's code, as a secret), `totp` (the TOTP secret, as a secret) or `ask` (the person types it each time). If there are several alike for parallel runs, use `pool` and `{n}` in the username. Google and SSO sign-ins can't be automated: the first run asks the person to sign in once in the verification browser.
7. **Payments.** The provider (`stripe`, `razorpay`, `other`, `none`) and `test` or `ask`. Offer the provider's published test values, checked against its docs now, never from memory. Test card numbers and test UPI ids are public, not secrets.
8. **Anything a tester should know** (a coupon that always works, a feature flag to turn on). This goes in `notes`.

## 3. Write it

Call `setup_verify` with `web` holding the answers in this shape:

```yaml
environments:
  - name: staging
    url: https://staging.acme.com
    kind: staging                                   # staging | preview | production
    analytics: { tool: mixpanel, project: "3281844", token: "a1b2…", region: us }
    read_key: STAGING                               # optional
    served_commit: header x-build-sha               # optional: header <name> | meta <name> | none
    bypass: { kind: vercel, secret: VERCEL_BYPASS } # optional; basic_auth adds username
allow_production: false
accounts:
  - { role: member, login: email_password, username: qa+member@acme.com, password: MEMBER_PASSWORD, otp: ask }
payments: { provider: stripe, mode: test, success: { card: "…" }, failure: { card: "…" } }
exclude_test_users: { property: distinct_id, op: starts_with, value: usr_qa_ }   # required with allow_production
notes: Coupon QA10 always works on staging.
```

If it's refused, nothing was written. Fix what each rule says and call it again.

## 4. Tell the person what's left

Repeat what `setup_verify` says, by name:
- each secret still empty, and that they fill it in `.datalyze/verify.local.yaml` themselves (open it in their editor);
- then restart the Claude Code session, or `/mcp` reconnect, because the browser reads the file once when the session starts;
- to commit `.datalyze/config.yaml` and `.datalyze/.gitignore`. Never the secrets file; `.claude/settings.local.json` stays theirs;
- if the sandbox is off, that turning it on (`/sandbox`) also stops scripts from reading the file.

Call `setup_verify` again once they say they've filled it in. It says `Setup is complete` only when the block passes, every secret is set, git ignores the file and reading it is denied.
