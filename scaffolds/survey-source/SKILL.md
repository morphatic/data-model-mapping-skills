---
name: survey-source
description: Build a localized model of an undocumented data source for one domain — the tables, join paths, lookup structures, and naming conventions that mapping will need. Use before mapping a domain against a source for the first time, or when an existing survey is stale.
argument-hint: <source> <domain>
user-invocable: true
disable-model-invocation: true
---

# Survey a source for one domain

Produce the document a mapper would need and does not have: what this source contains in the
neighborhood of one domain, how its pieces connect, and what its conventions are.

This is **not** mapping. You are not choosing fields or deciding which candidate wins. You are
producing leads, with enough structure behind them that adjudication becomes cheap. If you find
yourself weighing two candidates against each other, stop — that belongs to `/map-attribute`.

Read `.github/instructions/data-mapping.instructions.md` before starting. It carries the
environment facts, scope constraints, and evidence standards this workflow assumes. Those
instructions are scoped to mapping paths and may not have auto-loaded yet.

## Scope, and how to keep it

The source has hundreds of tables and most are irrelevant. A census is worthless; the useful
artifact is local.

**Anchor first, and confirm it with the human before expanding.** Everything downstream is
described by its distance from the anchor, so a wrong anchor does not degrade the survey — it
invalidates it. This is the one place to stop and check.

Read `configs/domains/<domain>.toml` for the grain and business key. Those names are abstract
and will usually not appear in the source. Do not search for them literally and do not assume a
column bearing the domain's name is the right one; a source can contain a column named exactly
like the domain's business key that means something else entirely.

Three separate things must be identified, and in most sources they are three different columns:

1. **The anchor table** — where this domain's grain lives. Its name may not contain the domain
   term at all.
2. **The business identifier** — the column a human would use to name one entity, and what the
   domain's `business_key` actually corresponds to.
3. **The join key** — the column other tables use to point at this one. Often a surrogate, often
   not the business identifier, and it is the one that matters for building join paths.

**The identifier may not live in the anchor table.** Some sources model a shared entity — one table
holding every party to any interaction, where a role determines which domain a given row belongs to.
In that shape the anchor is a table **plus a qualifying join**: the shared table restricted to rows
that a satellite table identifies as belonging to this domain. There may be more than one such
satellite, each covering a different product or program, in which case the qualification is a union
across them. Satellite tables are frequently named for something other than the domain, so a name
search for the domain term will not find them.

**Test whether the anchor table is shared before trusting any bounding rule based on connectivity.**
One query answers it: does this table contain rows that are not entities of this domain, and what
fraction? If most of its rows are not yours, connectivity is worthless as a filter — nearly
everything in the source will be reachable from it — and inclusion must be decided purely on whether
a table carries an attribute of *this* domain.

Cheap tests that discriminate between a real identifier and a decoy:

- **Match against the sample.** The member sample CSV is keyed by a real identifier. The column
  whose values intersect it is the business identifier; a same-named column whose values do not
  is a decoy. This settles the question in one query.
- **Distinctness against the claimed grain.** Count rows versus distinct values. A business
  identifier that repeats means the table is not at the grain you assumed — usually history or
  versioning, which changes every probe that follows.
- **Reach.** A join key appears in other tables. Query the schema catalog for columns sharing the
  name. A candidate key that appears nowhere else is not how this source joins.

Report all three with the evidence that chose them, name the decoys you rejected and why, and
state how current-state rows are selected if the anchor carries history or versions. Then stop
and confirm with the human before expanding. They know this source's core table and can settle
in one sentence what would otherwise cost you an hour.

**Expand outward, not alphabetically.** Follow join paths from the anchor. Include a table when
it is reachable from the anchor *and* plausibly carries a domain attribute. Stop expanding a
branch when it stops being either.

**Two hops is normal. One hop is not a survey.** This is the most expensive mistake available
here. When this workflow was evaluated, two independent agents each built only depth-1 stars off
the anchor, and between them missed every attribute living behind a bridge table, a code lookup
reached through a bridge, or a date dimension — mapping 8 and 9 attributes out of 21 where a person
surveying the same source reached 20. Anything ranked, dated, decoded, or many-per-entity is
usually two or three joins out. Follow the chain until it stops paying, and record the hop count
for every table you include.

**Small tables are cheap to read; large ones are not.** Reference and lookup tables can be read
in full. Fact and history tables should be examined by structure, and by values only through
the member sample.

**Report values, not counts of values.** A distinct count is not a finding. "Thirteen distinct type
keys appear across the sampled entities" tells a mapper nothing they can act on; the thirteen labels
tell them which attributes this source carries. Whenever you count distinct values of a
discriminator, code, or type column, print the values themselves. In the evaluation, an agent ran
exactly the join that would have resolved three unmapped attributes, reported how many distinct type
keys it had matched, discarded the labels, and then recorded all three attributes as absent from the
source.

**Say what you skipped.** A survey that claims completeness is worse than one with an honest
boundary.

**But your boundary is drawn from the inside.** A *Not examined* section can only list what you
know you skipped, which is never the category that hurts. In the evaluation, two agents each
declared the survey complete — with honest-sounding boundaries, explicit hop counts, and unfiltered
lookup reads — while both had missed the same generic-container structures, and with them the six
attributes that lived behind those structures. Neither *Not examined* section mentioned any of it.
Both agents found the missing branch within one pass of a human naming it out loud.

So do not judge completeness against the source, whose edges you cannot see. Judge it against the
domain model, which is finite and in front of you. See **Before you call it done**.

## The artifact

Write to `docs/sources/<SOURCE>/<domain>-survey.md`. Propose it before you write it.

Markdown, markdownlint-clean, hierarchical headings. Sections:

### Provenance

What was surveyed, when, against which schema, by what means, and what remains unexamined.

### Open questions

Things a human or SME must answer. This section exists so uncertainty has somewhere to live
other than a confident guess.

Three kinds belong here, and all three are cheap for a human to settle — often in one sentence.
Raising them is not an imposition. Answering them on their behalf is not efficiency.

1. **Facts about the world that a database cannot contain.** The shape of an external code
   standard, what a superseded acronym stood for, what format an issuing authority uses, whether
   two systems were ever meant to agree. No amount of querying produces these, and a person who
   works here knows most of them offhand.
2. **Labels you cannot adjudicate.** When an enumerated discriminator offers several values and
   their names do not tell you which one belongs to this domain, list the values with their
   populations and ask. Someone who knows this source will recognize it on sight.
3. **Scope exceptions you want.** If a question genuinely cannot be answered inside the sample,
   say what you would run, what it would settle, and wait.

### Source conventions

Patterns that hold across the source, each with the evidence that established it — naming
schemes, suffix conventions, how keys are formed, how history and effective-dating are handled,
whether names are stored alongside codes or only behind lookups.

Conventions inferred from two examples are hypotheses. Mark them as such.

**Record naming vintage.** Note abbreviations, superseded acronyms, and organization names that have
since changed. A table named for an agency renamed two decades ago is invisible to anyone searching
for the current name, and that single fact decides whether every subsequent attribute search in this
source succeeds or fails. List the dead terms you find alongside their modern equivalents, so the
next person searches for both.

### Anchor and identity

State plainly, as three labeled items, the anchor table, the business identifier, and the join
key — with the evidence that selected each. Downstream mapping needs the business identifier for
`[identity]` and the join key for `[[joins]]`, and they are usually not the same column.

If the anchor required qualification — a shared entity table restricted by a join to one or more
satellites — state the qualifying join and the satellites in full, and say what fraction of the
shared table belongs to this domain. A mapper who does not know the anchor is qualified will
silently return every party in the source.

Then record:

- **Decoys** — columns whose names suggest they are the identifier but are not, with the test
  that ruled them out. A future mapper will otherwise fall into the same trap.
- **Grain reality** — whether the anchor is one row per entity, or per version, or per history
  record, and the exact filter or selection that yields current state.
- **Anything else that makes a row unique** — composite keys, effective dating, soft deletes.

### Neighborhood

Every table you are including, with: name, one-line purpose, join path back to the anchor,
approximate row count, and which domain attributes it might serve.

### Reference and lookup tables

Their shape, what they decode, and the column that carries the human-readable value. Note where
a code appears to follow an external standard — fixed-width, numeric, a row count matching a
known universe — and say what would confirm it.

**Enumerate them completely, and unfiltered.** These tables are small; read all of their rows and
record the full value list. Do **not** filter a lookup or type table by a name pattern taken from
the attribute you happen to be chasing — that is exactly how a type whose label you would never
have guessed stays invisible. In the evaluation, both agents searched a shared identifier-type
table only for names matching the attribute they wanted, and both consequently missed identifier
types that were sitting in the same table under labels their search terms did not contain. A single
unfiltered enumeration would have handed them several attributes at once.

### Tables whose names do not announce them

A section that exists because it is the one most often missing — and the one most often filled in
wrongly. **This is a probe to run, not a slot to describe findings in.** When it was evaluated, both
agents populated it with bridge tables they had already found by name, and neither read it as an
instruction to go looking.

Some sources keep domain content in tables whose names and columns are entirely generic — a
value table keyed by an abstract type, a single shared lookup serving unrelated concepts, an
attribute-value pair structure, a JSON or enum column. These are invisible to any search over
names. They are found only by reading what is inside them.

**The sweep.** For every table in the neighborhood carrying a type or code discriminator beside a
payload column — a `*_TYPE_KEY`, a `*_TYPE_CODE`, an abstract `*_KEY` paired with a value column —
enumerate that discriminator's values *as they appear on sampled entities*, and list them in full.
Not how many there are. The values.

Run the sweep before you write the attribute leads, and run it without reference to the attributes
you are hoping to find. A discriminator filtered by a term you already had in mind can only return
what you already knew.

For each structure the sweep turns up, record the discriminating key, the join path to it, and which
of its values correspond to domain concepts. If the sweep turned up nothing, name the tables you
swept and say what their discriminators contained.

**If the labels do not adjudicate, do not choose.** Generic containers accumulate generic labels for
specific things. The value carrying what a domain needs is frequently the one whose name gave no
hint at all, while the plausibly-named values are empty, deprecated, or belong to a different
domain. Rank nothing on how a label reads. List the values with their populations and put the
question under **Open questions**, where a human settles it in a sentence.

### View definitions worth reading

Views are usually written by the organization to make a vendor's schema usable, so their SQL
encodes institutional knowledge about which tables and columns are actually authoritative.
Retrieve the definitions of views in this neighborhood and record what their joins and filters
reveal.

### Attribute leads

For each attribute in the domain model: candidate table and column in `TABLE.COLUMN` form, the
join path if it is not on the anchor, **the hop count of that lead**, and a one-line note on why it
is a lead. Hop count per lead matters more than hop count per table: it records how far out the
attributes actually live, which is what tells the next person where to start looking.

**"Not found" is the most expensive claim in this document.** It is the sentence that stops the next
person from looking. Before writing it for any attribute, do two things:

1. Search the lookup and discriminator values *you enumerated in this survey* for the attribute's
   name and its obvious synonyms. In the evaluation, an agent extracted a 105-row type dictionary
   holding an exact match for the attribute it was chasing, saved that extract into the survey's own
   folder, and recorded the attribute as not present in the source.
2. Confirm the sweep above actually ran, and name the discriminators it read.

A name search returning no hits is not the evidence. It is the thing that has to be overcome.

**Where a lead applies to only part of the domain — one program, product, or line of business —
say which part, and report its population within that subset rather than across all sampled
entities.** An identifier issued by a program covers the members of that program and nobody else.
Measured against everyone, it looks like a sparse and unpromising field; measured against the
population it is for, it may be nearly complete. A lead recorded without its qualifying subset
invites the mapper to reject it on a denominator that was never the right one.

**Where an attribute has several leads that cover different rows rather than competing for the same
ones, say so explicitly.** The signature is leads on separate join paths, or on the same column
separated by a type code, each well populated within its own branch and empty outside it. That is
one attribute stored several ways, not several candidates for one answer, and the two get confused
constantly. Note it as a *possible discriminated attribute* and record what appears to distinguish
the branches. Naming it here is cheap; leaving the mapper to discover it costs a session, and the
mistake it invites — concluding the attribute cannot be mapped, and proposing to redesign the domain
model until it can — is expensive to unwind.

Do not rank leads or select a primary. That is the mapper's job.

### Not examined

Tables, schemas, or structures deliberately left out, and the condition under which they would
be worth revisiting.

## Before you call it done

Completeness cannot be judged against the source; you cannot see its edges. Judge it against the
domain model, which is finite and sitting in front of you.

Walk the attribute list in `configs/domains/<domain>.toml` and check every one:

- Each attribute has either a lead with a join path and a hop count, or a "not found" that has
  survived both tests in **Attribute leads**.
- Every generic container the sweep identified has its discriminator values listed, not counted.
- No attribute is recorded as absent while a value you enumerated in this survey names it.

If any of those fails you are not done, whatever the section structure looks like. A survey with
every required section filled in and an entire branch of the source missing reads exactly like a
finished one. That is the failure this check exists to catch.

## Working rhythm

This is longer than one turn. Work in passes and report between them; the document is the work
product, so a session that ends mid-survey resumes by reading it.

Between passes, tell the human what you found, what changed your picture of the source, and what
you propose to look at next. If something you find contradicts an earlier section, revise that
section rather than appending to it.

Constraints in `.github/copilot-instructions.md` apply throughout — in particular, no
full-population queries without approval, and nothing written into the project without
proposing it first.
