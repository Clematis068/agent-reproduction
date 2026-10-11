---
name: "agent-reproduction"
description: "Plan, run and analyse a reproduction of an LLM-agent / CS paper from its official repo: audit data splits and metrics, run cheaply and robustly, add paired baselines, seeds and failure analysis."
---

# Agent / CS paper reproduction

Use when the user wants to reproduce, re-run, or check an LLM-agent or other CS/ML paper with a GitHub repo, or compare it with an alternative paper. Work in phases and tell the user plainly what can and cannot be claimed. "The code runs" is never evidence that the experiment is reliable.

## Phase 1 - Feasibility audit (before spending compute)

Read the code and the repo's GitHub issues, not only the README. Answer in a short table:

1. **Data**: full datasets or only demo subsets? Does the repo's data format drop information the official one has (types, enums, multi-turn parts, original ids)?
2. **Splits**: fixed in code/files with seeds, or only described? Record sizes and seeds. No train split in the benchmark -> ask where optimisation data came from.
3. **Metric**: official, re-implemented, or approximated (e.g. LLM judge)? Note where it is stricter/looser than the official checker.
4. **Environment**: live, replayed, or mocked tools; how much signal depends on missing data.
5. **Coverage gaps, hidden budget dependencies, cost estimate.**

Verdict: (a) reproducible as published, (b) reproducible on a subset with a self-defined protocol, (c) not worth it.

## Phase 2 - Preparation (before any real run)

**Write three lists first:**
- *Allowed to vary*: only the thing under test (e.g. evolved vs. initial policy), plus declared repeats/seeds.
- *Must stay fixed*: model, decoding settings, test items/split, step/token budgets, truncation rules, environment, retries, grader. Record them in run metadata so comparisons can be checked mechanically.
- *Scoring rules*: what is graded, what counts as success, how system errors are scored, the primary endpoint and test.

**Check the pipeline stage by stage** - data loading -> output parsing -> tool execution -> scoring -> saving -> statistics - for empty/null values, error types, length and budget limits, retries, interruption and resume. Typical traps: unparsable output treated as a normal finish; a truncation or limit applied to only one condition; by-design statuses (e.g. "not executed") left unexplained to LLM judges/blamers; gold keyed by function name so repeated calls are mis-scored; merging runs with different settings or incomplete repeats.

**Report each finding as:** code location / trigger condition / effect on results / minimal verification, classified as *confirmed defect*, *reasonable design choice*, or *unverified risk*.

**Include an in-setting baseline** (the method's unoptimised start, or e.g. ReAct through the same environment and grader) on the same test items; paper baseline numbers are not comparable once model/data/budget differ.

**Pilot with fake models before paying**: an oracle that returns gold must score 100%; malformed and over-budget outputs must hit the error paths; interrupted+resumed must equal uninterrupted.

## Phase 3 - Running

- OpenAI-compatible cheap APIs; keys out of commands, configs and git.
- Parallelise evaluation across processes (policy x repeat x chunk, append per item, resumable); don't add threads to non-thread-safe code.
- Long laptop runs: `caffeinate -dimsu`, tmux, retries tolerant of minutes-long outages, checkpoint the optimiser state and finished episodes.
- Log task failures and system errors separately.

## Phase 4 - Analysis

- Paired per-item comparison: bootstrap CI, permutation / McNemar; Wilson CI for single rates. n~30 non-significant = "not detected", not "no effect".
- Repeats measure API noise; seeds measure method variance. Selection-set scores are optimistic.
- Break failures down by data-defined subgroups to separate method from data/environment problems.
- For self-improving methods, audit the attribution step from logs (blame on passing episodes, misread statuses, environment errors blamed on components).
- Never compare absolute numbers with the paper's table when model/data/budget differ.
