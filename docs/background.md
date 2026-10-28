# Background: why humboldt exists

This is the owner's context for the project: how it started, what it is for, what
it must never do, and where it is heading. [`design.md`](design.md) says *how*
humboldt works; this file says *why*. Read both before proposing a change in
direction.

## How it started

On 2026-10-27 the owner received a Telegram message from someone posing as
"Telegram official", threatening to freeze their account. Their first instinct was
to retaliate: they hate scammers and wanted to be as irritating to that account as
possible, within the law, so they started spamming it with stickers.

That turned out to be the wrong lever:

- Spamming a scam account mostly risks your own account being limited for spam,
  and it costs the scammer nothing.
- What actually hurts a scam is taking its link down. A link that is blocked or
  pulled protects everyone else who got the same message, and the scammer has to
  pay for a new domain and start again.

The better first step is to report the account in the app as a scam and block it.
The owner wanted to do more than that, and went looking for a way.

## What the owner wants

The owner gets these messages on their personal phone, over Telegram, WhatsApp and
SMS, and wants to fight back. They looked at the scambaiting community and chose
its disruption side: **phishing reporting and takedown requests**, not talking to
scammers or wasting their time.

Reporting a link properly means finding the host, the registrar and the right
blocklists, collecting evidence and writing to each of them. That is about ten
minutes of lookups per link, so almost nobody does it. humboldt exists to make
that fast enough to do every time a scam message arrives.

## Why build something new

Before building anything we checked what already exists, so we could use it, learn
from it or contribute to it instead. The tools and why each one doesn't fit are in
the [design doc's table](design.md#problem). In short:

- Every existing tool starts from a URL or a phishing **email**. None of them takes
  the text of a **chat message**, which is how these scams reach the owner.
- The good ones are hosted services or commercial; the local ones are
  abandoned or send unreviewed email.
- We learn from phishing.support (MIT), and reuse its code where it helps, with
  its notice kept. Community channels such as Netcraft and PhishDestroy are
  where reports go; humboldt feeds them rather than competing with them.

The owner decided to open-source the result under their `squid-quarters`
organisation, so others who get the same messages can use it. humboldt may be the
first of a small suite of anti-scam tools there.

## The name

Phishing is fishing for people, and the organisation is squid-themed. Humboldt
squid hunt fish in packs; this one hunts phish.

## What matters to the owner

These are the owner's priorities, in their words where possible. They outrank
convenience.

1. **Nothing opens a scam link on the owner's machine.** The owner is "very
   fearful of using this tool/ai and it accidentally opens links on my PC". This is
   the project's top rule, and it applies to coding agents working on humboldt as
   much as to humboldt itself. See [`AGENTS.md`](../AGENTS.md) and the safety
   section of the design.
2. **But don't overdo it.** The owner accepts that opening a link is seldom
   dangerous in itself: "we'll keep this secure (lets not overdo it though)". The
   design's safety layers were chosen to that standard, and the heavier options (a
   runtime tripwire, a container with a proxy) were rejected as overkill. Don't add
   them back without asking.
3. **A human stays in the loop.** humboldt drafts abuse emails; the owner reviews
   and sends them. Unreviewed automated abuse mail is what gets reporters ignored,
   and every report goes out in the owner's name.
4. **Quality over volume.** One well-evidenced report to the right desk beats ten
   noisy ones. No duplicates, no bulk reporting.
5. **Stay within the law and keep the owner safe.** No hacking back, no touching
   the scammer's systems beyond public lookups, no doxxing, no harassment. Never
   share real personal details with a scammer, and never move money for one.
6. **Keep personal details out of the public repo.** The repo is public, so the
   reporter email address, API keys and real scam messages never go in it. The
   reporter address lives in local configuration only.

## Decisions so far

All made by the owner on 2026-10-27.

| Decision | Choice |
|---|---|
| Build or reuse | Build a new tool, learning from and reusing phishing.support |
| Home | A new repo, `squid-quarters/humboldt`, separate from the owner's site |
| Name | humboldt |
| Licence | MIT |
| Visibility | Public from day one |
| Reporter email | A dedicated project address, kept out of the repo |
| Safety | Secure, but not over-engineered |

## Where it is heading

The milestones are in the [design doc](design.md#milestones). In short:

- **M0:** the safety fence for coding agents first, then the parts that read a
  message and work out where to report it, printing a plan without sending
  anything.
- **M1:** evidence from urlscan, automatic Netcraft reports and drafted emails to
  hosts and registrars.
- **M2:** a local record of what was reported, follow-up checks and no
  duplicates.
- **M3:** a Claude Code skill, so the owner can paste a scam message and have
  Claude run humboldt and help word the drafts.

Later ideas, not yet decided: reading screenshots (many chat scams are images),
and more tools in the same organisation.
