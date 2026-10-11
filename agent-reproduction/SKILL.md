---
name: "agent-reproduction"
description: "Plan, run and analyse a reproduction of an LLM-agent / CS paper from its official repo: audit data splits and metrics, run cheaply and robustly, add paired baselines, seeds and failure analysis."
---

# Agent / CS paper reproduction

Use when the user wants to reproduce, re-run, or check an LLM-agent or other CS/ML paper (usually an arXiv/ACL/NeurIPS/ICLR paper with a GitHub repo), or compare it with an alternative paper before choosing one. Work in phases; finish each phase's checklist before moving on, and tell the user plainly what can and cannot be claimed at every step.

## Phase 1 - Feasibility audit (before spending any compute)

Clone the official repo (and any separate "artifact" repo the README links). Read the code, not only the README. Produce a short table answering:

1. **Data**: Which benchmarks? Are the FULL datasets used in the paper available, or only demo subsets? Where does each dataset come from (official source vs. third-party re-hosting)?
2. **Splits**: Are train / val / test splits defined in code with fixed seeds (good), in released files (good), or only described in the paper (bad)? Record exact sizes and seeds. If splits are unspecified, say so: absolute numbers will not be comparable.
3. **Metric**: Is each metric the benchmark's official one, a re-implementation, or an offline approximation (e.g. an LLM judge replacing a live-API evaluator)? Flag every non-official metric.
4. **Environment fidelity**: Are tools/APIs executed live, replayed from recordings, or mocked? Estimate how much of the signal depends on missing/replayed data.
5. **Coverage gaps**: List every table/setting in the paper that the code cannot produce (missing subsets, models, baselines).
6. **Hidden budget dependencies**: e.g. a method's budget copied from another method's run, caches shared between runs (a run that waits for another run's cache will hang).
7. **Cost**: models used, number of rollouts/LLM calls, GPU needs. Give a rough token or GPU-hour estimate from the paper or from a pilot.

Verdict: one of (a) reproducible as published, (b) reproducible on a subset with a self-defined protocol, (c) not worth it. If the user has an alternative paper, run the same audit on it and compare side by side.

## Phase 2 - Protocol (write it down before running)

- Fix model, decoding settings (temperature, thinking on/off, max tokens), seeds, splits, budget.
- If the paper's split is unavailable, define one yourself (official test set as test where possible, seed it) and document it.
- **Always include an in-setting baseline.** Numbers from the paper's baseline table are not comparable once model/data/budget differ. The cheapest strong control is the method's own unoptimized starting point (e.g. initial prompts / untrained policy) evaluated on the SAME test items; add external baselines only if budget allows.
- Pre-specify the primary endpoint (e.g. per-item success, paired difference) and the statistical test before seeing results.

## Phase 3 - Running cheaply and robustly

- Prefer OpenAI-compatible APIs for cheap models; keep API keys out of commands, configs and git. Note that generated launch scripts may contain keys.
- Start with a dry run / tiny pilot (e.g. 10% budget) to measure tokens and time, then extrapolate cost before the full run.
- **Evaluation is embarrassingly parallel; optimization loops usually are not.** Parallelise fixed-policy evaluation across processes (shard by policy x repeat x test chunk, one API client + log file per process, results appended per item so reruns can resume). Do not add threads to code whose logging/trace context is not thread-safe.
- Long runs on a laptop die from sleep and network drops. Before a multi-hour run: keep the machine awake and on power (macOS: `caffeinate -dimsu`, lid open), raise retry counts for connection errors (minutes of tolerance, not seconds), and run inside tmux.
- If the optimizer has no checkpointing and runs are long, add it: save loop state (population/params, RNG state, data-order position, token counters) after every step, and persist finished episodes so a resume reuses them instead of re-calling the API. Verify with a deterministic fake API that interrupted+resumed output equals uninterrupted output.
- Log failures by category as they happen: task failures (score below max, with benchmark-specific reasons) separately from system errors (rate limit, timeout, connection, context overflow, parse error). System errors silently scored as 0 corrupt results.

## Phase 4 - Analysis

- Paired comparison on identical test items: per-item difference, bootstrap 95% CI, paired permutation / exact McNemar p-value. Report Wilson CIs for single rates.
- **Repeats vs seeds**: repeating evaluation of a fixed policy measures API/decoding noise (usually small at temperature 0). Re-running the whole optimization with different seeds measures method variance (usually large; even the same seed may diverge through LLM nondeterminism). Claims about the method need multiple seeds.
- Selection-set scores are optimistic (winner's curse); only held-out test counts.
- Check that a re-run of the reported policy agrees item-by-item with the original logged result.
- Break failures down by data-defined subgroups (e.g. path length, call count, replay coverage) to separate method problems from data/environment problems.
- Audit the implementation against the paper: information the paper says flows between components but the code does not pass, prompt/parser mismatches, missing "blame the environment" options, one-tool-per-step limits. Mark each finding as confirmed (code/log evidence) or risk (needs an ablation).
- A non-significant result with n~30 means "not detected at this sample size", not "no effect". Never compare absolute numbers with the paper's table when model/data/budget differ.

## Things that went wrong before (check them)

- Demo subsets in the repo while the paper reports full benchmarks.
- A launcher that waits forever for another method's cache directory.
- Machine sleep freezing a run for hours, then retries exhausted within seconds on wake.
- A proxy/VPN drop killing a run after ~1 minute of failed retries.