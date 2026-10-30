# Project Brief: Fluent Journal Skill

_Created 2026-10-20. This is the source of truth for this project. Update it as decisions change._

## Goal
Add a `/fluent-journal` skill to my own fork of the Fluent plugin (`m98/fluent`). The fork should keep receiving upstream updates without overwriting my additions. This is also a learning project: I've never forked a repo before, so it's done in baby steps with explanations.

## How the skill works (MVP)
1. **Trigger:** manual only (`/fluent-journal`, `disable-model-invocation: true`), like the other Fluent skills.
2. **Blank page:** no prompts or suggested topics. I write whatever I want about my day, in Spanish.
3. **Wait for the full entry.** No corrections mid-entry.
4. **Rating:** communication, grammar, vocabulary, and naturalness (each /10), plus an overall score.
5. **Corrections:** tagged 🔴/🟡/🟢. Recurring patterns from `mistakes-db` get flagged. Accents count as 🟢 minor (existing preference). No vosotros.
6. **Two rewrites:** a *Corrected* version (minimal fixes) and a *Natural* version (how a native would write it).
7. **Natural-version level:** pitched at my **target level (B2)** by default, and changeable as a setting.
8. **Phrases → review, automatically:** 2–3 phrases from the natural version go into spaced repetition via `new_vocabulary[]`, with no asking.
9. **Save:** the entry goes to `~/.claude/fluent-data/journal/YYYY-MM-DD.md` (outside the plugin, so it survives updates). The DB update is one `update-db.py` call with `command_used: "/fluent-journal"` and `skills_practiced: ["writing"]`.

## Not in MVP
- Guided prompts or topic suggestions
- Changes to the Python scripts (none needed: `update-db.py` accepts any command name)
- Contributing back upstream (maybe later)
- Fix the vague `/results/` transcript path in my fork so every skill saves to `~/.claude/fluent-data/results/`. Sessions were saving relative to whatever folder they ran in, which scattered transcripts across three places. Could also be a good first upstream contribution.

## Decision: rewrite level (decided 2026-10-27)
Stored as `preferences.journal_rewrite_level` in the learner profile. When it isn't set, the skill uses `target_level` (B2). `/fluent-journal level B1` changes it. `fluent-setup` stays untouched, so it can't conflict with upstream.

## Decision: how Claude Code loads my fork (decided 2026-10-30)
`main` stays an untouched copy of `m98/fluent`. Claude Code loads the `fluent-journal` branch instead: `claude plugin marketplace add 'Bennett-JA/fluent#fluent-journal'`. The marketplace keeps the name `m98` (it comes from `marketplace.json`, and renaming it would conflict with every upstream version bump), so the plugin is still `fluent@m98`. "m98" is only a local label. It points at my fork. Auto-update is off, so after pushing: `claude plugin marketplace update m98`, then restart Claude Code. If updates don't seem to apply, check whether the unchanged `0.3.0` version number is the cause.

## Environment facts
- GitHub account: **Bennett-JA** (`gh` CLI already logged in). Git name and email are already configured.
- Plugin install: `fluent@m98` v0.3.0, loaded from my fork's `fluent-journal` branch (since 2026-10-30), in `~/.claude/plugins/cache/m98/fluent/0.3.0` (this gets overwritten on update, so don't edit it).
- My learning data: `~/.claude/fluent-data/` (separate from the plugin, safe).
- Clone location: `~/github.com/fluent` (alongside my other repos)

## Build order (✅ = done)
1. ✅ **Fork** `m98/fluent` on GitHub. Plan: use the website so I see it once (Fork button → keep name `fluent` → "copy main branch only" → Create).
2. ✅ **Clone** the fork to `~/github.com/fluent`
3. ✅ **Add `upstream` remote** pointing at `m98/fluent`
4. ✅ **Create a branch** for the journal work
5. ✅ **Move this brief** into the repo
6. ✅ **Build** `.claude/skills/fluent-journal/SKILL.md` (resolve the open decision first)
7. ✅ **Test** with a real journal entry
8. ✅ **Commit and push** to my fork
9. ✅ **Switch Claude Code** to load Fluent from my fork instead of `m98` (still to confirm: `/fluent-journal` works when Claude Code starts outside the clone)
10. ⬜ **Practice pulling an upstream update**

## Housekeeping
- ✅ (2026-10-21) Gathered all 11 session transcripts into `~/.claude/fluent-data/results/`. They had been scattered across `~/.claude/results/`, `~/results/`, and the plugin cache's `results/`. Sessions 005–007, 013, and 015 were never saved. Removed the two empty folders afterward.
