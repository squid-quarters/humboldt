# humboldt — design (draft)

## Problem

Scam messages reach personal phones over Telegram, WhatsApp and SMS. Reporting a
scam link properly — to the hosting provider, the registrar and the blocklists —
takes ten minutes of lookups per link, so almost nobody does it.

Existing tools don't fit an individual on a phone:

| Tool | Why it doesn't fit |
|---|---|
| [phishing.support](https://github.com/samuelscheit/phishing.support) | Email-centric (`.eml` intake); needs Tor, a browser fleet and CAPTCHA solving. MIT — we borrow from it. |
| [scammerlocker](https://github.com/richardvanorton/scammerlocker) | Registrar-only; sends LLM-written email unreviewed; inactive since 2025-05. |
| [abwhose](https://github.com/bradleyjkemp/abwhose) | Contact lookup only; inactive since 2023. |
| [Phish Report](https://phish.report/pricing) | Commercial, enterprise pricing. |

humboldt takes **chat-message text**, not email, and runs on a laptop with no
infrastructure.

## Principles

1. **A human sends every email.** humboldt drafts abuse emails; you review and send
   them. Only channels built for automated submission (Netcraft's reporting API)
   are submitted automatically.
2. **Nothing ever opens a scam link on your machine** — not humboldt, not the AI
   working on it, not you by accident. urlscan loads pages on its own servers.
   This is the project's top requirement; see [Safety](#safety-nothing-opens-a-link).
3. **Quality over volume.** Every report carries evidence (urlscan screenshot and
   result link). The same target is never reported to the same channel twice.
4. **Aggregators first, then host, then registrar.** Netcraft and Safe Browsing
   reach the most people fastest; the host can pull a page in hours; the registrar
   suspends the whole domain and is slowest.

## Safety: nothing opens a link

### What "opening" means here

Any of these counts as a failure:

- an HTTP(S) request, DNS query or any other connection from your machine to a host
  the scammer controls;
- a scam link handed to a browser, `open`, Quick Look or anything else that renders it;
- a scam link written anywhere as a *live* link — terminal output, a file, a drafted
  email — where one click or a link preview would load it.

### Layers

Each layer works on its own; any one of them holding is enough.

1. **Links are stored and shown defanged.** A link is parsed once into a `ScamUrl`
   value with no method that returns the live form — it prints as
   `hxxps://tg-verify[.]example/login` everywhere: terminal, ledger, logs, drafted
   emails. Only one function can produce the live string, and only to hand to an
   allowed reporting API (Netcraft, urlscan) as a JSON field — never as a request
   address. A drafted email carries the link defanged, in plain text, with no HTML.
2. **One egress module, with a fixed allowlist.** All network access goes through a
   single module that connects only to hosts written into the code: the DNS-over-HTTPS
   resolver, IANA's RDAP bootstrap and the RDAP servers it lists, Netcraft's API and
   urlscan's API. The destination host is never taken from a scam link. Redirects are
   not followed unless the next host is also on the allowlist. A DNS lookup of the
   scam domain goes to the DoH resolver, so the scammer's nameserver sees the
   resolver, not you.
3. **Runtime tripwire.** At startup humboldt wraps Node's `fetch`, `http`, `https`,
   `net`, `tls`, `dns` and `child_process` so that any connection to a host outside
   the allowlist, and any attempt to start a process, throws and aborts the run.
   (Node's own permission model will be checked as an extra layer, but not relied on.)
4. **It runs in a container behind an allowlisting proxy.** The default way to run
   humboldt is a container with no browser and no `open` command, whose only network
   route is a [Squid](https://www.squid-cache.org/) proxy (yes, really) allowing the
   same hosts as layer 2. Even a bug in layers 1–3 cannot reach a scam host or open
   anything on your Mac.
5. **The AI is fenced in too.** The repo ships Claude Code project settings that, in
   this repo, deny `WebFetch`, every browser tool, and `curl`, `wget`, `open` and
   `nc` in Bash, plus a `PreToolUse` hook that blocks any tool call whose input
   contains a live (non-defanged) URL to a host outside the allowlist. Test fixtures
   use only reserved names (`.test`, `.invalid`, `example.*`), so the AI never needs
   to see a real scam link while working on the code.

### Tests that prove it

Each of these must be shown to **fail** when its layer is deliberately broken; a
safety test that has never failed proves nothing.

- **Honeypot, end to end.** The test suite runs a trap server and a trap DNS name.
  It feeds messages pointing at them, in every form below, through the whole
  pipeline with real (allowlisted) network code, then asserts the trap saw **zero**
  connections and zero DNS queries.
- **Output never contains a live link.** Property-based tests generate messages with
  links in many shapes and assert that no output — stdout, ledger, logs, `.eml`
  drafts — contains anything a terminal or mail app would turn into a clickable link.
- **Allowlist bypass attempts are rejected,** each as its own test: IP literals (v4,
  v6, decimal, octal), `https://urlscan.io@evil.test`, look-alike and IDN hosts, a
  trailing dot, mixed case, redirects to outside hosts, and DNS answers pointing at
  private or loopback addresses.
- **Link forms the extractor must catch:** `hxxp`, `[.]`, `(dot)`, `[:]//`, missing
  scheme, `www.` prefixes, Unicode look-alikes, zero-width characters, and links
  split across lines.
- **Static checks:** no module outside the egress module imports a network or
  process API; no dependency is added that makes network calls on its own (link
  previews, update checkers, telemetry). CI fails on either.
- **AI fence:** tests feed the `PreToolUse` hook a live scam-style URL in a `WebFetch`,
  `Bash` and browser tool call and assert each is blocked, and that a defanged URL
  passes.
- **Container fence:** a CI job runs humboldt in its container and asserts that a
  request to a non-allowlisted host fails at the proxy.

### What the risk actually is

For calibration, not reassurance: on an up-to-date browser, merely loading a phishing
page usually does little damage on its own. The real harm comes from typing
credentials or codes into it, or downloading and running a file. Loading it does
confirm to the scammer that a real person clicked and reveals your IP address — reason
enough to never do it, and the layers above make sure humboldt can't.

## Usage (v0)

```console
$ humboldt report "Telegram: your account will be frozen. Verify at hxxps://tg-verify[.]example/login"
$ humboldt status          # re-check open reports
```

## Pipeline

1. **Extract.** Pull URLs, Telegram handles (`@user`, `t.me/…`), WhatsApp links
   (`wa.me/…`) and phone numbers from the text. Refang `hxxp`, `[.]`, `(dot)`.
   Deduplicate.
2. **Classify each target.**
   - Web URL → the full pipeline below.
   - Telegram account → report to `@notoscam` (manual; humboldt prints the steps).
   - WhatsApp number / phone number → in-app report + ScamShield (manual).
3. **Enrich a web URL** (read-only):
   - DNS: resolve A/AAAA.
   - RDAP for the domain → registrar and its abuse email.
   - RDAP for each IP → hosting org and its abuse email.
   - CDN check: if the IP belongs to Cloudflare (or another CDN), the host contact
     is the CDN's abuse form, not an email.
   - urlscan: submit as **unlisted**; keep the result link, screenshot and final URL
     (after redirects).
4. **Report.**

   | Channel | How | Needs |
   |---|---|---|
   | Netcraft | API v3 `POST /report/urls`, auto | Reporter email only (per phishing.support's live integration — verify against Netcraft's docs) |
   | urlscan | API, auto (evidence, not a report) | Free account + API key |
   | Google Safe Browsing | Manual form; humboldt opens it with the URL | — |
   | PhishDestroy | Manual: `@PhishDestroy_bot` on Telegram (web reporting paused as of 2026-10-27) | — |
   | Hosting provider | Drafted email or CDN form link | You send it |
   | Registrar | Drafted email, after the host | You send it |

5. **Record** every target × channel in a local SQLite ledger: when, what was sent,
   the provider's reference (e.g. Netcraft submission ID), and status.

`humboldt status` re-checks open reports: does the domain still resolve, what does
a fresh urlscan show, and what does Netcraft's submission status say.

## Borrowed from phishing.support (MIT, credited)

- Netcraft v3 client: submission body `{ email, reason, urls[] }`; strict validation
  of the returned submission ID; build the status URL ourselves, never follow a
  provider-supplied one.
- Never automatically re-POST a report after a network error or a 5xx response —
  a lost response may still have been a delivered report.
- Per-report, per-channel state instead of one status for the whole report.

## Stack

TypeScript on Node 22, pnpm, vitest, oxlint/oxfmt — the same toolchain as the rest
of squid-quarters, and it lets us reuse phishing.support's TypeScript directly.
Secrets (urlscan key, reporter email) come from environment variables or the macOS
keychain, never the repo.

## Non-goals

- Sending email automatically.
- Automating web forms or solving CAPTCHAs.
- Anything that touches the scammer's systems beyond public DNS/RDAP lookups.
- Bulk reporting from lists or feeds — humboldt reports what *you* received.
- Scambaiting (talking to scammers).

## Milestones

- **M0 — safety first:** the Claude Code fence (layer 5) lands in the first commit,
  before any code. Then the `ScamUrl` type, egress module, runtime tripwire and
  container, each with its tests (including the honeypot), proven by breaking them.
  Then extract, classify, DNS/RDAP enrichment and a printed report plan. No reports
  are sent.
- **M1 — report:** urlscan evidence, Netcraft submission, drafted host/registrar
  emails (`.eml` files to open in your mail app).
- **M2 — follow up:** SQLite ledger, `humboldt status`, duplicate prevention.
- **M3 — Claude Code skill:** paste a message into Claude Code; it runs humboldt and
  helps word the drafts.

## Open questions

1. **Licence.** MIT (matches phishing.support, simplest for reuse)?
2. **Visibility.** Public from day one, or private until M1 works?
3. **Reporter email.** A dedicated address (e.g. an alias) keeps provider replies and
   any scammer attention away from your personal inbox. Which one?
4. **Netcraft terms and rate limits.** Their API docs need JavaScript; read them in a
   browser before M1.
5. **urlscan quotas** for a free account — check before M1.
6. **Screenshots as input.** Many chat scams are images. OCR input is a possible
   later milestone.
