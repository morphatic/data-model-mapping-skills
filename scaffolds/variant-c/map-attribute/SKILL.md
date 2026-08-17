---
name: map-attribute
description: Map one domain attribute to candidate physical fields in a source system, gather evidence, and recommend a primary. Use when the human names a single attribute to map, or asks to continue mapping the next attribute in a domain.
argument-hint: <attribute-name>
---

# Map one attribute

You are mapping **one** attribute this turn. Not the domain. Not the next attribute after it.

Read `configs/mappings/README.md` before writing TOML. It defines the allowed values for
`candidate_resolution_mode`, `selection_status`, candidate `status`, `sample_scope`, and the
decision-state notation. Use it; do not invent vocabulary.

## Start with the survey

Look for `docs/sources/<SOURCE>/<domain>-survey.md`.

**If it is missing**, say so and recommend the human run `/survey-source <source> <domain>`
first. Mapping without it means rediscovering the same source structure once per attribute, and
arriving at a different picture each time. Offer to proceed unaided if they would rather — that
is their call — but do not quietly proceed as though nothing were missing.

**If it is present**, read four things: the leads for this attribute, the *Not examined*
section, the *Open questions*, and the provenance date. Then treat it as a **lead sheet, not an
answer key**:

- Verify a lead still exists before building on it. Surveys go stale and schemas move.
- Its *Not examined* section tells you where it stopped looking. For this attribute
  specifically, does that boundary matter?
- Its *Open questions* are still open. A question the survey raised correctly was not resolved by
  being written down, and this is the moment one of them becomes actionable. Carry forward any
  that bear on this attribute — under **What I need from you** if it blocks the mapping, under
  **Survey corrections** if you settled it yourself.
- If you find a candidate the survey does not have, that is a finding about the survey as well
  as the attribute. Say so, so it can be corrected.

The survey also records the anchor table, the business identifier, and the join key, which are
usually three different columns. Use them rather than re-deriving them.

What follows is a worked example. Work the way it works.

---

## Worked example: mapping `region` in a source with a code/name split

### Finding the candidates

The survey listed two leads for `region`: a `REGION_CODE` column on each of two address tables.
Both were confirmed to still exist, and the join paths it recorded were correct.

Before accepting that as the candidate list, two questions were worth asking. The survey's own
provenance said its neighborhood walk was built from object-name search and join-path expansion.
So: **what would a search conducted that way be structurally unable to find?** A name search
finds fields whose names announce themselves. It cannot find a field whose name is generic,
whose meaning lives in its values, or whose relevance only appears once you look inside
something. And the survey's *Not examined* section named two schemas it had skipped.

Neither turned out to matter here — but that was established by checking, not by assuming.

### The shallow pass — this is what going wrong looks like

The attribute `region` exists in the domain model. Two address tables ("primary" = `ap` and
"contact" = `cp`) carry a `REGION_CODE` column. A tiny dual-path probe returns:

```toml
sample_members=108
primary_rows=160
contact_rows=268
populated_primary_region=73
populated_contact_region=109
agree=109
```

Both paths agree. The obvious move is to write `region = ap.REGION_CODE`, note the alternate,
update the mapping file, and report the attribute complete.

**That report would be wrong, and every step in it was correct.** The probe ran, the numbers
are real, the file validates. What is missing is not a step. It is the questions nobody asked.

### The questions that were missing

1. **Semantic mismatch.** The domain model defines `region` as a business attribute. The
   mapping is a code. That runs against the stated preference for human-readable values.
2. **Code-system ambiguity.** Is `REGION_CODE` an external standard, a plan-internal key, or
   source-specific? Nothing in the probe answers this, and the answer changes everything
   downstream about comparability across sources.
3. **False confidence from agreement.** The two paths agreeing proves the two paths are
   consistent with each other. It does not prove either is correct against the canonical
   meaning. Agreement between two copies of the same mistake looks exactly like this.
4. **Coverage shape.** Population differs across the join paths (73 vs 109) on different row
   counts (160 vs 268). Before `agree=109` means anything, define whether these are per-member
   or per-row figures. An agreement count exceeding one side's population is a sign the units
   are mixed.

### The second pass

Probe the lookup rather than the column:

```toml
both code paths resolve to DB_REF.REGION_CODE — 73/73 and 109/109 matched, 0 unmatched
DB_REF.REGION_CODE columns: REGION_ID, REGION_CODE, REGION_DESC, REGION_NAME
3227 rows; every REGION_CODE value is 5-digit numeric
```

Now the picture inverts. A readable `REGION_NAME` exists and is reachable by join — so the
preference is satisfiable. *And* the code is fixed-width numeric across 3227 rows, which is
the signature of an external standard, which means the code carries cross-source
comparability the name does not.

The survey had recorded the lookup table's existence but not that its codes were fixed-width.
That is worth sending back.

### The conflict, named out loud

Two stated rules now point in opposite directions:

- prefer human-readable values over abstract codes → map `REGION_NAME`
- standardized codes are comparable across sources → map `REGION_CODE`

**Neither rule wins by itself, and picking one silently is the error.** The resolution was not
to choose a rule. It was to change the domain model: split `region` into `region_code` and
`region_name`, map both, and record why. That decision belonged to the human, and the job was
to surface the conflict, verify the facts that made it a real conflict, and propose the
options:

- A — rename the attribute to `region_code`, describe it as the standard code
- B — redefine it as `region_name` and map through the lookup
- C — model both

Recommendation stated, with a reason. Human chose. Then the edits were made.

---

## The other failure direction

The example above is about stopping too early. The opposite failure is just as real.

On an attribute with four candidates where the evidence converged quickly — one field
clearly best populated, clean agreement with its alternates — the right move was to stop.
Further characterization of value distributions, outliers, and quality belongs to **profiling**,
which is a later phase and not this job. Documenting the candidates, ranking them, and
recording the evidence that separated them is the whole task.

Stop when one more query would tell you about the *data* rather than about *which field to map*.

---

## Required response format

Emit every section, in the order below, every time. Everything above **Recommendation** is written
before it — a concern written after a conclusion tends to justify the conclusion.

Render each section name as a markdown heading. Do not collapse them into one numbered or bulleted
list: several sections contain lists of their own, and nesting those inside an outer list makes the
reply unreadable.

This reply is the deliverable. Writing a summary into a project document as well is useful, but the
reasoning must be here in full — anything that lives only in a file was not said.

### What the survey gave you

The leads it held for this attribute, whether they verified, and what its stated boundaries mean
for this attribute specifically.

### What you added

Objects you interrogated beyond the survey and what each turned up, including the searches that
returned nothing. If you added nothing, say why the survey was sufficient rather than leaving it
implied.

### Candidates

Each one traced to the lead or search that produced it.

When candidates come from the values of a discriminator, **the label is the weakest evidence you
have**. Rank them by population and by whether their format fits the attribute, never by which label
reads most like the attribute's name. Long-lived systems accumulate generic labels for specific
things: the value carrying what you want is frequently the one whose name gave no hint, while the
obviously-named ones turn out empty, deprecated, or owned by another domain. If the labels do not
adjudicate, list them with their populations under **What I need from you** instead of picking the
one that sounds right.

### What you did not check

And why it might matter.

### Conflicts

Between two stated preferences, or between the domain model and what the source actually offers.
Say "none" only after looking. If a conflict is not settled by the evidence, end with a direct
question and stop there rather than carrying it through to a recommendation — naming a conflict and
then resolving it yourself is the same as not raising it.

### Evidence

The probe, its scope, and the counts. State what result would have changed your conclusion, and
state the denominator — a population figure means nothing until you say what it is a fraction of.
If the attribute applies to only part of the domain, count against that part.

If a probe returns zero population on every candidate, that is not evidence of absence — and
widening the sample is the *second* thing to try. **Widen the hypothesis before you widen the
rows.** A zero across everything you tried usually means the candidate set is wrong, not that the
sample is too small. Go back to the enumeration the set came from and ask which values you passed
over and what made you pass over them. The same predicate against ten times the entities returns
the same zero, more slowly and with more confidence attached.

### What I need from you

The questions a person answers faster than you can investigate them. Three kinds belong here:

- **A fact the database cannot contain** — the format an issuing authority uses, what a superseded
  acronym stood for, the shape of an external code standard. Querying does not produce these at any
  budget, and the human usually knows them offhand.
- **A label you cannot adjudicate** — the enumerated values with their populations, and the
  question of which one is yours. Someone who knows this source recognizes it on sight.
- **A scope exception you want** — the query, what it would settle, and why the sample cannot.

Usually this is one word: none. When it is not, keep it short and answerable. A question that takes
ten seconds to answer is worth asking; one that requires the human to reconstruct your reasoning
first is not.

This is not a substitute for **Conflicts**, which stops the turn. These are cheap questions that do
not block — state them and carry on, marking the recommendation provisional if the answer would
change it.

### Recommendation

The primary, the alternates, and the reason each alternate lost.

**There is a third outcome besides mapped and not-found: the field exists and cannot be used.** A
column can be real, plainly named, and still carry nothing a mapping could rely on. Three facts
settle it, none of them needing a scope exception or a scan of the table:

1. **Population across the whole source, not the sample.** The schema catalog records a null count
   per column. A column empty for 99% of every row in the source is a different object from one
   that happens to be empty across a few hundred sampled entities.
2. **Whether a decode target exists at all.** No foreign key, no similarly-named lookup, values
   that join to nothing — the signature of a vendor feature nobody finished. An abstract key with
   nothing to decode it against is not a lead awaiting a lookup; the absent lookup is the finding.
3. **Cardinality of the populated remainder.** A handful of distinct opaque values across the
   populated fraction means an abandoned field. Tens of thousands means a real payload you have
   not decoded yet.

When all three agree you have a licence to stop, and stopping is correct. Say it as a positive
finding — *this field exists, and here is why it cannot be used* — recommend the decision state in
`configs/mappings/README.md` that fits, and move on. That is a complete answer, not a failure to
find one. Do not call it "not found": a column visible in the catalog reads as a miss when reported
missing, even where the practical conclusion is right.

### Survey corrections

Anything you learned that the survey should have said. Report it; do not edit the survey yourself
unless asked.

### Proposed TOML edit

Conforming to `configs/mappings/README.md`.

## Then stop

Wait for the human. Do not begin the next attribute.
