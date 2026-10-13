---
name: "agent-reproduction"
description: "Reproduce an LLM-agent / CS paper from its official repo in five steps: set up the environment and read the source, pin down metrics and compared methods, record runs and results systematically, analyse each method's weaknesses from the results, and turn those weaknesses into improvement proposals."
---

# Agent / CS paper reproduction

Use when the user wants to reproduce, re-run or check an LLM-agent or other CS/ML paper that has a GitHub repo, compare it with an alternative paper, or build their own research on top of it. Work through the five steps below in order; each ends with a concrete deliverable. Tell the user plainly what can and cannot be claimed: "the code runs" is never evidence that the experiment is reliable.

## Step 1 - Environment setup and source reading (了解环境配置、阅读项目源码)

**Set up**
- Clone the official repo; read its GitHub issues as well as the README. Find the commit that matches the paper version and diff later commits: post-paper features can silently change defaults or break the paper's code path, so check that every paper config still runs at the chosen commit.
- Install exactly as the README says, then record every pin you had to add (incompatible library versions, monkeypatches of private APIs). Download large data with a resumable tool and the repo's own unpack routine.
- Keep API keys out of commands, configs, logs and git: a gitignored secrets file read by a small launcher that passes the key through the environment.
- Verify the environment before spending money: replay ground-truth / oracle solutions through the official evaluator (must score 100%), and run the repo's own verify scripts (they may be stale; fix them in a scratch copy, not in the repo).

**Read the code along the real execution path**: entry point -> config loading -> data loading -> agent loop -> each role/module -> output parsing -> tool execution -> scoring -> saving -> statistics. Produce a code map with `file:line` for each stage. At each stage look for empty/null values, error types, length and budget limits, retries, interruption and resume.

Check in particular:
- **Paper vs code**: components the paper describes that the code does not implement or never calls; configs pointing to missing files; configs that overwrite shipped artifacts; settings that differ from the paper (epochs, rounds, warm starts).
- **LLM client layer**: which parameters are actually sent (seed, max_tokens, response_format are often dropped or defaulted); which errors are retried (bad key / no balance / bad request should fail fast); whether post-processing (e.g. cost computation) sits inside the retry loop; hidden client-side retries; what the response cache is keyed on (often model + messages only, not endpoint).
- **Typical traps**: unparsable output treated as a normal finish; a truncation or limit applied to only one condition; by-design statuses (e.g. "not executed") left unexplained to LLM judges/blamers; gold keyed by function name so repeated calls are mis-scored; time freezing or other global state; per-task output deletion; parent processes that return 0 when children crash.

**Pilot with fake models through the official entry point**: an oracle generator must score 100%; malformed and over-budget outputs must hit the error paths; interrupt + resume must equal an uninterrupted run; a local fake OpenAI-compatible server shows the exact request body.

Deliverable: setup notes (commit, pins, data), code map, and a findings list, each item as *code location / trigger / effect on results / minimal check*, labelled *confirmed defect*, *reasonable design choice* or *unverified risk*.

## Step 2 - Metrics and compared methods (清楚评价指标、实验对比方法)

- **Metrics**: the official definition and where it is computed; official vs re-implemented vs approximated (e.g. LLM judge), and where it is stricter or looser than the official checker; granularity (e.g. scenario-level metrics move in steps of 1/#scenarios); what counts as success and how system errors are scored.
- **Data and splits**: full data or demo subset; sizes and seeds, fixed in files or only described; what is withheld (test ground truth); what the repo's data format drops (types, enums, required fields, multi-turn parts, original ids). No train split in the benchmark -> ask where optimisation data came from.
- **Environment fidelity**: live, replayed or mocked tools, and how much signal depends on missing recordings or data (measure with an oracle/ceiling run).
- **Compared methods**: map every row of the target table to code and config. Mark rows with no code (baselines the repo never shipped) and rows whose reported number actually comes from a different setting (e.g. an "online" row that equals an "offline warm-up" ablation). Note paper settings that differ from the shipped configs, and hidden budget dependencies.
- **Baselines**: always add an in-setting baseline (the method's unoptimised start, or e.g. ReAct through the same runner, environment, step budget and grader) on the same items. Paper baseline numbers are not comparable once model, data or budget differ.
- **Three lists before any real run**:
  - *Allowed to vary*: only the thing under test, plus declared repeats/seeds.
  - *Must stay fixed*: model, decoding, test items/split, step/token budgets, truncation, environment, retries, grader.
  - *Scoring rules*: what is graded, primary endpoint, statistical test, how failures and system errors count.
- **Config diff**: diff your configs against the official ones field by field (script it) and keep a table of deviations with a reason for each.
- **Feasibility verdict**: (a) reproducible as published, (b) reproducible on a subset with a self-defined protocol, (c) not worth it. Include coverage gaps and a cost/time estimate from measured tokens per call, not guesses.

Deliverable: metric sheet, table-row -> code/config map, deviation table, verdict.

## Step 3 - Systematic records of models and results (系统地整理、记录实验模型与结果)

- **One experiment = one directory**, named with a model tag + method + split (e.g. `repro_<model>_<method>_<split>`), so runs with different models or endpoints never share outputs or caches. Generate configs from one script rather than editing copies.
- **Record per run** (in run metadata, so comparisons can be checked mechanically): repo commit and local diff, config deviations, model + endpoint + decoding, item ids and how the subset was drawn (whole scenarios, seed, difficulty mix vs the full split), start/end time, exit status, and per item: success, LM calls, prompt/completion/cached tokens, cost.
- **Separate task failures from system failures** (API errors, rate limits, crashes); log errors with wall-clock time. Verify per-item outputs exist instead of trusting exit codes. Archive aborted or partial runs; never merge runs with different settings or incomplete repeats.
- **Running**: process-level parallelism (method x split x chunk lanes) with per-item, resumable outputs; do not add threads to non-thread-safe code. Long laptop runs under `tmux` + `caffeinate`, with retries that tolerate minutes-long outages and checkpoints of optimiser state and finished items. Sequential stages (adaptation, evolution) are the critical path; do not let evaluations starve them.
- **Rate-limited / free APIs**: measure the real limit and per-model capacity first (429/503 can be model congestion, not your quota); pace requests evenly across processes (not just a per-minute total), apply a global back-off after a 429, make sure nothing (client retries) bypasses the pacer, and use a real clock if the environment freezes time. Re-estimate cost/time after the first minutes from measured calls per item and output tokens (with prefix caching, output usually dominates cost).
- **Results table**: one row per (method, model, split) with our number, the paper's number, n, CI, and the deviations that apply.

Deliverable: experiment log (status files, run directories, per-item scores) and the results table.

## Step 4 - Results, metrics and each method's weaknesses (结合指标与方法输出结果，分析各方法不足)

- **Paired comparisons** on the same items: bootstrap CI, sign-flip permutation / McNemar; Wilson CI for single rates. With n around 30, non-significant means "not detected", not "no effect".
- **Noise**: repeats measure API nondeterminism, seeds measure method variance; check whether identical requests return identical outputs. Selection/validation-set scores are optimistic; always report held-out test.
- **Failure breakdown per method** into data-defined, mutually exclusive categories (missing step, wrong argument type/value, wrong call count, environment/replay limits, judge rejections) so method weaknesses are separated from data and environment limits. Use an oracle/ceiling run when the environment itself caps performance.
- **Mechanism checks for self-improving methods**: audit the attribution/credit step from logs (blame on passing items, misread statuses, environment errors blamed on components); check whether updates are validated before being kept; track context or prompt growth and its token cost.
- **Gap to the paper**: explain each difference using the deviation table (model, subset, epochs, missing components) before calling anything a failure to reproduce.

Deliverable: for each method, a list of weaknesses, each backed by numbers and log/file references, separated into *method*, *implementation*, *evaluation* and *data/environment* causes.

## Step 5 - Improvement proposals (针对不足尝试提出改进方案)

For each weakness worth pursuing:
- **State it as a research question** with the evidence from Step 4 (e.g. "contexts grow without credit assignment: X tokens, Y% inert entries, Z x the baseline's cost").
- **Classify the contribution**: new problem (quantified), new method (minimal fix aimed at the cause), new finding (e.g. transfer across models), or new evaluation protocol (gains that vanish under noise-aware evaluation).
- **Check novelty** against the closest related work before investing; say which papers were checked.
- **Smallest intervention first**, piloted on the existing subset with a success criterion fixed in advance; then scale.
- **Experiment matrix**: tasks (>= 2 domains) x models (>= 3, different sizes) x methods (baselines, original method, method as the paper describes it, the proposal, ablations) x repeats, with call/token/time/cost estimates.
- **Anticipate reviewers**: "just engineering", single benchmark, gains within noise, missing baseline, single model; plan the experiment that answers each.

Deliverable: a short proposal per idea: question, evidence, method sketch, experiment matrix with cost, risks, and the claim the paper would make.

## Rules throughout

- Never compare absolute numbers with the paper's table when model, data or budget differ; compare methods within your own setting.
- Confirm with the user before changing the model, spending money, or deviating from the official configs, and record every deviation.
