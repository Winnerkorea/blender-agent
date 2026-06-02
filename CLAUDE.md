# Blender Agent for Claude Code

Read `AGENTS.md` first. It is the canonical project instruction file for this
repository.

## Claude Compatibility

- `.claude/skills/` is retained as a compatibility mirror for Claude Code.
- `.agents/skills/` is the canonical skill tree used for Codex/Gemini planning.
- If a skill fix is made in one tree, keep the mirror synchronized until the duplicate
  tree is removed.
- Do not create `MEMORY.md` or write to memory directories. Durable project knowledge
  belongs in `AGENTS.md`, `GEMINI.md`, `CLAUDE.md`, skills, or `docs/`.
