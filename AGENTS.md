# AGENTS.md

humboldt reports scam links that arrive over chat apps (Telegram, WhatsApp, SMS) to
the people who can take them down. Read [`docs/design.md`](docs/design.md) before
changing behaviour: it holds the motivation, the safety design, the milestones and
the decisions already made. Current status is in `README.md`.

## Why it exists

- The owner gets scam messages on their personal phone and wants the links taken
  down, not just blocked. The existing tools are email-centric, hosted, commercial
  or abandoned (see the table in the design doc).
- It runs locally and keeps a human in the loop on purpose. Unreviewed, automated
  abuse email is what gets reporters ignored, so humboldt drafts and the owner
  sends. Only channels built for automation (Netcraft's API) are submitted
  automatically.

## Never open a scam link

This is the project's top rule, and it binds every agent on every task:

- Never fetch, browse, screenshot, `open` or resolve (DNS, WHOIS, RDAP) a real scam
  or phishing URL — not to check it, not in a test. Ask the owner if you think you
  need to.
- Use only reserved names in fixtures and examples (`.test`, `.invalid`,
  `example.com`). Write any real link defanged: `hxxps://evil[.]test`.
- Don't run humboldt on real input; run it on fixtures. Real input is only for an
  installed release build, from M3 on.
- Network code lives only in the egress module and talks only to allowlisted
  hosts. The host it connects to never comes from a scam link or an API response.
- The Claude Code fence (sandbox network allowlist and hook) covers Claude Code
  only, and it doesn't exist until M0 lands it. Any other agent, and Claude before
  then, works with network access off and asks before each install or `gh` call.
- The owner wants this safe, not over-engineered. Don't add layers the design
  rejected (a runtime tripwire, a container proxy) without asking.

## Public repo

- Never commit the reporter email address, API keys, or real scam messages or
  links. Secrets come from environment variables or the macOS keychain.
- Code reused from phishing.support (MIT) keeps its copyright and permission
  notice in `THIRD_PARTY_NOTICES`, with the source commit SHA. Take no code from
  scammerlocker (GPL-3.0).

## Workflow

- Every change goes through a pull request to `main`. Get an independent review
  before merging, re-review after fixes, and record the reviewed head SHA on the PR.
- Conventional Commits.
- When a change alters the design, update `docs/design.md` in the same PR.
