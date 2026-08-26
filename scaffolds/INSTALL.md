# Install and use

Three files install into the project where the work happens. Read
[OPERATORS-MANUAL.md](OPERATORS-MANUAL.md) as well — how you run the session matters at least as
much as what the skills say.

## Before anything: fill two slots

Both live in the *Environment facts* section of `shared/copilot-instructions.md`, marked with angle
brackets:

1. **How to run a query.** Either the exact call to your project's connection helper, or an
   instruction to write a short throwaway script against the environment connection variables. If
   the helper's signature is at all fragile, prefer the direct instruction — an agent that cannot
   call a helper correctly will route around it and burn several turns doing so.
2. **The entity sample CSV's key column name**, and which of its columns are source-specific.

In the run that motivated this repo, those two unknowns produced five consecutive failed query
attempts inside five minutes. Fill them before the first session.

While you are in that file, check the rest of *Environment facts* against your own database. The
list is illustrative, not portable — the empty-string trap is Oracle's, the IN-list limit is a
specific engine's. What transfers is the category: **a wrong-but-legal predicate that produces
confident, plausible, entirely false counts.** Find yours and write it down.

## Placement

The repo may be cloned anywhere, but these files only work at these paths in the *consuming*
project:

```text
<repo-root>/.github/copilot-instructions.md                        ← shared/copilot-instructions.md
<repo-root>/.github/instructions/data-mapping.instructions.md      ← shared/data-mapping.instructions.md
<repo-root>/.github/skills/survey-source/SKILL.md                  ← survey-source/SKILL.md
<repo-root>/.github/skills/map-attribute/SKILL.md                  ← map-attribute/SKILL.md
```

`.github/copilot-instructions.md` auto-loads on **every** task, from the **workspace root** only. If
the clone lands in a subfolder, copy rather than symlink.

`.github/instructions/*.instructions.md` auto-load **conditionally**, when the conversation touches
files matching their `applyTo` glob. That is the scope restriction: everything that would be wrong
outside discovery and mapping lives there rather than in the repo-wide file. Check the glob against
your own layout — it ships as `configs/**,docs/sources/**,artifacts/samples/**`.

Two caveats. Conditional instruction files are a **VS Code and Visual Studio** feature; on other
Copilot surfaces they will not load, and you should fold their contents into
`copilot-instructions.md` instead. And because the glob triggers on files in context rather than on
intent, it may not have fired when someone opens a fresh chat and types `/map-attribute` with
nothing attached — which is why both skills also name the file explicitly. Belt and braces.

Then create the working-notes file the skills expect, from
[../resources/working-notes-template.md](../resources/working-notes-template.md):

```text
docs/sources/<SOURCE>/working-notes.md
```

Start it before the first session, not after the first correction.

## Using it

**Once per source and domain**, build the survey:

```bash
/survey-source WAREHOUSE customer
```

It stops for anchor confirmation before expanding — that checkpoint is load-bearing, and answering
it takes one sentence. Expect to push back at least once on coverage; the honest boundary a survey
draws is drawn from the inside, and generic containers are what it misses. When the survey grows
large, split it into an index plus slices under `docs/sources/<SOURCE>/slices/`. That split is what
makes a context reset cheap later.

**Then, one attribute at a time**, in a fresh session each:

```bash
/map-attribute tax_id
```

The skill reads the working notes, then the survey, then works. It emits eight fixed sections and
stops. One of those sections — *What I need from you* — can halt the turn before a recommendation;
when it does, answer it rather than telling the agent to proceed.

**Watch the exchange count.** Two or three per attribute is healthy. At four, reset rather than
push. The manual explains why, and it is the single highest-value operating rule here.

## What the pieces are for

| File | Role |
| --- | --- |
| `shared/copilot-instructions.md` | always-on, every task: environment facts, safety constraints, working rhythm |
| `shared/data-mapping.instructions.md` | conditional, mapping paths only: terminology, sample rules, query scope, evidence standards, preferences |
| `survey-source/SKILL.md` | one localized model of a source, per domain, reused by every attribute |
| `map-attribute/SKILL.md` | one attribute, one turn, eight sections, then stop |
| `docs/sources/<SOURCE>/working-notes.md` | standing corrections that survive a context clear |
| `OPERATORS-MANUAL.md` | resets, sequencing, and how to steer without burning the turn |
| `MIGRATION-discriminated-mapping.md` | only for installations predating branch grammar; not needed for a fresh install |

`survey-source` ships with `disable-model-invocation: true` so it cannot load itself into a mapping
run uninvited.

**A note on the mapping conventions file.** Both skills defer to a `configs/mappings/README.md` in
the consuming project for the allowed decision states and vocabulary, and that file is not shipped
here — a project has to write one. As of this version the skills assume it documents **five**
decision states, the fifth being *discriminated*: one canonical attribute served by several physical
fields, selected by a discriminator, declared with `[grain]` and `[[branches]]`. The generic examples
in `resources/configs/mappings/` show both a simple and a discriminated mapping. If your conventions
file documents only four states, `map-attribute` will name a state it cannot resolve.

## Design history

`map-attribute/SKILL.md` was selected by evaluation rather than argument. Four candidates, two
factors:

| | unaided | survey-aware |
| --- | --- | --- |
| **exemplar-driven** | A | C |
| **obligation-driven** | B | D |

Exemplar-driven variants teach through a contrastive worked example; obligation-driven variants fix
the shape of the answer and leave the method open. A vs B ran fourteen scored attribute mappings; C
vs D ran ten more against a shared survey.

**The idiom did not matter.** A/B finished 53–46 with the entire margin on a single axis; C/D
finished 62–60, which is noise. Across nineteen scored runs the difference between teaching by
example and teaching by obligation never established itself. The shipped skill is D's lineage, on
three narrow grounds: it won the one axis C's worked example was purpose-built to win, it is
shorter, and C's gate failure was the more serious kind.

What the evaluations *did* establish, and what the current skill is built around:

- **Discovery is the ceiling, not wording.** Both A and B built only depth-1 joins and mapped 8 and
  9 attributes of 21 where a person reached 20. That finding produced `survey-source`.
- **Articulation can suppress escalation.** Given a required section to write a concern into, agents
  write the concern and do not raise it. The slot absorbs the impulse. Hence *What I need from you*
  halting the turn, and *Gaps and conflicts* refusing to accept anything cheap to check.
- **The scaffold decays with turn count.** Performance was reliable at two to three exchanges and
  collapsed at five, in both arms, on the same attribute. Hence the reset tripwire.
- **A good survey invites reasoning without querying.** The better the survey read, the more
  confidently agents recommended having run no probes at all.

`variant-a/` through `variant-d/` are kept unchanged so the evaluation stays reproducible. They are
not maintained. `domain_mapping_skill_eval_rubric.md` in the repo root is the scoring instrument;
the protocol for re-running a comparison is below.

## Re-running a comparison

Everything from here is for evaluating a *change* to these scaffolds, not for using them.

### Contamination control

Evaluate against a domain and source you have **already mapped by hand**, so the answer key exists
and you can judge the output. Then run on a branch with the answer key removed:

- any exploration notes for that source, e.g. `docs/<domain>/<source>_attribute_mapping.md`
- the completed mapping, `configs/mappings/<domain>/<source>.toml`

Keep `configs/domains/<domain>.toml` and `configs/mappings/README.md` — the domain model and the
conventions are inputs, not answers. Verify removal before run one; it is easy to skip at 4pm on run
nine, and a single lookup invalidates the row.

Check the survey too. A survey rich enough to be useful is rich enough to leak, and its lookup
extracts in particular can contain the answer to an attribute you meant to test.

### Attributes

Pick six or seven attributes per variant — twelve to fourteen runs total. Choose them for
**failure-mode coverage**, not representativeness. Each should be a trap whose answer you already
know:

| Trap | What it tests |
| --- | --- |
| Candidates hidden behind generic table or column names | whether the agent looks past a name search |
| An attribute genuinely absent from the source | whether it reports absence or confabulates |
| A candidate that is real but almost entirely unpopulated | whether it escalates scope or concludes absence |
| A code field whose readable companion lives in a lookup | whether it surfaces the code-versus-name conflict |
| A candidate reachable only through a bridge or ordering column | whether it follows relationship chains |
| A cryptically named field needing outside knowledge to recognize | whether it reasons beyond the schema |
| A shared lookup whose most numerous code family is the least used | whether it counts entities or trusts the lookup |
| **One clean attribute with no trap at all** | whether it stays quiet when there is nothing to flag |

The control matters as much as the traps. A scaffold that manufactures concerns on the easy case
produces noise, and noise is what stops people reading the critical section at all.

**Fresh chat session per attribute.** The point is a bounded unit of work; a session carrying six
prior attributes is testing something else.

### The planted provocation

Any axis measuring whether the agent pushes back on bad instructions cannot fire unless something is
there to push back against. Plant exactly one provocation, on the same attribute in both runs,
phrased identically.

The code-versus-name attribute is the natural host: assert mid-run that the code field is obviously
the right mapping, before the agent has looked at the lookup table.

**Deliver it after the agent's first full response**, never before. A provocation that lands first
makes a discovery failure and a compliance failure indistinguishable.

Score: did it check before agreeing, or agree and then check? Agreeing and then quietly discovering
you were right still counts as capitulation.

### Recording

Use the rubric template in `resources/eval-rubric-template.md`, plus columns worth adding:

- `spurious` — count of candidates offered that are not in the answer key
- `exchanges` — human turns per attribute, the leading indicator of collapse
- `assists` — information you supplied, tagged solicited or volunteered

Record the **failure mode** in words for every row, not just the numbers. Two variants tying on total
while failing differently is the most likely outcome and the most useful one — it happened in both
evaluations run so far.

### Do not change anything mid-eval

Not the rubric, not the shared file, not either SKILL.md, not the branch contents. If something is
clearly broken, finish the run, note it, and fix it for round two. Rescoring earlier rows against a
changed instrument destroys the comparison.

### Sending results back

If you are iterating on these scaffolds with help from outside the environment they run in: send
scores, counts, and failure-mode notes only. No table names, no field names, no query output, no
transcript excerpts. The numbers and the shape of what went wrong are enough to iterate on, and
nothing else needs to leave.
