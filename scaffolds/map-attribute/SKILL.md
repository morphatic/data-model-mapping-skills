---
name: map-attribute
description: Map one domain attribute to candidate physical fields in a source system, gather evidence, and recommend a primary. Use when the human names a single attribute to map, or asks to continue mapping the next attribute in a domain.
argument-hint: <attribute-name>
---

# Map one attribute

You are mapping **one** attribute this turn. Not the domain. Not the next attribute after it.

There is no procedure here and no sequence of stages. How you find candidates and what SQL you
write is yours to decide — you are good at that. What is fixed is the **shape of the answer**, and
the small number of situations that stop the turn.

Read `.github/instructions/data-mapping.instructions.md` before starting. It carries the
environment facts, scope constraints, and evidence standards this workflow assumes. Those
instructions are scoped to mapping paths and may not have auto-loaded yet.

Read `configs/mappings/README.md` before writing TOML. It defines the allowed values for
`candidate_resolution_mode`, `selection_status`, candidate `status`, `sample_scope`, and the
decision-state notation. Use it; do not invent vocabulary.

## Before you start

Read two things, in this order.

**`docs/sources/<SOURCE>/working-notes.md`** — standing corrections the human has already given
you. Chat context gets cleared between attributes; these do not. Every line is an instruction you
have already received, and it outranks your own judgement about how to proceed.

**`docs/sources/<SOURCE>/<domain>-survey.md`** — the source survey.

If the survey is missing, say so and recommend the human run `/survey-source <source> <domain>`
first. Mapping without it means rediscovering the same source structure once per attribute, and
arriving at a different picture each time. Offer to proceed unaided if they would rather — that is
their call — but do not quietly proceed as though nothing were missing.

If it is present, it is a **lead sheet, not an answer key.** It records the anchor table, the
business identifier, and the join key, which are usually three different columns; use those rather
than re-deriving them. It may be an index pointing at slices — follow the links.

## Required response shape

Eight sections. Emit every one, in the order below, every time — including on the fourth exchange
of a difficult attribute, when it will feel redundant and will not be.

Render each section name as a markdown heading. Do not collapse them into one numbered or bulleted
list: several sections contain lists of their own, and nesting those inside an outer list makes the
reply unreadable.

Everything above **Recommendation** is written before it. A concern written after a conclusion tends
to justify the conclusion.

This reply is the deliverable. Writing a summary into a project document as well is useful, but the
reasoning must be here in full — anything that lives only in a file was not said.

### What the survey gave you

The leads it held for this attribute, and whether each still exists — surveys go stale and schemas
move. Verify before building on them.

Then read its *Not examined* section, its *Open questions*, and its provenance date, and answer:
**for this attribute specifically, does where the survey stopped looking matter?** Carry forward any
open question that bears on this attribute. One the survey raised correctly is not resolved by
having been written down; this is the moment it becomes actionable.

A survey can be wrong. If a lead contradicts what you find, or an absence it claims is refuted by
one of its own lookup extracts, that is a finding — record it under **Changes to apply**.

If there is no survey, say so here and record that the human chose to proceed without one.

### What you added

Objects you interrogated beyond the survey, and what each turned up, **including the searches that
returned nothing**. Then answer one question about your own search: **what would a search conducted
that way be structurally unable to find?** Answer it concretely, in terms of what this source
contains, and if the answer identifies a real blind spot, go look before continuing.

If you added nothing, say why the survey was sufficient for this attribute. Do not leave it implied.

### Candidates

For each, the expression and the lead or search that surfaced it. A candidate you cannot trace to
either was not found — it was assumed, and should be labeled as such.

"No candidates exist" is a claim about the source. It is only credible alongside the list of ways
you looked.

**When candidates come from the values of a discriminator, the label is the weakest evidence you
have.** Rank them by population and by whether their format fits the attribute — never by which
label reads most like the attribute's name. Long-lived systems accumulate generic labels for
specific things: the value that carries what you want is frequently the one whose name gave no
hint, while the obviously-named ones *may* turn out to be empty, deprecated, or owned by another domain.

**A lookup's own row counts point the wrong way.** Reading a code table tells you what the source
*could* represent. Only counting what entities actually carry tells you what it *does*. These come
apart badly: a shared lookup will often devote hundreds of rows to a fine-grained standard nobody
populates, while the family holding the real data is a handful of local values that look
unofficial. In an evaluation of a previous version of this skill, agents read a lookup unfiltered, correctly identified the
official-looking family, filtered to it, got zero rows, and reported the attribute absent — while
the family carrying it sat in the same table under a scruffier prefix. **Enumerate the lookup, then
group the bridge by family and count entities.** The second step is not optional, and the first is
misleading without it.

### Evidence

The probe, its `sample_scope`, and the counts.

**You must actually query the source in this turn.** A recommendation supported only by the survey
is not a mapping, it is a summary of a document someone else wrote. The survey tells you where to
point a probe, never what the probe would have returned. In evaluation, agents repeatedly produced
complete, well-argued, correctly formatted recommendations having run **zero queries**. If you find
yourself about to recommend without a probe, that is the signal to run one, not evidence that none
was needed.

**State the denominator, and build the sample to match it.** A population figure means nothing until
you say what it is a fraction of. Where an attribute applies to only part of the domain — one
program, product, or line of business — restrict the sample to that part *before* you count, rather
than counting across everyone and interpreting the shortfall. And draw the subset across strata, not
off the top of the file: the sample is stratified as a population, but any slice you take from it is
not.

Then state, in one sentence: **what result would have changed your recommendation.** If no possible
result would have changed it, you did not run a probe, you ran a confirmation.

Agreement between two fields establishes that they are consistent with each other. It does not
establish that either is correct. In particular, do not qualify a candidate by how well it agrees
with a candidate you already doubt.

If every candidate returns zero population, that is not evidence of absence — and widening the
sample is the *second* thing to try. **Widen the hypothesis before you widen the rows.** A zero
across every candidate usually means the candidate set is wrong, not that the sample is too small.
Go back to the enumeration the set came from and ask which values you passed over and what made you
pass over them. The same predicate against ten times as many entities returns the same zero, more
slowly and with more confidence attached.

### Gaps and conflicts

Three things belong here, and one habit does not.

**What you did not check, and why it might matter.** But this section is not a disposal site.
**Anything cheap to check that could change the recommendation must be checked before the turn ends,
not listed here.** If one query would resolve it, listing it is a failure to investigate wearing the
costume of disclosure. This section is for the genuinely expensive and the genuinely out of scope.

**Conflicts.** Does mapping this attribute put two stated rules in tension, or put the domain model
in tension with what the source actually contains? Common shapes:

- a preference points one way and a different preference points the other
- the domain model expects something the source does not represent
- the domain model's granularity does not match the source's granularity
- the best available field would satisfy this attribute but break a correlated one
- two attributes in the model resolve to the same physical path, separated only by value family

If a conflict exists, propose the resolution options and recommend one. Where the resolution is to
change the domain model rather than the mapping, say so — that is a legitimate outcome and the human
decides it. Write "none — checked for X and Y" rather than "none".

**The most plausible way your recommendation turns out to be wrong**, stated concretely enough that
someone could go check it.

### What I need from you

**This section can stop the turn, and often should.**

Some observations are not notes. They are questions with the question mark filed off, and writing
them into a passive section is how they die. If any of the following is true it belongs here — and
if the answer would change your recommendation, you state it here and **stop**, without proceeding
to a recommendation at all:

- **You do not know a fact about the world that the database cannot contain.** The format an issuing
  authority uses, what a superseded acronym stood for, the shape of an external code standard,
  whether two systems were ever meant to agree. Querying does not produce these at any budget.
- **A label will not adjudicate.** You have several enumerated values and their names do not settle
  which one is yours. List them with their populations and ask. Someone who knows this source
  recognizes it on sight.
- **A conflict above is not settled by the evidence.** Do not carry an unsettled conflict through to
  a recommendation. In evaluated runs of this workflow, agents named conflicts accurately,
  recommended a resolution, applied it, and moved on — and not once asked the human anything. A
  recommendation the human can only decline by noticing it is not a decision they made.
- **You want a scope exception.** Name the query, what it would settle, and why the sample cannot
  settle it. Then wait.

Two guards. **Asking is not automatically good:** a question you could answer yourself with one
query is outsourcing, not partnership — run the query instead. And **there is no credit for
volume**; one question that matters beats four that do not.

If none of the above applies, write "none". That will often be true, and it should be.

### Recommendation

The primary candidate, the alternates, and the specific reason each alternate lost. If evidence is
insufficient to choose, say so and mark it provisional rather than picking.

**There is a third outcome besides mapped and not-found: the field exists and cannot be used.** A
column can be real, plainly named, and still carry nothing a mapping could rely on. Persisting past
that point burns turns and produces nothing; mapping it anyway hands the human a field that fails
the first time someone reads it.

Three facts settle it, none requiring a scope exception or a scan of the table:

1. **Population across the whole source, not the sample.** The schema catalog records a null count
   per column. A column empty for 99% of every row in the source is a different object from one that
   happens to be empty across a few hundred sampled entities.
2. **Whether a decode target exists at all.** No foreign key, no similarly-named lookup, and values
   that join to nothing is the signature of a vendor feature nobody finished. An abstract key with
   nothing to decode it against is not a lead awaiting a lookup — the absent lookup is the finding.
3. **Cardinality of the populated remainder.** A handful of distinct opaque values across the
   populated fraction means an abandoned field. Tens of thousands means a real payload you have not
   decoded yet.

When all three agree you have a licence to stop, and stopping is correct. Say it as a positive
finding — *this field exists, and here is why it cannot be used* — recommend the decision state in
`configs/mappings/README.md` that fits, and move on. That is a complete answer, not a failure to
find one. Do not phrase it as "not found": a column visible in the catalog reads as a miss when
reported missing, even where the practical conclusion is right.

Finally, **do not propose profiling as a next action.** Value distributions, quality issues,
outliers, and population characteristics belong to a later phase. Where you noticed something a
profiler will want, put it in one line under **Changes to apply** and let it go.

### Changes to apply

Not propose — apply. A correction that is only described has not been made.

**The mapping TOML**, conforming to `configs/mappings/README.md`. Read it before writing, every
time; this is the most frequently failed part of the response, and the same two defects recur:

- **Every candidate needs its `[[sources]]` and `[[joins]]` entries.** A candidate whose table is
  not declared is not addressable.
- **A candidate expression is almost always a plain aliased field reference** — `table_alias.field_name`. If
  you are writing a `CASE`, a `COALESCE`, or a concatenation, stop: you are encoding a decision into
  the expression that belongs in the decision state, or merging fields the model wants kept apart.

**Survey corrections**, if you found any — a missing lead, a stale one, a structural fact it did not
record, an absence claim its own artifacts refute. Edit the survey; do not merely report it.

**A working note**, if this turn produced a correction from the human that would otherwise be lost
at the next context clear. Propose the line and let them approve it.

## If this turn keeps going

A hard attribute takes several exchanges. **This is where the work reliably falls apart**, and it
falls apart the same way every time: by the fourth or fifth exchange the response format is gone,
the sample constraint has been forgotten, scratch files are accumulating in the repo, and the
original question — *which field should this attribute map to* — has been replaced by whatever
narrower thing the last message was about.

In evaluation this was the single largest failure. On the hardest attribute, agents ran the right
probe, obtained a near-perfect result for the correct field, and then **did not recommend it**,
because by then they were answering the most recent sub-question rather than mapping the attribute.
The evidence was there. The goal was not.

So on **every** exchange after the first, before writing anything:

1. **Re-emit the full required response shape.** All eight sections — not a freeform paragraph
   continuing the conversation. Later exchanges are where the sections earn their keep.
2. **Restate the attribute you are mapping**, in one line, at the top. If your last three messages
   were about a single candidate, that line is what pulls you back out.
3. **Re-read the constraints.** Sample scope in particular: a widened or improvised sample from an
   earlier exchange does not carry forward as approved, and a scratch file written mid-turn belongs
   in a temp directory outside the project, not in the repo.
4. **Ask whether new evidence changes the recommendation.** A candidate that tested well three
   exchanges ago and has not been beaten is still your primary. Say so plainly rather than letting
   it drop off because the conversation moved on.

**At the fourth exchange, consider stopping.** If the attribute is not converging, say so and
propose restarting it in a fresh session, with the findings so far written into the survey and the
working notes first. That is a cheap, reasonable request and a better outcome than a recommendation
nobody can trust.

## Then stop

Wait for the human. Do not begin the next attribute, do not run reports, do not open a PR.
