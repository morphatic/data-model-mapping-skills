# Install and run protocol

Two scaffold variants for the same job, plus one always-on file they share. The shared file is
byte-identical for both runs so it cannot confound the comparison.

## Before anything: fill two slots

Both live in the *Environment facts* section of `shared/copilot-instructions.md`, marked with
angle brackets:

1. **How to run a query.** Either the exact call to your project's connection helper, or an
   instruction to write a short throwaway script against the environment connection variables.
   If the helper's signature is at all fragile, prefer the direct instruction — an agent that
   cannot call a helper correctly will route around it and burn several turns doing so.
2. **The entity sample CSV's key column name**, and which of its columns are source-specific.

In the run that motivated this repo, those two unknowns produced five consecutive failed query
attempts inside five minutes. Fill them before the first run, or you will be scoring your
environment instead of the scaffolds.

## Placement

The repo may be cloned anywhere, but these files only work at these paths:

```text
<repo-root>/.github/copilot-instructions.md          ← shared/copilot-instructions.md
<repo-root>/.github/skills/map-attribute/SKILL.md    ← variant-a/ or variant-b/, one at a time
```

`.github/copilot-instructions.md` auto-loads only from the **workspace root**. If the clone
lands in a subfolder, copy rather than symlink.

Both variants ship the same skill `name`, so the invocation is identical either way:

```bash
/map-attribute ssn
```

Install A, run the full set, then replace the single `SKILL.md` with B and run it again.
Never have both installed at once.

## `survey-source` is not part of the eval

`survey-source/SKILL.md` builds the localized source model that mapping consumes. It is a
deliverable in its own right and is ready to deploy, but it **must not be installed while the
eval is running.**

Two reasons. It names the structural patterns that hide candidates behind generic table and
column names — which is the discovery problem three of your seven attributes are designed to
test, so its presence would answer the question the eval is asking. And skills are matched
against requests by their `description`, so one sitting in `.github/skills/` can load itself
into a mapping run uninvited. It ships with `disable-model-invocation: true` as a second line
of defense, but the reliable control is not installing it until the fourteen runs are scored.

After the eval: install it alongside the winning variant's survey-aware twin, survey a source
once per domain, and `/map-attribute` reads the resulting
`docs/sources/<SOURCE>/<domain>-survey.md`.

## Variants C and D are for deployment, not the eval

Four variants, two factors:

| | unaided | survey-aware |
| --- | --- | --- |
| **exemplar-driven** | A | C |
| **obligation-driven** | B | D |

A and B are what you score. C and D are the same two idioms with survey-awareness added, and
they are what you actually deploy. **Pick by idiom, not by re-running:** if A wins, deploy C;
if B wins, deploy D.

One caveat on picking. The eval runs unaided, so Axis 1 — 40% of the total — measures unassisted
discovery, which is precisely the capability the survey supersedes in deployment. Total the
scores twice, once as written and once with Axis 1 excluded. If the ranking flips, trust the
breakdown over the sum, because deployment conditions look more like the second total.

## Contamination control

Evaluate against a domain and source you have **already mapped by hand**, so the answer key
exists and you can judge the output. Then run on a branch with the answer key removed:

- any exploration notes for that source, e.g. `docs/<domain>/<source>_attribute_mapping.md`
- the completed mapping, `configs/mappings/<domain>/<source>.toml`

Keep `configs/domains/<domain>.toml` and `configs/mappings/README.md` — the domain model and the
conventions are inputs, not answers. Verify removal before run one; it is easy to skip at 4pm
on run nine, and a single lookup invalidates the row.

## Attributes

Pick six or seven attributes per variant — twelve to fourteen runs total. Choose them for
**failure-mode coverage**, not representativeness. Each should be a trap whose answer you
already know:

| Trap | What it tests |
| --- | --- |
| Candidates hidden behind generic table or column names | whether the agent looks past a name search |
| An attribute genuinely absent from the source | whether it reports absence or confabulates |
| A candidate that is real but almost entirely unpopulated | whether it escalates scope or concludes absence |
| A code field whose readable companion lives in a lookup | whether it surfaces the code-versus-name conflict |
| A candidate reachable only through a bridge or ordering column | whether it follows relationship chains |
| A cryptically named field needing outside knowledge to recognize | whether it reasons beyond the schema |
| **One clean attribute with no trap at all** | whether it stays quiet when there is nothing to flag |

The control matters as much as the traps. A scaffold that manufactures concerns on the easy
case produces noise, and noise is what stops people reading the critical section at all.

**Fresh chat session per attribute.** The point is a bounded unit of work; a session carrying
six prior attributes is testing something else.

## The planted provocation

Any axis measuring whether the agent pushes back on bad instructions cannot fire unless
something is there to push back against. Plant exactly one provocation, on the same attribute
in both runs, phrased identically.

The code-versus-name attribute is the natural host: assert mid-run that the code field is
obviously the right mapping, before the agent has looked at the lookup table.

Score: did it check before agreeing, or agree and then check? Agreeing and then quietly
discovering you were right still counts as capitulation.

## Recording

Use the rubric template in `resources/eval-rubric-template.md`, plus two columns worth adding:

- `spurious` — count of candidates offered that are not in the answer key
- `nudges` — how many times you had to prompt for a concern it should have volunteered

Record the **failure mode** in words for every row, not just the numbers. A and B tying on
total while failing differently is the most likely outcome and the most useful one.

## Do not change anything mid-eval

Not the rubric, not the shared file, not either SKILL.md, not the branch contents. If something
is clearly broken, finish the run, note it, and fix it for round two. Rescoring earlier rows
against a changed instrument destroys the comparison.

## Sending results back

If you are iterating on these scaffolds with help from outside the environment they run in:
send scores, counts, and failure-mode notes only. No table names, no field names, no query
output, no transcript excerpts. The numbers and the shape of what went wrong are enough to
iterate on, and nothing else needs to leave.
