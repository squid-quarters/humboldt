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
2. **Never load a scam page from your own IP.** All page fetching goes through
   urlscan. humboldt itself only does DNS and RDAP lookups.
3. **Quality over volume.** Every report carries evidence (urlscan screenshot and
   result link). The same target is never reported to the same channel twice.
4. **Aggregators first, then host, then registrar.** Netcraft and Safe Browsing
   reach the most people fastest; the host can pull a page in hours; the registrar
   suspends the whole domain and is slowest.

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

- **M0 — look, don't touch:** extract, classify, DNS/RDAP enrichment, printed report
  plan. No network writes.
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
