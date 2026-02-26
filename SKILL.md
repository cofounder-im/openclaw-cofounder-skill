---
name: cofounder-im
description: Pull startup project data and AI-generated build specifications from CoFounder.im. Use when a user wants to build a project that was validated and planned on CoFounder.im — lists projects, fetches the full build spec (tech stack, MVP plan, UI/UX, implementation plan, OpenClaw builder output), and helps orchestrate sub-agents to build it.
version: 1.1.0
metadata:
  openclaw:
    requires:
      env:
        - COFOUNDER_API_TOKEN
      bins:
        - curl
        - jq
      primaryEnv: COFOUNDER_API_TOKEN
    emoji: "\U0001F680"
    homepage: https://cofounder.im/openclaw
    safety:
      autoApproval: false
      requiresUserConfirmation: true
---

# CoFounder.im Skill

## Overview

Connect OpenClaw to [CoFounder.im](https://cofounder.im) to pull AI-validated startup projects and autonomously build them. CoFounder.im runs 20+ AI agents that validate ideas, research markets, plan MVPs, design UI/UX, and generate implementation plans. This skill lets you fetch those results and use them as the foundation for building the project.

## Safety policy

This skill fetches build specifications from a remote API (cofounder.im). To ensure safe execution:

1. **User approval required** — Always show the user a summary of the build plan and get explicit confirmation before spawning any sub-agents or running any commands.
2. **Preview before execution** — Before each phase, display the sub-agent's goal, requirements, and verification command. Ask the user to approve.
3. **No blind execution** — Do not execute the build spec verbatim. Review each phase for reasonableness before presenting it to the user.
4. **Sandboxed environment recommended** — Run builds in a dedicated directory or container. Do not build in directories containing existing projects or sensitive files.
5. **Credential scope** — This skill only requires `COFOUNDER_API_TOKEN` for the CoFounder.im API. Build plans may reference additional tools (databases, cloud CLIs, etc.) depending on the project's tech stack — the user should review and install these as needed.
6. **No network access by default** — Sub-agents should only access local files and the project repository. Any external network calls (package installs, API integrations) should be reviewed by the user.

## Core rules

- Always authenticate with the bearer token from `COFOUNDER_API_TOKEN`.
- Use `list-projects` first to show available projects and let the user choose.
- Use `get-build-spec` to pull the full build specification for the chosen project.
- The build spec contains agent outputs keyed by type (e.g., `tech_stack`, `mvp_planner`, `ui_ux_assistant`, `implementation_plan_generator`, `openclaw_builder`).
- The `openclaw_builder` output is the most important — it contains a structured multi-agent build plan designed specifically for OpenClaw.
- Parse the `openclaw_builder` output to identify sub-agent tasks, then present them to the user for approval before spawning.
- **Always get user confirmation before spawning sub-agents or running verification commands.**

## Quick start

```bash
# Set your API token (generate at https://cofounder.im/users/settings)
export COFOUNDER_API_TOKEN="cfr_your_token_here"

# List your projects
curl -s https://cofounder.im/api/v1/projects \
  -H "Authorization: Bearer $COFOUNDER_API_TOKEN" | jq '.projects[] | {id, name, status}'

# Get build spec for a project
curl -s https://cofounder.im/api/v1/projects/PROJECT_ID/build-spec \
  -H "Authorization: Bearer $COFOUNDER_API_TOKEN" | jq .
```

## Workflow

### Step 1: List projects

Fetch the user's projects and present them for selection:

```bash
curl -s https://cofounder.im/api/v1/projects \
  -H "Authorization: Bearer $COFOUNDER_API_TOKEN" | jq .
```

Response shape:
```json
{
  "projects": [
    {
      "id": "uuid",
      "name": "ProjectName",
      "description": "...",
      "status": "completed",
      "package_type": "basic",
      "inserted_at": "2026-02-20T10:30:00Z"
    }
  ]
}
```

Show the user a numbered list of projects with name, status, and description. Ask which one to build. Only projects with status `"completed"` have full build specs.

### Step 2: Fetch build specification

```bash
curl -s https://cofounder.im/api/v1/projects/PROJECT_ID/build-spec \
  -H "Authorization: Bearer $COFOUNDER_API_TOKEN" | jq .
```

Response shape:
```json
{
  "project": {
    "id": "uuid",
    "name": "ProjectName",
    "description": "...",
    "status": "completed"
  },
  "agent_outputs": {
    "tech_stack": "## Tech Stack\n\n...",
    "mvp_planner": "## Core Features\n\n...",
    "ui_ux_assistant": "## Design System\n\n...",
    "implementation_plan_generator": "## Phase 1: Setup\n\n...",
    "openclaw_builder": "# OpenClaw Build Plan\n\n..."
  }
}
```

### Step 3: Review and approve the build plan

Parse the `openclaw_builder` agent output and present the user with a summary:

1. **Show the overall plan** — project name, tech stack, number of phases, dependency graph
2. **List each sub-agent phase** — name, goal, dependencies, verification command
3. **Highlight any additional tools required** — databases, CLIs, package managers beyond curl/jq
4. **Ask the user to confirm** before proceeding to execution

Example summary to show the user:

```
Build Plan: ProjectName (6 phases)
Tech: Next.js, TypeScript, PostgreSQL

Phase 1: project-setup (no deps) — Initialize repo and install dependencies
Phase 2: database (depends: project-setup) — Create schema and migrations
Phase 3: auth (depends: project-setup) — User registration and login
Phase 4: core-features (depends: database, auth) — Main business logic
Phase 5: frontend (depends: core-features) — UI components and pages
Phase 6: testing (depends: frontend) — Test suite and verification

Additional tools needed: node, npm, psql

Proceed with this build plan? (yes/no)
```

### Step 4: Execute the build (after user approval)

For each sub-agent defined in the build plan:

1. **Show the user what will be spawned** — the sub-agent's goal, requirements, and context
2. **Get user approval** for each phase (or batch-approve all phases upfront)
3. Create the sub-agent using `sessions_spawn` with the requirements from its section
4. Include relevant context from other agent outputs (tech_stack, ui_ux_assistant, etc.)
5. Monitor sub-agent completion
6. **Show verification results** before proceeding to the next phase

**Constraints:**
- Maximum 5 concurrent sub-agents (OpenClaw limit)
- Sub-agents run at depth 1 (they cannot spawn their own sub-agents)
- Each sub-agent session is isolated — pass all needed context in the spawn instruction
- Follow the phasing in the build plan — complete Phase 1 before starting Phase 2

### Step 5: Verify and report

After all sub-agents complete:
1. Verify the project structure matches the implementation plan
2. Run any test commands specified in the build plan
3. Report completion status to the user

## API reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/projects` | GET | List all projects for the authenticated user |
| `/api/v1/projects/:id/build-spec` | GET | Get project details and all completed agent outputs |

**Authentication:** Bearer token in the `Authorization` header.

```
Authorization: Bearer cfr_your_token_here
```

**Error responses:**
- `401` — Missing or invalid token
- `404` — Project not found or does not belong to user
- `429` — Rate limit exceeded (20 requests/minute)

## Agent output keys

| Key | Description |
|-----|-------------|
| `tech_stack` | Technology stack recommendation |
| `mvp_planner` | MVP feature plan and roadmap |
| `ui_ux_assistant` | UI/UX design system and guidelines |
| `implementation_plan_generator` | Step-by-step implementation plan |
| `openclaw_builder` | Multi-agent build specification for OpenClaw |
| `idea_validator` | Idea validation analysis |
| `market_research` | Market size and trends |
| `competitor_analysis` | Competitive landscape |
| `customer_persona` | Target customer profiles |
| `business_model` | Business model canvas |
| `monetization_strategy` | Revenue and pricing strategy |
| `go_to_market` | Go-to-market strategy |

## Additional runtime dependencies

Build plans generated by CoFounder.im may require tools beyond `curl` and `jq`, depending on the project's tech stack. Common examples:

| Tech Stack | Additional Tools |
|------------|-----------------|
| Node.js / Next.js / React | `node`, `npm` or `yarn` |
| Elixir / Phoenix | `elixir`, `mix`, `postgres` |
| Python / Django / FastAPI | `python`, `pip`, `postgres` |
| Ruby / Rails | `ruby`, `bundler`, `postgres` |
| Go | `go` |

The skill will identify required tools during the build plan review (Step 3) so you can install them before execution begins.

## Getting your API token

1. Sign up at [cofounder.im](https://cofounder.im)
2. Create a project and run the AI agents
3. Go to [Settings](https://cofounder.im/users/settings) and generate an API token
4. Configure it: `openclaw config set skills.entries.cofounder-im.env.COFOUNDER_API_TOKEN "cfr_..."`
5. Restart: `openclaw gateway restart`
