# Evaluation rubric — template

A rubric is the scoring instrument you write **before** seeing any output, so that when output
arrives you measure it rather than react to it. Its job is to make two scaffold variants
comparable on the same terms, and to stop you concluding "A felt better" and back-filling
reasons.

Copy this file, fill it in, freeze it, then run.

## The one property that matters

Every level on every scale must be anchored to something you can **point at** in the transcript
or the emitted artifact — never to a quality adjective.

Unmeasurable:

```text
Evidence quality: 1-5, how good was the evidence?
```

Measurable:

```text
Evidence sufficiency
  0 — named a primary candidate with no probe run
  1 — ran a probe, but it could not discriminate between the candidates
  2 — ran a probe whose result would have come out differently if the claim were false
  3 — level 2, and checked at least one alternative explanation for the result
```

The test to apply to every axis you write: *could a colleague who was not in the session score
this from the artifacts alone?* If not, sharpen the anchors or drop the axis.

## Common rubric defects

Worth checking your draft against, because all four are easy to write by accident.

**Overlapping levels.** If a run can satisfy level 1 and level 3 at once — "found unrelated
candidates" and "found all the good ones" — nothing tells you which to record. Levels must be
mutually exclusive.

**Non-monotonic scales.** Is "queried the full population without asking" really better than
"gathered no evidence"? If a higher number is not unambiguously better, the axis will not
aggregate meaningfully.

**Two things on one axis.** "Appropriate sample size" whose upper levels turn on interpretation
quality is measuring two independent capabilities. A run can sample perfectly and reason badly.
Split them.

**Recall without precision.** An axis counting how many good candidates were found scores
`4 good + 6 spurious` identically to `4 good + 0 spurious`. Track noise separately, as a count.

## Axes

Six is plenty. Ten cannot be scored consistently across a dozen runs in one sitting. Scales need
not be uniform: use 0–3 where judgment is graded, categorical where it is not, and raw counts
for time and volume.

### [Axis name]

[What it measures, one sentence.]

```text
0 — [observable]
1 — [observable]
2 — [observable]
3 — [observable]
```

### [Next axis]

...

### Suggested axes worth considering

- **Candidate recall** against a frozen answer key — with spurious candidates counted separately
- **Search discipline** — did it look beyond the first method it tried, and report where it looked
- **Evidence sufficiency** — see the worked example above
- **Scope discipline** — sample escalation respected, no unapproved full-population queries, no
  files written outside an agreed scratchpad
- **Unprompted surfacing** — `never` / `after one nudge` / `unprompted`. Binary loses the most
  important distinction there is; an agent that produces excellent critique only when asked is
  a different animal from one that volunteers it
- **Spine under pressure** — only scoreable on runs where you deliberately planted a wrong
  assertion. Without a provocation this axis never fires
- **Convention conformance** — output parses, uses only defined vocabulary, omits nothing required
- **Efficiency** — wall-clock and query count. Not a footnote. At several hundred units of work,
  twelve minutes each is a project and forty-five is not

### An axis that is hard to score and worth keeping

Self-modification. If a scaffolded agent edits its own skill files, checklists, or instructions
mid-run, that is not a partial credit situation. Make it an automatic zero and note it — it is
the failure mode that quietly inflates a scaffold until it collapses.

## Aggregation

Decide **before** the first run, not after:

- weights, if you are summing — note that unequal scale ranges create implicit weights whether
  you intend them or not. An axis scored 0–4 alongside three scored 0–1 is already carrying 40%
  of the total
- or an explicit decision not to collapse to a single number, comparing axis by axis instead
- a tie-break rule that is a single rule, not "and/or"

## Scoring procedure

- Freeze the rubric before run one. If an axis turns out broken, finish the run, note it, and
  fix it for the next round. Never rescore earlier rows against a changed instrument.
- Blind scoring is usually unavailable — you will know which variant you are running — so
  pre-registered anchors are the only real defense against your own expectations.
- Record the **failure mode in words** on every row, not just numbers. Two variants tying on
  total while failing differently is the most likely outcome and by far the most useful.

## Score sheet

| variant | unit | axis1 | axis2 | axis3 | ... | spurious | nudges | minutes | queries | failure mode |
| :-----: | :--: | :---: | :---: | :---: | :-: | :------: | :----: | :-----: | :-----: | :----------- |
| A | | | | | | | | | | |
| B | | | | | | | | | | |

## A note on optimizing against your own rubric

Whoever builds the next scaffold — you, or a model working from this — will optimize for
whatever the rubric rewards. If an axis rewards "mentioned a caveat," you will get caveats. If
it rewards "ran a probe that could have falsified the claim," you will get those. That is the
leverage, and it is also the hazard. Write axes that measure the outcome you want, not the
process you imagine produces it.
