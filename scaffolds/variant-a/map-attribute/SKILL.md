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

What follows is a worked example. Work the way it works.

---

## Worked example: mapping `region` in a source with a code/name split

### Finding the candidates

Nothing documents this source. The search that produced the candidate list was:

```text
searched object names for the domain term and its plausible abbreviations
  → two address tables carrying a REGION_CODE column
checked what those two tables join to, and what joins to them
  → one shared parent, no other path to an address
```

Two candidates, from one method. Before accepting that as the candidate list, the question
worth asking was: **what would a search like that miss?** A name search finds fields whose
names announce themselves. It cannot find a field whose name is generic, whose meaning lives
in its values, or whose relevance only appears once you look inside something. Whether this
source contains any of those is an empirical question, and it had not been asked yet.

Hold that thought — the first pass did not ask it.

### The shallow pass — this is what going wrong looks like

The attribute `region` exists in the domain model. Two address tables ("primary" = `ap` and "contact" = `cp`) carry a `REGION_CODE` column. A tiny dual-path probe returns:

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

## What to produce

In your reply, in this order:

1. **Where you looked.** The objects you actually interrogated and what each turned up,
   including the searches that returned nothing. A candidate list with no search behind it is
   a guess wearing a citation.
2. **Candidates**, each with the search that produced it — and, for the list as a whole, what
   a search of that kind would be structurally unable to find.
3. **What you did not check**, and why it might matter.
4. **Conflicts** — between two stated preferences, or between the domain model and what the
   source actually offers. Say "none" only after looking.
5. **Evidence** — the probe, its scope, and the counts. State what result would have changed
   your conclusion. If a probe returns zero population on every candidate, that is not evidence
   of absence; escalate scope or run a targeted count before concluding anything.
6. **Recommendation** — the primary, the alternates, and the reason each alternate lost.
7. **Proposed TOML edit**, conforming to `configs/mappings/README.md`.

Items 1–5 come before item 6. A concern written after a conclusion tends to justify it.

Then stop and wait. Do not begin the next attribute.
