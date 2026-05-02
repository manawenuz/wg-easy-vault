---
title: Kimi prompt template
type: handoff
---

# Kimi prompt template

Standard wrapper for handing a PRD to Kimi (256K context). Copy-paste and fill in the blanks.

## System prompt

```
You are an implementation engineer working on the manawenuz/wg-easy fork.
You write production code, not prose. You output a single unified diff plus a
self-test plan. You DO NOT modify files outside the PRD's "Modify these files"
list. If the PRD is ambiguous, list the ambiguities at the top of your reply
and STOP — do not guess.

Stack: Nuxt 4 + Nitro, Vue 3, Pinia, Tailwind, Drizzle ORM (SQLite),
TypeScript strict, pnpm, ESLint configured.

Rules:
- Match existing code style. Look at neighboring files.
- No new top-level dependencies without justifying in the diff message.
- Migrations are additive when possible; never edit a shipped migration.
- Tests live next to the code (*.test.ts) unless an integration suite exists.
- Don't introduce new abstractions beyond what the PRD specifies.
- Every shell command in the self-test plan must actually run; do not invent flags.
```

## User prompt

````
# Task: implement PRD <PRD-ID>

## PRD
<paste full PRD markdown here>

## Architecture spine (reference)
<paste docs/obsidian/architecture.md here, or the relevant diagram section anchors>

## Glossary
<paste docs/obsidian/glossary.md here>

## Source files (read-only context)
For each file listed in the PRD's "Read before implementing" block, paste the
file with a header:

  ===== FILE: <relative/path> =====
  <full file contents>
  ===== END =====

## Source files (you will modify these)
Same format, for the files listed in "Modify these files".

## Output format

1. **Ambiguities** (if any) — list and STOP. Otherwise write "None."
2. **Plan** — bullet list, ≤10 lines, what you'll change and why.
3. **Diff** — single unified diff against the current tree. Include new files.
4. **Self-test commands** — copy-pasteable shell block that exercises the
   acceptance tests in the PRD.
5. **Open follow-ups** — items the PRD didn't cover but came up during impl.
````

## Context budget guidance

| Component | Approx tokens |
| --- | --- |
| System prompt | <1K |
| PRD | 3–8K |
| `architecture.md` (relevant sections) | 5–10K |
| `glossary.md` | 2K |
| Source files (5–10 files, 200–400 lines each) | 30–60K |
| Output (diff + plan) | 10–20K |
| **Total** | **~50–100K** |

Well inside Kimi's 256K window. If a PRD is pushing past 120K of context, **split the PRD** rather than dropping context.

## Trimming source files

If a single file is huge (e.g., generated migrations, a long Vue component you only touch in one place), **excerpt it** with line markers:

```
===== FILE: src/server/utils/WireGuard.ts (lines 10-80, 194-230) =====
<lines 10-80>
... [snip lines 81-193] ...
<lines 194-230>
===== END =====
```

The PRD's `touches:` and "Read before implementing" lists should cite exact line ranges where helpful.

## After Kimi returns

1. Apply the diff: `git apply /tmp/kimi.diff`. If it fails, hand the rejected hunks back to Kimi — do not hand-fix beyond trivial whitespace.
2. Run Kimi's self-test commands as-is. Don't paraphrase.
3. Read the diff yourself before merging. The PRD is the contract; the diff is the artifact.
4. Update the PRD's `status:` to `shipped` and verify `touches:` matches the diff's actual file list.
5. If Kimi flagged ambiguities and stopped, resolve them in the PRD (not in chat) and re-run.

## Anti-patterns

- ❌ Sending Kimi multiple PRDs in one session. One PRD per session.
- ❌ Sending Kimi the whole repo. Send only the listed files.
- ❌ Letting Kimi write the PRD itself. PRDs are written by the orchestrator (this vault); Kimi only implements.
- ❌ Accepting "I added a small refactor while I was there." Reject and re-prompt — scope creep compounds.
