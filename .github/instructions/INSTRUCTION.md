# Instructions — Onboarding Guide

## What Are Instructions?

GitHub Copilot instruction files (`.instructions.md`) provide persistent context rules that apply automatically to Copilot Chat sessions. They're not invoked manually — they load based on the `applyTo` glob pattern in their frontmatter.

## Files in This Directory

| File | applyTo | Purpose |
|---|---|---|
| `HIVE_Hierarchical.instructions.md` | `**` | Multi-agent coordination model, workflow protocol, delegation rules |

## How to Add a New Instruction File

1. Create `<topic>.instructions.md` with YAML frontmatter:
   ```yaml
   ---
   applyTo: 'internal/**/*.go'   # glob pattern for when this applies
   ---
   ```
2. Write concise, actionable rules (not prose)
3. Keep each instruction file focused on one concern
4. Document it in the table above

## Scope Recommendations

| Glob | Use for |
|---|---|
| `**` | Project-wide conventions |
| `internal/**/*.go` | Go backend-specific rules |
| `ui/web/**` | Web UI-specific rules |
| `ui/desktop/**` | Desktop-specific rules |
| `migrations/**` | SQL migration rules |
| `tests/**` | Test-specific conventions |
