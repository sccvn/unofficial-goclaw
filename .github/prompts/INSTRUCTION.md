# Prompts — Onboarding Guide

## What Are Prompts?

GitHub Copilot prompt files (`.prompt.md`) are reusable prompt templates that provide domain-specific context. They can be referenced in Copilot Chat using `/` followed by the prompt name, or attached to agent definitions.

## Files in This Directory

| File | Audience | Domain |
|---|---|---|
| `go-gateway-backend.prompt.md` | Backend developers | Go idioms, pipeline, HTTP, WS, store |
| `react-web-ui.prompt.md` | Frontend developers | React 19, mobile UX, routing, i18n |
| `postgresql-store-layer.prompt.md` | Backend + DBA | Store interface, queries, migrations, indexes |
| `agent-pipeline-ai.prompt.md` | Backend developers | 8-stage pipeline, providers, tools, memory |
| `deployment-infra.prompt.md` | DevOps / Cloud | Docker, CI/CD, releases, Wails desktop |

## How to Use a Prompt

In Copilot Chat:
```
/go-gateway-backend implement a new handler for webhook delivery
/react-web-ui add a mobile-responsive agent card component
/postgresql-store-layer design the schema for webhook_deliveries table
/agent-pipeline-ai add a new pre-summarize pipeline callback
/deployment-infra create a release for v3.1.0
```

## Adding a New Prompt

1. Create `<domain>-<expertise>.prompt.md` with YAML frontmatter:
   ```yaml
   ---
   description: 'One-line description of what this prompt provides'
   ---
   ```
2. Focus on **patterns** and **examples** specific to this codebase
3. Keep it actionable — code snippets preferred over prose
4. Add it to the table above
