# Operator's manual

How to run a mapping session so the agent stays useful. This is the human-facing half of the
scaffold, and on a long domain it matters more than the skill text does.

It exists because of one measured fact: **the agent degrades with turn count, not with difficulty.**
In evaluation, three attributes in a row went well at two to three exchanges each — format followed,
questions asked, survey used correctly. The fourth attribute took five exchanges and performance
collapsed inside that single turn: response format abandoned, sample constraints forgotten, scratch
files written into the repo, and the original question replaced by whatever the last message was
about. Both variants, same turn, same way.

You cannot prompt your way out of that. You engineer around it.

## The core loop

1. **One attribute, one fresh session.** Not one session for the domain.
2. Run the attribute. Two or three exchanges is normal and healthy.
3. **At the fourth exchange, stop and reset** — see the tripwire below.
4. Before moving on, make sure anything you had to correct is written down somewhere durable.

## The tripwire

**Four exchanges on one attribute is the signal to reset, not to push harder.**

Pushing on is the tempting move, because by exchange four you have usually built up real context and
throwing it away feels wasteful. The evaluation says otherwise: the fifth and sixth exchanges
produced the worst output of the entire run, including one case where the agent obtained a
near-perfect result for the correct field and then failed to recommend it.

To reset:

1. Have the agent write what it has established into the survey and propose a working note.
2. Clear the context.
3. Reload (below) and restart the attribute.

The skill also asks the agent to propose this itself. Do not wait for it to.

## Working notes

`docs/sources/<SOURCE>/working-notes.md` is the single most valuable file in this setup.

Every time you correct the agent's *working method* — not its answer, its method — that correction
currently lives only in chat, and the next clear destroys it. In evaluation this happened
repeatedly: the same correction about sample construction had to be given to each arm, and both
arms lost it at the next clear.

So the rule is:

> **Every steering message you send is a symptom of a missing working note.**

When you catch yourself typing a correction, ask whether you will need to type it again next
attribute. If yes, it is a note. The skill reads this file at the start of every attribute, before
the survey.

### What belongs in it

- how to construct a working subset from the sample
- standing scope permissions and standing scope refusals
- conventions the agent keeps getting wrong
- "when you propose X, also apply X"
- facts about the world you have already supplied once and would rather not supply again

### What does not

- findings about the source — those belong in the survey
- decisions about a specific attribute — those belong in the mapping TOML
- anything you would not want applied to *every* attribute

Keep it short. It competes for attention with everything else, and a note file that grows past a
page has become documentation rather than instruction.

## Reload order after a clear

1. `docs/sources/<SOURCE>/working-notes.md`
2. The relevant slice of `docs/sources/<SOURCE>/<domain>-survey.md` — only the slice, not the whole
   survey
3. `configs/mappings/<domain>/<source>.toml` as it stands
4. The attribute

If the survey is a single large file rather than an index over slices, split it. The slice structure
is what makes a reset cheap, and a reset being cheap is what makes the tripwire usable.

## Sequencing

**Order attributes easy to hard**, and treat a known-hard attribute as its own session from the
start rather than as the fourth item in a run. In evaluation, the hardest attribute arrived fourth
and inherited a session that was already long; it would have had a better chance cold.

Signals that an attribute is hard, decided before you start:

- the survey's leads for it are vague, or its *Open questions* mention it
- it needs a fact from outside the database
- it applies to only part of the population
- more than one candidate is plausible on name alone

Any of those, give it a fresh session.

## Steering

**Prefer front-loading over correcting.** A constraint stated in the working notes before the turn
costs one line; the same constraint delivered as a correction costs an exchange, and exchanges are
the scarce resource.

When you do correct mid-turn, be aware you are spending from the same budget the attribute needs to
converge. Two corrections plus the original prompt is already the tripwire.

**Answer questions the agent asks.** Getting it to ask was hard-won and it is fragile; a question
met with "figure it out" trains the behavior away. Answer briefly, answer only what was asked, and
add the answer to the working notes if it will recur.

## When the agent stops querying

Watch for a recommendation that cites only the survey. A good survey is a liability here: the better
it reads, the more the agent treats it as an answer key rather than a lead sheet, and the more
confidently it will reason without touching the database. The skill now requires a probe per turn,
but check it. A mapping turn with zero queries is not a mapping turn.

## What to record

For each attribute, whatever else you keep, keep these three: the number of exchanges, the
corrections you had to make, and whether it ended in a recommendation you trust. Exchanges are your
leading indicator — when the average creeps up, something upstream has gone stale, and it is usually
the survey or the working notes rather than the skill.
