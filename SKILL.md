---
name: swarm-mode
description: >-
  Use when the user asks to trigger swarm mode, 触发蜂群功能, 开启蜂群,
  用蜂群模式, 并发拆分任务, or asks Codex to work like a swarm. Turn Codex
  into a main orchestrator that breaks work into independent subagents. This
  skill is the single source of truth for subagent model routing, scheduling,
  reassignment, and concurrency policy. Keep at most 30 workers running at
  once, read `config/model-routing.json` before routing any worker, and follow
  the configured `default_profile` unless the user explicitly asks for another
  profile alias defined there.
---

# Swarm Mode

Use this skill when the user explicitly wants a swarm-style workflow.
The main Codex stays responsible for the whole task. Subagents are workers, not owners.
Read [config/model-routing.json](config/model-routing.json) before assigning any worker model.

## Trigger intent
Treat the following as direct triggers:

- `触发蜂群功能`
- `启用蜂群模式1档`
- `启用蜂群模式2档`
- `启用蜂群模式3档`
- `开启蜂群模式1`
- `开启蜂群模式2`
- `开启蜂群模式3`
- `用蜂群模式处理`
- `进入 swarm mode`
- `像 Claude Code 一样用蜂群做`
- requests to split work into many parallel agents

## Core behavior

1. Reframe the request as an orchestration job.
Write a short execution plan with clear parallel lanes before delegating.

2. Keep one main orchestrator.
The main Codex owns the user conversation, the final answer, risk checks, and the integration pass.

3. Hard cap concurrency at 30.
Never run more than 30 active subagents at the same time.
Default to fewer when the task has fewer than 30 meaningful independent slices.
If the host profile still uses the default subagent cap, make sure `config.toml` includes:

```toml
[agents]
max_threads = 30
```

4. Route models from one shared config policy.
This skill is the default source of truth for subagent decomposition, model allocation, scheduling, reassignment thresholds, and concurrency rules unless a repo-local decision explicitly records `respect_project_override`.
Always read [config/model-routing.json](config/model-routing.json) before routing any worker.
Use `default_profile` unless the user explicitly asks for another alias defined in `profile_aliases`.
Set the main orchestrator to `profiles.<active_profile>.main_orchestrator_model`.
For each lane, classify the work using `task_kinds`, then assign the model from `profiles.<active_profile>.task_kind_models`.
If any item in `hard_escalation_to_gpt_5_4` matches, override the lane to `gpt-5.4` immediately even if the active profile mapped it lower.
Use `gpt-5.2` only when the lane also satisfies `hard_floor_for_gpt_5_2`.
Only choose the subagent role after the final lane model is locked.
If the locked model is anything other than `gpt-5.4`, do not use `explorer`; Codex Desktop currently has a host-side issue where `explorer` may ignore non-`gpt-5.4` model overrides.
In that case, automatically fall back to `worker` first, then `default` if a worker shape is awkward, while keeping the lane read-only or bounded as planned.
Reserve `explorer` for lanes whose final locked model still remains `gpt-5.4` and whose scope is read-only discovery.
Tier `1` preserves the current conservative routing: almost all code-owning or decision-heavy work stays on `gpt-5.4`.
Tier `2` is intentionally more aggressive: first split mixed work into narrower discovery, bounded implementation, focused verification, targeted review, and docs lanes, then route those lanes to lower-cost models when the config says they are safe to do so.
Tier `3` pushes narrow discovery, execution, probe, and targeted review lanes down further, but still keeps the main orchestrator, architecture, complex implementation, refactors, test strategy, and authoritative review on `gpt-5.4`.
Other skills may note that a task could benefit from delegation, but they must hand subagent decomposition, routing, and scheduling to `swarm-mode` instead of redefining those rules locally.
If you find duplicated per-skill routing guidance, fold it back into `swarm-mode` and simplify the other skill to a reference.
If a lane changes category mid-task, reassign the model before continuing.

4a. Check repo-level routing overrides before spawning workers.
For repo work, read `.codex-runtime/swarm-routing-policy.json` first if it exists.
Then scan likely project docs such as `AGENTS.md`, current project entry docs, and local prompt templates for explicit subagent model-routing rules that could override `swarm-mode`.
Look for signals such as `gpt-*`, `codex-*`, `subagent`, `子代理`, `模型分配`, `同一模型`, `默认使用`, `必须使用`, `profile`, or `lane 路由`.
If the repo-local decision file says `respect_project_override`, do not interrupt the task again; keep the project's routing rule and continue.
If the repo-local decision file says `follow_swarm_mode`, treat `swarm-mode` as the only routing authority. If conflicting project docs still exist, remove or neutralize those rules before delegating when mutation is allowed; otherwise stop and report the drift.
If conflicting project docs are found and there is no repo-local decision file yet, stop before spawning workers and ask the user whether to:
- fully follow `swarm-mode`, remove the conflicting project rule now, and continue with `swarm-mode` routing
- keep the project's routing rule and continue without further routing prompts in future swarm runs for that repo
When the user chooses `follow_swarm_mode`, patch out the conflicting project rule, then write `.codex-runtime/swarm-routing-policy.json` with `mode: "follow_swarm_mode"`.
When the user chooses `respect_project_override`, leave the project rule in place and write `.codex-runtime/swarm-routing-policy.json` with `mode: "respect_project_override"` so future swarm runs do not ask again unless the user explicitly changes policy.
Use a small JSON object such as `{"mode":"follow_swarm_mode","updated_at":"ISO-8601","source":"user_decision","note":"..."}` for that repo-local policy file.

5. Delegate only independent work.
Give each subagent a concrete, bounded job with a disjoint responsibility when possible.
Do not send the same unresolved question to multiple workers unless you explicitly want competing approaches.

6. Keep the critical path local when needed.
If the very next decision depends on a result, prefer doing that part in the main Codex instead of blocking on a worker.

7. Merge before answering.
Collect worker outputs, reconcile conflicts, verify the combined result, and present one unified response.

## Runtime pairing

If workers will be resumed, handed off, or revisited across multiple rounds, pair with `$agent-runtime-plus`.
Give each worker a stable agent name and keep scoped memory, checkpoints, and snapshots per lane.

If you need stronger cleanup discipline than prompt-only behavior, pair swarm work with the shared runtime helper at `../../runtime-kit/codex_runtime.py`.
Use the `swarm` commands below to register spawned workers, track which ones are still open, and block finalization until they are closed.

Minimal setup:

```bash
python3 ../../runtime-kit/codex_runtime.py bootstrap --repo .
python3 ../../runtime-kit/codex_runtime.py swarm init --repo .
```

## Routing config

`config/model-routing.json` defines the active routing policy.

- `default_profile`: default lane-routing profile when the user does not ask for another one
- `profile_aliases`: words that switch to a named profile
- `task_kinds`: the shared task categories every lane must map to before a model is chosen
- `profiles.<name>.main_orchestrator_model`: model for the main coordinating agent
- `profiles.<name>.task_kind_models`: task-kind to model mapping for that profile
- `hard_escalation_to_gpt_5_4`: situations that always force the lane back to `gpt-5.4`
- `hard_floor_for_gpt_5_2`: minimum conditions that must still be true before `gpt-5.2` can own the lane

Recommended `task_kinds` for routing:

- `orchestrator`: final integration, risk checks, conflict resolution
- `architecture`: fix-shape choice, cross-module design, complex tradeoffs
- `implementation_complex`: multi-file or expanding code changes
- `implementation_bounded`: single-file or narrow write-scope code changes
- `test_strategy`: test design, failure attribution, acceptance decisions
- `test_execution`: run existing checks and report results
- `codebase_discovery`: locate files, read symbols, answer one bounded repo question
- `debug_probe`: repro, log triage, fault isolation, quick disposable probes
- `review_targeted`: checklist-based review on a small diff or bounded surface
- `review_authoritative`: final review conclusions and prioritized findings
- `docs_summary`: notes, migration docs, summaries, reviewer guidance
- `data_extraction`: template-driven extraction, classification, and formatting

## Swarm sizing
- Tiny task: 0-2 workers
- Normal task: 3-5 workers
- Large task: 6-8 workers
- Maximum: 30 workers

Do not force 30 workers onto a task that does not naturally split into 30 useful parts.

## Delegation pattern
For each worker, define:
- objective
- model (read the active profile in `config/model-routing.json`, map the lane to one `task_kind`, then apply `hard_escalation_to_gpt_5_4` and `hard_floor_for_gpt_5_2` before locking the model)
- role (use `explorer` only when the final locked model is `gpt-5.4`; otherwise use `worker`, or `default` as the fallback)
- scope
- file or module ownership when code is involved
- expected output
- constraints or validation focus

Good swarm lanes:

- codebase discovery
- implementation in separate modules
- test authoring
- regression review
- docs or migration notes

Avoid:

- overlapping write ownership
- duplicate exploration
- vague prompts like "look into this"

## Working rules

- Share a brief progress update before substantial orchestration work.
- If the task is substantial, provide a compact swarm plan before launching workers.
- Run the repo-level routing-override check before the first `spawn_agent` on repo work.
- Use the smallest number of workers that still creates real speedup.
- Reuse existing worker context when follow-up work is tightly related.
- After workers return, review their outputs before integrating.
- After a worker has finished and its result has been captured, close it with `close_agent` so it does not keep consuming the parent thread's active-agent budget.
- Before sending the final answer, close any completed or no-longer-needed workers unless the user explicitly asked to keep them open for follow-up work.
- If you launch many workers in batches, treat agent cleanup as part of each batch: `spawn_agent` -> `wait_agent` -> collect result -> `close_agent`.
- For repo work, register each spawned batch in the runtime helper, then mark each worker closed after `close_agent`.
- Before the final answer on repo work, run `python3 ../../runtime-kit/codex_runtime.py swarm assert-clean --repo .` and do not finalize while it reports pending workers.
- If a worker result conflicts with another, the main Codex resolves it and explains the chosen path if it matters.

## Cleanup Guard

Use this stronger pattern when you want the workflow itself to enforce cleanup:
1. After spawning a batch, register it immediately.
```bash
python3 ../../runtime-kit/codex_runtime.py swarm batch-start --repo . --label "analysis-batch-1" --worker /root/worker_a --worker /root/worker_b
```
2. After each `close_agent`, mark that worker closed in the runtime registry.
```bash
python3 ../../runtime-kit/codex_runtime.py swarm worker-close --repo . --worker /root/worker_a
```
3. Before the final answer, assert that no tracked workers remain open.
```bash
python3 ../../runtime-kit/codex_runtime.py swarm assert-clean --repo .
```
4. If you want a blocking guardrail, pair with `$hook-runtime-lite` and add:
```bash
python3 ../../runtime-kit/codex_runtime.py hooks add --repo . --phase before_finalize --type command --value "python3 ../../runtime-kit/codex_runtime.py swarm assert-clean --repo ." --description "block finalize if swarm workers remain open" --halt-on-nonzero
```

## Output style
When swarm mode is active:

- mention that you are using swarm mode
- summarize the parallel lanes
- keep the final answer unified rather than dumping raw worker transcripts
- call out anything that could not be verified

## Profile note

This skill is installed separately in each Codex profile so any one of them can enter swarm mode on demand.
It does not by itself create cross-app IPC between `codex`, `codex2`, and `codex3`.
