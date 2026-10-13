# agent-reproduction

A Claude skill for reproducing LLM-agent / CS papers from their official repositories and building research on top of them.

It works in five steps, each with a concrete deliverable:

1. **Environment setup and source reading** (了解环境配置、阅读项目源码) – pick the commit that matches the paper, record dependency pins, verify the environment with oracle replays, read the code along the real execution path, and list paper-vs-code gaps, LLM-client pitfalls and pipeline traps; fake-model pilot.
2. **Metrics and compared methods** (清楚评价指标、实验对比方法) – official metric definitions and granularity, data and splits, environment fidelity, map every table row to code/config, in-setting baselines, what may vary vs. what stays fixed, field-by-field config diff, feasibility verdict with measured cost.
3. **Systematic records of models and results** (系统地整理、记录实验模型与结果) – one tagged directory per run, per-item calls/tokens/cost, task vs. system failures, resumable parallel runs, pacing for rate-limited free APIs, results table against the paper.
4. **Results and each method's weaknesses** (结合指标与方法输出结果，分析各方法不足) – paired statistics, repeats vs. seeds, data-defined failure breakdowns, mechanism audits for self-improving methods, explaining gaps to the paper.
5. **Improvement proposals** (针对不足尝试提出改进方案) – turn weaknesses into research questions, classify the contribution, check novelty, pilot the smallest fix, plan the experiment matrix and reviewer questions.

## Install

Copy the `agent-reproduction/` folder into your skills directory (e.g. `~/.claude/skills/`), or upload it as a skill in Claude.
