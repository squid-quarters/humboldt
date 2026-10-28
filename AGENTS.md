# AGENTS.md

humboldt reports scam links that arrive over chat apps (Telegram, WhatsApp, SMS) to
the people who can take them down. Before changing behaviour or direction, read
[`docs/background.md`](docs/background.md) (why the project exists and what
matters to the owner) and [`docs/design.md`](docs/design.md) (the safety design,
the milestones and the decisions already made). Current status is in `README.md`.

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
  or phishing URL — not to check it, not in a test. If you need something from
  one, ask the owner to get it for you.
- Use only reserved names in fixtures and examples (`.test`, `.invalid`,
  `example.com`). Write any real link defanged: `hxxps://evil[.]test`.
- Don't run humboldt on real input; run it on fixtures. Real input is only for an
  installed release build, from M3 on (see M3 in the design).
- The safety layers in the design are binding on the code: scam text arrives on
  stdin and is never put in argv, logged or echoed; every output goes through the
  one defanging function; network code lives only in the egress module and never
  takes its host from a scam link or an API response; humboldt starts no
  processes.
- M0 lands the Claude Code fence (sandbox network allowlist and hook) before any
  other code.
- That fence covers Claude Code only. Any other agent runs in its own sandbox with
  network access disabled, so each command that needs the network (package
  installs, `gh`) goes to the owner for approval. If your network access isn't
  off, stop and tell the owner before doing anything else. Until M0 lands, Claude Code
  follows the same rule. Never use ad-hoc network tools (`curl`, `wget`,
  `node -e`, `python3 -c`) or web-fetch and browser tools in this repo.
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
