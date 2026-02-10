---
name: codex
description: Orchestrate OpenAI Codex CLI across three roles (Planner, Coder, Verifier) to decompose large requirements, code in parallel, and verify results. Also supports single-task mode for quick Codex commands.
---

# Codex Orchestration Skill

Claude acts as the **orchestrator** managing Codex instances across three specialized roles:

| Role | Purpose | Sandbox | Parallel |
| --- | --- | --- | --- |
| **Planner** | Decompose requirement into independent sub-tasks | `read-only` | No (single instance) |
| **Coder** | Implement one sub-task | `workspace-write` | Yes (N instances) |
| **Verifier** | Review one Coder's work for correctness | `read-only` | Yes (N instances) |

Claude **never delegates orchestration to Codex**. Claude owns: task scheduling, conflict detection, dependency ordering, result aggregation, and user communication.

---

## Mode Selection

When the user invokes Codex, determine which mode to use:

- **Orchestration mode** — the requirement is large, spans multiple files/modules, or the user explicitly asks for planning/parallel work. Follow the full 5-phase workflow below.
- **Single-task mode** — the requirement is small, focused, or the user just wants a quick Codex run. Skip to the [Single-Task Mode](#single-task-mode) section.

If unsure, ask the user via `AskUserQuestion`.

---

## Pre-flight — Git Safety Check

**This check is MANDATORY before any Codex work (orchestration or single-task).**

1. Run `git status --porcelain` in the target workspace directory.
2. If the output is **empty** (working tree clean), proceed to Phase 0.
3. If the output is **non-empty** (uncommitted changes exist), **stop and ask the user** via `AskUserQuestion`:

| Option | Description |
| --- | --- |
| **Commit current changes** | Claude helps stage and commit all pending changes on the current branch before proceeding. |
| **Create a new branch** | Claude creates a new feature branch from the current state, commits pending changes there, then proceeds with Codex work on the new branch. |
| **Stash changes** | Claude runs `git stash` to temporarily shelve changes, proceeds with Codex work, and reminds the user to `git stash pop` afterward. |
| **Proceed anyway** | The user accepts the risk of uncommitted changes being mixed with Codex modifications. |

### Why This Matters
- Codex Coders modify files directly in the workspace. Without a clean baseline, it becomes impossible to distinguish Codex's changes from the user's pre-existing uncommitted work.
- `git diff` conflict detection in Phase 2 relies on a known clean state.
- If something goes wrong, a clean commit point allows easy rollback with `git checkout .` or `git reset`.

### Commit Helper
If the user chooses "Commit current changes" or "Create a new branch":
1. Run `git status` to list all modified/untracked files.
2. Present the file list and ask the user to confirm which files to stage (default: all).
3. If "Create a new branch" was chosen, run `git checkout -b <branch-name>` first. Ask the user for the branch name, or suggest one based on the requirement (e.g., `feat/user-auth`).
4. Stage and commit with a descriptive message.
5. Confirm the commit succeeded, then proceed to Phase 0.

---

## Phase 0 — Preparation

1. **Model & reasoning effort**: Ask the user (via `AskUserQuestion` — **one prompt, two questions**) which model (`gpt-5.3-codex` or `gpt-5.2`) and which reasoning effort (`xhigh`, `high`, `medium`, `low`) to use. If the user says `codex` without specifying a model or reasoning effort, default to `gpt-5.3-codex` / `high` without asking. If the user specifies only one (e.g., model but not effort), ask only about the unspecified parameter.
2. **Validate installation**: Run `codex --version 2>/dev/null`. If it fails, stop and tell the user to install Codex.
3. **Identify workspace**: Note the current working directory or `-C <DIR>` target. All Codex commands will operate in this directory.
4. **Prepare schemas**: Two JSON output schemas are shipped alongside this skill (relative to this SKILL.md):
   - `plan-schema.json` — used by the Planner for structured task decomposition output
   - `verify-schema.json` — used by each Verifier for structured review reports
   Resolve the absolute paths to both files; you will pass them via `--output-schema`.

---

## Phase 1 — Planning

Launch a single Codex **Planner** to decompose the user's requirement.

### Command Template

```bash
codex exec \
  -m <MODEL> \
  --config model_reasoning_effort="<EFFORT>" \
  --config hide_agent_reasoning=true \
  --config model_reasoning_summary="none" \
  --sandbox read-only \
  --full-auto \
  --skip-git-repo-check \
  --ephemeral \
  --output-schema "<ABSOLUTE_PATH_TO>/plan-schema.json" \
  -o "<TEMP_DIR>/codex-plan.json" \
  "<PLANNER_PROMPT>" 2>"<TEMP_DIR>/codex-plan.stderr"
```

**Notes**:
- Stderr is redirected to a file (not `/dev/null`) so errors can be diagnosed. On success, Claude ignores the stderr file. On failure (non-zero exit), Claude reads it for error details.
- `hide_agent_reasoning=true` and `model_reasoning_summary="none"` suppress thinking tokens from both stdout and stderr, dramatically reducing output size and saving Claude orchestrator tokens.

### Concrete Example

```bash
codex exec \
  -m gpt-5.3-codex \
  --config model_reasoning_effort="high" \
  --config hide_agent_reasoning=true \
  --config model_reasoning_summary="none" \
  --sandbox read-only \
  --full-auto \
  --skip-git-repo-check \
  --ephemeral \
  --output-schema "/home/user/.claude/skills/codex/plan-schema.json" \
  -o "/tmp/codex-plan.json" \
  "Analyze this codebase and decompose the following requirement into independent sub-tasks that can be implemented separately. REQUIREMENT: Add user authentication with JWT tokens, login/register endpoints, and role-based access control. RULES: 1. Each task MUST operate on a DIFFERENT set of files. No two tasks may modify the same file. 2. If two pieces of work MUST touch the same file, merge them into one task. 3. List exact file paths in files_to_modify. Use files_to_read for context-only references. 4. If a task depends on another task's output, mark it in the dependencies array. 5. Write clear acceptance_criteria that can be verified by reading the code. 6. Keep tasks granular enough for one focused coding session but not so small they lack context. 7. Respond ONLY with valid JSON matching the output schema. No markdown, no commentary." 2>"/tmp/codex-plan.stderr"
```

### Planner Prompt Construction

Build the prompt by combining these elements. **Do NOT mention other roles (Coder, Verifier) or the orchestration system. The Planner only knows it is analyzing and decomposing a requirement.**

```
Analyze this codebase and decompose the following requirement into independent sub-tasks that can be implemented separately.

REQUIREMENT:
<user's requirement verbatim>

RULES:
1. Each task MUST operate on a DIFFERENT set of files. No two tasks may modify the same file.
2. If two pieces of work MUST touch the same file, merge them into one task.
3. List exact file paths in files_to_modify. Use files_to_read for context-only references.
4. If a task depends on another task's output, mark it in the dependencies array.
5. Write clear acceptance_criteria that can be verified by reading the code.
6. Keep tasks granular enough for one focused coding session but not so small they lack context.
7. Respond ONLY with valid JSON matching the output schema. No markdown, no commentary.
```

### After the Planner Completes

1. Read `<TEMP_DIR>/codex-plan.json` and parse the task list.
2. **Validate no file overlap**: Build a map of `files_to_modify` across all tasks. If any file appears in more than one task, flag it.
   - If overlap exists: either merge the conflicting tasks into one, or mark them as sequential (respect dependency order).
3. **Present the plan to the user** via a clear summary table. Include task ID, title, files, and dependencies. Ask the user to approve, adjust, or reject tasks before proceeding.
4. **Resolve dependencies**: Sort tasks topologically. Tasks with no dependencies form the first parallel batch. Tasks depending on completed ones form subsequent batches.

---

## Phase 2 — Parallel Coding

For each batch of independent tasks, launch Codex **Coder** instances in parallel.

### Command Template (per Coder)

```bash
codex exec \
  -m <MODEL> \
  --config model_reasoning_effort="<EFFORT>" \
  --config hide_agent_reasoning=true \
  --config model_reasoning_summary="none" \
  --sandbox workspace-write \
  --full-auto \
  --skip-git-repo-check \
  -o "<TEMP_DIR>/codex-coder-<TASK_ID>.txt" \
  "<CODER_PROMPT>" 2>"<TEMP_DIR>/codex-coder-<TASK_ID>.stderr" &
```

After launching all Coders in a batch, run `wait` to block until they all finish.

### Concrete Example (3 parallel Coders)

```bash
# Launch all coders in parallel
codex exec -m gpt-5.3-codex --config model_reasoning_effort="high" --config hide_agent_reasoning=true --config model_reasoning_summary="none" --sandbox workspace-write --full-auto --skip-git-repo-check -o "/tmp/codex-coder-1.txt" "Implement the following task. TASK: Add JWT authentication middleware. DESCRIPTION: Create auth middleware that validates JWT tokens from Authorization header... FILES YOU MUST MODIFY (and ONLY these files): src/middleware/auth.ts, src/types/auth.ts. FILES TO READ FOR CONTEXT (do NOT modify these): src/app.ts, package.json. ACCEPTANCE CRITERIA: 1. Middleware extracts Bearer token from header 2. Validates token signature 3. Attaches decoded user to request object 4. Returns 401 on invalid/missing token. RULES: 1. ONLY modify listed files 2. Follow existing code style 3. Implement ALL criteria 4. Explain any blockers in output 5. No unnecessary additions 6. Do NOT run package managers or build tools." 2>"/tmp/codex-coder-1.stderr" &

codex exec -m gpt-5.3-codex --config model_reasoning_effort="high" --config hide_agent_reasoning=true --config model_reasoning_summary="none" --sandbox workspace-write --full-auto --skip-git-repo-check -o "/tmp/codex-coder-2.txt" "Implement the following task. TASK: Create user registration endpoint. DESCRIPTION: Build POST /api/register endpoint... FILES YOU MUST MODIFY (and ONLY these files): src/routes/auth.ts, src/models/user.ts. FILES TO READ FOR CONTEXT (do NOT modify these): src/middleware/auth.ts, src/db/connection.ts. ACCEPTANCE CRITERIA: 1. Accepts email and password 2. Hashes password with bcrypt 3. Creates user in database 4. Returns JWT token. RULES: 1. ONLY modify listed files 2. Follow existing code style 3. Implement ALL criteria 4. Explain any blockers 5. No unnecessary additions 6. Do NOT run package managers or build tools." 2>"/tmp/codex-coder-2.stderr" &

codex exec -m gpt-5.3-codex --config model_reasoning_effort="high" --config hide_agent_reasoning=true --config model_reasoning_summary="none" --sandbox workspace-write --full-auto --skip-git-repo-check -o "/tmp/codex-coder-3.txt" "Implement the following task. TASK: Add role-based access control. DESCRIPTION: Create RBAC utility and route guards... FILES YOU MUST MODIFY (and ONLY these files): src/middleware/rbac.ts, src/config/roles.ts. FILES TO READ FOR CONTEXT (do NOT modify these): src/middleware/auth.ts. ACCEPTANCE CRITERIA: 1. Define admin/user/viewer roles 2. Create requireRole middleware 3. Guard routes based on user role. RULES: 1. ONLY modify listed files 2. Follow existing code style 3. Implement ALL criteria 4. Explain any blockers 5. No unnecessary additions 6. Do NOT run package managers or build tools." 2>"/tmp/codex-coder-3.stderr" &

# Wait for all coders to finish
wait
```

### Coder Prompt Construction

**Do NOT mention other roles (Planner, Verifier), the orchestration system, or that other Coders exist. Each Coder only knows about its own task.**

```
Implement the following task.

TASK: <TITLE>

DESCRIPTION:
<description from plan>

FILES YOU MUST MODIFY (and ONLY these files):
<files_to_modify list>

FILES TO READ FOR CONTEXT (do NOT modify these):
<files_to_read list>

ACCEPTANCE CRITERIA:
<acceptance_criteria list>

RULES:
1. ONLY create or modify files listed in FILES YOU MUST MODIFY. Do not touch any other files.
2. Follow existing code style and conventions in the project.
3. Implement ALL acceptance criteria.
4. If you cannot complete a criterion, explain why in your final output instead of skipping it silently.
5. Do not add unnecessary abstractions, comments, or features beyond what is specified.
6. Do NOT run package managers (npm install, pip install, etc.) or build tools. Only modify source files.
```

### After All Coders Complete

1. Read each `codex-coder-<TASK_ID>.txt` output file.
2. Check for errors — if any Coder exited non-zero or reported failure, note the failed task.
3. **Conflict detection**: Run `git diff --name-only` (or compare file modification timestamps) to verify that Coders only modified their assigned files. If unexpected overlaps are detected:
   - Analyze the conflicting changes.
   - Attempt automatic resolution if the changes are additive/non-conflicting (e.g., both added different functions to a shared utility file).
   - If resolution is ambiguous, present the conflict to the user via `AskUserQuestion` and ask how to proceed.
4. Report Coder results summary to the user before starting verification.

---

## Phase 3 — Parallel Verification

For each completed Coder task, launch a corresponding Codex **Verifier** instance. One Verifier per Coder (1:1 mapping).

### Command Template (per Verifier)

```bash
codex exec \
  -m <MODEL> \
  --config model_reasoning_effort="<EFFORT>" \
  --config hide_agent_reasoning=true \
  --config model_reasoning_summary="none" \
  --sandbox read-only \
  --full-auto \
  --skip-git-repo-check \
  --ephemeral \
  --output-schema "<ABSOLUTE_PATH_TO>/verify-schema.json" \
  -o "<TEMP_DIR>/codex-verify-<TASK_ID>.json" \
  "<VERIFIER_PROMPT>" 2>"<TEMP_DIR>/codex-verify-<TASK_ID>.stderr" &
```

After launching all Verifiers, run `wait` to block until they all finish.

### Concrete Example (3 parallel Verifiers)

```bash
codex exec -m gpt-5.3-codex --config model_reasoning_effort="high" --config hide_agent_reasoning=true --config model_reasoning_summary="none" --sandbox read-only --full-auto --skip-git-repo-check --ephemeral --output-schema "/home/user/.claude/skills/codex/verify-schema.json" -o "/tmp/codex-verify-1.json" "Review the following completed implementation task for correctness and completeness. TASK: Add JWT authentication middleware. ORIGINAL TASK DESCRIPTION: Create auth middleware that validates JWT tokens... FILES THAT SHOULD HAVE BEEN MODIFIED: src/middleware/auth.ts, src/types/auth.ts. ACCEPTANCE CRITERIA TO CHECK: 1. Middleware extracts Bearer token 2. Validates token signature 3. Attaches decoded user to request 4. Returns 401 on invalid/missing token. INSTRUCTIONS: 1. Read every file listed above 2. Check each criterion - mark PASS or FAIL with explanation 3. Look for edge cases, security issues, broken imports, type errors, logic bugs 4. Check no unexpected files were modified 5. Respond ONLY with valid JSON matching the output schema." 2>"/tmp/codex-verify-1.stderr" &

codex exec -m gpt-5.3-codex --config model_reasoning_effort="high" --config hide_agent_reasoning=true --config model_reasoning_summary="none" --sandbox read-only --full-auto --skip-git-repo-check --ephemeral --output-schema "/home/user/.claude/skills/codex/verify-schema.json" -o "/tmp/codex-verify-2.json" "Review the following completed implementation task for correctness and completeness. TASK: Create user registration endpoint. ORIGINAL TASK DESCRIPTION: Build POST /api/register endpoint... FILES THAT SHOULD HAVE BEEN MODIFIED: src/routes/auth.ts, src/models/user.ts. ACCEPTANCE CRITERIA TO CHECK: 1. Accepts email and password 2. Hashes password with bcrypt 3. Creates user in database 4. Returns JWT token. INSTRUCTIONS: 1. Read every file listed above 2. Check each criterion - mark PASS or FAIL with explanation 3. Look for edge cases, security issues, broken imports, type errors, logic bugs 4. Check no unexpected files were modified 5. Respond ONLY with valid JSON matching the output schema." 2>"/tmp/codex-verify-2.stderr" &

codex exec -m gpt-5.3-codex --config model_reasoning_effort="high" --config hide_agent_reasoning=true --config model_reasoning_summary="none" --sandbox read-only --full-auto --skip-git-repo-check --ephemeral --output-schema "/home/user/.claude/skills/codex/verify-schema.json" -o "/tmp/codex-verify-3.json" "Review the following completed implementation task for correctness and completeness. TASK: Add role-based access control. ORIGINAL TASK DESCRIPTION: Create RBAC utility and route guards... FILES THAT SHOULD HAVE BEEN MODIFIED: src/middleware/rbac.ts, src/config/roles.ts. ACCEPTANCE CRITERIA TO CHECK: 1. Define admin/user/viewer roles 2. Create requireRole middleware 3. Guard routes based on user role. INSTRUCTIONS: 1. Read every file listed above 2. Check each criterion - mark PASS or FAIL with explanation 3. Look for edge cases, security issues, broken imports, type errors, logic bugs 4. Check no unexpected files were modified 5. Respond ONLY with valid JSON matching the output schema." 2>"/tmp/codex-verify-3.stderr" &

wait
```

### Verifier Prompt Construction

**Do NOT mention other roles (Planner, Coder) or the orchestration system. The Verifier only knows it is reviewing a completed task.**

```
Review the following completed implementation task for correctness and completeness.

TASK: <TITLE>

ORIGINAL TASK DESCRIPTION:
<description from plan>

FILES THAT SHOULD HAVE BEEN MODIFIED:
<files_to_modify list>

ACCEPTANCE CRITERIA TO CHECK:
<acceptance_criteria list>

INSTRUCTIONS:
1. Read every file listed above.
2. Check each acceptance criterion — mark it PASS or FAIL with a brief explanation.
3. Look for: missing edge cases, security issues, broken imports, type errors, logic bugs, incomplete implementations.
4. Check that no files OUTSIDE the listed files were unexpectedly modified.
5. Respond ONLY with valid JSON matching the output schema. No markdown, no commentary.
```

### After All Verifiers Complete

1. Read each `codex-verify-<TASK_ID>.json` output file and parse the JSON.
2. Extract `overall_verdict` for each task. Collect all `criteria_results` and `issues_found`.

---

## Phase 4 — Synthesis

Claude aggregates all results and reports to the user.

### If All Verifiers Pass
- Report success: list each task with its PASS status.
- Inform the user that all tasks were implemented and verified.
- Offer to run tests, linting, or any project-specific validation.

### If Any Verifier Reports Failure
- Present a clear failure report showing which tasks failed and why.
- Ask the user via `AskUserQuestion` how to handle each failure:
  - **Re-code**: Launch a new Coder for the failed task with the Verifier's feedback included in the prompt.
  - **Manual fix**: The user (or Claude) fixes the issue directly.
  - **Accept as-is**: The user acknowledges the issue and moves on.
- If re-coding, repeat Phase 2 → Phase 3 → Phase 4 for only the failed tasks.
- **Maximum 2 retry cycles per task.** After 2 failures, escalate to the user for manual resolution.

### Post-Synthesis
- Inform the user: "You can resume any Codex session at any time. Each Coder and Verifier session is preserved."
- Clean up temp files only if the user confirms, or leave them for reference.

---

## Conflict Prevention & Resolution

### Prevention (Planner-Level)
- The Planner prompt explicitly requires non-overlapping `files_to_modify` between tasks.
- Claude validates this constraint before launching Coders. If overlap is found, Claude either:
  - Merges overlapping tasks into one, or
  - Converts them from parallel to sequential with the earlier task as a dependency.

### Detection (Post-Coding)
- After all Coders finish, Claude checks which files were actually modified.
- Compare actual modifications against the plan's `files_to_modify` assignments.

### Resolution (Claude-Level)
- **Non-conflicting overlap** (e.g., additions to different sections of same file): Claude merges automatically.
- **Conflicting overlap** (e.g., both modified same function): Claude presents both versions to the user and asks which to keep, or attempts an intelligent merge.
- **Unauthorized modifications** (Coder touched files outside its scope): Claude flags the change and asks the user whether to keep or revert it.

---

## Single-Task Mode

For simple, focused tasks where orchestration is unnecessary.

### Workflow
1. Ask the user for model and reasoning effort (same as Phase 0), unless the user says `codex` without specifying a model or reasoning effort (then default to `gpt-5.3-codex` / `high`).
2. Select sandbox mode based on the task:

| Use case | Sandbox | Key flags |
| --- | --- | --- |
| Read-only analysis | `read-only` | `--sandbox read-only` |
| Apply local edits | `workspace-write` | `--sandbox workspace-write --full-auto` |
| Network or broad access needed | `danger-full-access` | `--sandbox danger-full-access --full-auto` |

3. Run the command:
```bash
codex exec \
  -m <MODEL> \
  --config model_reasoning_effort="<EFFORT>" \
  --config hide_agent_reasoning=true \
  --config model_reasoning_summary="none" \
  --sandbox <MODE> \
  --full-auto \
  --skip-git-repo-check \
  -o "<TEMP_DIR>/codex-single-task.txt" \
  "<USER_PROMPT>" 2>"<TEMP_DIR>/codex-single-task.stderr"
```
4. Summarize the outcome and ask about follow-up.

### Resuming a Session
```bash
echo "<NEW_PROMPT>" | codex exec --skip-git-repo-check resume --last 2>"<TEMP_DIR>/codex-resume.stderr"
```
When resuming, do **not** pass model, reasoning effort, or sandbox flags — they are inherited from the original session. Only add flags if the user explicitly requests a change.

---

## Command Reference

### Global Flags (apply to all roles)
| Flag | Purpose |
| --- | --- |
| `-m, --model <MODEL>` | Model override |
| `--config model_reasoning_effort="<LEVEL>"` | Reasoning effort: `xhigh`, `high`, `medium`, `low` |
| `--config hide_agent_reasoning=true` | Suppress reasoning/thinking events from output. **Always use.** |
| `--config model_reasoning_summary="none"` | Disable reasoning summaries entirely. **Always use.** |
| `--sandbox <MODE>` | `read-only`, `workspace-write`, `danger-full-access` |
| `--full-auto` | Auto-approve all tool calls without prompting (does NOT set sandbox mode — use `--sandbox` separately) |
| `--skip-git-repo-check` | Allow running outside Git repos. **Always use this flag.** |
| `-C, --cd <DIR>` | Set working directory |
| `-o, --output-last-message <PATH>` | Write final message to file |
| `--output-schema <PATH>` | JSON Schema for structured output validation |
| `--json` | Newline-delimited JSON event stream |
| `--ephemeral` | Do not persist session files |
| `--image, -i <PATH>` | Attach images to prompt |
| `--add-dir <PATH>` | Grant additional directory write access |
| `2>"<file>.stderr"` | Redirect stderr (thinking tokens + errors) to file. Read on failure for diagnostics. |

### Role-Specific Defaults
| | Planner | Coder | Verifier |
| --- | --- | --- | --- |
| Sandbox | `read-only` | `workspace-write` | `read-only` |
| `--full-auto` | Yes | Yes | Yes |
| `--output-schema` | Yes (plan-schema.json) | No | Yes (verify-schema.json) |
| `-o` (output file) | Yes (.json) | Yes (.txt) | Yes (.json) |
| `--ephemeral` | Yes | No (preserve session for resume) | Yes |
| Stderr redirect | `2>"<file>.stderr"` | `2>"<file>.stderr"` | `2>"<file>.stderr"` |

---

## Error Handling

### General
- If `codex --version` fails, stop and instruct the user to install/configure Codex.
- If any `codex exec` exits non-zero, **read the corresponding `.stderr` file** for error details and report it to the user. Do not silently continue.
- On success (exit code 0), the `.stderr` files can be ignored (they contain only thinking tokens).
- Before using `--full-auto`, `--sandbox danger-full-access`, or `--skip-git-repo-check` for the first time in a session, ask the user for permission via `AskUserQuestion` (unless already granted).

### Per-Role Errors

**Planner fails:**
- Report the error to the user.
- Offer to retry with adjusted reasoning effort or a different model.
- If retry also fails, offer to have Claude do the planning manually.

**Coder fails (one or more):**
- Report which Coders failed and their error output.
- Offer to: retry the failed task, adjust the task description, or let Claude implement it directly.
- Do NOT re-run Coders that succeeded.

**Verifier fails:**
- Report the failure. Offer to retry or have Claude do a manual review of that task.
- A Verifier failure means the review process failed, not that the code is bad.

### Graceful Degradation
- If parallel execution causes issues (e.g., resource constraints, sandbox conflicts), fall back to sequential execution and inform the user.
- If `--output-schema` is not supported by the installed Codex version, fall back to unstructured output and have Claude parse the Planner's response manually.

---

## Critical Evaluation of Codex Output

Codex is powered by OpenAI models with their own knowledge cutoffs and limitations. Treat Codex as a **colleague, not an authority**.

### Guidelines
- **Trust your own knowledge** when confident. If Codex claims something you know is incorrect, push back directly.
- **Research disagreements** using WebSearch or documentation before accepting Codex's claims.
- **Remember knowledge cutoffs** — Codex may not know about recent releases, APIs, or changes that occurred after its training data.
- **Don't defer blindly** — Codex can be wrong. Evaluate its suggestions critically, especially regarding model names, recent library versions, API changes, and best practices.

### When Codex is Wrong
1. State your disagreement clearly to the user.
2. Provide evidence (your own knowledge, web search, docs).
3. Optionally resume the Codex session to discuss:
   ```bash
   echo "This is Claude (<your current model name>) following up. I disagree with [X] because [evidence]. What's your take?" | codex exec --skip-git-repo-check resume --last
   ```
4. Frame disagreements as discussions — either AI could be wrong.
5. Let the user decide how to proceed if there is genuine ambiguity.

---

## Following Up

- After every orchestration cycle, use `AskUserQuestion` to confirm next steps.
- When resuming, pipe the new prompt via stdin: `echo "new prompt" | codex exec --skip-git-repo-check resume --last`.
- Restate the chosen model, reasoning effort, and sandbox mode when proposing follow-up actions.
- Individual Coder sessions can be resumed by session ID if the user wants to iterate on a specific task.
