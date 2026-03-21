---
name: aif-note
description: >-
  Extract a concise research summary from the current conversation context
  and save it as a numbered note file. Use when finishing a research session,
  deep-dive, or exploration and you want to persist the key findings
  for future reference. Keywords: note, research, summary, findings, extract.
argument-hint: "[topic-slug]"
disable-model-invocation: true
user-invocable: true
allowed-tools: Read Write Bash(ls *) Bash(mkdir *) Glob
---

# Note — Research Summary Extraction

Extracts key findings from the current conversation context and saves them as a numbered note file.

## Workflow

### Step 1: Analyze Context

Review the full conversation to identify:
- What was researched or explored
- Key discoveries, conclusions, or insights
- Code patterns, architecture decisions, or constraints identified
- Relevant file paths and their roles
- Unresolved questions or areas needing further investigation

### Step 2: Determine Slug

Use `$1` if provided. Otherwise derive a short descriptive slug from the research topic (lowercase, hyphens).

### Step 3: Save Note to File

**Note file path:** `.ai-factory/notes/<NN>-<slug>.md` where:
- `<slug>` is derived from the topic (lowercase, hyphens)
- `<NN>` is a zero-padded two-digit sequence number (`01`, `02`, `03` …)

To determine `<NN>`, find the highest existing `NN` prefix among files matching `[0-9][0-9]-*.md` in `.ai-factory/notes/` and add 1. If no numbered files exist yet, start at `01`.

**Before saving, ensure directory exists:**
```bash
mkdir -p .ai-factory/notes
```

**Note file must follow this template:**

```markdown
# <Topic Title>

**Date:** <YYYY-MM-DD>
**Source:** conversation context

## Key Findings

<Bulleted list of the most important discoveries, conclusions, or insights.
Each bullet should be self-contained and actionable.>

## Details

<Organized sections expanding on the findings above. Use subheadings to group
related information. Include code patterns, architecture decisions, relevant
file paths, API contracts, gotchas, and edge cases as appropriate.>

## Open Questions

<Unresolved questions or areas needing further investigation.
Remove this section if there are none.>
```

### Step 4: Report

```
Note saved: .ai-factory/notes/<NN>-<slug>.md

Key findings:
- <1-3 bullet summary>
```

---

## Important Rules

1. **Be concise** — capture the essence, not the full conversation. A note is a distilled reference, not a transcript.
2. **Focus on findings** — what was learned, not the process of learning it.
3. **Include file paths** — when the research identified specific files or code locations, include them so the note is actionable.
4. **All content in English** — regardless of conversation language.
5. **Duplicates are OK** — multiple notes on the same topic are expected. Research is iterative — notes capture the flow of thought and how understanding evolves over time. By default, always create a new file. Update an existing note only if the user explicitly asks to.

## Note File Handling

Notes live at `.ai-factory/notes/<NN>-<slug>.md`:
- `<NN>` determined by scanning existing `[0-9][0-9]-*.md` files and incrementing the highest
- `<slug>` derived from the topic or `$1` argument (lowercase, hyphens)
