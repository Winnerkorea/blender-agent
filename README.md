# Blender Agent

[한국어 README](README.ko.md)

Run Python in Blender via HTTP. No MCP, no protocol, no dependencies -- just `curl`.

Use Codex, Gemini CLI, or Claude Code to create 3D scenes, motion graphics, and
video edits by describing what you want in plain English.

Targets **Blender 5.1+** only.

## Fork

This repository is forked from
[ptrthomas/blender-agent](https://github.com/ptrthomas/blender-agent). The original
project is maintained by Peter Thomas and provides the Blender HTTP addon plus the
initial Claude Code skill workflow. This fork adapts the project for Codex and Gemini
CLI, keeps skill-guided Blender API usage as the source of truth, and adds a
conversational design workflow for iterating on scene forms.

## Quick start

### 1. Install the addon

Symlink into Blender's extensions directory (one-time):

```bash
mkdir -p ~/Library/Application\ Support/Blender/5.1/extensions/user_default
ln -sf $(pwd)/blender_agent ~/Library/Application\ Support/Blender/5.1/extensions/user_default/blender_agent
```

Then enable it in Blender: **Edit > Preferences > Add-ons**, search for "Blender Agent" and check the box.

The symlink means edits to the addon source are live — just restart Blender to pick up changes.

### 2. Start Blender with the server

```bash
python3 start_server.py
```

This launches Blender, waits until the HTTP server is ready, and prints the version.
If Blender is already running, it connects to the existing instance.

**Alternative — start manually from the UI:**
Open the 3D Viewport sidebar (press `N`), find the **Agent** tab, and click **Start**.

### 3. Verify it works

```bash
curl -s localhost:5656 --data-binary @- <<< 'bpy.app.version_string'
```

Should return something like: `{"ok": true, "result": "5.1.0", "output": ""}`

## Using with Agents

The repo includes Agent Skills that teach coding agents how to drive Blender:

| Skill | Triggers on |
|-------|-------------|
| `blender` | General Blender automation, screenshots, scene inspection |
| `blender-3d` | 3D objects, materials, cameras, lights, animation, rendering |
| `blender-vse` | Video editing, timelines, text overlays, transitions |
| `blender-geometry-nodes` | Geometry nodes, procedural geometry, instancing, scatter |
| `blender-laser` | Laser beams, raycast reflections, bouncing light paths |
| `blender-projector` | Spotlight projection patterns, gobos, volumetric beams |
| `blender-audioviz` | Audio extraction, beat-synced lights, reactive materials |

Agent entry points:

- Codex: read `AGENTS.md`; use `.agents/skills/` as the canonical skill tree.
- Gemini CLI: read `GEMINI.md`; it imports `AGENTS.md` and points to the same skills.
- Claude Code: read `CLAUDE.md`; `.claude/skills/` is kept as a compatibility mirror.

The intended workflow is conversational design. Describe the form you want to create
or change, discuss shape, proportions, materials, lighting, and motion with the agent,
then let the agent make small skill-guided Blender edits and inspect the result with
you. Skill docs should drive Blender API usage more than model training data.

Just ask the agent what you want:

```
> create a glowing neon cube that rotates and render it to video with bloom

> add subtitles to output/video.mp4 at these timestamps: ...

> set up a 3-point lighting rig and render a turntable animation
```

The agent sends Python code to Blender, renders frames, inspects the output visually,
and iterates until it looks right.

### Tips

- **Start Blender first.** Agents can do this for you if you ask, but it's faster to have it running already.
- **Output goes to `output/`.** An `OUTPUT` variable is injected into all code, so use `f"{OUTPUT}/render.mp4"` for output paths.
- **Visual feedback.** Agents render test frames and inspect them to iterate on aesthetics; this is normal and useful.

## Manual usage (curl)

For multi-line Python, use a heredoc to avoid shell quoting issues:

```bash
curl -s localhost:5656 --data-binary @- <<'PYEOF'
import bpy
bpy.ops.mesh.primitive_cube_add(location=(1, 2, 3))
bpy.data.objects.keys()
PYEOF
```

Or send a script file:

```bash
curl -s localhost:5656 --data-binary @script.py
```

### Response format

```json
{"ok": true, "result": "<last expression>", "output": "<stdout>"}
{"ok": false, "error": "<message>", "output": "<stdout before error>"}
```

- `result` — the return value of the last expression in your code (auto JSON-serialized)
- `output` — captured stdout from `print()` statements
- `error` — the exception message if something went wrong

## How it works

The addon starts a tiny HTTP server (default port 5656) inside Blender. When it receives a POST request, it queues the Python code to run in Blender's main thread via `bpy.app.timers`, waits for it to complete, and returns JSON with the result.

This means the code has full access to `bpy` and runs in the correct context for all Blender operations — no restrictions.

## One-time setup tips

Disable the splash screen (persists across restarts):

```bash
curl -s localhost:5656 --data-binary @- <<'PYEOF'
bpy.context.preferences.view.show_splash = False
bpy.ops.wm.save_userpref()
PYEOF
```

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
