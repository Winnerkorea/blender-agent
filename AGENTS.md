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
