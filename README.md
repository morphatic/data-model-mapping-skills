# Data Model Mapping Skills

This repo contains:

* A `survey-source` skill directing an agent to create a map of tables and fields in a new or not-well-known data source relevant to a specific, abstract data domain
* 4 candidate implementations of a `map-attribute` skill designed to then identify candidate fields that map to the attributes of the data domain, and do a shallow evaluation of them to determine the field most likely to be authoritative

It was designed primarily for GitHub Copilot in mind, but should be adaptable for other agents.

## Assumptions

These skills were developed in the context of an ongoing project and reflect dependencies on elements that may not exist in your project. The assumptions include:

* Some mechanism for enabling the agent to make direct queries against a data source
* A `configs/domains/<domain>.toml` file for each data domain to be mapped
* The `map-attribute` skill will create or update a `configs/mappings/<domain>/<source>.toml` file
* The `<domain>.toml` domain definition and `<source>.toml` mapping files have defined structure, syntax, and semantics as defined respectively in `configs/domains/README.md` and `configs/mappings/README.md`
* A domain sample file at `artifacts/samples/<domain>/<source>_<domain>_sample_<YYYY-MM-DD>.csv`

### About the domain sample

The skills assume a small, pre-built CSV of entity keys — a stratified sample of the domain
entity drawn across whatever dimensions matter for coverage, such as product, segment, or
status. Probes target the specific keys in that file rather than scanning the source, which is
what makes exploratory queries survivable against tables holding millions of rows.

Two things the skills care about: the file's key column, which is frequently *not* the column
whose name suggests it, and which of its other columns are source-specific rather than portable
across sources. Both are recorded in `scaffolds/shared/copilot-instructions.md`.

**Generating that sample is outside the scope of this repo.** Stratification design, sizing, and
refresh cadence are properties of your data and your domain, not of these skills.

## Examples

`resources/` holds generic, domain-agnostic versions of the config files the skills expect,
plus a template for the evaluation rubric used to compare skill variants against each other.

Use at your own risk!

## Design Philosophy

These skills exist in four variants because the first two attempts at building one failed
badly, and the reason turned out to be more interesting than the fix.

The workflow originally ran with no skill at all — just a carefully written opening prompt, a
human steering attribute by attribute, and an agent that wrote and ran its own queries. It went
well. The obvious next step was to have the agent read that session's transcript, extract the
steps, and write a skill so the process could be repeated. The result performed far worse than
the unscaffolded session. Adding more determinism — scripts, checklists, stage gates — made it
worse again.

The diagnosis: **a transcript is not a procedure, and stage gates define "done" as
steps-executed.** Open-ended judgment cannot satisfy a checkbox, so it loses to the thing that
can. Every guardrail added another way for the agent to feel finished without having thought.

The clearest evidence came from a single exchange. The agent completed an attribute — probe run,
files updated, validation passed — and reported it done, raising no concerns. Asked *"did you
not have a critical gap to point out?"*, it immediately produced four substantive and correct
critiques. The capability had been there the whole time. Only the trigger was missing, and the
scaffold had removed it.

So the design principle throughout is to **constrain the artifact rather than the process**:

* Say what must be true of the answer; leave the method to the agent, which is genuinely good at choosing one
* Require output slots that a process which did not think cannot fill — where you looked, what you did not check, what would make this wrong
* Put those slots *before* the recommendation, because a concern written after a conclusion tends to justify it
* Prefer the question to the technique list. Listing methods an agent already knows adds obligation without information, and turns into a checklist the moment it is written down

Procedure is what you write down once the questions have become boring. Writing it down first is
how the questions get lost.
