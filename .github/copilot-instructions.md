# GitHub Copilot Instructions

## Communication style

- Do not use emojis.
- Be direct and concise.
- Use plain prose and markdown tables where appropriate.

## Repository context

This repository contains skill files for the Nonica Revit MCP, which enables AI assistants to query and control Autodesk Revit models via natural language.

Two skill files are provided:

- `SKILL.md` — full reference for frontier models (Claude, GPT-4o, Gemini, etc.)
- `SKILL_LocalLLM.md` — lean edition optimised for Gemma 4 27B and smaller local LLMs

## Contribution guidelines

- Keep `SKILL.md` and `SKILL_LocalLLM.md` consistent where they overlap (category IDs, rule numbers, gotcha descriptions).
- When adding a new tool to `SKILL.md`, also consider whether it belongs in the Quick Tool Reference table in `SKILL_LocalLLM.md`.
- Category IDs must match the standard Revit built-in values — do not invent or approximate IDs.
- All coordinate values in examples must use feet (Revit's internal unit), not metres.
- Gotcha entries in `SKILL.md` are numbered sequentially; update the count in `README.md` when adding new ones.
- The three-step creation workflow (`create_tool_names_explorer` → `create_tool_arguments_explorer` → `create_tools_invoker`) must always be documented as three steps — never collapse or skip the middle step.
