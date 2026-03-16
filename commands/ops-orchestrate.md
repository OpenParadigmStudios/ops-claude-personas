---
description: Orchestrate full Epic implementation — plan, delegate, test, review
argument-hint: <epic-file>
allowed-tools: [Agent, Read, Glob, Grep, Bash, TaskCreate, TaskUpdate, TaskList]
---

# Orchestrate Epic

Run a full Epic implementation end-to-end: analyze dependencies, delegate Stories to specialist agents, validate with QA after each Story, and run a code review at Epic completion.

**Arguments**: `$ARGUMENTS` — path to the Epic spec file (e.g., `spec/implementation/phase1-scaffolding-db.md`)

---

## Workflow

### Step 1: Validate the Epic

Read the Epic file specified in `$ARGUMENTS`. If the file doesn't exist or isn't a valid Epic spec, report the error and stop.

Extract:
- Epic number and title
- All Stories with their numbers, titles, files to create, spec refs, and acceptance criteria
- Dependencies between Stories
- Total Story count

### Step 2: Load Context

Read these supporting files:
1. `spec/MASTER.md` — project overview and current status
2. `spec/glossary.md` — canonical terminology
3. `spec/implementation/README.md` — dependency graph across Epics
4. `CLAUDE.md` — project conventions and tech stack
5. All spec documents referenced by Stories in this Epic

### Step 3: Plan Execution

Analyze the Epic's Stories for dependencies and parallelism:

1. Identify explicit dependencies (Story A depends on Story B)
2. Identify implicit dependencies (Story A creates a file that Story B modifies)
3. Group Stories into sequential **phases** — within each phase, independent Stories can run in parallel
4. Identify the critical path (longest sequential chain)

Present the execution plan to the user before proceeding:

```markdown
## Execution Plan: Epic [X.Y] — [Title]

### Phase 1 (parallel)
| Story | Title | Agent | Dependencies |
|-------|-------|-------|-------------|
| X.Y.1 | ... | db-admin | none |
| X.Y.2 | ... | db-admin | none |

### Phase 2 (after Phase 1)
| Story | Title | Agent | Dependencies |
|-------|-------|-------|-------------|
| X.Y.3 | ... | backend-dev | X.Y.1, X.Y.2 |

**Critical path**: X.Y.1 → X.Y.3 → X.Y.5 (3 phases)
**Total Stories**: N
```

Wait for user confirmation before proceeding with implementation.

### Step 4: Create Tasks

Create a task for each Story using TaskCreate:
- Title: `Story X.Y.Z — [title]`
- Status: not started

Create additional tasks for:
- `QA validation — Epic X.Y`
- `Code review — Epic X.Y`

### Step 5: Execute Stories

For each phase in the execution plan:

**5a. Delegate Stories**

For each Story in the current phase, spawn the appropriate specialist agent using the Agent tool:

- **db-admin** — for Stories creating database models, migrations, data layer
- **backend-dev** — for Stories creating API endpoints, services, schemas, business logic
- **frontend-dev** — for Stories creating UI components or pages

When spawning an agent:
- Use the agent name as `subagent_type` (e.g., `subagent_type: "backend-dev"`)
- Set description to `Story X.Y.Z — [title]`
- In the prompt, instruct the agent to run `/implement-story <epic-file> <story-number>`
- Include a brief summary of the Story's scope and key spec refs
- If Stories within the same phase are independent, spawn their agents concurrently

Update the Story's task to "in progress" when the agent is spawned.

**5b. QA Validation**

After each Story's implementation agent completes, spawn **qa-engineer**:
- `subagent_type: "qa-engineer"`
- Prompt: instruct it to run `/run-tests` targeting the files created/modified by the Story
- Include the Story's acceptance criteria in the prompt so QA knows what to verify

**5c. Handle Results**

For each Story:
- **If implementation + QA pass**: update the Story's task to "completed" and proceed
- **If implementation fails**: update the task with the failure details, report the blocker to the user, and stop the current phase
- **If QA fails**: spawn the same specialist agent again with the test failures and acceptance criteria that weren't met. If it fails a second time, report to the user and stop

**5d. Advance to Next Phase**

Only proceed to the next phase when all Stories in the current phase are complete and QA-verified.

### Step 6: Epic Code Review

After all Stories are complete and QA-verified, spawn **code-reviewer**:
- `subagent_type: "code-reviewer"`
- Prompt: instruct it to run `/review-changes` across all files created/modified during this Epic
- Include the Epic's scope and spec refs for context

Handle review results:
- **PASS**: proceed to completion
- **PASS WITH NOTES**: report the notes, proceed to completion
- **NEEDS CHANGES**: report the critical/moderate issues. Spawn the appropriate specialist to address critical issues, then re-run review

### Step 7: Completion

1. Update all tasks to reflect final status
2. Update `spec/implementation/README.md` — mark the Epic and its Stories as complete
3. Report the final summary:

```markdown
## Epic [X.Y] — [Title] Complete

### Stories Implemented
| Story | Title | Agent | Status |
|-------|-------|-------|--------|
| X.Y.1 | ... | db-admin | complete |

### Files Created
- [list of all new files]

### Files Modified
- [list of all modified files]

### Test Results
- Total tests: X
- All passing: yes/no

### Code Review
- Verdict: [PASS / PASS WITH NOTES]
- Notes: [if any]

### What's Next
- [Next Epic that's unblocked]
- [Any follow-up items from review notes]
```

---

## Rules

- Always present the execution plan and wait for user confirmation before starting implementation
- Never skip QA — every Story must have passing tests
- Never skip the code review gate at Epic completion
- If a Story fails twice (implementation or QA), stop and escalate to the user rather than retrying in a loop
- Track all progress via tasks so the user can see status at any time
- Report progress at natural milestones: phase completion, QA results, review results
- If the Epic has dependencies on other Epics that aren't complete, report the blocker immediately
