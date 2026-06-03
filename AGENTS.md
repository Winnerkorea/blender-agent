# Blender Agent

Drive Blender via HTTP POST to `localhost:5656`. Use this repository with Codex,
Gemini CLI, or Claude Code by following the same core workflow:

1. Discuss the intended shape, motion, material, and visual direction with the user.
2. Read the relevant skill file before using Blender APIs.
3. Start or connect to Blender with `python3 start_server.py`.
4. Send Blender Python through the local HTTP server.
5. Render or screenshot to `output/`, inspect the result, and iterate through dialogue.

## Agent Entry Points

This file is the canonical project instruction file for Codex and the shared source
for other agents.

- Codex: reads `AGENTS.md`; use `.agents/skills/` for task-specific Blender skills.
- Gemini CLI: reads `GEMINI.md`, which imports or points back to this file. Gemini
  users should keep long procedural knowledge in skills rather than in always-loaded
  context.
- Claude Code: reads `CLAUDE.md`, which is now a compatibility entry point. The
  historical `.claude/skills/` tree is a mirror of the Blender skills.

## Project Structure

```
blender_agent/__init__.py    # Blender addon: HTTP server + exec engine
blender_agent/blender_manifest.toml
start_server.py              # Launcher: connects or starts Blender, blocks until ready
output/                      # Render output and agent.log (gitignored)
AGENTS.md                    # Canonical project instructions
GEMINI.md                    # Gemini CLI entry point
CLAUDE.md                    # Claude Code compatibility entry point
.agents/skills/              # Canonical Agent Skills for Codex-style workflows
  blender/                   # General Blender automation (start here)
  blender-3d/                # 3D scenes, materials, animation, rendering
  blender-vse/               # Video Sequence Editor
  blender-geometry-nodes/    # Geometry Nodes (procedural geometry, instancing)
  blender-laser/             # Laser beams with raycast reflection
  blender-projector/         # Spotlight projection patterns (grid, dots, radial, circles)
  blender-audioviz/          # Audio visualization (extraction, patterns, lighting rigs)
.claude/skills/              # Compatibility mirror; keep in sync until removed
docs/agent-integration.md    # Codex/Gemini/Claude integration plan
```

## Source of Truth

Target **Blender 5.1+** only. No backwards compatibility is needed.

The Blender API patterns in `.agents/skills/` are the source of truth for 5.1 usage.
Follow the skills instead of relying on older training data. When you hit an API
error, fix the code and update the relevant skill so the documentation stays accurate.

## Conversational Design Workflow

This project is not a one-shot prompt-to-render pipeline. The agent should help the
user design Blender scenes through conversation.

- Treat the user's shape intent as the primary input: silhouette, proportions, surface
  treatment, motion, lighting, constraints, and references.
- Prefer asking and sharing design opinions before making large scene changes.
- Convert vague requests into concrete Blender-editable decisions, then implement the
  next small step.
- Use skill documentation for Blender API details. Do not invent API behavior from
  training data when a skill covers the topic.
- After each meaningful edit, inspect or render, describe what changed, and invite the
  next design adjustment.
- Keep the process collaborative: propose alternatives, explain tradeoffs, and let the
  user's evolving intent steer the final form.

## Question-Driven Add-on Design

The main product is not only a rendered scene. The main product is a Blender add-on
workflow that lets the user keep designing inside Blender.

For feature work, proceed in this order:

1. Ask what the user wants to change in Blender: shape, control surface, UI panel,
   operator behavior, generated geometry, material response, or render workflow.
2. Restate the design problem as a Blender add-on capability.
3. Check the relevant skill first for known Blender 5.1 API patterns.
4. Check official Blender documentation when the skill does not cover the API,
   extension packaging, add-on UI, preferences, operators, or command-line behavior.
5. Propose a small implementation step that can be tested inside Blender.
6. Implement through the add-on or skill-guided Blender script.
7. Verify in Blender, then discuss what the result means for the design.

Use official Blender documentation as the reference for extension structure and user
installation:

- Extension creation: https://docs.blender.org/manual/en/latest/advanced/extensions/
- Add-ons preferences and installation: https://docs.blender.org/manual/en/latest/editors/preferences/addons.html
- Extension command line: https://docs.blender.org/manual/en/dev/advanced/command_line/extension_arguments.html
- Blender Python API and release notes: https://developer.blender.org/docs/release_notes/5.1/python_api/

## Modeling Issue Support

Users often discover the real problem while modeling: a shape does not deform as
expected, a material is hard to control, geometry nodes become confusing, an operator
does not expose the right option, or an add-on workflow is unclear. Treat these moments
as the core use case.

When an issue appears during modeling:

1. Ask the user what they expected to happen and what actually happened.
2. Identify whether the issue is conceptual, Blender UI workflow, add-on behavior,
   Python API usage, geometry, material, animation, rendering, or packaging.
3. Explain the likely cause in user-friendly terms before proposing code.
4. Use skills for established local patterns.
5. Use official Blender documentation or a clearly cited guide when the skill does not
   answer the issue.
6. Turn the answer into an actionable Blender step: a UI action, add-on feature,
   script change, scene adjustment, or documentation update.
7. Verify the fix in Blender and add the learned pattern back to a skill or docs when
   it will help future users.

This workflow should serve both experts and general users. Experts may want direct API
details; general users may need the same answer framed as design choices and Blender UI
steps. Prefer explaining enough that the user learns how to solve the next related
problem, not only the current one.

## Company and Local LLM Workflows

Some teams have internal design systems, modeling conventions, approval processes, or
private asset rules. Some environments also use local LLMs because project files,
assets, or design discussions cannot leave the company network. Support these workflows
without changing the core principle: skills and verified documentation are the source of
truth.

When working in a company or local-LLM setup:

1. Ask whether there are internal design rules, naming conventions, asset libraries,
   review steps, or restricted data before proposing broad changes.
2. Treat internal guides as project-specific references, but keep them separate from
   public Blender facts and official API behavior.
3. Use local LLMs for private brainstorming, summarizing internal guides, and proposing
   design options when cloud tools are not allowed.
4. Do not send confidential scene files, assets, prompts, screenshots, logs, or company
   design rules to external services unless the user explicitly approves it.
5. Prefer local docs, local skills, and official Blender documentation for API and
   add-on implementation decisions.
6. When an internal rule conflicts with a public Blender pattern, explain the conflict
   and ask which constraint should win.
7. Capture reusable non-confidential patterns in skills or docs. Keep confidential
   company knowledge in private local documentation controlled by the team.

If no internal design guide exists, fall back to Blender's official design guidance:

- Human Interface Guidelines: https://developer.blender.org/docs/features/interface/
- Layout rules: https://developer.blender.org/docs/features/interface/human_interface_guidelines/layouts/
- Add-on Guidelines: https://developer.blender.org/docs/handbook/extensions/addon_guidelines/
- Add-on Development Setup: https://developer.blender.org/docs/handbook/extensions/addon_dev_setup/

Use these as practical defaults:

- Follow Blender UI conventions instead of inventing a separate interface style.
- Expose the most important and frequently used controls first.
- Group related properties with clear headings or sub-panels.
- Use dropdowns for enums with many choices; use compact toggles only when quick
  switching matters.
- Avoid complex or decorative layouts that do not integrate with Blender's UI system.
- Keep add-on behavior self-contained and avoid global side effects outside the add-on
  namespace.

Recommended documentation layers:

- Public base: `AGENTS.md`, `README.md`, `README.ko.md`, public skills, official Blender docs.
- Team layer: private company design rules, asset naming rules, review checklists, local examples.
- Local LLM layer: private prompts, local retrieval indexes, and offline summaries that never
  leave the company environment.

Do not use auto-memory. Do not create or write to `MEMORY.md` or memory directories.
All durable project knowledge belongs in `AGENTS.md`, `GEMINI.md`, `CLAUDE.md`, the
skill files, or explicit docs under `docs/`.

If stuck, search current Blender documentation:

- Python API: https://docs.blender.org/api/5.1/
- Release notes: https://developer.blender.org/docs/release_notes/5.1/python_api/
- VSE changes: https://developer.blender.org/docs/release_notes/5.1/sequencer/

## Blender HTTP Workflow

See `.agents/skills/blender/SKILL.md` before sending commands. In short:

- Always use a quoted heredoc such as `<<'PYEOF'` for Python sent through `curl`.
- Inspect the existing scene before modifying it; never assume a clean scene.
- Use the injected `OUTPUT` variable for screenshots, renders, exports, and logs.
- The server returns JSON only. Do not use `curl -o` for screenshots or renders.
- Watch `output/agent.log` while debugging long-running or hung requests.

## Skill Selection

Use the smallest relevant skill set:

- `blender`: connection, scene inspection, screenshots, logging, recovery.
- `blender-3d`: objects, materials, lights, cameras, animation, rendering.
- `blender-vse`: video timelines, strips, subtitles, transitions, VSE rendering.
- `blender-geometry-nodes`: procedural geometry, instancing, node trees.
- `blender-laser`: raycast reflection beams and laser frame handlers.
- `blender-projector`: Cycles spotlight projection through volumetric fog.
- `blender-audioviz`: audio extraction, beat curves, pulse lights, reactive materials.

## Skill Usability Tests

When asked to test a skill, launch a background sub-agent that builds something
non-trivial using only the relevant skill documentation. This validates whether the
skills are accurate, complete, and usable by an agent without prior knowledge.

### How to Run

1. Pick a test project that exercises multiple sections of the skill:
   - `blender-3d`: animated scene with materials, lighting, camera, and rendered output
   - `blender-vse`: multi-strip timeline with text, transitions, and video render
   - `blender-geometry-nodes`: procedural scene with instancing, keyframed inputs, and render
   - `blender-projector`: multi-pattern spotlight scene with fog, animated params, and render
2. Launch a background sub-agent with instructions to:
   - Read the relevant skill files from `.agents/skills/`
   - Verify Blender is running
   - Build the project step by step, checking for errors after each command
   - Render a test frame and report the path
   - Report what it built, confusing/wrong/missing docs, errors hit, and how they were resolved
3. Monitor progress by tailing the sub-agent output or `output/agent.log`.
4. Inspect the rendered output, then fix any skill documentation bugs the sub-agent found.

### Sub-Agent Restrictions

- Do not modify project files during the test.
- Do not use knowledge outside the skill files.
- Only send commands to Blender and report findings.

## Known Blender 5.1 Bugs

- **Strip modifiers crash**: `strip.modifiers.new()` segfaults. Avoid until fixed.
- **Render can crash**: threading segfault in `libIlmThread`. Use
  `resolution_percentage = 50` for test renders.
- **Never call `depsgraph.update()` in frame handlers**: this can crash or stutter
  playback. For hiding objects from `ray_cast`, use `hide_viewport = True` without
  `depsgraph.update()`. See the laser skill.
