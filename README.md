# agent-reproduction

A Claude skill for reproducing LLM-agent / CS papers from their official repositories.

It walks through four phases:

1. **Feasibility audit** – data and splits (including what the repo's demo data drops), official vs. approximate metrics, environment fidelity, coverage gaps, cost.
2. **Preparation** – list what may vary, what must stay fixed and how scoring works; check the pipeline stage by stage (parsing, tool execution, scoring, saving, statistics) and report each finding with code location, trigger, effect and a minimal check; in-setting baseline; fake-model pilot.
3. **Running** – cheap APIs, process-level parallel evaluation, sleep/network resilience, checkpoint & resume, separate task and system failures.
4. **Analysis** – paired comparisons with CIs, repeats vs. seeds, data-defined failure subgroups, attribution audit for self-improving methods.

## Install

Copy the `agent-reproduction/` folder into your skills directory (e.g. `~/.claude/skills/`), or upload it as a skill in Claude.
