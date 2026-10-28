# Background: why humboldt exists

This file records the owner's intent: how the project started, what the owner
asked for, and what they decided. [`design.md`](design.md) says *how* humboldt
works, and [`AGENTS.md`](../AGENTS.md) holds the rules agents follow. When the
owner's intent changes, update this file.

## In short

- humboldt helps the owner get scam links taken down. It does not talk to
  scammers or retaliate against them.
- The owner's top concern is that nothing, including a coding agent, opens a scam
  link on their machine. They also asked that the safety work not be overdone.
- The repo is public. The owner's reporter email address never goes in it.

## How it started

On 2026-10-27 the owner received a Telegram message from someone posing as
"Telegram official", threatening to freeze their account. They hate scammers, and
their first move was to be as irritating to that account as possible, "within
legal limits", by spamming it with stickers.

Claude advised against that. Spamming risks the owner's own account being limited
for spam, and costs the scammer little. Reporting the account in the app and
blocking it is a better first step. Getting the scam link taken down does more
still: it protects everyone else who received the same message.

## What the owner asked for

The owner gets these messages on their personal phone, over Telegram, WhatsApp
and similar apps. They asked how to get involved in the scambaiting community, then
narrowed it down: "I think I want to help with the Phishing reporting and takedown
requests", and asked for help building a setup for it.

Reporting a link properly means finding the host, the registrar and the right
blocklists, collecting evidence and writing to each. That takes about ten minutes
per link, so most people don't do it. humboldt exists to make it quick enough to
do for every scam message that arrives.

## Why build something new

The owner asked whether such a tool already exists, and whether to learn from it,
before building anything. Claude looked; the tools found and why each doesn't fit
are in the [design doc's table](design.md#problem). None of them takes the text of
a chat message, which is how these scams reach the owner. humboldt reuses code
from phishing.support (MIT) where it helps, and sends its reports to existing
channels such as Netcraft and PhishDestroy rather than competing with them.

The owner suggested open-sourcing the work under their `squid-quarters`
organisation, perhaps as a suite of tools, and agreed to a new repo for it.

## The name

The owner asked for a name with something to do with squid, "since phishing is like
fishing", and chose humboldt from Claude's suggestions. The README's line about
Humboldt squid hunting in packs is Claude's.

## What the owner said matters

In the owner's words:

1. **Safety.** "I'm very fearful of using this tool/ai and it accidentally opens
   links on my PC... we need comprehensive tests and systems to avoid that ever
   happening." This is why the project's top rule is that nothing opens a scam
   link, and why that rule binds coding agents as well as humboldt itself.
2. **But not overdone.** After Claude pointed out that opening a link is seldom
   dangerous in itself: "I care about the security, yes you're right opening a link
   is seldom dangerous, but we'll keep this secure (lets not overdo it though)."
   The design's safety layers were chosen to that standard. It left out a runtime
   tripwire and a container proxy as overkill for now; ask the owner before adding
   them.
3. **Legal.** The owner wants to fight scammers "within legal limits".
4. **Privacy.** "Yes, don't put the address in the repo, thank you." The reporter
   email address lives in local configuration only.

The other principles, such as a human sending every email and quality over
volume, are in the [design](design.md#principles). Claude proposed them there;
they stand unless the owner changes them.

## Decisions the owner made

All on 2026-10-27.

| Decision | Choice |
|---|---|
| Where | A new repo under `squid-quarters` |
| Name | humboldt |
| Licence | MIT |
| Visibility | Public |
| Reporter email | Kept out of the repo |
| Safety | Secure, but not over-engineered |

## Where it is heading

The milestones are in the [design](design.md#milestones). The end goal is a Claude
Code skill that lets the owner paste a scam message and have Claude run humboldt
and help word the reports. Reading screenshots, since many chat scams are images,
is an open question.
