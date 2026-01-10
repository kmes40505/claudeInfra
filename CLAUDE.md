# CLAUDE.md

## Definitions

- **Concept**: A distinct idea, component, or behavior described in a README. Each concept has a name, description, and relations to other concepts.
- **Upstream**: Agent/phase whose outputs are consumed by another. Example: Planning is upstream of Coding because Coding reads what Planning produces.
- **Downstream**: Agent/phase that consumes outputs from upstream.
- **Flavor**: A variant of the implementation (e.g., platform, build config, environment).

## Flavor Marks

Marks used in requirement checklists to track which flavors have implemented a requirement.

**Location**: Active marks are defined in `project/projectDescription/README.md` based on project description.

**Format**: `{mark}: {flavor_name}`

**Example**:
- w: windows
- l: linux

**Validation**: Before any Coding Phase task, verify flavor marks are defined. If not set or invalid, warn and fail the command. Do not proceed without valid flavor marks.

## Directory Structure

```
root/
├── CLAUDE.md
├── project/
│   ├── projectDescription/
│   ├── code/{flavor}/
│   └── tests/{flavor}/
```

- `CLAUDE.md`: This file. Global configuration and rules.
- `project/projectDescription/`: Documentation (.md files) and shared resources (JSON, images, assets, etc.).
- `project/code/{flavor}/`: Generated implementations and copied shared resources, one folder per flavor.
- `project/tests/{flavor}/`: Test cases organized by flavor, mirroring `projectDescription/` paths.

## Cross-Flavor Documentation

All .md files in `projectDescription/` must be generic and cross-flavor. Do not include flavor-specific details. Flavor-specific implementation belongs in code only.

## Test Structure

Tests mirror `projectDescription/` paths, organized by flavor.

**Unit tests**: Mirror individual {class/function}.md files
- `project/tests/{flavor}/{path}/{File}.test.{ext}`

**Flow tests**: Test code flow/UX flow at directory level
- `project/tests/{flavor}/{path}/flow.test.{ext}`

**Integrity tests**: Test that all code in a directory (including subfolders) integrates correctly
- `project/tests/{flavor}/{path}/integrity.test.{ext}`
- What to test depends on what the code does

**Example:**
- Documentation: `project/projectDescription/auth/login/UserAuth.md`
- Unit test: `project/tests/windows/auth/login/UserAuth.test.cpp`
- Flow test: `project/tests/windows/auth/login/flow.test.cpp` (tests login flow)

## Path Mirroring

Documentation location determines implementation location.

**Rule**: All generated code MUST be written only to its corresponding mirror path. No exceptions.

**How to calculate:**
1. Take documentation path: `project/projectDescription/{path}/{File}.md`
2. Extract relative path: `{path}/`
3. Mirror to: `project/code/{flavor}/{path}/{File}.{ext}`

**Example:**
- Documentation: `project/projectDescription/auth/login/UserAuth.md`
- Relative path: `auth/login/`
- Code output: `project/code/windows/auth/login/UserAuth.cpp`

## README.md Structure

Each folder in `projectDescription/` contains a README.md. All content in README.md is organized as concept entries.

### Concept Entry Format

```markdown
## Concept: {Name}
{Description at this abstraction level only}

- Related (Active): {Path}/README.md: {ConceptName}
- Related (Passive): {Path}/README.md: {ConceptName} [{keyword1}, {keyword2}]
- Looked Up By: {Path}/README.md
```

**Fields:**
- **Description**: What this concept is, at current level. For more detail, follow relations.
- **Related (Active)**: Always read this referenced concept to understand the current concept.
- **Related (Passive)**: Only read if keywords match what you're looking for.
- **Looked Up By**: Which README files reference this concept. Used for change propagation.

### Code Dependencies Section

At the end of each README.md, list which code definition files depend on which concepts:

```markdown
## Code Dependencies
- {ClassName}.md: {ConceptA}, {ConceptB}
- {functionName}.md: {Path}/README.md: {ConceptC}
```

**Can reference:**
- Concepts in same README: just the concept name
- Concepts in other READMEs: `{Path}/README.md: {ConceptName}`

### Integrity Tests Section

At the end of README.md, if directory has components that integrate:

```markdown
## Integrity Tests
- {integration test case 1}:
- {integration test case 2}: {mark}
```

## {class/function}.md Structure

Each class or function has its own .md file in `projectDescription/`.

```markdown
## Description
{What this class/function does}

## Signature
```
{Generic signature that works across all flavors}
```

## Requirements
- signature:
- {requirement1}:
- {requirement2}: {mark}
- {requirement3}: {mark}, {mark}
```

**Rules:**
- `signature` is always a required checklist item
- Format: `- {requirement}: {marks}`
- Empty after colon means no flavor has implemented yet
- Marks must match Active Marks defined in this file
- If a requirement references a custom class, use the actual class name from another .md file

## Phases

### Planning Phase

**What**: Creating and modifying .md files in `projectDescription/`.

**Tasks:**
1. Understand project requirements
2. Create README.md in each folder with concept entries
3. Create {class/function}.md files with description, signature, and requirements
4. If design needs more detail, create subfolder and repeat
5. Maintain concept relations and Looked Up By entries
6. Generate shared resources (JSON, images, assets) in `projectDescription/`

**Outputs**: .md files and shared resources in `projectDescription/`

### Coding Phase

**What**: Generating code in `code/{flavor}/` based on .md files.

**Tasks:**
1. Read the {class/function}.md file
2. Follow Code Dependencies to understand required concepts
3. Generate code to mirror path: `project/code/{flavor}/{path}/`
4. Copy shared resources from `projectDescription/` to mirrored path in `code/{flavor}/`
5. Generate test cases in `project/tests/{flavor}/`
6. Read README.md for Integrity Tests section
7. Implement integrity tests in `project/tests/{flavor}/{path}/integrity.test.{ext}`
8. Run integrity tests - fix issues before proceeding

**Outputs**: Code files and copied resources in `code/{flavor}/`, test cases in `tests/{flavor}/`

## Requirement Verification

Coding agents implement and test. Verification agent (or main agent) marks requirements.

### Verification Rules

Before marking a requirement:
1. Implementation exists at correct mirror path
2. Corresponding test exists
3. Test passes

If any condition fails, do not mark the requirement.

### Verification Process

1. Coding agent implements code and creates tests
2. Coding agent runs tests
3. Verification agent (or main agent) confirms test results
4. Only verification agent marks requirements in {class/function}.md

Coding agents must NOT mark requirements themselves. Report completion to main/verification agent instead.

## Concept Tracking

### When You Add a Relation

If concept A references concept B:
1. Go to the README.md containing concept B
2. Add your README path to concept B's `Looked Up By` list

This enables change propagation.

### When You Modify a Concept

Run change propagation:

1. Check the concept's `Looked Up By` list
2. For each README path listed:
   - Open that README
   - Find the concept with the same name (it references this one)
   - Verify logic is still valid, modify if needed
3. Check `Code Dependencies` in your README
4. For each {class/function}.md that depends on this concept:
   - **Remove marks only for affected requirements** - only those implementations need re-verification
5. In Coding Phase: re-verify and re-mark requirements after updating code

## Agent Limits

Maximum 8 agents running concurrently to avoid context over-consumption. Wait for agents to complete before spawning new ones if limit is reached.

## Agent Spawning

Sub-agents can be used for both Planning and Coding Phase tasks.

When spawning sub-agents, include in the task prompt:
- "Read CLAUDE.md first before starting work"

This ensures sub-agents follow project rules.

### Dependency Handling

**Planning Phase**: Directory hierarchy determines dependency order. Complete parent directories before child directories.

**Coding Phase**: Code Dependencies in README.md determines dependency order. Complete dependencies before dependents.

- **Independent tasks**: Can spawn sub-agents in parallel
- **Dependent tasks**: Wait for dependency to complete before spawning dependent task

### Shared File Updates

When sub-agent needs to update a file outside its scope:
- Append request to AgentTalk.md in the directory under `## Updates` section
- Sub-agents only append, never modify existing entries
- Main agent processes requests and cleans up AgentTalk.md

### Planning Verification

Planning quality is verified after Coding Phase completes. If flow tests fail, the issue may be in Planning (requirements/design) or Coding (implementation). Use AgentTalk.md to report issues upstream if Planning needs revision.

## Agent Communication

### AgentTalk.md

Agents report issues to upstream agents. Upstream agents address issues by modifying their outputs.

**Location**: Create AgentTalk.md in the directory where the issue occurs. Issues stay localized to their context.

**Key rule**: AgentTalk.md is for reporting problems only. Never put solutions or fix instructions in AgentTalk.md.

Agents fix problems by following their phase's tasks as defined in the Phases section. Do not bypass the normal workflow.

**Example flow for a bug:**
1. Bug identified (by user, test failure, or other means)
2. Planning agent modifies requirements in {class/function}.md, removes affected marks
3. Coding agent sees updated requirements, follows normal coding process
4. Verification agent marks requirements after tests pass

### Upstream-First Principle

Fix issues at their source. Update upstream phase outputs so downstream phases handle the fix through their normal workflow.

**Example:** Bug found → Planning Phase adds requirement to .md file → Coding Phase generates test and fix → verification marks complete

### When to Use

- Agent encounters an issue it cannot resolve at current phase
- Issue requires upstream phase to modify its outputs

**Example**: Coding agent finds a function is too complex → writes issue to AgentTalk.md → Planning agent splits the function in the .md files → Coding agent regenerates from updated .md files.

### Format

```markdown
## Updates

### {Short description}
- **From**: {agent identifier}
- **File**: {path to file to update}
- **Action**: {what update is needed}

## Open

### {Short description}
- **From**: {phase or agent identifier}
- **To**: {upstream phase or agent identifier}
- **File**: {path to relevant file}
- **Issue**: {description of the problem, not the solution}

## Resolved

### {Short description}
- **Addressed in**: {path to modified file(s)}
```

### Cleanup Process

1. Upstream agent addresses issue by modifying outputs
2. Upstream agent moves issue from `Open` to `Resolved`, noting which files were modified
3. Downstream agent continues work based on updated outputs
4. Once downstream agent confirms success, delete the entry from `Resolved`
5. Goal: AgentTalk.md should be empty when all issues are addressed
