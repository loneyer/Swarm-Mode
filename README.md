# Swarm Mode

Swarm Mode is a Codex skill for swarm-style orchestration.

It keeps one main orchestrator, splits work into parallel lanes, caps active workers at 30, and routes each lane through a shared model policy in `config/model-routing.json`.

## How It Differs From Codex Built-in Agent Team / Subagents

Swarm Mode does not replace Codex's built-in subagent capability. It sits one layer above it.

Built-in Agent Team or subagents are the execution primitive: Codex can spawn workers, assign them bounded tasks, wait for results, and merge the outputs.

Swarm Mode adds a reusable orchestration policy on top of that primitive:

- one main orchestrator stays responsible for final judgment, integration, and risk control
- every lane is classified into a task kind before a model is chosen
- model routing is centralized in `config/model-routing.json` instead of being decided ad hoc each run
- concurrency is explicitly capped and scaled by task size instead of "spawn as needed"
- worker prompts are structured around objective, scope, ownership, expected output, and validation focus
- repo-level routing overrides are checked before delegation so project-specific rules do not silently conflict
- cleanup is treated as part of the workflow, with optional runtime assertions to ensure no workers are left open

In short, built-in subagents give you delegation. Swarm Mode gives you delegation discipline.

## Why Use Swarm Mode

Swarm Mode is useful when plain subagent delegation starts to become inconsistent, expensive, or hard to review.

- repeatable routing: the same class of work is routed through the same policy instead of depending on whoever writes the prompt
- lower coordination overhead: lanes have explicit ownership and expected outputs, which reduces overlapping edits and duplicate exploration
- better cost control: tiered profiles let you keep architecture and final review on stronger models while pushing narrow, verifiable work downward
- safer multi-agent runs: escalation rules move risky or ambiguous work back to the main model instead of letting weaker lanes drift into decision-heavy work
- better repo compatibility: project-level routing rules can be detected and resolved before workers are launched
- cleaner shutdown: the workflow includes close-out rules and optional runtime guards so long multi-agent runs do not leak workers
- easier adoption by teams: the policy lives in files, so it can be reviewed, versioned, and tuned like other project configuration

If you only need one or two temporary helpers, Codex built-in subagents may already be enough. Swarm Mode is for the cases where you want multi-agent execution to be repeatable, auditable, and policy-driven.

## Files

- `SKILL.md`: main skill definition and operating rules
- `config/model-routing.json`: routing profiles, escalation rules, and role policy
- `agents/openai.yaml`: agent metadata and default prompt
- `references/prompt-templates.md`: reusable prompt scaffolds for main-agent and worker lanes

## Usage

Install this skill into your Codex profile, then invoke `$swarm-mode` when you want Codex to decompose a task into parallel worker lanes.
