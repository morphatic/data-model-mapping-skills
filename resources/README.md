# Resources

Generic, domain-agnostic examples of the two config files the skills in `scaffolds/` expect a
consuming project to have. Nothing here is tied to any particular industry or organization —
the `customer` domain and `warehouse` source are invented, and every table, column, and figure
is fabricated.

```text
configs/domains/customer.toml              a domain model
configs/mappings/customer/warehouse.toml   that domain mapped onto one source
eval-rubric-template.md                    scoring instrument for comparing skill variants
```

Copy them into a project as `configs/domains/<domain>.toml` and
`configs/mappings/<domain>/<source>.toml`, then replace the contents.

## What the examples demonstrate

**The domain model is independent of any source.** Attributes exist because the business needs
them, not because a source happens to carry them. An attribute no source can fill is a finding
worth recording, not an error to delete.

**A source is a table plus an optional filter.** One physical table appears under several
aliases when it stores different things per row type. That is why `[[sources]]` binds
`(table, filter) → alias` rather than listing tables.

**The join graph is a tree rooted at the base alias.** Every non-base alias reaches the base by
exactly one path, so the joins needed for any expression can be derived by walking to the root.
Worth preserving as an invariant — it is what keeps a growing mapping file from accumulating
join paths nobody can prune.

**The business identifier is usually not the join key.** In the example, identity is
`CUSTOMER_PUBLIC_ID` while joins run on `CUSTOMER_HISTORY_FACT_KEY`. Conflating them is a
common and expensive mistake, and a column bearing the domain's name may be neither.

**Grain is a claim the filter has to earn.** The base table holds one row per version; the
declared one-row-per-customer grain is true only because of the current-version filter. Remove
it and every count silently multiplies.

**Four decision states, deliberately distinct:**

| State | Shown by | In `[fields]`? |
| --- | --- | --- |
| mapped | `region_code`, `region_name`, names, `birth_date` | yes |
| unresolved | `external_id` — provisional, semantics unconfirmed | yes, flagged |
| withheld | `geo_latitude` — candidate exists, coherence unverified | no |
| missing in source | `loyalty_tier` — nothing found, and how that was checked | no |

Withheld is not missing. A withheld attribute has a real, populated candidate that is being
deliberately held back for a governance reason, with a named condition for revisiting. Losing
that distinction turns a decision into an absence.

**A standardized code and its readable name are two attributes, not two candidates.** When a
code turns out to be an external standard, it carries cross-source comparability the name does
not, and the name carries meaning the code does not. Modeling both is usually right; picking one
silently is usually wrong.

**Evidence lives next to the decision it supports.** Probe counters are aggregate only, and the
`inference` line states what the numbers support — often less than they appear to. Uniform
format does not establish semantics; agreement between two fields establishes only that they
agree.

## Note

The skills reference a `configs/mappings/README.md` that defines the allowed values for
`candidate_resolution_mode`, `selection_status`, candidate `status`, `sample_scope`, and the
decision-state notation. That conventions file is not included here. Anyone adopting these
skills will need to write one, or the vocabulary used above has no authority behind it.
