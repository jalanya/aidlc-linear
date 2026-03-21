---
name: create-unit-tasks
description: Converts AI-DLC code generation plans into a structured Linear task hierarchy (Epic → Stories → Tasks) optimized for parallel AI agent execution via Symphony
---

# Create Unit Tasks

Converts an AI-DLC code generation plan into a structured hierarchy of Linear issues — **Epic (Project) → Stories (Parent Issues) → Tasks (Sub-Issues)** — optimized for parallel AI agent execution via Symphony or similar orchestrators.

Each task is self-contained with full context, explicit scope boundaries, a validation matrix, and traceability. Every task passes the **"cold start" test**: an agent with zero prior context can read only the task description and produce the correct output without asking questions.

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
- **unit-name** (required): The unit name matching the plan filename (e.g., `shared-foundation`, `core-api`, `agent`)
- **team-name** (required): The Linear team name (e.g., `aura-core`)
- **--dry-run** (optional flag): Preview what would be created without making any Linear write API calls

---

## Execution Steps

### Step 1: Validate Inputs & Load Context

1. Parse `$ARGUMENTS` for `unit-name`, `team-name`, and `--dry-run` flag
2. If either is missing, show usage examples and ask the user to provide the missing values
3. Locate and read the following files:
   - **Code generation plan**: `aidlc-docs/construction/plans/{unit-name}-code-generation-plan.md`
   - **User stories**: `aidlc-docs/inception/user-stories/stories.md`
   - **Guide** (reference methodology): `aidlc-docs/guides/code-gen-plan-to-linear-tasks.md`
4. If the plan file does not exist, list available plans and ask the user to choose one
5. Read the unit's design documents (all that exist):
   - `aidlc-docs/construction/{unit-name}/functional-design/` — all `.md` files
   - `aidlc-docs/construction/{unit-name}/nfr-design/` — all `.md` files
   - `aidlc-docs/construction/{unit-name}/infrastructure-design/` — all `.md` files
6. Resolve the Linear team via `mcp__linear-server__list_teams`. If not found, list available teams and ask the user to choose
7. Extract the **Unit Context** from the plan: unit number, unit type, stories, dependencies, consumers, workspace root

### Step 2: Parse the Code Generation Plan

1. Read the plan file completely
2. Extract the **Source Artifacts** section (design doc paths)
3. Extract all **Steps** from the Execution Plan. For each step, capture:
   - Step number and title
   - All checkbox items (files to create/modify, with exact paths)
   - Story references (from `Stories:` lines)
   - File count per step
4. Count total steps and total files across the entire plan

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

**Rule**: Take the first letter of each hyphen-separated word, uppercase. If only one word, take first two letters.

### Step 4: Group Steps into Stories

Analyze plan steps and group them by **functional cohesion** — files that must exist together to be testable or meaningful.

**Grouping heuristic**: Steps that produce files in the same module/package AND are needed together for a test to pass belong in the same story.

**Granularity Decision Matrix** — use this to decide whether to split or merge steps:

| Signal | Split into finer tasks | Merge into coarser tasks |
|--------|------------------------|--------------------------|
| File count per step | >10 files | ≤5 files |
| Module coupling | Low coupling between files | High coupling, files co-depend |
| Test isolation | Files independently testable | Files only testable together |
| Review complexity | Mixed concerns (models + routes) | Single concern (all models) |
| Agent collision risk | Multiple agents might touch same file | Single-agent file ownership |

Each story gets:
- **Name**: `{PREFIX}: {Functional Group Name}` (e.g., `SF: Database Engine & Models`)
- **Plan steps**: Which steps from the plan are covered
- **Rationale**: Why these steps belong together

Present the story grouping to the user as a table before proceeding.

### Step 5: Decompose Stories into Atomic Tasks

Each story breaks into **atomic tasks** — the unit of work an agent picks up and completes in one focused session (roughly 1-20 files).

**Splitting rules**:
1. **One concern per task**: Don't mix model definitions with repository logic
2. **Testable boundary**: Each task produces something verifiable (compiles, passes lint, tests pass)
3. **File-level granularity**: List every file the agent must create or modify
4. **No circular dependencies** between tasks in the same story

**Collision avoidance rules** (critical for parallel execution):

**Core rule**: No two concurrent tasks may create or modify the same file.

| Collision Type | Rule | Examples by Language |
|----------------|------|---------------------|
| **Module index / barrel files** | Assign to the LAST task in the module. Prior tasks create their files but do NOT update the index. | Python: `__init__.py`; TypeScript/JS: `index.ts`; Go: package-level exports; Rust: `mod.rs`; Java: package-info |
| **Dependency manifest** | One task owns the manifest. Other tasks document their deps in the description; the owner task aggregates. | Python: `pyproject.toml`, `requirements.txt`; Node: `package.json`; Go: `go.mod`; Rust: `Cargo.toml`; Java: `pom.xml`, `build.gradle` |
| **Shared config files** | Assign to a single scaffolding task that runs first. | Linter configs, formatter configs, CI base configs, `.env.example` |
| **Test fixtures / setup** | One task creates the shared test setup. Subsequent test tasks declare dependency on it. | Python: `conftest.py`; JS: `jest.setup.ts`, `testUtils.ts`; Go: `testmain_test.go`; Java: `TestBase.java` |
| **Shared type definitions** | One task owns the types file. Consumers import from it and declare the dependency. | TypeScript: `types.ts`; Go: `types.go`; Rust: `types.rs`; Java: shared interfaces/enums |

Each task gets a sequential number: `{PREFIX}-{NN}` (e.g., SF-01, SF-02, ...).

### Step 6: Define Dependencies & Parallel Groups

For each task, determine:
- **blocked_by**: Tasks that must complete before this one starts
- **blocks**: Tasks that cannot start until this one is done
- **parallel_group**: Name of the group of tasks safe to run concurrently (or `N/A`)

Assign tasks to **sequencing phases**:

```
Phase 1 (Sequential):    Scaffolding — must run first
Phase 2 (Parallel A):    Core modules — independent of each other
Phase 3 (Sequential):    Integration modules — depend on core
Phase 4 (Parallel B):    Tests — run after implementation
Phase 5 (Parallel C):    Infrastructure — parallel with app code
Phase 6 (Parallel D):    Automation & docs — run last
```

Identify the **critical path** — the longest chain determines minimum wall-clock time.

### Step 7: Write Task Descriptions

For each task, generate a description using the **full template** below. Populate it by reading the unit's design documents (loaded in Step 1).

````markdown
## Context

**Epic**: Unit {N}: {Unit Name}
**Story**: {Parent story name}
**Code Gen Plan Step(s)**: Step {N}
**Unit**: {Unit Name}
**Plan Reference**: `aidlc-docs/construction/plans/{unit}-code-generation-plan.md` → Step {N}

## Objective

{One sentence: what this task produces and why it exists.}

## Scope

### In Scope
- {What this task MUST deliver — be explicit}
- {Specific modules, files, or behaviors}

### Out of Scope
- {What this task must NOT touch — prevents agent drift}
- {Adjacent work that belongs to another task: "Repository layer — see {PREFIX}-{NN}"}

## Design References

{Links to specific design documents and sections this task implements. Include only directories that exist for this unit. Be precise — section or heading level.}

- Functional Design: `aidlc-docs/construction/{unit}/functional-design/{file}.md` → Section "{heading}"
- NFR Design: `aidlc-docs/construction/{unit}/nfr-design/{file}.md` → Section "{heading}"
- Infrastructure Design: `aidlc-docs/construction/{unit}/infrastructure-design/{file}.md` → Section "{heading}"

## Files to Create/Modify

| Action | File Path | Description |
|--------|-----------|-------------|
| CREATE | `{exact/path/from/repo/root}` | {What this file contains} |
| MODIFY | `{exact/path/from/repo/root}` | {What changes to make} |

## Technical Specifications

### Dependencies (imports / packages)
{Exact package names and versions the agent should use.}

### Key Implementation Details
- {Class/function signatures with type hints}
- {Design patterns to use (e.g., "Repository pattern with BaseRepository[T]")}
- {Specific libraries/APIs to call}
- {Error handling approach}
- {Naming conventions}

### Constraints
- {Hard constraints: "Must use async/await throughout"}
- {Security: "Never log secrets", "Validate all external input"}
- {NFR: "Connection pool size = 1 per Lambda", "Use TLS for DB connections"}

## Dependencies

**Blocked by**: {Task IDs and titles, or "None — first task"}
**Blocks**: {Task IDs and titles}
**Parallel group**: {Group name, or "N/A" if sequential}

## Validation

### Validation Matrix

| Level | Applicable | Test Location | Run Command |
|-------|-----------|---------------|-------------|
| Unit | {Yes/No} | `{path}` | `{command}` |
| Integration | {Yes/No} | `{path}` | `{command}` |
| Smoke | {Yes/No} | N/A | `{command}` |
| E2E | {Yes/No} | `{path}` | `{command}` |

> **Agent instruction**: Generate ALL applicable test levels. If a level is marked "Yes", the task is NOT done until those tests exist and pass. If "No", skip entirely — do not create placeholder test files.

## Acceptance Criteria

- [ ] All files listed in "Files to Create/Modify" exist at the specified paths
- [ ] {Specific functional criterion — machine-verifiable}
- [ ] Code passes `ruff check` with zero errors
- [ ] Code passes `mypy --strict` with zero errors
- [ ] {Test criterion: "All tests in `{path}` pass"}
- [ ] No secrets or credentials in committed code

## Definition of Done

- [ ] All acceptance criteria met
- [ ] Files match the design references
- [ ] No `TODO`, `FIXME`, or placeholder implementations
- [ ] Imports resolve correctly (no circular imports)
- [ ] Code is production-ready, not scaffolding

## Traceability

| Artifact | Reference |
|----------|-----------|
| User Stories | {Story IDs, or "N/A — enabling infrastructure"} |
| Requirements | `aidlc-docs/inception/requirements/{file}` → {section} |
| Design | {Design doc links listed above} |
| Plan Step | Step {N} in `{plan-file-path}` |
| Code Output | {File paths listed above} |
````

**Key rules for populating the template**:
- Read the unit's functional design to fill **Technical Specifications** (class signatures, patterns, constraints)
- Read the unit's NFR design to fill **Constraints** and **Validation** sections
- Read the unit's infrastructure design for Terraform/infra tasks
- **Acceptance criteria must be machine-verifiable** — lint commands, test commands, import checks. Never "code quality is good"
- **Out of Scope** must cross-reference other task IDs to prevent agent drift

### Step 8: Validate Decomposition (Pre-Flight Check)

Before creating anything in Linear, run the **Symphony Readiness Checklist**:

- [ ] Every file in the code generation plan is covered by exactly one task
- [ ] No task references a file created by another task without declaring the dependency
- [ ] Every task has at least one acceptance criterion that can be machine-verified
- [ ] No task requires more than 20 files to be created/modified
- [ ] The dependency graph has no cycles
- [ ] Tasks in the same parallel group share no output files
- [ ] Each task's description is self-contained (passes the "cold start" test)

Present any violations to the user. **Do not proceed until all checks pass or the user acknowledges exceptions.**

---

## Linear Creation Phase

> **Dry-run mode**: If `--dry-run` is active, present everything below as a preview (what WOULD be created) without making any write API calls. After the preview, ask: "Ready to create these in Linear? (yes/no)". If confirmed, re-execute the creation steps with actual API calls.

### Pre-Execution Warning

Before any write operations, show:

```
About to create in Linear:

  Team:       {team-name}
  Epic:       Unit {N}: {Unit Name}
  Stories:    {count}
  Tasks:      {count} (across {phase-count} parallel phases)
  Relations:  {dependency-count} blocked-by links

Continue? (yes/no)
```

Wait for explicit confirmation before proceeding.

### Step 9: Create the Epic Project

1. List existing projects using `mcp__linear-server__list_projects`
2. If a project named `Unit {N}: {Unit Name}` already exists, reuse it
3. Otherwise, create it using `mcp__linear-server__save_project` with:
   - Name: `Unit {N}: {Unit Name}`
   - Description: The Unit Context section from the plan

### Step 10: Create Story Issues

For each story group from Step 4, create a Linear issue:

- **Title**: `{PREFIX}: {Story Name}` (e.g., `SF: Database Engine & Models`)
- **Project**: The epic project from Step 9
- **Priority**: High (2)
- **Description**:
  ```
  ## {Story Name}

  **Plan Steps**: {step numbers}
  **Rationale**: {why these steps are grouped}

  ### Tasks in this Story
  - {PREFIX}-{NN}: {Task Title}
  - {PREFIX}-{NN}: {Task Title}
  - ...
  ```

Build a map of `story name → issue ID` for setting parent references.

### Step 11: Create Task Issues

For each task from Step 5, create a Linear issue:

- **Title**: `{PREFIX}-{NN}: {Task Title}`
- **Parent**: The story issue from Step 10 (using `parentId`)
- **Project**: The epic project from Step 9
- **Priority**: Based on position in the dependency graph:
  - Critical path / no dependencies: `Urgent` (1)
  - Parallel group tasks: `High` (2)
  - Test tasks: `Normal` (3)
  - Documentation tasks: `Low` (4)
- **Description**: The full template from Step 7

### Step 12: Set Dependencies

After ALL issues are created:

1. Build a map of `{PREFIX}-{NN} → Linear issue ID`
2. For each task, parse its Dependencies section to find `blocked_by` references
3. Use `mcp__linear-server__save_issue` to set blocking relations
4. Dependency rules:
   - Steps referencing earlier steps → blocked by them
   - Test tasks → depend on the implementation tasks they test
   - Documentation tasks → depend on all other tasks
   - Infrastructure/Terraform tasks → can run in parallel with application code (unless they share outputs)

### Step 13: Summary Report

Present the final summary:

```
## Linear Tasks Created for Unit {N}: {Unit Name}

**Team**: {team-name}
**Epic Project**: {project name}
**Stories**: {count}
**Tasks**: {count}
**Dependencies**: {count} blocked-by relations

### Stories & Tasks

#### {Story Name} (Steps {N, N, N})
| Task | Title | Size | Parallel Group | Blocked By |
|------|-------|------|---------------|------------|
| {PREFIX}-{NN} | {title} | {xs/s/m/l} | {group or —} | {deps or —} |

#### ...

### Parallel Execution Phases

Phase 1 (Sequential):    {task list}
Phase 2 (Parallel A):    {task list}
Phase 3 (Sequential):    {task list}
Phase 4 (Parallel B):    {task list}
...

### Critical Path
{PREFIX}-{NN} → {PREFIX}-{NN} → ... → {PREFIX}-{NN}
Minimum phases: {count}

### Symphony Readiness: PASS/FAIL
{Checklist results from Step 9}
```

---

## Requirements

This skill requires access to the following MCP tools:
- `mcp__linear-server__list_teams`
- `mcp__linear-server__list_issues`
- `mcp__linear-server__list_projects`
- `mcp__linear-server__save_issue`
- `mcp__linear-server__save_project`

The project must follow the AI-DLC directory structure with:
- Code generation plans at `aidlc-docs/construction/plans/{unit-name}-code-generation-plan.md`
- User stories at `aidlc-docs/inception/user-stories/stories.md`
- Execution guide at `aidlc-docs/guides/code-gen-plan-to-linear-tasks.md`
- Design documents at `aidlc-docs/construction/{unit-name}/{functional,nfr,infrastructure}-design/`
