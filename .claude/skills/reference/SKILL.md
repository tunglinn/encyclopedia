---
name: reference
description: Turn the question just discussed into a reference page in this repo's Quartz encyclopedia (content/*.md) — either a new page or an extension of an existing one, wikilinked into the rest of the notes. Use when the user runs /reference or otherwise asks to save/write up/keep what was just discussed as a reference/note/page.
metadata:
  origin: encyclopedia
---

# Reference

Turns the question(s) just discussed in this conversation into a page in the
Quartz site rooted at `content/`. The user asked a question, got an answer,
and now wants it kept — organized like an encyclopedia, not logged like a
chat transcript.

## 1. Identify the topic

Use the immediately preceding question/answer exchange in this conversation
as the source material. If the user passed an explicit topic as an argument
to `/reference`, that overrides inferring one from context.

## 2. Find related existing notes

Search `content/*.md` (frontmatter `title`, `aliases`, `tags`, and the
filename itself) for anything covering the same or an adjacent topic. A
plain `grep -il` over titles/tags is enough — this is a personal wiki, not a
large corpus.

## 3. Decide: extend or create

- **Same concept, more detail** (a follow-up that deepens or corrects the
  existing page) → extend that note: add/expand a section in place.
- **Related but distinct concept** → create a new note, and link it from the
  existing one (add a line/section referencing it) as well as linking back
  from the new note. This is what makes the site a graph instead of a pile
  of unrelated pages.
- **No related note found** → create a new note.
- If it's genuinely unclear which of the above applies, ask the user instead
  of guessing.

## 4. Write it like an encyclopedia entry, not a chat log

The page must read as clean, self-contained reference prose — a textbook
entry — because the user edits directly into these files afterward and
wants no seam between what Claude wrote and what they added. Concretely,
avoid:
- "The user asked..." / "As discussed above..." framing
- Restating the question as a heading
- Any inline marker distinguishing Claude's writing from anyone else's —
  that distinction lives entirely in git history (see step 5), not the page

New note file: `content/<kebab-case-title>.md`. Frontmatter:

```yaml
---
title: Agent-Based Modeling
tags:
  - simulation
  - modeling
date: 2026-09-16
---
```

Use `[[Wikilinks]]` (Quartz/Obsidian syntax, already configured) to
cross-reference other notes in `content/` — this is the primary way the
"encyclopedia" becomes browsable rather than a flat list. Add tags that
reflect the topic's category so Quartz's tag pages and graph view stay
useful.

Reach for extras only when they genuinely help understanding, not by
default:
- Images/diagrams: save under `content/attachments/` and reference with
  normal markdown image syntax. Mermaid diagrams work directly in a
  ` ```mermaid ` fence (Obsidian template support is already configured).
- A small interactive check (e.g. a collapsible self-test) can use plain
  `<details><summary>...</summary>...</details>` HTML inline in the
  markdown — no extra tooling needed for something that simple.

## 5. Commit

Stage **only the file(s) this note touches** and commit them on their own
— do not bundle in unrelated changes. This one-file/one-topic-per-commit
discipline is what makes `git log --follow -- content/<file>.md` /
`git blame` a reliable way to separate Claude-authored material from the
user's later hand edits, entirely outside the rendered page. Use this
session's standard commit attribution trailer.

Commit message: `Add reference: <Title>` for a new note, or
`Extend reference: <Title>` for an addition to an existing one.

Do **not** `git push` automatically — committing is safe and local;
pushing publishes to the live site. Tell the user the note is committed
locally and that pushing (or asking Claude to push) will publish it.

## 6. Report back

Tell the user the file path, whether it was a new page or an extension, and
what it's now linked to.
