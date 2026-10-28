---
name: fluent-journal
description: Run a free-form journaling session — the learner writes about their day in the target language on a blank page, then gets severity-tagged corrections, four-part scoring, a Corrected and a Natural rewrite, and 2-3 phrases auto-added to spaced repetition. Triggered only when the learner types /fluent-journal. Accepts `level <CEFR>` to change the Natural rewrite's level.
allowed-tools: Read, Write, Bash
disable-model-invocation: true
---

# Journal Session

## Overview

Free writing with no prompt. The learner writes whatever they want, gets one round of detailed feedback, sees two rewrites (minimal fixes vs. how a native would say it), and walks away with 2-3 new phrases in their review queue. The entry is saved to the data directory so it survives plugin updates.

## When to Use

Trigger this skill only when the learner types `/fluent-journal`. Gated with `disable-model-invocation: true` — it writes to the databases and the journal folder.

Use `/fluent-writing` instead when the learner wants a structured task (email, form, letter).

## Arguments

`$ARGUMENTS`

- Empty → normal session.
- `level <CEFR>` (A1, A2, B1, B2, C1, C2) → save that as the Natural rewrite level (step 2), then run a normal session.
- Anything else → ignore and run a normal session.

## Instructions

### 1. Load context

```bash
python3 "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/read-db.py"
```

Need:
- `learner_profile.learner` — `name`, `target_language`, `target_level`
- `learner_profile.preferences` — `journal_rewrite_level`, `avoid_topics`, `notes`
- `learner_profile.current_streak_days`
- `mistakes_db.error_patterns` — ids, descriptions, frequency (for recurring-error flags)
- `computed.next_session_id`, `computed.today`

Resolve the data directory (used for the journal file):

```bash
python3 -c "import sys; sys.path.insert(0, sys.argv[1]); from fluent_paths import data_dir; print(data_dir())" "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks"
```

**Rewrite level** = `preferences.journal_rewrite_level` if set, else `learner.target_level`.

### 2. Save a new level (only with `level <CEFR>`)

```bash
python3 - "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks" B1 <<'EOF'
import json, sys
sys.path.insert(0, sys.argv[1])
from fluent_paths import data_dir
path = data_dir() / "learner-profile.json"
profile = json.loads(path.read_text(encoding="utf-8"))
profile.setdefault("preferences", {})["journal_rewrite_level"] = sys.argv[2]
path.write_text(json.dumps(profile, indent=2, ensure_ascii=False) + "\n", encoding="utf-8")
EOF
```

Replace `B1` with the requested level. Confirm in one line ("Natural rewrites will now be pitched at B1.") and continue. `update-db.py` preserves unknown profile keys, so the setting survives later sessions.

### 3. Blank page

```markdown
## 📔 Journal — {date}

¡Hola, {name}! 🔥 {streak}-day streak.

Write about anything — your day, a thought, a plan — in {target_language}, as long or short as you like. Send it when you're done.
```

No topic suggestions, no prompts, no example sentences.

### 4. Wait for the full entry

Don't correct, ask follow-ups, or comment until the learner submits. If they ask a vocabulary question mid-entry, answer it briefly and let them keep writing.

### 5. Feedback

Check every sentence for grammar, vocabulary, spelling, and naturalness. Tag each finding 🔴 critical, 🟡 moderate, or 🟢 minor.

- **Accent-only errors are always 🟢 minor.**
- If a finding matches an existing `error_patterns` id, add **🔁 Recurring (×{frequency + 1})**.
- Respect `avoid_topics` — e.g. never introduce vosotros in corrections or rewrites.

```markdown
## Feedback

### ✅ What You Did Well
- {specific strength}
- {specific strength}

### ❌ Corrections

**Critical:** 🔴
- `{category}`: "{wrong}" → **"{correct}"** — {why} {🔁 Recurring (×N) if applicable}

**Moderate:** 🟡
- ...

**Minor:** 🟢
- ...

### 📊 Scores

| Communication | Grammar | Vocabulary | Naturalness | **Overall** |
|---|---|---|---|---|
| {X}/10 | {X}/10 | {X}/10 | {X}/10 | **{X}/10** |
```

Omit any severity group that has no findings. If there are no errors at all, say so and skip to the rewrites.

### 6. Two rewrites

```markdown
### 📝 Corrected
{the learner's text with only the fixes above — same sentences, same voice}

### 🌟 Natural ({rewrite level})
{how a native speaker would write the same content, pitched at the rewrite level}
```

The Natural version keeps the learner's meaning and facts. Don't add events they didn't mention.

### 7. Phrases → review

Pick 2-3 reusable phrases from the Natural version that the learner didn't use themselves. Prefer chunks (collocations, connectors, set expressions) over single words. Don't ask — just show:

```markdown
### ➕ Added to your reviews
- **{phrase}** — {English meaning}
- **{phrase}** — {English meaning}
```

### 8. Save the entry

Write `{data_dir}/journal/{YYYY-MM-DD}.md` (create `journal/` if missing). If that file exists, use `{YYYY-MM-DD}-2.md`, `-3`, etc.

```markdown
# Journal — {YYYY-MM-DD}

**Session:** {session_id} · **Rewrite level:** {level} · **Overall:** {X}/10

## Entry
{original text, unchanged}

## Corrections
| Severity | Wrong | Correct | Category | Note |
|---|---|---|---|---|
| 🔴 | ... | ... | ... | ... |

## Scores
Communication {X} · Grammar {X} · Vocabulary {X} · Naturalness {X} · **Overall {X}**

## Corrected
{corrected text}

## Natural ({level})
{natural text}

## Added to reviews
- {phrase} — {meaning}
```

### 9. Update all databases

Use the `fluent-db-updater` skill — one `update-db.py` call:

- `session_id: {computed.next_session_id}`, `date: {computed.today}`
- `command_used: "/fluent-journal"`, `skills_practiced: ["writing"]`
- `skill_scores.writing: {exercises: 1, correct: 1_if_overall_≥_7_else_0, time_minutes}`
- `errors[]` — one per distinct pattern. **Reuse the existing `pattern_id`** when a finding matches one (the script bumps frequency); only invent a new id for a genuinely new pattern.
- `new_vocabulary[]` — one per phrase from step 7:
  - `item_id: "phrase_{snake_case_slug}"`, `item_type: "vocabulary"`
  - `content`: the {target_language} phrase, `answer`: English meaning
  - `category: "journal_phrases"`, `difficulty`: the rewrite level
  - `initial_quality: 3`, `priority: "medium"`
- `topics_covered[]` — what the entry was about
- `focus_next_session[]` — top 1-2 patterns
- `session_notes` — one line

### 10. Close

```markdown
**Overall: {X}/10** · Takeaway: {one sentence}

¡Hasta mañana! 📔
```

## Example

Learner (A2 Spanish, level B2): "Hoy fue un dia largo. Trabaje mucho y despues fui al gimnasio. Estaba cansado pero feliz."

> ### ✅ What You Did Well
> - Correct preterite for *fue* and *fui*.
> - *Estaba cansado* — right choice of *estar* for a temporary state.
>
> ### ❌ Corrections
>
> **Minor:** 🟢
> - `spelling`: "dia" → **"día"** — written accent on the stressed *í*. 🔁 Recurring (×5)
> - `spelling`: "Trabaje" → **"Trabajé"** — *yo* preterite of -ar verbs carries an accent. 🔁 Recurring (×2)
> - `spelling`: "despues" → **"después"**
>
> ### 📊 Scores
> | Communication | Grammar | Vocabulary | Naturalness | **Overall** |
> |---|---|---|---|---|
> | 9/10 | 9/10 | 7/10 | 7/10 | **8/10** |
>
> ### 📝 Corrected
> Hoy fue un día largo. Trabajé mucho y después fui al gimnasio. Estaba cansado pero feliz.
>
> ### 🌟 Natural (B2)
> Hoy ha sido un día larguísimo. Me pasé la mañana trabajando y, al salir, me fui directo al gimnasio. Acabé agotado, pero contento.
>
> ### ➕ Added to your reviews
> - **me pasé la mañana trabajando** — I spent the morning working
> - **acabé agotado** — I ended up exhausted

## Critical Rules

- **Never auto-invoke.** Fires only on explicit `/fluent-journal`.
- **Blank page.** Never suggest topics or prompts.
- **Wait for the full entry** before any feedback.
- **Accents are always 🟢 minor.**
- **Respect `avoid_topics`** (e.g. no vosotros) in corrections and rewrites.
- **Add phrases without asking.**
- **One `update-db.py` call** at the end, and **always save the journal file.**
