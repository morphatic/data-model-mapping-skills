# Migration: discriminated mapping

A manual migration guide. Apply these changes by hand, in order. Nothing here requires running a
script, and nothing here is source-specific — every physical table, column, and code value appears as
a `<PLACEHOLDER>` you fill in from your own environment.

**What this adds.** A way to say that one canonical attribute is served by several physical fields,
selected by a discriminator, without writing dialect-specific SQL into a mapping file.

**Why.** The mapping layer previously required exactly one physical field per canonical attribute.
That rule is an extraction-layer constraint, but because it was unwritten it propagated upward: when
a source could not satisfy it, the only thing that could give was the domain model. The result was
domain decomposition driven by source structure — which is the one thing a canonical model exists to
prevent. This change puts the constraint where it belongs and gives it a defined escape hatch.

**Scope.** Six work items across four areas: mapping grammar, harness code, domain files, and skill
text. Items 1–2 are the substance; 3–6 are cleanup that follows from them.

---

## Contents

1. [The grammar](#1-the-grammar)
2. [Harness and validator changes](#2-harness-and-validator-changes)
3. [Mapping README: the fifth decision state](#3-mapping-readme-the-fifth-decision-state)
4. [Domain file changes](#4-domain-file-changes)
5. [Skill text changes](#5-skill-text-changes)
6. [Working note](#6-working-note)
7. [Order of operations](#7-order-of-operations)

---

## 1. The grammar

### 1.1 The core idea

A **branch** is a set of rows that carries a known value of a discriminator, plus the expressions
that are valid for those rows. A mapping file with branches produces the union of its branches.

Two things establish which rows are in a branch, and a branch may use either or both:

| Key | Meaning | Use when |
| --- | --- | --- |
| `implies` | The branch **asserts** the discriminator value. Membership is established by the join path. | The source does not store the discriminator anywhere. Arriving through this path *is* what makes the row an email. |
| `when` | The branch **tests** a physical predicate. | The source stores the discriminator, or a related type code, in a column. |

`when` selects rows. `implies` labels them. When both are present, the branch is *the rows matching
the predicate, arriving through this path, labeled with this value*.

Both compile to the same thing — a leg of a `UNION ALL`. When every branch in a file uses only
`when` against the same base rows, the branches partition one row set and the harness may emit a
`CASE` instead. That is an optimization, not a semantic difference, and nothing in a mapping file
should depend on which one it picks.

### 1.2 Grain

Declare the mode at the top of the mapping file, next to `[defaults]`.

```toml
[grain]
mode = "discriminated"        # "simple" is the default and may be omitted
discriminator = "<attribute>" # must name an attribute in the domain file
```

`mode = "simple"` is every mapping file you have today: one row per business key, `[fields]` holds
one expression per attribute. Nothing changes for those files.

`mode = "discriminated"` means the file produces one row per *branch member*. This is a real grain
change and it is the point — a discriminated attribute is not a column-level trick, it is a
row-producing construct that defines the grain of the mapping.

### 1.3 Branch blocks

Branches reuse the existing `[[sources]]` and `[[joins]]` machinery. A branch does not redeclare
aliases; it references aliases declared at file level, exactly as `[fields]` does. The harness
derives the join subset a branch needs by walking from its expressions to the base alias — the same
rule already documented for `[[joins]]`.

```toml
[[branches]]
branch_id = "email"
implies = { contact_type = "email" }

[branches.fields]
contact_value = "<EMAIL_ALIAS>.<EMAIL_COLUMN>"

[[branches]]
branch_id = "phone"
implies = { contact_type = "phone" }
when = { "<PHONE_TYPE_ALIAS>.<TYPE_CODE_COLUMN>" = ["<HOME>", "<WORK>", "<MOBILE>"] }

[branches.fields]
contact_value = "<PHONE_ALIAS>.<PHONE_NUMBER_COLUMN>"

[[branches]]
branch_id = "fax"
implies = { contact_type = "fax" }
when = { "<PHONE_TYPE_ALIAS>.<TYPE_CODE_COLUMN>" = "<FAX>" }

[branches.fields]
contact_value = "<PHONE_ALIAS>.<PHONE_NUMBER_COLUMN>"
```

Note the phone/fax pair. Both reach the same physical column through the same path and are separated
only by a type code — `when` does that work. Both then assert different canonical values — `implies`
does that. Neither key alone expresses this branch; that pair is the reason to keep both.

### 1.4 Branch-invariant attributes

`[fields]` keeps its current meaning and gains one: in a discriminated file it holds the attributes
that are the same in every branch. The harness projects them into every leg.

```toml
[fields]
member_id = "<BASE_ALIAS>.<MEMBER_ID_COLUMN>"
```

An attribute appearing in both `[fields]` and a `[branches.fields]` block is an error, not an
override. It is ambiguous and should fail validation.

### 1.5 The discriminator attribute itself

Do not map the discriminator in `[fields]`. It is produced by branch membership — the harness emits
each branch's `implies` value as a literal in that leg's projection.

This has a useful consequence. If your source has a communication-method column that is largely
unpopulated, you are no longer blocked by it, because you were never depending on it. The branch
structure carries the mode. The unpopulated column becomes a **governance finding** — *the
enterprise has a mode field and does not use it* — recorded in `[notes]` rather than a reason the
attribute cannot be mapped.

### 1.6 Predicate grammar for `when`

Deliberately restricted, so it is dialect-neutral by construction rather than by review. Everything
below is identical across Oracle, Databricks SQL, Postgres, and T-SQL.

| Form | Emits |
| --- | --- |
| `when = { "alias.COLUMN" = "value" }` | `alias.COLUMN = 'value'` |
| `when = { "alias.COLUMN" = 42 }` | `alias.COLUMN = 42` |
| `when = { "alias.COLUMN" = ["a", "b"] }` | `alias.COLUMN IN ('a','b')` |
| `when_like = { "alias.COLUMN" = "R%" }` | `alias.COLUMN LIKE 'R%'` |
| `when_null = ["alias.COLUMN"]` | `alias.COLUMN IS NULL` |
| `when_not_null = ["alias.COLUMN"]` | `alias.COLUMN IS NOT NULL` |
| `when_not = { "alias.COLUMN" = ["a", "b"] }` | `(alias.COLUMN IS NULL OR alias.COLUMN NOT IN ('a','b'))` |
| `when_not_like = { "alias.COLUMN" = "R%" }` | `(alias.COLUMN IS NULL OR alias.COLUMN NOT LIKE 'R%')` |

Multiple keys within one branch combine with `AND`. **There is no author-visible `OR`** — express it
as multiple branches carrying the same `implies` value. This keeps every predicate a flat
conjunction and removes the need for precedence rules. The parenthesized `OR` inside the negation
forms is emitted by the compiler within a single key's expansion; it is not a disjunction you can
write.

#### Why negation is null-inclusive

`when_not` does **not** emit bare `NOT IN`, and this is the one place the grammar deliberately
departs from SQL semantics.

Under SQL's three-valued logic, `COLUMN NOT IN ('a','b')` is *unknown* — and therefore false — for
every row where `COLUMN IS NULL`. In an ordinary query that costs you some rows in one result set.
In a discriminated mapping it is worse: a row matching no branch is absent from the union entirely.
Not miscounted, not mislabeled — gone, with no error. That is the same failure class as a predicate
that is legal, plausible, and silently false for every row.

So `when_not` is defined as **set complement including nulls**, which is what the phrase "not one of
these" means to a person reading the file. It preserves the property the construct depends on:
branches partition the rows.

If you genuinely want strict SQL semantics — complement *excluding* nulls — compose it, since keys
combine with `AND`:

```toml
when_not = { "alias.COLUMN" = ["a", "b"] }
when_not_null = ["alias.COLUMN"]
```

That reduces to plain `NOT IN`, and it is now visible in the file that nulls were excluded on
purpose rather than by accident.

#### What stays excluded, and why

Two different reasons, which the earlier draft of this section wrongly collapsed into one phrase:

- **Ordering comparisons** (`<`, `>`, `<=`, `>=`, `BETWEEN`) are excluded for *portability*. They
  are almost always applied to dates, and date literal syntax is where dialects genuinely diverge.
  A range predicate is also usually a filter on the population rather than a definition of a branch,
  which makes it a `[[sources]].filter` concern.
- **String functions, date arithmetic, null coalescing** (`NVL` / `COALESCE` / `ISNULL`),
  **concatenation, and anything containing a parenthesis** are excluded because they encode a
  *decision* — a normalization, a fallback, a derivation — and decisions belong in a candidate block
  with a recorded rationale, not buried in a predicate where nobody reviews them.

Negation was in neither category. It is portable (`NOT IN` and `<>` are ANSI and identical across
Oracle, Databricks SQL, Postgres, and T-SQL) and it encodes no decision. It was excluded by an
imprecise word, and the null trap is a reason to *define it carefully*, not a reason to omit it.

> **Note on the existing `filter` key.** `[[sources]].filter` already accepts a raw SQL string. In
> practice every use of it in current mappings is within the grammar above. Bringing `filter` under
> the same restricted grammar is worth doing, but it is a separate migration — do not bundle it
> here.

### 1.7 Candidates inside a branch

Multi-candidate adjudication is unchanged and composes with branches. A branch may carry its own
`[[attribute_candidates.<attribute>.candidates]]` list, scoped by `branch_id`, so side-by-side
redundancy profiling keeps working *within* a branch:

```toml
[attribute_candidates.contact_value]
attribute = "contact_value"
candidate_resolution_mode = "discriminated"
discriminator = "contact_type"

[[attribute_candidates.contact_value.candidates]]
candidate_id = "email_primary"
branch_id = "email"
expression = "<EMAIL_ALIAS>.<EMAIL_COLUMN>"
status = "primary_recommended"
rationale = "<why this won within the email branch>"
```

`candidate_resolution_mode = "discriminated"` is the new fifth value, alongside `single_candidate`,
`multi_candidate`, and `not_mappable_yet`.

### 1.8 Full worked example

In the repo's generic vocabulary. A `customer_contact` domain over the `warehouse` source, where
contact payload is split across an email column, a phone table, and a messaging table.

```toml
system = "warehouse"

[defaults]
catalog = "EXAMPLE_CATALOG"
schema = "EXAMPLE_SCHEMA"

[grain]
mode = "discriminated"
discriminator = "contact_type"

[[sources]]
alias = "c"
table = "CUSTOMER_VERSION_HIST_FACT"
role = "base"
filter = "CURRENT_VERSION_FLAG = 'Y'"

[[sources]]
alias = "ci"
table = "CONTACT_INFO"
role = "contact_branch"

[[sources]]
alias = "ph"
table = "CONTACT_PHONE_FACT"
role = "phone_branch"

[[sources]]
alias = "pt"
table = "PHONE_TYPE_CODE"
role = "phone_type_lookup"

[[joins]]
left = "c.CUSTOMER_HISTORY_FACT_KEY"
right = "ci.CUSTOMER_HISTORY_FACT_KEY"
type = "left"

[[joins]]
left = "ci.CONTACT_INFO_KEY"
right = "ph.CONTACT_INFO_KEY"
type = "left"

[[joins]]
left = "ph.PHONE_TYPE_KEY"
right = "pt.PHONE_TYPE_KEY"
type = "left"

[identity]
business_key = ["customer_id"]

# Branch-invariant: projected into every leg.
[fields]
customer_id = "c.CUSTOMER_PUBLIC_ID"

# --- branches -------------------------------------------------

[[branches]]
branch_id = "email"
implies = { contact_type = "email" }
when_not_null = ["ci.EMAIL_ADDRESS"]

[branches.fields]
contact_value = "ci.EMAIL_ADDRESS"
preferred_flag = "ci.PREFERRED_EMAIL_FLAG"

[[branches]]
branch_id = "phone"
implies = { contact_type = "phone" }
when = { "pt.PHONE_TYPE_CODE" = ["HP", "WP", "CP"] }

[branches.fields]
contact_value = "ph.PHONE_NUMBER"
preferred_flag = "ph.PREFERRED_FLAG"

[[branches]]
branch_id = "fax"
implies = { contact_type = "fax" }
when = { "pt.PHONE_TYPE_CODE" = "FX" }

[branches.fields]
contact_value = "ph.PHONE_NUMBER"
preferred_flag = "ph.PREFERRED_FLAG"

[notes]
grain_warning = "This file is discriminated and produces one row per contact endpoint, not one row per customer. Counting rows here does not count customers."
discriminator_provenance = "contact_type is asserted by branch membership, not read from a column. The source's own communication-method column is present but populated for a small minority of rows and carries a single non-null value, so it is not usable as the discriminator. That is a governance finding, recorded here, not a mapping blocker."
mutual_exclusivity = "Phone and fax branches partition the phone type lookup between them. If a new type code is added upstream, rows carrying it fall into no branch and disappear silently. Re-check the lookup when the source version changes."
```

---

## 2. Harness and validator changes

Code changes you will need to make yourself. Listed as behavior plus acceptance criteria rather than
implementation, since this document cannot carry code.

### 2.1 Mapping loader

- Parse `[grain]`. Absent means `mode = "simple"`; preserve all current behavior for those files.
- Parse `[[branches]]` including the nested `[branches.fields]` tables.
- Compile a discriminated file to a `UNION ALL` over branch legs. Each leg projects: `[fields]`
  expressions, that branch's `[branches.fields]` expressions, and the branch's `implies` value as a
  literal aliased to the discriminator attribute.
- Derive each leg's join subset by walking from its expressions to the base alias. A leg must not
  join tables no expression in it references — otherwise the phone leg inherits the email join and
  the grain inflates.
- Optional, and safe to defer: when every branch uses only `when` against an identical alias set,
  emit a single `CASE`-based query instead of a union.

### 2.2 Predicate compiler

- Accept only the eight forms in §1.6. Reject anything else with a message naming the offending key.
- Reject any predicate value containing a parenthesis, and reject any key that is not a
  `alias.COLUMN` reference against a declared alias.
- Emit parameterized values where the driver supports it rather than interpolating literals.

### 2.3 Validator rules to add

- `discriminator` names an attribute present in the domain file. Fail otherwise.
- Every branch has at least one of `implies`, `when`, `when_like`, `when_null`, `when_not_null`.
- Every `implies` key equals the declared `discriminator`. A branch asserting a different attribute
  is a modeling error.
- `branch_id` values are unique within a file.
- No attribute appears in both `[fields]` and any `[branches.fields]`.
- Every expression resolves to a declared alias.
- Every attribute named across `[fields]` and all branches exists in the domain file.
- The discriminator is **not** present in `[fields]` or any `[branches.fields]`.
- Warn — do not fail — when branches carry only `implies` and no `when`, since mutual exclusivity
  cannot then be checked statically.
- Warn — do not fail — when a branch's predicates are **entirely negative** (`when_not` /
  `when_not_like` / `when_null` with no positive form). A negatively-defined branch is a catch-all:
  it absorbs any value added upstream after the mapping was written and labels it with whatever that
  branch asserts. Positive enumeration leaves an unknown value unmatched instead, which the coverage
  assertion can detect. Silence is the risk, so make the choice visible.

### 2.4 Consumers

- Anything that reads `[fields]` as the complete runtime surface needs to also read branches, or it
  will silently see a discriminated file as having almost no mapped attributes.
- Coverage and completeness metrics must use the branch-aware grain. A discriminated file's row
  count is endpoint-grain; member-grain questions ("has at least one contact of any type") are a
  rollup on the business key, not a row count.

### 2.5 Tests

- Existing domain-definition tests that assert on the contact domain files will fail once §4
  removes them. Update those before deleting, or the suite goes red mid-migration.
- Add: a discriminated fixture compiles to the expected number of legs; a branch does not inherit
  another branch's joins; each of the eight predicate forms compiles; each rejected form is rejected;
  the duplicate-attribute rule fires.
- Add specifically for negation: a row whose discriminator column is NULL is **retained** by
  `when_not`, and is **dropped** when `when_not` and `when_not_null` are combined. This is the
  behavior most likely to regress under a well-meaning refactor toward plain `NOT IN`, and it is
  invisible without a null row in the fixture.

---

## 3. Mapping README: the fifth decision state

`configs/mappings/README.md` currently documents four decision states — mapped, unresolved,
withheld, missing. Add a fifth.

> **discriminated** — the attribute is mapped, but to several physical fields rather than one,
> selected by a discriminator. This is a complete and final outcome, not a provisional one. Use it
> when a single canonical attribute is instantiated by different columns depending on a row's type,
> path, or mode, and no single field is the right answer for all rows. The attribute appears in
> `[branches.fields]` rather than `[fields]`, and the file declares `[grain]`.
>
> Reach for this before reaching for `unresolved`. An attribute with several correct answers is not
> the same as an attribute with no answer, and recording it as unresolved understates what is known
> and invites a redesign of the domain model that is not warranted.

The scaffold repo's generic examples already ship with this state documented — see
`resources/configs/mappings/customer_contact/warehouse.toml` for a worked discriminated mapping and
`resources/README.md` for the five-state table. Nothing to do there; they are listed so you can copy
the wording rather than invent it.

---

## 4. Domain file changes

These reverse the decomposition that the missing grammar forced. Do them **after** §1–2 work, so you
are never without a way to express contact payload.

### 4.1 Delete

| File | Reason |
| --- | --- |
| `configs/domains/member_contact_endpoint.toml` | Existed to hold `contact_endpoint_key`, which existed to join subtypes. With branches there is no cross-branch join. |
| `configs/domains/member_contact_email.toml` | Payload is a branch of `member_contact`, not a domain. |
| `configs/domains/member_contact_phone.toml` | Same. |
| `configs/domains/member_contact_fax.toml` | Same. Note the source stores fax as a type value inside the phone structure — the split was never source-shaped either. |
| `configs/domains/member_contact_address.toml` | See §4.3. |

`contact_endpoint_key` disappears with them. It was a synthetic integration key with no source
column, and integration keys are a materialization concern — not something a profiling stage needs,
since profiling generates a query per source rather than persisting a joined model. It also
violated the one-home rule in `configs/domains/README.md`, which grants a multi-file exception to
the aggregate root identifier only.

### 4.2 Keep and adjust `member_contact.toml`

It returns to being the single contact domain, and `contact_type` / `contact_value` work as
originally designed.

- **`contact_type`** — confirm the description says *communication mode* and that its allowed values
  are modes (email, phone, fax, text). It is now produced by branch membership.
- **Consider adding `contact_purpose`** — a genuinely separate business concept covering the
  correspondence-role values your source's contact-type lookup actually carries (billing,
  correspondence, return mail, and so on). These are well populated and are *not* modes. Conflating
  them with `contact_type` is what made the original mapping look impossible. This is a judgment
  call and a real modeling addition; treat it as optional and decide it on its own merits.
- **`business_key`** — currently includes `contact_value`, which puts payload in a key. It is a
  defensible natural key for an endpoint, but flag it for the governance review rather than leaving
  it unexamined.

### 4.3 Address: a reversal of an earlier decision

Earlier you decided to keep both `member_address.toml` and `member_contact_address.toml`, on the
grounds that it was confusing but that the redundancy was a useful governance signal. **This
migration recommends dropping `member_contact_address.toml`** — for a better reason than the one
that led you to keep it.

An address is compound (line, city, state, postal code), so it cannot be a branch of
`contact_value`, which is single-valued. That much stands. But the distinction between the two
address structures in your source is *purpose* — where the member is versus where mail goes — and
both structures already carry a type or classification code expressing exactly that. So the
discriminator you thought you had to invent as `address_context` already exists, is a real business
concept, and is called `address_type`.

That makes the two physical address structures two **candidates** for the same canonical attributes,
adjudicated on evidence and profiled side by side — the pattern you have said already works well.
No invented field, no second domain.

Two caveats before you commit to this:

1. Your survey flagged that the classification codes on the two branches differ in character — one
   looks like business classification, the other operational. Confirm they express the same concept
   before treating them as one attribute. If they do not, that is itself the finding.
2. Your survey also found geospatial columns on only one of the two branches. That is already
   handled by the `withheld_pending_coherence_validation` state and does not change here.

If either caveat resolves badly, keeping both domains stays defensible — but decide it on that
evidence, not on the absence of a grammar.

---

## 5. Skill text changes

Two edits to `.github/skills/map-attribute/SKILL.md`. Both are find-and-replace on exact existing
text.

### 5.1 In `### Changes to apply`

**Find:**

```markdown
- **A candidate expression is almost always a plain aliased field reference** — `table_alias.field_name`. If
  you are writing a `CASE`, a `COALESCE`, or a concatenation, stop: you are encoding a decision into
  the expression that belongs in the decision state, or merging fields the model wants kept apart.
```

**Replace with:**

```markdown
- **A candidate expression is always a plain aliased field reference** — `table_alias.field_name`.
  If you are writing a `CASE`, a `COALESCE`, or a concatenation, stop. Three different things get
  mistaken for each other here, and only one of them is an expression problem:
  - You are merging fields the model wants kept apart. Split them.
  - You are encoding a decision into the expression that belongs in the decision state. Record the
    decision instead.
  - **The attribute is discriminated** — several physical fields are each correct for a different
    subset of rows, selected by type, path, or mode. This is not a defect and not a `CASE`. Declare
    `[grain]` and `[[branches]]` per `configs/mappings/README.md`. Each branch expression stays a
    plain field reference; the branch structure carries the selection.
```

### 5.2 In `### Recommendation`

**Find:**

```markdown
**There is a third outcome besides mapped and not-found: the field exists and cannot be used.** A
```

**Replace with:**

```markdown
**Before concluding that no single field is right, check whether the attribute is discriminated.**
Several fields each being correct for a different subset of rows is not the same as no field being
correct. Signals: the candidates are each highly populated but on disjoint row sets; they sit on
different join paths from the anchor; the attribute's name is a generic container (`value`, `id`,
`amount`) and the source stores a type alongside it. When that is what you are looking at, the
answer is a discriminated mapping and a complete recommendation — not `unresolved`, and never a
proposal to restructure the domain model. **The domain model is not yours to redesign in a mapping
turn.** If you believe it is genuinely wrong, say so under *What I need from you* and stop.

**There is also an outcome besides mapped and not-found: the field exists and cannot be used.** A
```

---

## 6. Working note

Add to `docs/sources/<SOURCE>/working-notes.md`:

```markdown
- When an attribute has no single best field, check whether it is discriminated — several fields
  each correct for a different subset of rows — before concluding `unresolved`. Discriminated
  attributes are mapped with `[grain]` and `[[branches]]`, not with a `CASE` and not by changing
  the domain model.
```

One line, because the operator's-manual rule holds: every steering message you send is a symptom of
a missing working note, and this one cost six exchanges the first time.

---

## 7. Order of operations

The sequence matters in two places — §2.5 before §4, and §1–2 before §4 — so that the suite never
goes red and you are never without a way to express contact payload.

1. **§3** — mapping README gains the fifth decision state. Documentation only, no risk.
2. **§1** — write the grammar into the mapping README as the normative spec.
3. **§2.1–2.3** — loader, predicate compiler, validator. Land with the fixtures from §2.5.
4. **§2.4** — consumers. This is where a silent failure hides: a reader that only knows `[fields]`
   sees a discriminated file as nearly empty rather than erroring.
5. **§5, §6** — skill text and working note. Independent of the code; can be done any time.
6. **§2.5** — update the domain-definition tests that assert on files about to be deleted.
7. **§4.1–4.2** — delete the five contact domain files, adjust `member_contact.toml`.
8. **Re-map the contact attribute** in a fresh session, using the new grammar. One attribute, one
   session, four-exchange tripwire.
9. **§4.3** — address consolidation, only after the two caveats are checked. This one is a reversal
   of a previous decision and should be made deliberately, not as migration momentum.

---

## What this does not solve

- **`[[sources]].filter` is still a raw SQL string.** It is in practice already within the
  restricted grammar, but it is not enforced. Separate migration.
- **Mutual exclusivity of `implies`-only branches cannot be validated statically.** If two branches
  overlap, rows are double-counted; if they under-cover, rows vanish silently. The `[notes]` entry in
  §1.8 is the mitigation, and it is a weak one. Worth a coverage assertion in the harness later:
  branch leg row counts should sum to the unioned base row count.
- **There is no `define-domain` skill.** This migration exists because a domain-modeling problem was
  solved as a mapping problem, badly, across six exchanges with no scaffold. The working note in §6
  is the cheap mitigation. The gap remains.
