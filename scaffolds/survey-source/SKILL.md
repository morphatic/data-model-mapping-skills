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

**Say what you skipped.** A survey that claims completeness is worse than one with an honest
boundary.

## The artifact

Write to `docs/sources/<SOURCE>/<domain>-survey.md`. Propose it before you write it.

Markdown, markdownlint-clean, hierarchical headings. Sections:

### Provenance

What was surveyed, when, against which schema, by what means, and what remains unexamined.

### Open questions

Things a human or SME must answer. This section exists so uncertainty has somewhere to live
other than a confident guess.

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

A section that exists because it is the one most often missing.

Some sources keep domain content in tables whose names and columns are entirely generic — a
value table keyed by an abstract type, a single shared lookup serving unrelated concepts, an
attribute-value pair structure, a JSON or enum column. These are invisible to any search over
names. They are found only by reading what is inside them.

List what you found this way, what the discriminating key is, and which values in it correspond
to domain concepts. If you found none, say how you checked.

### View definitions worth reading

Views are usually written by the organization to make a vendor's schema usable, so their SQL
encodes institutional knowledge about which tables and columns are actually authoritative.
Retrieve the definitions of views in this neighborhood and record what their joins and filters
reveal.

### Attribute leads

For each attribute in the domain model: candidate table and column in `TABLE.COLUMN` form, the
join path if it is not on the anchor, and a one-line note on why it is a lead. Where nothing was
found, say so explicitly — that is a finding, not a gap in the survey.

Do not rank leads or select a primary. That is the mapper's job.

### Not examined

Tables, schemas, or structures deliberately left out, and the condition under which they would
be worth revisiting.

## Working rhythm

This is longer than one turn. Work in passes and report between them; the document is the work
product, so a session that ends mid-survey resumes by reading it.

Between passes, tell the human what you found, what changed your picture of the source, and what
you propose to look at next. If something you find contradicts an earlier section, revise that
section rather than appending to it.

Constraints in `.github/copilot-instructions.md` apply throughout — in particular, no
full-population queries without approval, and nothing written into the project without
proposing it first.
