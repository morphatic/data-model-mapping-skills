# Working notes — `<SOURCE>`

Standing corrections for anyone, human or agent, mapping this source. The mapping skill reads this
file at the start of every attribute, before the survey.

Each line is an instruction already given. It outranks the agent's own judgement about how to
proceed, and it survives a context clear — which is the whole reason the file exists.

**Keep it short.** It competes for attention with everything else in context. If it grows past a
page it has become documentation instead of instruction, and it will start being skimmed.

**What belongs here:** how to draw a working subset, standing scope permissions and refusals,
conventions that keep getting missed, "when you propose X, also apply X", and facts about the world
you have supplied once already.

**What does not:** findings about the source (those go in the survey), decisions about one attribute
(those go in the mapping TOML), and anything you would not want applied to every attribute.

Delete the examples below and replace them with your own.

## Sampling

- Draw working subsets **across strata** — n per product per status — never the first n rows of the
  sample file and never a random n. Escalate by raising n per cell.
- The sample file's own columns identify the strata. Read its header before drawing.

## Scope

- Sample-scoped probes need no approval. Anything wider needs an explicit request, stating what the
  query is and why the sample cannot answer the question.
- `<add standing permissions granted once and re-granted every session>`

## Follow-through

- When you propose a survey correction, apply it in the same turn. A correction that is only
  described has not been made.
- When you propose a TOML edit, apply it in the same turn.

## Facts already supplied

Things established once that should not need re-deriving or re-asking.

- `<external formats, code standards, superseded acronyms, program boundaries>`

## Conventions this source keeps tripping over

- `<the thing you have now corrected twice>`
