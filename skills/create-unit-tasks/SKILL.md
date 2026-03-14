---
name: create-unit-tasks
description: Creates Linear tasks from an AI-DLC code generation plan, organized by deliverable-based projects with full context for agent delegation
---

# Create Unit Tasks

Converts an AI-DLC code generation plan into Linear tasks organized by deliverable-based projects. Each task includes full context (objective, design references, files, dependencies, patterns, stories, acceptance criteria) so it can be delegated to an agent team.

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

## Arguments

Parse `$ARGUMENTS` to extract:
- **unit-name** (required): The unit name as it appears in the plan filename (e.g., `shared-foundation`, `core-api`, `agent`)
- **team-name** (required): The Linear team name to create tasks in (e.g., `aura-core`)
- **--dry-run** (optional flag): Preview what would be created without making any Linear API calls

## Execution Steps

### Step 1: Validate Inputs

1. Parse `$ARGUMENTS` to extract `unit-name`, `team-name`, and `--dry-run` flag
2. If either `unit-name` or `team-name` is missing, show usage examples and ask the user to provide the missing values
3. Locate the code generation plan file at: `aidlc-docs/construction/plans/{unit-name}-code-generation-plan.md`
4. If the plan file does not exist, list available plans and ask the user to choose one
5. Locate the user stories file at: `aidlc-docs/inception/user-stories/stories.md`
6. Look up the Linear team by name using `mcp__linear-server__list_teams` and find the matching team. If not found, list available teams and ask the user to choose

### Step 2: Parse the Code Generation Plan

1. Read the plan file completely
2. Extract the **Unit Context** section: unit type, stories, dependencies, consumers
3. Extract the **Deliverables** section (if present) — these become project grouping hints
4. Extract all **Steps** from the Execution Plan. For each step, capture:
   - Step number and title
   - All checkbox items (files to create)
   - Stories references (from `**Stories**:` lines)
   - Any dependency hints (from step ordering and naming)

### Step 3: Derive Task Prefix

Generate a 2-letter prefix from the unit name initials:

| Unit Name | Prefix |
|-----------|--------|
| shared-foundation | SF |
| core-api | CA |
| agent | AG |
| slack-bot | SB |
| scheduler | SC |
| frontend | FE |

Rule: Take the first letter of each hyphen-separated word, uppercase. If only one word, take first two letters.

### Step 4: Auto-Infer Project Grouping

Group steps into deliverable-based projects using keyword matching on step titles and content:

| Keywords in Step Title/Content | Project Suffix |
|-------------------------------|----------------|
| test, tests, unit test, conftest | Unit Tests |
| terraform, infra, module, tfvars | Terraform Infrastructure |
| CI/CD, workflow, github actions, deploy | CI/CD Pipelines |
| migration, alembic, schema | Database Migrations |
| documentation, summary, code-summary | Documentation |
| docker, makefile, .env, pre-commit, development environment | Development Environment |
| _(default — application code)_ | Application Code |

Project names follow the pattern: `{Unit Display Name}: {Project Suffix}`
Example: `Shared Foundation: Application Code`, `Core API: Unit Tests`

If the plan has an explicit **Deliverables** section, prefer those names over auto-inference.

### Step 5: Read User Stories

1. Read `aidlc-docs/inception/user-stories/stories.md`
2. Parse all story headings to build a lookup map of `Story ID → Story Title`
   - Pattern: `## STORY-NNN: Title` → e.g., `PARK-001` → `Check Parking Availability`
3. This map is used for:
   - Creating labels with descriptive names (Step 6)
   - Adding story references in task descriptions (Step 8)

### Step 6: Create Story Labels (Check-and-Create-if-Missing)

1. List all existing labels in the team using `mcp__linear-server__list_issue_labels`
2. Collect all unique story IDs referenced across ALL steps in the plan
3. For each story ID that does NOT already exist as a label:
   - Create it using `mcp__linear-server__create_issue_label`
   - Name format: `{STORY-ID}` (e.g., `PARK-001`)
   - Use a consistent color per epic group:
     - PARK-*: `#4EA8DE` (blue)
     - ADMIN-*: `#9B59B6` (purple)
     - AGENT-*: `#2ECC71` (green)
     - AUTH-*: `#E67E22` (orange)
     - SLACK-*: `#3498DB` (light blue)
     - ERR-*: `#E74C3C` (red)
4. If the plan references "all stories" or "enabling infrastructure", create labels for ALL story IDs found in the stories file
5. Build a map of `label name → label ID` for use in Step 8

### Step 7: Create Projects

1. List existing projects in the team using `mcp__linear-server__list_projects`
2. For each project group identified in Step 4:
   - If a project with that name already exists, reuse it
   - Otherwise create it using `mcp__linear-server__save_project`
3. Build a map of `project name → project ID` for use in Step 8

### Step 8: Create Tasks with Full Context

For each step in the plan, create a Linear issue using `mcp__linear-server__save_issue`:

**Title**: `{PREFIX}-{NN}: {Step Title}`
Example: `SF-01: Project Structure Setup`, `CA-03: Parking Router & Service`

**Priority**: Derive from step position and dependencies:
- Steps with no dependencies (can start immediately): `High` (priority: 2)
- First step / critical path: `Urgent` (priority: 1)
- Test steps: `Normal` (priority: 3)
- Documentation steps: `Low` (priority: 4)

**Project**: Assign to the project determined in Step 4

**Labels**: Apply all story ID labels referenced in the step. When referencing stories in the description, use the format: `{STORY-ID} — {Story Title}` (em dash, no parentheses)

**Description**: Use this markdown template:

```markdown
## Objective
[Derived from the step title and context — 2-3 sentences describing what to build]

## Design References
Read these artifacts before starting:
- `aidlc-docs/construction/{unit-name}/functional-design/[relevant files]`
- `aidlc-docs/construction/{unit-name}/nfr-design/[relevant files]`
- `aidlc-docs/construction/{unit-name}/infrastructure-design/[relevant files]`

[Include only the design reference directories that exist for this unit. Check the plan's "Source Artifacts" section.]

## Files to Create
- `path/to/file1.py` — [brief description from checkbox item]
- `path/to/file2.py` — [brief description from checkbox item]

[Copy directly from the plan's checkbox items for this step]

## Dependencies
- Blocked by: {PREFIX}-{NN} — {title} (must complete first)
- Blocks: {PREFIX}-{NN} — {title}

[Derive from step ordering: later steps typically depend on earlier ones. Use the plan's explicit dependency hints when available. Reference by task prefix-number, NOT Linear issue IDs — those are set separately via the API.]

## Key Patterns & Decisions
- [Relevant NFR design pattern, if referenced in the plan]
- [Relevant tech stack decision]

[Pull from the plan's step content and the unit's NFR design artifacts]

## Stories Enabled
- {STORY-ID} — {Story Title}
- {STORY-ID} — {Story Title}

[List all stories referenced in this step, using the story lookup map from Step 5. If the step says "enabling infrastructure" with no specific stories, note that.]

## Acceptance Criteria
- [ ] All files listed above are created and syntactically valid
- [ ] Code follows patterns described in design references
- [ ] [Step-specific criteria derived from the plan content]
- [ ] All files pass linting (ruff) and type checking (mypy)
```

### Step 9: Set Task Dependencies in Linear

After ALL tasks are created:
1. Build a map of `{PREFIX}-{NN} → Linear issue ID`
2. For each task, parse its Dependencies section to find which tasks it depends on
3. Use `mcp__linear-server__save_issue` to set the `parentId` or relation, linking dependent tasks
4. Dependency rules:
   - Steps that reference earlier steps are blocked by them
   - Test steps depend on the implementation steps they test
   - Documentation steps depend on all other steps
   - Infrastructure/Terraform steps can often run in parallel with application code

### Step 10: Summary Report

After all tasks are created, present a summary:

```
## Linear Tasks Created for {Unit Display Name}

**Team**: {team-name}
**Tasks**: {count} created
**Projects**: {list of project names}
**Labels**: {count} created / {count} reused

### Tasks by Project

#### {Project Name}
- {PREFIX}-{NN}: {Title} (Priority: {priority}) [{story labels}]
- ...

### Dependencies Set
- {PREFIX}-{NN} → blocks → {PREFIX}-{NN}
- ...
```

## Dry-Run Mode

When `--dry-run` is passed:
- Execute Steps 1-5 (read and parse everything)
- For Steps 6-9, show what WOULD be created but do NOT call any Linear write APIs
- Present the same summary report with a `[DRY RUN]` banner
- Ask: "Ready to create these tasks in Linear? (yes/no)"
- If user confirms, re-execute Steps 6-9 with actual API calls

## Warning Before Execution

Before creating any tasks (Step 6), always warn the user:

```
⚠️  About to create tasks in Linear:
- Team: {team-name}
- Unit: {unit-name} ({step-count} steps → {task-count} tasks)
- Projects: {project-count} projects
- Labels: {new-label-count} new / {existing-label-count} existing

Continue? (yes/no)
```

Wait for confirmation before proceeding.

## Requirements

This skill requires access to the following MCP tools:
- `mcp__linear-server__list_teams`
- `mcp__linear-server__list_issues`
- `mcp__linear-server__list_issue_labels`
- `mcp__linear-server__list_projects`
- `mcp__linear-server__create_issue_label`
- `mcp__linear-server__save_issue`
- `mcp__linear-server__save_project`

The project must follow the AI-DLC directory structure with:
- Code generation plans at `aidlc-docs/construction/plans/{unit-name}-code-generation-plan.md`
- User stories at `aidlc-docs/inception/user-stories/stories.md`
