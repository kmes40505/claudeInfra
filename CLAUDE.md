# CLAUDE.md

## Compliance

All agents (including main agent) must follow every rule in this file without exception. No shortcuts, even for "simple" tasks. If a workflow is defined (bug fixes, requirements, verification), follow it completely.

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

All .md content in `projectDescription/` must be generic and cross-flavor. If anything is condition-specific, annotate inline (e.g., `[windows only]`, `[requires admin]`).

## Test Structure

Tests mirror `projectDescription/` paths, organized by flavor.

**Unit tests**: Mirror individual {class/function}.md files
- `project/tests/{flavor}/{path}/{File}.test.{ext}`

**Flow tests**: Test code flow/UX flow at directory level
- `project/tests/{flavor}/{path}/flow.test.{ext}`

**Integration tests**: Test that all code in a directory (including subfolders) integrates correctly
- `project/tests/{flavor}/{path}/integration.test.{ext}`
- What to test depends on what the code does

**Example:**
- Documentation: `project/projectDescription/auth/login/UserAuth.md`
- Unit test: `project/tests/windows/auth/login/UserAuth.test.cpp`
- Flow test: `project/tests/windows/auth/login/flow.test.cpp` (tests login flow)

## Running WinUI Tests

WinUI tests cannot run via `dotnet test` due to MSBuild integration limitations. Run the executable directly:
```bash
dotnet build project/tests/winUITests -p:Platform=x64
./project/tests/winUITests/bin/x64/Debug/net8.0-windows10.0.19041.0/Tidbits.Tests.WinUI.exe
```

**WinUI Requirement Condition**: Requirements using WinUI controls (Button, RadioButton, FrameworkElement, etc.) should be marked with `[windows: winUI]`. Tests go in `project/tests/winUITests/{path}/{File}.WinUI.test.cs`.

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

### Integration Tests Section

At the end of README.md, if directory has components that integrate:

```markdown
## Integration Tests
- {integration test case 1}:
- {integration test case 2}: {mark}
```

### Cross-Directory Integration Tests

When a concept has a `Related (Active/Passive)` link to another directory's README, add integration tests to verify all dependent behaviors.

**Placement**: Test entry goes in the README that defines the Related link.

**Integration Helpers Scope**: Each README's test file provides helpers for parts defined in that README. Specify helpers in README.md so code changes trigger helper updates.

**Integration Helpers Purpose**: Single source of truth for data or behaviors that cross-directory tests need, but aren't exposed as a single production function. Update ONE helper, all dependent tests automatically reflect the change.

**Helper needed:**
- `GetAllComponentTypes()`: Cross-directory tests iterate over all components. Add component #28 → update helper → all tests include it.
- `ExecuteWarnFlow()`: Cross-directory tests verify workflow. Workflow changes func1→func2→func3 to func1→func3 → update helper → all tests verify new flow.

**No helper needed:**
- Production has `Registry.GetCategoriesForComponent(type)` - call it directly, no duplication.
- If production later adds a function that a helper provides, remove the helper and use production.

**Demand-Driven Creation**: Only create integration helpers when a test actually requires one. Do not speculatively define helpers in README.md before tests need them. Write the test first—if production code doesn't expose the needed functionality, then add the helper.

```markdown
## Integration Helpers
- GetAllComponentTypes(): returns all component type names
```

```csharp
// In components integration test (components has Related to fields)
foreach (var type in ComponentsTestHelper.GetAllComponentTypes()) {
    // Use production code directly - no helper needed
    var categories = FieldCategoryRegistry.GetInstance().GetCategoriesForComponent(type);
    Assert.IsNotNull(categories);
}
```

Each module owns verification for its own concepts. Test fails if either side breaks the contract.

### {class/function}.md Relationship Rules

{class/function}.md files have no Related links or Looked Up By. They relate only to their directory's README.md implicitly.

**External References**: Other READMEs reference concepts in README.md, never {class/function}.md directly.

**Concept Surfacing**: When a {class/function}.md introduces a referenceable concept:
- Add concise concept entry in README.md (purpose, Related links)
- Keep detailed definition in {class/function}.md (description, signature, requirements)

```
fields/README.md:
  ## Concept: Field Category Registry   ← concise, has Related links

fields/FieldCategoryRegistry.md:
  ## Description                        ← detailed definition
  ## Signature
  ## Requirements
```

**Integration Helper Check**: When modifying {class/function}.md, check README.md's Integration Helpers section. If the change affects data returned by a helper (e.g., adding new component type), update the helper in the test file.

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

## Agent-Specific Rules

### Agent Independence

Each agent must independently verify all information required for its tasks. Never rely on or trust results reported by other agents. Always:
- Read source files directly (don't trust summaries from other agents)
- Check file existence independently (don't assume files were created)
- Run tests independently (don't trust pass/fail reports from other agents)
- Validate paths and locations independently (don't trust path calculations from other agents)

This ensures each agent catches errors that previous agents may have missed or misreported.

### Scope Boundaries

Do not perform tasks belonging to other phases. If you encounter work outside your scope (e.g., coding agent finds design issues, validation agent finds bugs to fix), stop and report to the main agent rather than handling it directly.

### Planning Agent

**What**: Creating and modifying .md files in `projectDescription/`.

**Tasks:**
1. Understand project requirements
2. Create README.md in each folder with concept entries
3. Create {class/function}.md files with description, signature, and requirements
4. If design needs more detail, create subfolder and repeat
5. Maintain concept relations and Looked Up By entries
6. Generate shared resources (JSON, images, assets) in `projectDescription/`
7. When removing planning phase outputs:
   a. Update all references (remove from Looked Up By)
   b. Add final requirement to the file: `- remove corresponding code, resources, and this file:`
   c. Do not delete the file - Coding Agent handles the actual deletion

**Outputs**: .md files and shared resources in `projectDescription/`

### Coding Agent

**What**: Generating code in `code/{flavor}/` based on .md files.

**Tasks:**
1. Read the {class/function}.md file
2. Read the directory's README.md to understand required concepts
3. Generate code to mirror path: `project/code/{flavor}/{path}/`
4. Copy shared resources from `projectDescription/` to mirrored path in `code/{flavor}/`
5. Generate test cases in `project/tests/{flavor}/`
6. Read README.md for Integration Tests section
7. Implement integration tests in `project/tests/{flavor}/{path}/integration.test.{ext}`
8. Run integration tests - fix issues before proceeding
9. When a planning phase output contains a removal requirement:
   a. Delete corresponding code at mirror path
   b. Delete corresponding tests
   c. Delete associated resources only if not referenced by other files
   d. Delete the file itself

**Test Rules:**
- Never create ignored, skipped, or disabled tests. All tests must be active and executable.
- Run tests for the code you wrote/modified to verify correctness before reporting completion.
- Do NOT run all tests - Testing Agent handles that.
- Requirements may have flavor-specific conditions: `[{flavor}: {condition}]`
  - `[windows: winUI]` - windows flavor uses winUI setup for this requirement
  - `[all: mock server]` - all flavors require mock server for testing
  - `[windows only]` - requirement applies to windows flavor only
- Do NOT mark requirements. Report completion to main agent.

**Outputs**: Code files and copied resources in `code/{flavor}/`, test cases in `tests/{flavor}/`

### Testing Agent

**What**: Running all tests for a specific setup.

**Tasks:**
1. Run all tests for the specified setup
2. Report results (passed, failed, skipped)
3. Report any skipped/ignored tests as an issue

### Validation Agent

**What**: Verifying implementations and marking requirements.

Applies to {class/function}.md requirements and README.md Integration Tests.

Coding agents implement and test. Validation agent (or main agent) marks requirements.

**Verification Rules:**

For {class/function}.md requirements:
1. Implementation exists at correct mirror path
2. Implementation signature matches the .md signature (class/function names, parameters, return types)
3. Corresponding test exists
4. Test passes
5. Test is not ignored, skipped, or disabled
6. Test actually verifies the requirement's intended behavior

For README.md Integration Tests:
1. Test file exists at `project/tests/{flavor}/{path}/integration.test.{ext}`
2. Test covers the requirement to be marked
3. Test passes
4. Test is not ignored, skipped, or disabled
5. Test actually verifies the requirement's intended behavior

Mark each requirement individually. If any condition fails, do not mark; if already marked, remove the mark.

**Flavor-Specific Marking:**
When verifying a specific flavor, only modify marks for that flavor:
- To add: append the flavor mark (e.g., `w` → `w, a`)
- To remove: remove only that flavor's mark (e.g., `w, a` → `w`)
- Never remove or modify marks for other flavors

**Test Logic Validation:**
Before marking, verify the test logic matches the requirement's intent. A passing test is insufficient if it doesn't actually test the intended behavior. If a test is flawed (tautologies, wrong assertions, testing unrelated functionality, or not matching the .md requirement), do not mark. Report the issue to the coding agent for correction.

**Verification Process:**
1. Coding agent implements code and creates tests
2. Coding agent runs tests
3. Validation agent (or main agent) confirms test results
4. Only validation agent marks requirements in {class/function}.md or README.md

Coding agents must NOT mark requirements themselves. Report completion to main/validation agent instead.

**Requirement Update Rule:**

Any change to code, tests, or documentation may invalidate existing requirements or marks. Before making changes:

1. Planning Agent: Update affected requirements in .md files (unmark, modify, add, or remove as needed)
2. Coding Agent: Make the changes
3. Validation Agent: Re-verify and re-mark after tests pass

This ensures requirements always reflect the current state.

**Examples (not exhaustive):**
- Modifying code → update requirements in corresponding {class/function}.md (use path mirroring)
- Modifying tests → update requirements in corresponding .md files (use path mirroring to find them)
- Modifying concepts → update requirements that depend on that concept (via Looked Up By)

## Agent Dispatch

Main agent uses the Task tool with `subagent_type="general-purpose"` to dispatch work to phase agents. This enables parallel execution and automatic workflow continuation.

### Task Prompts

**Planning Agent:**
```
Read CLAUDE.md first.

You are a Planning Agent. Follow Planning Agent rules in CLAUDE.md.

Task: {describe what needs to be planned}
Path: project/projectDescription/{path}

Report when complete:
- Files created/modified
- Any issues requiring upstream attention
```

**Coding Agent:**
```
Read CLAUDE.md first.

You are a Coding Agent for flavor: {flavor}. Follow Coding Agent rules in CLAUDE.md.

Task: Implement requirements from {path}
Documentation: project/projectDescription/{path}
Code output: project/code/{flavor}/{path}
Test output: project/tests/{flavor}/{path}

Do NOT mark requirements.

Report when complete:
- Files created/modified
- Completion status
```

**Testing Agent:**
```
Read CLAUDE.md first.

You are a Testing Agent. Follow Testing Agent rules in CLAUDE.md.

Task: Run all tests for setup: {setup}

Report when complete:
- Setup name
- Total tests, passed, failed
- List of failed tests with failure details
- Any skipped/ignored tests found
```

**Validation Agent:**
```
Read CLAUDE.md first.

You are a Validation Agent for flavor: {flavor}. Follow Validation Agent rules in CLAUDE.md.

Task: Verify and mark requirements for {path}
Documentation: project/projectDescription/{path}
Code location: project/code/{flavor}/{path}
Test location: project/tests/{flavor}/{path}

Report when complete:
- Requirements marked
- Requirements that failed verification (with reasons)
```

### Workflow Continuation

Main agent MUST continue the workflow after each phase completes:

1. **After Planning completes:**
   - Check if coding is needed for the planned items
   - Launch Coding Agent for each flavor (parallel if independent)

2. **After Coding completes:**
   - Launch Testing Agent for relevant setup(s)

3. **After Testing completes:**
   - If tests fail: return to Coding phase
   - If tests pass: launch Validation Agent

4. **After Validation completes:**
   - If issues found: return to Coding phase
   - If all valid: report final status to user

### Workflow Entry Point

When user requests implementation work, main agent determines starting phase:

1. **No .md files exist for the feature** → Start with Planning Agent
2. **.md files exist with unmarked requirements** → Start with Coding Agent
3. **Code exists but requirements unmarked** → Start with Testing or Validation Agent
4. **All requirements marked** → Report complete, ask user what's next

Main agent checks file existence before dispatching, does not assume.

### Dispatch Patterns

**Single flavor implementation:**
```
1. Launch Task: Coding Agent for {flavor} {path}
2. Wait for completion
3. Launch Task: Testing Agent for {setup}
4. Wait for completion
5. If tests fail: return to step 1
6. Launch Task: Validation Agent for {flavor}
   - Verify all marks targeted for change (additions and removals)
   - Verify marks for any requirements whose tests were modified
7. Wait for completion
8. If issues found: return to step 1
9. Report results
```

**Multi-flavor implementation (parallel):**
```
1. Launch Tasks in parallel: Coding Agent for each {flavor} {path}
2. Wait for all to complete
3. Launch Tasks in parallel: Testing Agent for each relevant {setup}
4. Wait for all to complete
5. If any setup fails: return to step 1
6. Launch Tasks in parallel: Validation Agent for each {flavor}
   - Verify all marks targeted for change (additions and removals)
   - Verify marks for any requirements whose tests were modified
7. Wait for all to complete
8. If any issues found: return to step 1
9. Report results
```

**Full workflow from planning:**
```
1. Launch Task: Planning Agent for {path}
2. Wait for completion
3. For each flavor, launch Coding + Testing + Validation sequence
4. Report results
```

**Grouped fixes:**
When test fixes conflict with each other (fixing one breaks another), group the related tests and fix them together in a single Coding Agent.

### Dependency Handling

**Planning Phase**: Directory hierarchy determines dependency order. Complete parent directories before child directories.

**Coding Phase**: Related (Active) links in README.md determine dependency order. Complete dependencies before dependents.

- **Independent tasks**: Can spawn sub-agents in parallel
- **Dependent tasks**: Wait for dependency to complete before spawning dependent task

### Dispatch Rules
1. Main agent coordinates workflow, does NOT do phase work directly
2. Use `run_in_background=true` for parallel agents, check with TaskOutput
3. Include "Read CLAUDE.md first" in every agent prompt

## Concept Tracking

### When You Add a Relation

If concept A references concept B:
1. Go to the README.md containing concept B
2. Add your README path to concept B's `Looked Up By` list
3. If relation crosses directories, add integration tests for all behaviors that depend on the referenced concept (see Cross-Directory Integration Tests)

This enables change propagation.

### When You Modify a Concept

Run change propagation:

1. Check the concept's `Looked Up By` list
2. For each README path listed:
   - Open that README
   - Find the concept with the same name (it references this one)
   - Verify logic is still valid, modify if needed
3. For each {class/function}.md in the same directory that depends on this concept:
   - **Remove marks only for affected requirements** - only those implementations need re-verification
4. In Coding Phase: re-verify and re-mark requirements after updating code

## Agent Limits

Maximum 8 agents running concurrently to avoid context over-consumption. Wait for agents to complete before spawning new ones if limit is reached.

## Agent Communication

Agents report issues in their completion response to the main agent. The main agent coordinates resolution by dispatching appropriate agents.

When an agent encounters an issue it cannot resolve:
- Report the issue in the completion response
- Main agent receives the report and dispatches the appropriate phase agent

### Upstream-First Principle

Fix issues at their source:
- Design/requirements issues → dispatch Planning Agent
- Implementation/test code issues → dispatch Coding Agent
- Test execution → dispatch Testing Agent
- Verification issues → dispatch Validation Agent

**Example flow for a bug:**
1. Bug identified (by user, test failure, or agent report)
2. Main agent dispatches Planning Agent to update requirements in {class/function}.md
3. Main agent dispatches Coding Agent to implement fix
4. Main agent dispatches Testing Agent to run all tests
5. Main agent dispatches Validation Agent to verify and mark
