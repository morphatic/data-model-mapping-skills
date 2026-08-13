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

## Required response shape

Emit every section, in the order below, every time. Everything above **Recommendation** is
written before it — a concern written after a conclusion tends to justify the conclusion.

Render each section name as a markdown heading. Do not collapse them into one numbered or
bulleted list: several sections contain lists of their own, and nesting those inside an outer
list makes the reply unreadable.

### Where you looked

The source is undocumented. Undocumented is not unknowable — the database can be interrogated,
and doing so is part of this job rather than something to be asked for.

List the objects you actually interrogated and what each turned up, **including the searches
that returned nothing**. Then answer one question about your own search: **what would a search
conducted that way be structurally unable to find?** Answer it concretely, in terms of what
this source contains, and if the answer identifies a real blind spot, go look before continuing.

"No candidates exist" is a claim about the source. It is only credible alongside the list of
ways you looked.

### Candidates

For each, the expression and the specific search that surfaced it. A candidate you cannot trace
to a search was not found — it was assumed, and should be labeled as such.

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

### Proposed TOML edit

Conforming to `configs/mappings/README.md`.

### Out of scope

What you noticed that belongs to **profiling** rather than mapping — value distributions,
quality issues, outliers, population characteristics. Name it here and do not pursue it. Deeper
characterization of the data is a later phase.

## Then stop

Wait for the human. Do not begin the next attribute, do not run reports, do not open a PR.
