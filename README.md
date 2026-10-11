# agent-reproduction

A Claude skill for reproducing LLM-agent / CS papers from their official repositories.

It walks through four phases:

1. **Feasibility audit** – data availability, split definitions, official vs. approximate metrics, environment fidelity, coverage gaps, hidden budget dependencies, cost.
2. **Protocol** – fixed model/decoding/seeds/splits, an in-setting baseline on the same test items, pre-specified endpoint and test.
3. **Running cheaply and robustly** – pilot runs, process-level parallel evaluation, sleep/network resilience, checkpoint & resume, categorized failure logging.
4. **Analysis** – paired comparisons with CIs, repeats vs. seeds, winner's-curse caveats, data-defined failure subgroups, implementation-vs-paper audit.

## Install

Copy the `agent-reproduction/` folder into your skills directory (e.g. `~/.claude/skills/`), or upload it as a skill in Claude.
