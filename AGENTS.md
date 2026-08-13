# Agent instructions for this repository

This repo authors and evaluates **agent skills** for a data governance workflow: mapping an
abstract data domain onto the physical models of undocumented source systems. It is a
skill-building repo. The skills here are the product; there is no application to run.

## Layout

```text
scaffolds/
  shared/copilot-instructions.md   always-on working agreements; installs to .github/
  survey-source/SKILL.md           builds a localized model of an undocumented source
  variant-a/ variant-b/            two competing map-attribute designs, unaided
  variant-c/ variant-d/            the same two designs, survey-aware
  INSTALL.md                       placement, eval protocol, contamination control
resources/                         generic config examples the skills expect to find
domain_mapping_skill_eval_rubric.md   scoring instrument for comparing variants
```

The skills target GitHub Copilot and install to `.github/skills/<name>/SKILL.md` in the
*consuming* project, not here. `shared/copilot-instructions.md` installs to
`.github/copilot-instructions.md` at that project's workspace root.

## The hard rule

**This repository is public, and it mirrors work done inside a client environment.**

Never add, and remove on sight:

- client or employer names, product names, or internal package paths
- real table names, column names, schema names, or lookup key values
- real query output, row counts, or sample statistics
- session transcripts, file paths from the client repo, or dated references to client work
- anything that narrows down which organization or which vendor system is involved

Generic structural patterns are the point and are welcome. Identifiers are not. When an example
needs a table or column, invent one. If you are unsure whether something is identifying, assume
it is and ask.

## Authoring principles

These are conclusions from failed attempts, not preferences. They cost real evaluation cycles.

- **Constrain the artifact, not the process.** Require an output shape. Do not write stages,
  exit criteria, or checklists — they define "done" as steps-executed, which crowds out judgment.
- **Atomize the work, not the knowledge.** One bounded unit per invocation. Splitting knowledge
  across many small skills makes each one individually skippable.
- **Environment facts belong in always-on instructions**, never in a skill. A skill is retrieved
  on description match, and an agent that is confidently wrong never triggers the retrieval.
- **A stance cannot be instructed.** Do not write "be skeptical." Require an output slot that a
  non-skeptical process cannot fill, and place it *before* the recommendation.
- **Prefer the question to the procedure.** Listing techniques an agent already knows adds
  obligation without information. Ask the question only investigation can answer.
- **Never let a skill modify itself or its own scaffold.** Scaffold changes are deliberate,
  human, and between runs.

## Conventions

- Markdown, markdownlint-clean, US English.
- Skill frontmatter requires `name` (matching the parent directory) and `description`. The
  description is the load trigger — write it as when-to-use, not what-it-is.
- Keep `shared/copilot-instructions.md` short. It competes for attention with everything else.
- When variants exist to be compared, change **one** factor between them and keep everything
  else byte-identical.
