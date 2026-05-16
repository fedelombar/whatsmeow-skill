# whatsmeow-skill

A [Claude Code skill](https://code.claude.com/docs/en/skills) for the [whatsmeow](https://github.com/tulir/whatsmeow) Go library (`go.mau.fi/whatsmeow`).

When installed, it teaches Claude the canonical API patterns, JID rules, event handling, and gotchas — so it stops hallucinating WhatsApp library APIs in your Go projects.

## Install

The repo IS the skill. Clone it directly into your Claude Code skills directory:

### Personal — available in every project

```bash
git clone https://github.com/fedelombar/whatsmeow-skill ~/.claude/skills/whatsmeow
```

### Project-scoped — committed into one repo

```bash
cd your-go-project
git clone https://github.com/fedelombar/whatsmeow-skill .claude/skills/whatsmeow
# optional: remove the inner .git if you want to commit it as your own files
rm -rf .claude/skills/whatsmeow/.git
git add .claude/skills/whatsmeow
```

Either way, the skill activates automatically when Claude encounters a Go project importing `go.mau.fi/whatsmeow`. You can also invoke it manually with `/whatsmeow` after it loads.

> **Skill name vs directory name:** Claude Code uses the *directory name* as the skill name unless the `name` field in `SKILL.md` overrides it. This skill's frontmatter declares `name: using-whatsmeow`, so you can rename the clone directory to anything you like (`whatsmeow`, `wa`, etc.) — the skill name stays `using-whatsmeow`.

## What this skill covers

- **Client setup & connection** — `sqlstore.Container` → `store.Device` → `whatsmeow.Client` → `Connect`, plus QR and phone-code pairing.
- **Sending messages** — `SendMessage`, JID construction, text/media/reply/reaction/edit/revoke, mentions, typing indicators, read receipts.
- **Receiving events** — `AddEventHandler`, the event-type catalog, downloading media, and the `RemoveEventHandler` deadlock.
- **Groups & contacts** — group operations, contact store, business profiles.
- **Cross-cutting gotchas** — LID JIDs, AD-JIDs, `AutoTrustIdentity`, sentinel errors, `HistorySync`.

Every code pattern traces back to a `file:line` reference in the whatsmeow source so it stays honest.

## File layout

```
whatsmeow-skill/
├── SKILL.md            # Main skill entrypoint (frontmatter + quickstart + navigation)
├── connecting.md       # Container/device/client bootstrap, QR + phone pairing
├── sending.md          # SendMessage, JIDs, media, mentions, MarkRead, SendChatPresence
├── events.md           # AddEventHandler, event catalog, downloading media
├── groups-contacts.md  # Group operations, contact store, business profiles
└── gotchas.md          # LID JIDs, AD-JIDs, identity trust, sentinel errors
```

Follows the [official Claude Code skills layout](https://code.claude.com/docs/en/skills): `SKILL.md` at the root with companion markdown files alongside, loaded on demand.

## Versioning

whatsmeow has no formal API stability guarantee. The skill cites specific `file:line` locations; if you upgrade whatsmeow significantly, spot-check that the citations still resolve.
