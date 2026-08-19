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

**A lookup's own row counts point the wrong way.** Reading a code table tells you what the source
*could* represent. Only counting what entities actually carry tells you what it *does*. These come
apart badly: a shared lookup will often devote hundreds of rows to a fine-grained standard nobody
populates, while the family that holds the real data is a handful of local values that look
unofficial. In evaluation, both variants read a lookup unfiltered, correctly identified the
official-looking family, filtered to it, got zero rows, and reported the attribute absent — while
the family carrying it sat in the same table under a scruffier prefix. **Enumerate the lookup, then
group the bridge by family and count members.** The second step is not optional, and the first is
misleading without it.

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

**You must actually query the source in this turn.** A recommendation supported only by the survey
is not a mapping, it is a summary of a document someone else wrote. The survey is a lead sheet: it
tells you where to point a probe, never what the probe would have returned. In evaluation, agents
repeatedly produced complete, well-argued, correctly formatted recommendations having run **zero
queries** — reasoning entirely from the survey and from the schema catalog. If you find yourself
about to recommend without a probe, that is the signal to run one, not evidence that none was
needed. If you genuinely believe no probe is warranted, say so explicitly and say why; do not let it
pass unmentioned.

Then state, in one sentence: **what result would have changed your recommendation.** If no
possible result would have changed it, you did not run a probe, you ran a confirmation.

Agreement between two fields establishes that they are consistent with each other. It does not
establish that either is correct.

**State the denominator, and build the sample to match it.** A population figure means nothing
until you say what it is a fraction of. Where an attribute applies only to part of the domain — one
program, product, or line of business — **restrict the sample to that part before you count, rather
than counting across everyone and interpreting the shortfall.** Check what the sample file already
carries: it frequently holds the very column that identifies the qualifying subset, which makes
this one predicate rather than a research problem. A drifting-down coverage number across
candidates that all apply to a subpopulation tells you nothing about any of them.

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

Conforming to `configs/mappings/README.md`. Read it before writing, every time — this is the most
frequently failed part of the response, and the same two defects recur:

- **Every candidate needs its `[[sources]]` and `[[joins]]` entries.** A candidate whose table is
  not declared is not addressable.
- **A candidate expression is almost always a plain aliased field reference** — `moi.field_name`.
  If you are writing a `CASE`, a `COALESCE`, or a concatenation, stop: you are encoding a decision
  into the expression that belongs in the decision state, or merging fields that the model wants
  kept apart.

Apply the edit as well as proposing it, along with any survey correction you named above. A
correction that is only described has not been made.

### Out of scope

What you noticed that belongs to **profiling** rather than mapping — value distributions,
quality issues, outliers, population characteristics. Name it here and do not pursue it. Deeper
characterization of the data is a later phase.

## If this turn keeps going

A hard attribute takes several exchanges. **This is where the work reliably falls apart**, and it
falls apart the same way every time: by the fourth or fifth exchange the response format is gone,
the sample constraint has been forgotten, scratch files are accumulating in the repo, and the
original question — *which field should this attribute map to* — has been replaced by whatever
narrower thing the last message was about.

In evaluation this was the single largest failure. On the hardest attribute, agents ran the right
probe, obtained a near-perfect result for the correct field, and then **did not recommend it**,
because by that point they were answering the most recent sub-question rather than mapping the
attribute. The evidence was there. The goal was not.

So on **every** exchange after the first, before you write anything:

1. **Re-emit the full required response shape.** Every section, every time — not a freeform
   paragraph continuing the conversation. Later exchanges are where the sections earn their keep,
   not where they become redundant.
2. **Restate the attribute you are mapping**, in one line, at the top. If your last three messages
   were about a single candidate, that line is what pulls you back out.
3. **Re-read the constraints you are working under.** Sample scope in particular: a widened or
   improvised sample from an earlier exchange does not carry forward as approved, and a scratch
   file written mid-turn belongs in a temp directory outside the project, not in the repo.
4. **Ask whether new evidence changes the recommendation.** A candidate that tested well three
   exchanges ago and has not been beaten is still your primary. Say so plainly rather than letting
   it drop off because the conversation moved on.

If a turn has become tangled enough that you cannot do this cleanly, say so and propose starting
the attribute fresh. That is a cheap, reasonable request and it is a better outcome than a
recommendation nobody can trust.

## Then stop

Wait for the human. Do not begin the next attribute, do not run reports, do not open a PR.
