# AGENTS.md

This repository is a single [Claude Code skill](https://code.claude.com/docs/en/skills) for working with the [whatsmeow](https://github.com/tulir/whatsmeow) Go library (`go.mau.fi/whatsmeow`).

## Repository structure

The repo IS the skill — clone the whole thing into a Claude Code skills directory and you're done. There is no `skills/` wrapper, no plugin manifest.

```
whatsmeow-skill/
├── SKILL.md            # Required. Frontmatter (name, description) + entrypoint content
├── connecting.md       # Companion file loaded on demand
├── sending.md          # Companion file loaded on demand
├── events.md           # Companion file loaded on demand
├── groups-contacts.md  # Companion file loaded on demand
├── gotchas.md          # Companion file loaded on demand
├── README.md           # For humans browsing on GitHub
├── AGENTS.md           # For AI agents browsing the repo
└── .gitignore
```

## How Claude Code finds this skill

Claude Code auto-discovers skills under any of these paths (see the [official docs](https://code.claude.com/docs/en/skills)):

| Location | Path |
|---|---|
| Personal (all projects) | `~/.claude/skills/<name>/SKILL.md` |
| Project (one repo) | `.claude/skills/<name>/SKILL.md` |
| Plugin | `<plugin>/skills/<name>/SKILL.md` |

The directory name becomes the skill's invocation slug unless `SKILL.md`'s frontmatter overrides it via `name:`. This skill declares `name: using-whatsmeow`, so the slash command is `/using-whatsmeow` regardless of where you clone it.

## How a Claude session uses this skill

1. On session start, Claude Code reads `SKILL.md`'s frontmatter (`name`, `description`) into its skill registry — the body stays on disk.
2. When the user's message matches the trigger description (working in a Go project that imports `go.mau.fi/whatsmeow`, etc.), Claude loads `SKILL.md`'s full body.
3. `SKILL.md`'s navigation table points to the companion `*.md` files. Claude reads only the ones relevant to the current task (e.g. just `sending.md` when the user asks how to send an image).

This is "progressive disclosure": companion files cost zero context tokens until loaded.

## Authoring conventions used here

- `SKILL.md` is under 500 lines (Anthropic's recommendation).
- Companion files are flat and lowercase, alongside `SKILL.md`. No `references/` subfolder — the official Claude Code docs example uses flat lowercase filenames.
- Every API claim cites a `file:line` in the whatsmeow source so the skill stays verifiable as upstream evolves.
- The `description` field is third-person, "Use when…", and lists concrete trigger imports.
