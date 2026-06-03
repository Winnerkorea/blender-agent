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
- Design add-on capabilities through questions, official Blender documentation, and
  small Blender-verifiable implementation steps.
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

## Add-on Problem-Solving Loop

Use this loop whenever the user is designing functionality, not just scene content:

1. Ask what Blender behavior the user wants to change or add.
2. Translate the answer into an add-on capability: panel, operator, property,
   generated geometry, material driver, render workflow, or extension packaging.
3. Read the relevant skill for known API patterns.
4. Check official Blender documentation for missing or uncertain behavior.
5. Propose a small implementation that can be tested inside Blender.
6. Implement and verify through the HTTP server, UI, render, screenshot, or log.
7. Discuss whether the result solves the design problem before expanding scope.

Official references to use:

- Extensions: https://docs.blender.org/manual/en/latest/advanced/extensions/
- Add-ons preferences: https://docs.blender.org/manual/en/latest/editors/preferences/addons.html
- Extension command line: https://docs.blender.org/manual/en/dev/advanced/command_line/extension_arguments.html
- Blender 5.1 Python API notes: https://developer.blender.org/docs/release_notes/5.1/python_api/

## Modeling Issue Support

Treat "I got stuck while modeling" as a first-class workflow. The agent should help the
user learn how to solve the issue, not only patch the current scene.

For each issue:

1. Capture expected vs actual behavior.
2. Classify the issue: concept, Blender UI, add-on feature, Python API, geometry nodes,
   material, animation, rendering, or packaging.
3. Search the local skills first.
4. Use official Blender documentation or a clearly cited guide for gaps.
5. Convert the answer into a small testable action.
6. Verify inside Blender.
7. Add the learned pattern to docs or skills when it is reusable.

## Company and Local LLM Support

Company workflows may include private design systems, asset rules, approval stages, or
security restrictions. Local LLMs may be used when scene files, screenshots, logs,
prompts, or design rules cannot leave the company environment.

Design the integration around layered knowledge:

- Public layer: Blender official documentation, public skills, README files, open-source
  add-on code.
- Team layer: private design rules, naming conventions, asset libraries, review
  checklists, examples, and approved workflows.
- Local LLM layer: offline retrieval indexes, local prompts, private summaries, and
  internal Q&A over team documents.

Rules:

1. Ask whether company-specific design constraints exist before broad changes.
2. Keep internal guides separate from public Blender facts.
3. Use local LLMs for private ideation and retrieval when cloud tools are disallowed.
4. Do not send confidential files, images, logs, or rules to external services without
   explicit user approval.
5. Use official Blender docs for API behavior and extension packaging.
6. Use skills for project-tested patterns.
7. If company constraints conflict with Blender defaults, explain the tradeoff and ask
   which constraint wins.

If no company guide exists, use Blender's official design guidance as the fallback:

- Human Interface Guidelines: https://developer.blender.org/docs/features/interface/
- Layout rules: https://developer.blender.org/docs/features/interface/human_interface_guidelines/layouts/
- Add-on Guidelines: https://developer.blender.org/docs/handbook/extensions/addon_guidelines/
- Add-on Development Setup: https://developer.blender.org/docs/handbook/extensions/addon_dev_setup/

Fallback design rules:

- Match Blender's existing UI conventions.
- Put important and frequent controls first.
- Group related controls with headings or sub-panels.
- Use dropdowns for enum-heavy controls and toggles for quick mode switching.
- Avoid complex visual layouts that do not fit Blender's UI system.
- Keep add-ons self-contained and avoid global side effects.

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
