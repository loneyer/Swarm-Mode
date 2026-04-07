# Swarm Prompt Templates

Use this file only when you want a concrete delegation scaffold.

## Main-agent preflight

Before the first `spawn_agent` on repo work:

```text
1. Read `.codex-runtime/swarm-routing-policy.json` if it exists.
2. Scan `AGENTS.md`, current project entry docs, and local prompt templates for project-defined subagent model-routing rules.
3. If no repo-local decision exists and a conflicting routing rule is found, stop and ask the user once:
   - follow `swarm-mode` fully and remove the project override now
   - keep the project override and suppress future routing prompts for this repo
4. Record the answer in `.codex-runtime/swarm-routing-policy.json`.
5. Only route and spawn workers after the conflict is resolved or a stored policy says what to do.
```

## Worker prompt template

```text
Objective:
<one clear result this worker owns>

Scope:
<files, modules, APIs, or questions this worker may touch>

Expected output:
<what the worker should return to the main agent>

Constraints:
- Read `config/model-routing.json`, choose the active profile, and route by `profiles.<active_profile>.task_kind_models`
- Map the lane to exactly one `task_kind` before choosing a model
- If any item in `hard_escalation_to_gpt_5_4` matches, hand the lane back to `gpt-5.4` immediately
- Use `gpt-5.2` only when every item in `hard_floor_for_gpt_5_2` is still true
- In tiers `2` and `3`, split mixed work into narrower discovery, bounded implementation, focused verification, targeted review, and docs lanes before routing them to lower-cost models
- Escalate back to `gpt-5.4` for strategy, synthesis, conflict resolution, final review conclusions, or cross-domain coordination
- Lane type does not choose the model by itself; route by the actual work
- If the lane changes category, hand it back for reassignment before continuing
- Do not overlap ownership with other workers unless explicitly told
- Do not revert unrelated work
- Flag uncertainty instead of guessing
```

## Batch cleanup checklist

Use this after a swarm batch finishes:

```text
1. Wait for the current worker batch to finish.
2. Capture only the result you need for integration.
3. Close each completed worker with close_agent.
4. If using the runtime helper, mark each one closed with swarm worker-close.
5. Before the final answer, run swarm assert-clean and do not finalize if it reports pending workers.
```

## Suggested lane types

- Discovery lane: map code, locate files, answer one bounded question
- Implementation lane: change one module or one disjoint write area
- Verification lane: run focused checks, regression review, or targeted tests
- Docs lane: summarize impact, migration notes, or reviewer guidance

Lane type does not choose the model by itself; route by the work's nature using the rules above.

## Model routing hints

- Tier `1` keeps almost all code-owning or decision-heavy work on `gpt-5.4`.
- Tier `2` can push bounded implementation, targeted review, focused verification, discovery, and debug lanes to cheaper models when they remain narrow and verifiable.
- Tier `3` keeps the main orchestrator and all complex or final-decision lanes on `gpt-5.4`, but pushes narrow discovery, test execution, debug probes, and targeted review lanes down another step when they remain template-friendly and easy to verify.
- Keep `gpt-5.2` lanes template-friendly, single-source, and easy to verify.
- Escalate to `gpt-5.4` as soon as the worker must change final code shape, choose between implementations, judge test or review outcomes, resolve ambiguity, or coordinate across sources.

## Main-agent integration prompt

```text
Review all worker outputs, resolve conflicts, verify the combined result, and produce one unified answer for the user.
Do not paste raw worker transcripts unless they are directly useful.
```
