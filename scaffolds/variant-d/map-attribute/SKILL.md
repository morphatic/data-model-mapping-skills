---
name: map-attribute
description: Map one domain attribute to candidate physical fields in a source system, gather evidence, and recommend a primary. Use when the human names a single attribute to map, or asks to continue mapping the next attribute in a domain.
argument-hint: <attribute-name>
---

# Map one attribute

You are mapping **one** attribute this turn. Not the domain. Not the next attribute after it.

There is no procedure here and no sequence of stages. How you find candidates and what SQL you
write is yours to decide — you are good at that. What is fixed is the **shape of the answer**.

Read `configs/mappings/README.md` before writing TOML. It defines the allowed values for
`candidate_resolution_mode`, `selection_status`, candidate `status`, `sample_scope`, and the
decision-state notation. Use it; do not invent vocabulary.

## Start with the survey

Look for `docs/sources/<SOURCE>/<domain>-survey.md`.

**If it is missing**, say so and recommend the human run `/survey-source <source> <domain>`
first. Mapping without it means rediscovering the same source structure once per attribute, and
arriving at a different picture each time. Offer to proceed unaided if they would rather — that
is their call — but do not quietly proceed as though nothing were missing.

**If it is present**, it is a **lead sheet, not an answer key.** It also records the anchor
table, the business identifier, and the join key, which are usually three different columns.
Use those rather than re-deriving them.

## Required response shape

Emit every section, in the order below, every time. Everything above **Recommendation** is
written before it — a concern written after a conclusion tends to justify the conclusion.

Render each section name as a markdown heading. Do not collapse them into one numbered or
bulleted list: several sections contain lists of their own, and nesting those inside an outer
list makes the reply unreadable.

### What the survey gave you

The leads it held for this attribute, and whether each still exists — surveys go stale and
schemas move. Verify before building on them.

Then read its *Not examined* section and its provenance date, and answer: **for this attribute
specifically, does where the survey stopped looking matter?**

If there is no survey, say so here and record that the human chose to proceed without one.

### What you added

Objects you interrogated beyond the survey, and what each turned up, **including the searches
that returned nothing**. Then answer one question about your own search: **what would a search
conducted that way be structurally unable to find?** Answer it concretely, in terms of what this
source contains, and if the answer identifies a real blind spot, go look before continuing.

If you added nothing, say why the survey was sufficient for this attribute. Do not leave it
implied.

### Candidates

For each, the expression and the lead or search that surfaced it. A candidate you cannot trace
to either was not found — it was assumed, and should be labeled as such.

"No candidates exist" is a claim about the source. It is only credible alongside the list of
ways you looked.

### Not checked

Tables, columns, join paths, or lookup values you did not examine, and why each might matter.
This section is never empty. In a source of this size, there is always something you did not
look at.

### Conflicts

Does mapping this attribute put two stated rules in tension, or put the domain model in tension
with what the source actually contains? Common shapes:

- a preference points one way and a different preference points the other
- the domain model expects something the source does not represent
- the domain model's granularity does not match the source's granularity
- the best available field would satisfy this attribute but break a correlated one

If a conflict exists, propose the resolution options and recommend one. Where the resolution
is to change the domain model rather than the mapping, say so — that is a legitimate outcome
and the human decides it.

Write "none — checked for X and Y" rather than "none".

### Evidence

The probe, its `sample_scope`, and the counts.

Then state, in one sentence: **what result would have changed your recommendation.** If no
possible result would have changed it, you did not run a probe, you ran a confirmation.

Agreement between two fields establishes that they are consistent with each other. It does not
establish that either is correct.

If every candidate returns zero population, that is not evidence of absence. Say so, and either
escalate scope or run a targeted count before drawing any conclusion.

### What would make this wrong

The most plausible way your recommendation turns out to be mistaken, stated concretely enough
that someone could go check it.

### Recommendation

The primary candidate, the alternates, and the specific reason each alternate lost. If evidence
is insufficient to choose, say so and mark it provisional rather than picking.

### Survey corrections

Anything you learned that the survey should have said — a missing lead, a stale one, a structural
fact it did not record. Report it. Do not edit the survey yourself unless asked.

Write "none" if the survey held up.

### Proposed TOML edit

Conforming to `configs/mappings/README.md`.

### Out of scope

What you noticed that belongs to **profiling** rather than mapping — value distributions,
quality issues, outliers, population characteristics. Name it here and do not pursue it. Deeper
characterization of the data is a later phase.

## Then stop

Wait for the human. Do not begin the next attribute, do not run reports, do not open a PR.
