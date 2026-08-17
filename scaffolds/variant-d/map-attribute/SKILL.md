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

This reply is the deliverable. Writing a summary into a project document as well is useful, but the
reasoning must be here in full — anything that lives only in a file was not said.

### What the survey gave you

The leads it held for this attribute, and whether each still exists — surveys go stale and
schemas move. Verify before building on them.

Then read its *Not examined* section and its provenance date, and answer: **for this attribute
specifically, does where the survey stopped looking matter?**

Read its *Open questions* as well, and carry forward any that bear on this attribute. A question
the survey raised correctly is not resolved by having been written down — it is still open, and
this is the moment it becomes actionable. If one of them blocks this mapping, repeat it under
**What I need from you**. If you settled one during this pass, say so under **Survey corrections**.

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

**When candidates come from the values of a discriminator, the label is the weakest evidence you
have.** Rank them by population and by whether their format fits the attribute — never by which
label reads most like the attribute's name. Long-lived systems accumulate generic labels for
specific things: the value that carries what you want is frequently the one whose name gave no
hint, while the obviously-named ones turn out to be empty, deprecated, or owned by another domain.
If the labels do not adjudicate, list them with their populations under **What I need from you**
rather than picking the one that sounds right.

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

**If the conflict is not settled by the evidence, end this section with a direct question and stop
there.** Do not carry an unsettled conflict through to a recommendation. In fourteen evaluated runs
of this workflow, agents named conflicts accurately, recommended a resolution, applied it, and moved
on — and not once asked the human anything. A recommendation the human can only decline by
noticing it is not a decision they made.

Write "none — checked for X and Y" rather than "none".

### Evidence

The probe, its `sample_scope`, and the counts.

Then state, in one sentence: **what result would have changed your recommendation.** If no
possible result would have changed it, you did not run a probe, you ran a confirmation.

Agreement between two fields establishes that they are consistent with each other. It does not
establish that either is correct.

**State the denominator.** A population figure means nothing until you say what it is a fraction
of. If the attribute applies only to part of the domain, count against that part.

If every candidate returns zero population, that is not evidence of absence — and widening the
sample is the *second* thing to try. **Widen the hypothesis before you widen the rows.** A zero
across every candidate you tried usually means the candidate set is wrong, not that the sample is
too small. Go back to the enumeration the set came from and ask which values you passed over and
what made you pass over them. Running the same predicate against ten times as many entities
returns the same zero, more slowly and with more confidence attached.

### What would make this wrong

The most plausible way your recommendation turns out to be mistaken, stated concretely enough
that someone could go check it.

### What I need from you

The questions a person answers faster than you can investigate them. Three kinds belong here:

- **A fact the database cannot contain.** The format an issuing authority uses, what a superseded
  acronym stood for, the shape of an external code standard, whether two systems were ever meant
  to agree. Querying does not produce these at any budget, and the human usually knows them
  offhand.
- **A label you cannot adjudicate.** When an enumerated discriminator offers several values and
  their names do not settle which one is yours, list the values with their populations and ask.
  Someone who knows this source recognizes it on sight.
- **A scope exception you want.** Name the query, what it would settle, and why the sample cannot
  settle it. Then wait.

Usually this is one word: none. When it is not, keep it short and answerable — a question that
takes ten seconds to answer is worth asking, and a question that requires the human to reconstruct
your reasoning first is not.

This section is not a substitute for **Conflicts**. A conflict the evidence cannot settle stops
the turn there. These are cheap questions that do not block: state them, and carry on to a
recommendation marked provisional if the answer would change it.

### Recommendation

The primary candidate, the alternates, and the specific reason each alternate lost. If evidence
is insufficient to choose, say so and mark it provisional rather than picking.

**There is a third outcome besides mapped and not-found: the field exists and cannot be used.** A
column can be real, plainly named, and still carry nothing a mapping could rely on. Persisting past
that point burns turns and produces nothing, and mapping it anyway hands the human a field that
fails the first time someone reads it.

Three facts settle it, and none requires a scope exception or a scan of the table:

1. **Population across the whole source, not the sample.** The schema catalog records a null count
   per column. A column empty for 99% of every row in the source is a different object from one
   that happens to be empty across three hundred sampled entities.
2. **Whether a decode target exists at all.** No foreign key, no similarly-named lookup, and values
   that join to nothing is the signature of a vendor feature nobody finished. An abstract key with
   nothing to decode it against is not a lead awaiting a lookup — the absent lookup is the finding.
3. **Cardinality of the populated remainder.** A handful of distinct opaque values across the
   fraction that is populated means an abandoned field. Tens of thousands means a real payload you
   have not decoded yet.

When all three point the same way you have a licence to stop, and stopping is the right call. Say
it as a positive finding — *this field exists, and here is why it cannot be used* — recommend the
decision state in `configs/mappings/README.md` that fits, and move on. That is a complete answer,
not a failure to find one. Do not phrase it as "not found": a column visible in the catalog reads
as a miss when it is reported missing, even where the practical conclusion is right.

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
