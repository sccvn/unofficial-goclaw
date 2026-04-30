# Agents — Onboarding Guide

## What Are Agents?

GitHub Copilot agents are role-specific AI assistants scoped to a bounded domain of this codebase. Each agent file in this directory defines:
- **When** to invoke it (trigger conditions)
- **What** it needs to start working (inputs)
- **What** it produces (output artifacts)
- **How** to verify its work is correct (quality gates)

## How to Invoke an Agent

In GitHub Copilot Chat (VS Code), type `@` followed by the agent name:

```
@senior-Backend_Developer implement a new HTTP endpoint for agent export
@senior-Frontend_Developer add a mobile-responsive settings page
@senior-Cloud_Architect debug why the release workflow failed
@senior-Automation_Tester write integration tests for the new vault handler
@lead-Solution_Architect design the multi-tenant webhook delivery system
@senior-Manual_Tester create a test plan for the Telegram channel integration
@senior-Software_Engineer-DBA add a migration for the new agent_hooks_v2 table
```

## Available Agents

| Agent File | Role | Primary Domain |
|---|---|---|
| `ai-augmented-engineer-v2.agent.md` | Orchestrator | Bootstrap + docs generation |
| `senior-Backend_Developer.agent.md` | Senior | Go backend, gateway, pipeline, store |
| `senior-Frontend_Developer.agent.md` | Senior | React web UI, desktop frontend |
| `lead-Solution_Architect.agent.md` | Lead | Architecture, ADRs, security model |
| `senior-Automation_Tester.agent.md` | Senior | Go tests, vitest, CI |
| `senior-Cloud_Architect.agent.md` | Senior | Docker, CI/CD, releases |
| `senior-Manual_Tester.agent.md` | Senior | Test plans, acceptance criteria |
| `senior-Software_Engineer-DBA.agent.md` | Senior | PostgreSQL, SQLite, migrations |

## Multi-Agent Coordination Model

GoClaw uses the **HIVE model with Hierarchical topology**:

```
AI Augmented Engineer (Orchestrator)
         │
         ├── Lead Solution Architect (Architecture + cross-cutting)
         │         │
         │         ├── Senior Backend Developer
         │         ├── Senior DBA
         │         └── Senior Cloud Architect
         │
         └── Quality Guild
                   ├── Senior Automation Tester
                   ├── Senior Manual Tester
                   └── Senior Frontend Developer
```

For complex features, start with `@lead-Solution_Architect` for design, then delegate implementation to specialized agents.

## Adding a New Agent

1. Copy an existing agent file as a template
2. Name it `<role>-<Domain>.agent.md` (snake_case domain after dash)
3. Fill in all 5 sections: trigger conditions, inputs, responsibilities, output artifacts, quality gates
4. Add it to the table in `.github/copilot-instructions.md`
5. Add it to the table in this `INSTRUCTION.md`
