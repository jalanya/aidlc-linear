# aidlc-linear

A skill plugin that converts [AI-DLC](https://github.com/awslabs/aidlc-workflows) code generation plans into a structured Linear task hierarchy — **Epic (Project) → Stories (Parent Issues) → Tasks (Sub-Issues)** — optimized for parallel AI agent execution via Symphony or similar orchestrators.

<img src="docs/demo.gif" />

## What It Does

Takes a code generation plan produced by the AI-DLC Construction phase and creates:

- **Epic project** in Linear representing the unit (e.g., "Unit 1: Shared Foundation")
- **Stories** as parent issues grouped by functional cohesion (e.g., "SF: Database Engine & Models")
- **Atomic tasks** as sub-issues with full context, explicit scope boundaries, validation matrix, and traceability
- **Dependency relations** between tasks for parallel execution scheduling
- **Parallel execution phases** identifying which tasks can run concurrently

Each task passes the **"cold start" test**: an agent with zero prior context can read only the task description and produce the correct output without asking questions.

## Installation

### Claude Code

```bash
claude plugin install jalanya/aidlc-linear --scope project
```

Or load directly during development:

```bash
claude --plugin-dir /path/to/aidlc-linear
```

### Opencode

Copy the skill into your project or global config:

```bash
# Project-level (recommended)
cp -r /path/to/aidlc-linear/skills/create-unit-tasks .opencode/skills/create-unit-tasks

# Global
cp -r /path/to/aidlc-linear/skills/create-unit-tasks ~/.config/opencode/skills/create-unit-tasks
```

Or clone the repo and symlink:

```bash
git clone https://github.com/jalanya/aidlc-linear.git ~/.local/share/aidlc-linear
ln -s ~/.local/share/aidlc-linear/skills/create-unit-tasks .opencode/skills/create-unit-tasks
```

## Usage

```
/aidlc-linear:create-unit-tasks <unit-name> in <team-name>
/aidlc-linear:create-unit-tasks <unit-name> in <team-name> --dry-run
```

### Examples

```
/aidlc-linear:create-unit-tasks shared-foundation in aura-core
/aidlc-linear:create-unit-tasks core-api in aura-core --dry-run
/aidlc-linear:create-unit-tasks agent in my-team
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `unit-name` | Yes | Unit name matching the plan filename (e.g., `shared-foundation`, `core-api`, `agent`) |
| `team-name` | Yes | Linear team name to create tasks in |
| `--dry-run` | No | Preview what would be created without calling Linear write APIs |

## Prerequisites

1. **AI-DLC project structure** with:
   - Code generation plan at `aidlc-docs/construction/plans/{unit-name}-code-generation-plan.md`
   - User stories at `aidlc-docs/inception/user-stories/stories.md`
   - Guide at `aidlc-docs/guides/code-gen-plan-to-linear-tasks.md`
   - Design documents at `aidlc-docs/construction/{unit-name}/{functional,nfr,infrastructure}-design/`

2. **Linear MCP server** configured in your Claude Code or Opencode setup

## How It Works

The skill follows a 13-step process:

### Analysis Phase (Steps 1-8)

1. **Validate inputs** — parse unit name, team name, locate plan and design documents
2. **Parse the code generation plan** — extract all steps, files, story references
3. **Derive task prefix** — 2-letter prefix from unit name initials (e.g., SF, CA, AG)
4. **Group steps into stories** — cluster by functional cohesion using a granularity decision matrix
5. **Decompose stories into atomic tasks** — apply splitting rules and collision avoidance rules
6. **Define dependencies & parallel groups** — assign sequencing phases, identify critical path
7. **Write task descriptions** — populate full template from design documents
8. **Validate decomposition** — run Symphony Readiness Checklist (pre-flight check)

### Linear Creation Phase (Steps 9-13)

9. **Create epic project** — one Linear project per unit
10. **Create story issues** — parent issues for each functional group
11. **Create task issues** — sub-issues with full descriptions under their parent story
12. **Set dependencies** — blocked-by relations between tasks
13. **Summary report** — parallel execution phases, critical path, Symphony readiness

### Collision Avoidance

The skill enforces language-agnostic rules to prevent concurrent agents from modifying the same file:

| Collision Type | Rule |
|----------------|------|
| Module index / barrel files | Assigned to the LAST task in the module |
| Dependency manifests | One task owns the manifest; others document deps |
| Shared config files | Assigned to a single scaffolding task |
| Test fixtures / setup | One task creates shared setup; consumers declare dependency |
| Shared type definitions | One task owns the types file; consumers import and declare dependency |

## Task Description Template

Each task includes a comprehensive template designed for agent consumption:

- **Context** — Epic, story, plan step reference
- **Objective** — What this task produces and why
- **Scope** — Explicit in-scope and out-of-scope with cross-references to other tasks
- **Design References** — Links to specific design document sections
- **Files to Create/Modify** — Exact paths with action (CREATE/MODIFY) and description
- **Technical Specifications** — Dependencies, implementation details, constraints
- **Dependencies** — Blocked by, blocks, parallel group assignment
- **Validation Matrix** — Unit, integration, smoke, and E2E test applicability with commands
- **Acceptance Criteria** — Machine-verifiable criteria (lint, type check, test commands)
- **Definition of Done** — Production-readiness checklist
- **Traceability** — Links back to user stories, requirements, design docs, and plan steps

## Required MCP Tools

- `mcp__linear-server__list_teams`
- `mcp__linear-server__list_issues`
- `mcp__linear-server__list_projects`
- `mcp__linear-server__save_issue`
- `mcp__linear-server__save_project`

## License

MIT
