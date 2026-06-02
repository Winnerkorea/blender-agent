# Blender Agent for Gemini CLI

Use the canonical project instructions:

@./AGENTS.md

## Gemini CLI Notes

- Keep this file thin. Put durable project rules in `AGENTS.md`.
- Use `.agents/skills/` as the canonical skill documentation for Blender API work.
- When a task is complex or occasional, read the relevant `SKILL.md` only when needed
  instead of loading all skill docs into the main context.
- Gemini CLI can inspect loaded context with `/memory show` and reload it with
  `/memory refresh`.
- If this repository is wrapped as a Gemini extension later, keep the extension context
  file thin and point it back to `AGENTS.md`.
