# Swarm Mode

Swarm Mode is a Codex skill for swarm-style orchestration.

It keeps one main orchestrator, splits work into parallel lanes, caps active workers at 30, and routes each lane through a shared model policy in `config/model-routing.json`.

## Files

- `SKILL.md`: main skill definition and operating rules
- `config/model-routing.json`: routing profiles, escalation rules, and role policy
- `agents/openai.yaml`: agent metadata and default prompt
- `references/prompt-templates.md`: reusable prompt scaffolds for main-agent and worker lanes

## Usage

Install this skill into your Codex profile, then invoke `$swarm-mode` when you want Codex to decompose a task into parallel worker lanes.
