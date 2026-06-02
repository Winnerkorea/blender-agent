# Agent Integration Plan

This repository should support Codex and Gemini CLI without duplicating long Blender
API documentation. Claude Code remains supported as a compatibility path while the
project migrates to a shared skill layout.

## Goals

- Keep one canonical set of Blender 5.1 skill docs.
- Keep always-loaded context files short.
- Make Codex and Gemini use the same Blender workflow.
- Make Blender work conversational: design intent first, skill-guided implementation
  second.
- Preserve Claude Code compatibility until the duplicate `.claude/skills/` tree can be
  removed or replaced by a generated mirror.

## Current State

- `AGENTS.md`: canonical project instructions for Codex and shared agent behavior.
- `GEMINI.md`: Gemini CLI entry point; imports `AGENTS.md`.
- `CLAUDE.md`: Claude Code compatibility entry point.
- `.agents/skills/`: canonical skill tree for Codex-style workflows.
- `.claude/skills/`: compatibility mirror, currently almost identical to `.agents/skills/`.

## Codex Workflow

Codex should:

1. Read `AGENTS.md`.
2. Discuss the target form before editing: shape, scale, proportions, material, motion,
   lighting, and constraints.
3. Load only the relevant skill from `.agents/skills/`.
4. Start Blender with `python3 start_server.py` if needed.
5. Send Python through `curl -s localhost:5656 --data-binary @- <<'PYEOF'`.
6. Use `OUTPUT` for renders, screenshots, and logs.
7. Share what changed, compare it to the user's intent, and continue iterating.
8. Update the relevant skill when a Blender 5.1 API correction is discovered.

## Gemini CLI Workflow

Gemini CLI should:

1. Read `GEMINI.md`; it imports `AGENTS.md`.
2. Use `/memory show` to verify loaded context when behavior is unclear.
3. Discuss the user's design intent before making broad Blender edits.
4. Read only the relevant `.agents/skills/<skill>/SKILL.md` file for the task.
5. Avoid loading every skill file into the main session context.
6. Use `output/agent.log` for long-running Blender request debugging.
7. Iterate in small design steps: propose, edit, inspect, discuss, refine.

Gemini CLI supports extension-packaged skills. If this repository later becomes a
Gemini extension, keep the extension manifest and context lightweight, and either:

- point the extension context to `AGENTS.md`, or
- generate extension-local `skills/` files from `.agents/skills/`.

Do not maintain a third hand-written skill tree for Gemini.

## Duplicate Document Policy

Canonical:

- `AGENTS.md`
- `GEMINI.md`
- `.agents/skills/`
- `docs/`

Compatibility:

- `CLAUDE.md`
- `.claude/skills/`

Avoid copying large instruction blocks between root files. Root files should either
import, point to, or summarize the canonical instructions.

## Migration Options

### Option A: Keep Mirrors Temporarily

Keep `.agents/skills/` and `.claude/skills/` both present, but treat `.agents/skills/`
as canonical. Add a sync check to CI or a local script to fail when the mirror drifts.

Pros:

- Lowest risk for existing Claude Code users.
- Codex and Gemini can immediately use `.agents/skills/`.

Cons:

- Still duplicates skill files.
- Requires a sync step after edits.

### Option B: Generate Compatibility Mirrors

Store canonical skills in `.agents/skills/` and generate `.claude/skills/` from it.

Pros:

- One editable source of truth.
- Keeps Claude compatibility.

Cons:

- Requires a small sync script and contributor discipline.

### Option C: Remove Claude Mirror

Remove `.claude/skills/` after confirming Claude Code can reliably use `CLAUDE.md` to
read `.agents/skills/` on demand.

Pros:

- Cleanest repository.

Cons:

- May reduce automatic skill discovery in Claude Code.

## Recommended Next Step

Use Option A now, then move to Option B once the Codex/Gemini workflow is stable.
That keeps the project usable while making `.agents/skills/` the clear source of truth.
