# Spec-to-Code Autopilot -- MVP Specification

## 1. Overview

**Spec-to-Code Autopilot** is the first core feature of **MaxAICTO** -- an autonomous technical director agent that watches the `specs` GitHub repository, detects new or existing specifications, evaluates implementation complexity, and orchestrates code delivery through **Claude Code** (primary coding agent for MVP).

The agent works through two triggers:
- **Autonomous trigger**: new spec detected in the `specs` repo
- **Manual trigger**: command from Telegram

It should understand project-to-repository mappings, prepare an implementation plan, decide whether the task is simple or complex, execute coding runs in isolated work directories cloned from GitHub, create commits/PRs, update progress in the spec repo, and report everything back in Telegram.

---

## 2. Goal

Turn GitHub specs into an execution pipeline: when a spec appears, **MaxAICTO** should be able to assess it, decide how to act, implement it via coding agents, and report outcomes with minimal manual coordination.

---

## 3. Problem

Right now, specs exist separately from implementation. Someone still has to:
- notice new specs
- identify the target code repo
- decide whether implementation is safe to auto-start
- run a coding agent manually
- check tests and output quality
- create commits/PRs
- update progress files
- report status back in Telegram

MaxAICTO should automate this coordination layer.

---

## 4. MVP Scope

### In Scope
- Watch `specs` repo for new specs
- Accept manual run commands from Telegram
- Audit old specs and propose candidates on first run
- Map spec project -> code repo from config, with override support
- Classify tasks as `simple`, `complex`, `very_complex`
- Auto-start simple tasks
- Require approval for complex/unclear tasks
- Use **Claude Code** as primary coding agent
- Work with one primary code repo per spec plus optional related repos
- Clone/pull from GitHub for each run
- Execute each run in a fresh isolated working directory
- Produce preliminary implementation status before coding
- Run tests / lint / typecheck / local app run / self-review when applicable
- Commit, push, create PR
- Update spec/progress files at key stages
- Send Telegram reports
- Retry failed implementation attempts, then create issue + report next steps if still failing
- Track execution state in Supabase

### Out of Scope (MVP)
- Managing a human dev team
- Multi-user approvals
- Auto-merging PRs into `main`
- Full roadmap planning
- Budget / velocity analytics
- Autonomous production deploys

---

## 5. Core User Stories

### US1. Detect new specs automatically
**As Max**, I want MaxAICTO to detect new specs in the `specs` repo so implementation can start without manual monitoring.

**Acceptance Criteria**
- Agent reacts to new specs from GitHub webhook
- Agent also performs periodic fallback checks every 10 minutes
- Duplicate processing is prevented via internal state + spec status checks

### US2. Run specs manually from Telegram
**As Max**, I want to trigger work from Telegram so I can force or direct execution myself.

**Acceptance Criteria**
- Manual trigger supports free text and explicit command format
- Example supported styles:
  - `запусти спеку researcher-agent`
  - `/run-spec researcher/researcher-agent.md`
- If manual request conflicts with current autonomous state, bot warns and allows manual force

### US3. Prepare preliminary status before coding
**As Max**, I want a preliminary technical assessment before coding starts so I understand what will happen.

**Acceptance Criteria**
- Preliminary status includes:
  - target project
  - target repo
  - complexity assessment
  - whether a new branch is needed
  - selected coding agent
  - implementation plan
  - blockers / ambiguities
- This report is sent before implementation begins

### US4. Auto-start only simple tasks
**As Max**, I want simple tasks to start automatically, while risky tasks wait for approval.

**Acceptance Criteria**
- `simple` tasks auto-start
- `complex` and `very_complex` tasks wait for approval
- Incomplete or contradictory specs trigger a technical review + approval request

### US5. Execute implementation via Claude Code
**As Max**, I want MaxAICTO to implement specs through a coding agent.

**Acceptance Criteria**
- Primary coding agent in MVP is **Claude Code**
- Later expansion to Codex is possible but out of core MVP behavior
- Execution runs happen in fresh temp working copies cloned from GitHub
- Each spec has one primary repo and optional related repos

### US6. Validate output before delivery
**As Max**, I want MaxAICTO to verify changes before creating a PR.

**Acceptance Criteria**
- Runs tests when available
- Runs lint / typecheck when available
- Attempts local app run when practical
- Produces a brief self-review before PR creation

### US7. Deliver implementation result
**As Max**, I want MaxAICTO to complete the dev loop and report back.

**Acceptance Criteria**
- Agent commits changes
- Pushes branch
- Creates PR
- Sends Telegram report including:
  - spec name
  - target project/repo
  - summary of changes
  - branch link
  - PR link
  - test/check results
  - items requiring manual review
- Updates spec/progress status at key milestones

### US8. Handle failure safely
**As Max**, I want failures to be visible and actionable.

**Acceptance Criteria**
- Failed runs are retried
- If still failing, agent sends failure report with reason and recommended next step
- If implementation is incomplete, agent creates a linked GitHub issue in code repo and ties it back to the spec

### US9. Audit old specs
**As Max**, I want MaxAICTO to review past specs so I can choose what to implement next.

**Acceptance Criteria**
- On first run, agent audits existing specs
- Produces a candidate list instead of auto-running everything
- Waits for user confirmation/selection before executing audited backlog items

---

## 6. Complexity Rules

### simple
- One primary repo
- Small/localized change
- No migrations
- No major integration work
- Low regression risk
- Can auto-start

### complex
- Multiple files/modules
- New integration or architectural touch points
- Migration or higher regression risk
- Requires approval before coding

### very_complex
- Multiple repos or broad refactor
- Incomplete/conflicting spec
- High uncertainty or operational risk
- Requires approval before coding

---

## 7. Triggers

### Autonomous
- GitHub webhook on new/updated spec in `specs`
- Periodic fallback scan every **10 minutes**

### Manual
- Free-text Telegram commands
- Explicit commands like `/run-spec <path>`

Manual commands can override autonomous flow, but the agent must warn about conflicts first.

---

## 8. Source of Truth and Mapping

### Specs Source
- GitHub `specs` repository
- Agent must support both newly created specs and existing historical specs

### Repo Mapping
- Primary source: config file / config store
- Must support manual override
- Each spec maps to:
  - one primary code repo
  - optional related repos

---

## 9. Execution Model

1. Detect or receive spec
2. Resolve project and repo mapping
3. Check if spec is already in progress / done
4. Produce preliminary status
5. Classify complexity
6. If simple -> start automatically
7. If complex/unclear -> wait for approval
8. Clone target repo(s) from GitHub into fresh temp workspace
9. Run coding agent (Claude Code)
10. Validate output
11. Commit + push branch
12. Create PR
13. Update spec progress/state
14. Send Telegram report

---

## 10. Progress Updates in Spec Repo

Spec/progress files should be updated at these key stages:
- `taken` / taken in work
- `in-progress`
- `pr-created`
- `done`

The system should avoid ultra-granular noise; milestone-level updates are enough.

---

## 11. Supabase Data Model (Draft)

### `spec_task`
Tracks one spec as an executable work item.

Suggested fields:
- `id`
- `spec_repo`
- `spec_path`
- `spec_project`
- `feature_slug`
- `status`
- `complexity`
- `primary_repo`
- `related_repos`
- `current_run_id`
- `approval_required`
- `created_at`
- `updated_at`

### `repo_mapping`
Maps project/spec space to code repos.

Suggested fields:
- `id`
- `project_slug`
- `primary_repo_url`
- `related_repo_urls`
- `config_source`
- `override_active`
- `updated_at`

### `execution_run`
Tracks a concrete implementation run.

Suggested fields:
- `id`
- `spec_task_id`
- `agent_type`
- `run_status`
- `workspace_path`
- `branch_name`
- `commit_sha`
- `pr_url`
- `attempt_number`
- `test_summary`
- `self_review`
- `started_at`
- `finished_at`

### `approval_state`
Tracks waiting/approval flow.

Suggested fields:
- `id`
- `spec_task_id`
- `state` (`pending`, `approved`, `rejected`, `forced`)
- `reason`
- `requested_at`
- `resolved_at`

### `delivery_report`
Tracks what was delivered back to Telegram.

Suggested fields:
- `id`
- `spec_task_id`
- `run_id`
- `telegram_chat_id`
- `telegram_message_refs`
- `report_type`
- `sent_at`

---

## 12. Error Handling

### Duplicate work prevention
- Use internal Supabase state
- Also check spec/progress status in spec repo
- If conflict is found, report instead of starting blindly

### Incomplete or conflicting spec
- Generate technical review
- Highlight ambiguities / blockers
- Wait for approval

### Implementation failure
- Retry automatically
- If retries fail:
  - send detailed Telegram report
  - suggest next step
  - create linked GitHub issue in target code repo

### Manual conflict override
- Warn user if task is already running / has PR / waiting approval
- Allow manual force after warning

---

## 13. Implementation Phases

### Phase 1 — Intake and Task Registry
- Watch `specs` repo via webhook
- Add 10-minute fallback polling
- Telegram command intake
- Supabase schema for task registry
- Deduplication logic
- Old-spec audit flow

### Phase 2 — Planning and Approval
- Project/repo resolution
- Complexity classification
- Preliminary status report
- Approval flow for complex/unclear work

### Phase 3 — Execution Engine
- GitHub clone/pull workflow
- Fresh temp workspace per run
- Claude Code execution wrapper
- Retry logic
- Validation pipeline (tests/lint/typecheck/run/self-review)

### Phase 4 — Delivery and Traceability
- Commit/push/PR creation
- Spec progress updates
- Telegram delivery report
- Failure issue creation and linking

---

## 14. Definition of Done

MVP is complete when:
1. MaxAICTO detects new specs automatically
2. Max can trigger runs manually from Telegram
3. Agent maps specs to target repos
4. Agent produces preliminary status before coding
5. Simple tasks auto-start
6. Complex/unclear tasks require approval
7. Claude Code executes implementation in fresh cloned workspaces
8. Tests/checks/self-review run when applicable
9. Agent commits, pushes, and creates PRs
10. Agent updates spec/progress at key stages
11. Agent reports outcomes in Telegram
12. Failures retry, then generate actionable report + linked issue if unresolved
13. Old specs can be audited and proposed as candidates

---

## 15. Out of Scope / Future Ideas
- Developer team management
- Multi-user approvals
- Auto-merge to `main`
- Full roadmap planning
- Budgeting / velocity analytics
- Autonomous production deploys
