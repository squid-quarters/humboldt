# humboldt — design (draft)

## Problem

Scam messages reach personal phones over Telegram, WhatsApp and SMS. Reporting a
scam link properly — to the hosting provider, the registrar and the blocklists —
takes ten minutes of lookups per link, so almost nobody does it.

Existing tools don't fit an individual reporting chat scams from their own machine:

| Tool | Why it doesn't fit |
|---|---|
| [phishing.support](https://github.com/samuelscheit/phishing.support) | A hosted service (and MIT code) that takes a URL or an email and reports it from its operator's infrastructure (Tor, a browser fleet, CAPTCHA solving). It doesn't parse chat messages and doesn't run locally. We borrow from its code. |
| [scammerlocker](https://github.com/richardvanorton/scammerlocker) | Registrar-only; sends LLM-written email unreviewed; inactive since 2025-05. GPL-3.0 — no code is taken from it. |
| [abwhose](https://github.com/bradleyjkemp/abwhose) | Contact lookup only; last commit 2022. |
| [Phish Report](https://phish.report/pricing) | Commercial, enterprise pricing. |

humboldt takes **chat-message text**, runs on a laptop with no infrastructure, and
leaves you in control of what is sent in your name.

## Principles

1. **A human sends every email.** humboldt drafts abuse emails; you review and send
   them. Only channels built for automated submission (Netcraft's reporting API)
   are submitted automatically.
2. **Nothing opens a scam link on your machine** — not humboldt, not Claude Code
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
  email, shell history — where one click or a link preview would load it.

### Layers

Each layer covers a different way a link could be opened; together they cover all
three above.

1. **Scam text never touches the shell.** The message is read from stdin
   (`pbpaste | humboldt report`), never passed as an argument, so the shell can't
   expand anything in it and it never lands in shell history. humboldt never
   echoes the raw input.
2. **Every output is defanged at one choke point.** A link is parsed into a
   `ScamUrl` value with no method that returns the live form. Everything humboldt
   writes — terminal, ledger, drafted emails — passes through one output function
   that defangs every URL-like or hostname-like substring (`tg-verify[.]example`,
   `hxxps://…`), including bare domains, `www.` forms, URLs nested in query strings,
   and URLs that come back from APIs (urlscan's final URL, Netcraft's status), and
   strips control characters so no terminal hyperlink escape gets through. Drafted
   emails are plain text with no HTML. Only one function produces the live string,
   and only as a JSON field for Netcraft or urlscan — never as a request address.
3. **One egress module, with a fixed allowlist.** All network access goes through a
   single module that connects only to hosts in the code: Cloudflare's
   DNS-over-HTTPS resolver (which doesn't forward your network's subnet to the
   scammer's nameserver), the RDAP servers from a **committed snapshot** of IANA's
   bootstrap file, Netcraft's API and urlscan's API. The destination host is never
   taken from a scam link or from an API response. Redirects and RDAP `related`
   links are not followed. humboldt never starts another process, so nothing can
   hand a link to a browser or `open`.
4. **Claude Code is fenced in.** The repo's Claude Code project settings turn on
   the sandbox with a network allowlist (package registries and GitHub), so no
   command Claude runs here — `curl`, `node -e`, `python3`, anything — can reach
   another host. Deny rules for `WebFetch`, browser tools and `agent-browser`, plus a
   `PreToolUse` hook that blocks tool calls containing a live URL outside the
   allowlist, are the backstop. Test fixtures use only reserved names (`.test`,
   `.invalid`, `example.*`). This fence covers Claude Code only; don't use other
   coding agents on this repo without an equivalent.

### Tests that prove it

- **Egress recorder.** The egress module takes an injected transport. Tests run the
  whole pipeline with a recording transport and assert that every destination is on
  the allowlist and none is a host extracted from the message. Shown to fail once by
  planting code that fetches the scam URL — a safety test that has never failed
  proves nothing.
- **Output never contains a live link.** Every output — stdout, ledger, `.eml`
  drafts — is checked for anything a terminal or mail app would make clickable,
  bare domains included.
- **Allowlist bypass attempts are rejected:** IP literals,
  `https://urlscan.io@evil.test`, look-alike hosts, a trailing dot.
- **Link forms the extractor must catch:** `hxxp`, `[.]`, `(dot)`, missing scheme,
  `www.` prefixes, nested URLs and zero-width characters.
- **Static check:** oxlint's `no-restricted-imports` (`net`, `http`, `https`, `dns`,
  `tls`, `dgram`, `child_process`, `worker_threads`, `undici`) and
  `no-restricted-globals` (`fetch`, `WebSocket`, `EventSource`) outside the egress
  module. CI fails otherwise.
- **Claude Code fence:** the hook blocks a live scam-style URL in a `WebFetch`,
  `Bash` and browser tool call and lets a defanged one through; a sandboxed command
  to a non-allowlisted host fails. Shown to fail once with the hook disabled.

Considered and left out as overkill for now: a runtime tripwire wrapping Node's
network APIs, and running humboldt in a container behind an allowlisting
[Squid](https://www.squid-cache.org/) proxy. Either can be added later if the tool
grows beyond one person's laptop.

### What the risk actually is

For calibration, not reassurance: on an up-to-date browser, merely loading a phishing
page usually does little damage on its own. The real harm comes from typing
credentials or codes into it, or downloading and running a file. Loading it does
confirm to the scammer that a real person clicked and reveals your IP address — reason
enough to never do it. The layers are sized to that risk.

## Usage (v0)

```console
$ pbpaste | humboldt report     # copy the scam message, then run this
$ humboldt status               # re-check open reports
```

## Pipeline

1. **Extract.** Pull URLs, Telegram handles (`@user`, `t.me/…`), WhatsApp links
   (`wa.me/…`) and phone numbers from the text. Refang `hxxp`, `[.]`, `(dot)`.
   Deduplicate.
2. **Classify each target.**
   - Web URL → the full pipeline below.
   - Telegram account → in-app *Report → Scam*; `@notoscam` only when it
     impersonates you (manual; humboldt prints the steps).
   - WhatsApp number / phone number → in-app report + ScamShield (manual).
3. **Enrich a web URL** (read-only):
   - DNS: resolve A/AAAA via DoH.
   - RDAP for the domain → registrar and its abuse email. Some TLDs (e.g. `.de`,
     `.io`, `.ru`) have no RDAP server; humboldt prints manual lookup steps instead.
   - RDAP for each IP → hosting org and its abuse email.
   - CDN check: if the IP belongs to Cloudflare (or another CDN), the host contact
     is the CDN's abuse form, not an email.
   - urlscan: submit as **unlisted**; keep the result link, screenshot and final URL
     (after redirects, defanged).
4. **Report.**

   | Channel | How | Needs |
   |---|---|---|
   | Netcraft | API v3 `POST /report/urls`, auto | Reporter email only (per phishing.support's live integration — verify against Netcraft's docs) |
   | urlscan | API, auto (evidence, not a report) | Free account + API key |
   | Google Safe Browsing | Manual; humboldt prints the form address and the defanged link | — |
   | PhishDestroy | Manual: `@PhishDestroy_bot` on Telegram (web reporting paused as of 2026-10-27) | — |
   | Hosting provider | Drafted email or CDN form link | You send it |
   | Registrar | Drafted email, after the host | You send it |

5. **Record** every target × channel in a local SQLite ledger: when, what was sent,
   the provider's reference (e.g. Netcraft submission ID), and status.

`humboldt status` re-checks open reports: does the domain still resolve, what does
a fresh urlscan show, and what does Netcraft's submission status say.

## Borrowed from phishing.support

Code reused from phishing.support keeps its MIT copyright and permission notice in
`THIRD_PARTY_NOTICES`, with the source commit SHA it was taken from.

- Netcraft v3 client: submission body `{ email, reason, urls: [{ url }] }`; strict
  validation of the returned submission ID; build the status URL ourselves, never
  follow a provider-supplied one.
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

- **M0 — safety first:** the Claude Code fence (layer 4) lands before any code.
  Then stdin input, the output choke point, `ScamUrl` and the egress module with
  their tests. Then extract, classify, DNS/RDAP enrichment and a printed report
  plan. No reports are sent.
- **M1 — report:** urlscan evidence, Netcraft submission, drafted host/registrar
  emails (`.eml` files to open in your mail app).
- **M2 — follow up:** SQLite ledger, `humboldt status`, duplicate prevention.
- **M3 — Claude Code skill:** Claude runs `pbpaste | humboldt report` (no link in the
  command, so the fence stays intact) and helps word the drafts.

## Decisions

- **Licence:** MIT.
- **Visibility:** public from day one.
- **Reporter email:** a dedicated project address, set in local configuration and
  never committed, so the public repo doesn't hand it to scrapers.

## Open questions

1. **Netcraft terms and rate limits.** Their API docs need JavaScript; read them in a
   browser before M1.
2. **urlscan quotas** for a free account — check before M1. Also whether an abuse
   desk without a urlscan account can open an unlisted result link; if not, attach
   the screenshot to the email instead.
3. **`.eml` drafts in Apple Mail** may open as received messages, not drafts;
   `X-Unsent: 1` may fix it. Check in M1.
4. **Screenshots as input.** Many chat scams are images. OCR input is a possible
   later milestone.
