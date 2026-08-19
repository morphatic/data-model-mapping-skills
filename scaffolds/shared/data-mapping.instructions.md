---
applyTo: "configs/**,docs/sources/**,artifacts/samples/**"
description: Working agreements for source discovery and attribute mapping
---

# Source discovery and attribute mapping

These apply to **this workflow only** — surveying a source, and mapping domain attributes onto its
physical fields. They are deliberately not repo-wide: several would be wrong for profiling,
reporting, or any other work in this repository. Repo-wide facts and safety rules are in
`.github/copilot-instructions.md`.

The `/survey-source` and `/map-attribute` skills both assume everything below.

## Terminology

- **attribute** — an element of an abstract domain model in `configs/domains/`
- **field** — a physical column in a source system that may instantiate an attribute

## What this work is

For each attribute, the deliverable is a decision about **which physical field to map**, with just
enough evidence to justify it and to document the alternatives:

- find every plausible candidate field
- gather the minimum evidence that separates them
- name a primary, and record the others with the reason each one lost
- record what remains uncertain

**Profiling is a separate, later phase.** Value distributions, data quality, outlier analysis,
population characteristics, and completeness reporting are not part of *this* work. Do not pursue
them, do not propose them as a next action, and do not use them to justify a wider query. When you
notice something a future profiler will want to know, write it down in one line as a note and move
on — that is the correct handling, and it is all of it.

The bar for evidence is *enough to make a well-reasoned choice and show your reasoning*, not *enough
to be certain*. Attributes frequently converge after one or two probes. That is the expected
outcome, not a sign you stopped too early.

## The entity sample

- The entity sample CSV's key column is `<COLUMN_NAME>` — read the header before binding
  parameters, and note which of its columns are source-specific rather than portable across sources.
- **The sample file is a list of entity keys, not a data extract.** It deliberately contains
  identifiers and nothing else. Its purpose is to make exploration cheap: restrict every exploratory
  query to a few dozen keys drawn from it — `WHERE <key column> IN (<keys from the file>)` — instead
  of scanning tables holding millions of rows. Finding that it lacks the column you are investigating
  is expected and is not a reason to set it aside.
- **The sample file is stratified as a population. Any subset you draw from it is not.** It covers
  the products, programs, and statuses that matter for coverage — but the first fifty rows of it do
  not, and neither does a random fifty. When you draw a working subset, draw *across* the strata: n
  per product per status. Escalate by raising n per cell, never by taking a larger undifferentiated
  slice. Read the sample file's header first; the columns identifying the strata are usually already
  in it. Do not `GROUP BY` a large slice of the source to compensate for bias the sample has already
  handled.

## Scope constraints

These hold for discovery and mapping work. They do **not** bind other tasks in this repository.

- **Every exploratory query is restricted to keys from the sample file.** An unrestricted aggregate
  against these tables takes minutes to hours, and there is no cheap way to find that out except by
  hanging. Widening beyond the sample — including a "bounded", "limited", or row-capped query of
  your own construction — requires you to stop, state what you want to run and why the sample cannot
  answer the question, and wait for approval. **A row limit is not a scope approval.**
- Within the sample, escalate deliberately: one entity per stratum, then three, then ten, then the
  whole sample. Move up only when the narrower scope left the question genuinely unresolved, and say
  which question it left open.

## Where to look when you are stuck

Load these only when the trigger fires. Available is not the same as active.

| Source | Location | Authority | Load when |
| --- | --- | --- | --- |
| Standing corrections from the human | `docs/sources/<SOURCE>/working-notes.md` | canonical | at the start of every attribute, before anything else |
| Mapping conventions and allowed vocabularies | `configs/mappings/README.md` | canonical | before writing or editing any mapping TOML |
| Validated table names, joins, and aliases | `configs/mappings/<domain>/<source>.toml` | canonical | an object reference is uncertain, or a query fails to resolve |
| Domain model definitions | `configs/domains/<domain>.toml` | canonical | you need an attribute's intended meaning, type, or required status |
| Entity sample | `artifacts/samples/<domain>/` | supporting | any probe |

`working-notes.md` is where corrections the human has already made to your working method live. It
exists because chat context gets cleared and those corrections would otherwise have to be repeated
every time. Treat each line as an instruction already given to you. If you are about to do something
a note forbids, the note wins.

When two sources disagree, the domain TOML defines intent and the mapping TOML defines what exists
in the source. A conflict between them is a finding to report, not a discrepancy to silently
resolve.

## What counts as evidence

- **A zero from a filtered query is a fact about your filter, not about the data.** Before
  concluding that something is absent, enumerate the unfiltered distribution of whatever you
  filtered on. A name-shaped filter cannot find a thing whose name does not match, no matter how
  many times you run it.
- **Counting is not looking.** Select actual values from a column before reporting counts about it.
  A population count you have never sanity-checked against real values is a guess with a number
  attached, and a wrong predicate will produce one just as readily as a right one.
- **What you are looking for may be more than one join from the anchor.** A single-hop star is not a
  search. Attributes routinely sit behind a bridge table, a lookup, or a dimension.
- **Sparse is not the same as wrong.** An attribute may be populated only for the entities it
  applies to — a program, a product, a line of business. Coverage has to be measured against that
  qualifying subset, not against every entity in the sample. A field that looks weak at 20% of all
  entities may be complete at 98% of the ones it is for. Before rejecting a candidate as sparse,
  state which population it is supposed to cover and count against that one.
- **A conflict you resolved yourself is not a conflict you surfaced.** Naming it in a document and
  then deciding alone has the same effect as never raising it. State the options, recommend one, and
  ask.

## Preferences

These may bend when evidence supports it — **but bending one is a finding you must surface, not a
judgment call you make silently.**

- Prefer a human-readable NAME or VALUE over an abstract CODE when both represent the same encoded
  value. *Exception worth surfacing:* when the code is a recognized external standard (ISO country
  codes, FIPS county codes), it may carry cross-source comparability that the name does not. Say so
  and propose how to resolve it.
- Prefer fewer candidates with evidence over more candidates without.
