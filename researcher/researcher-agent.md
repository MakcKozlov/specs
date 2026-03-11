# Researcher Agent -- MVP Specification

## 1. Overview

**Researcher** is a personal Telegram bot (single-user) that accepts free-form research requests along with files and links, processes them through **Google NotebookLM** via the `notebooklm-py` library, and returns generated research outputs back to Telegram.

The bot lives as a **separate repository/project** that later plugs into the user's **OpenClaw** environment as a standalone agent with its own Telegram routing.

### Key Integration Points

| Component | Role |
|-----------|------|
| **Telegram** | User interface (only interface in MVP) |
| **OpenClaw** | Runtime / routing layer on Mac mini |
| **NotebookLM** | Main research execution engine |
| **notebooklm-py** | Python adapter for NotebookLM ([GitHub](https://github.com/teng-lin/notebooklm-py)) |
| **Supabase** | Metadata index, project registry, search layer |

---

## 2. Problem Statement

Currently the user has to manually:
- Collect research materials from multiple formats (PDF, YouTube, audio, links, images, video)
- Upload them into Google NotebookLM by hand
- Track separate research projects across sessions
- Remember which materials were already submitted
- Return to old research and find past results
- Manage parallel research tasks simultaneously

**Researcher** solves this by providing a single Telegram-based entry point where the user sends materials and instructions, and the agent handles everything else -- ingestion, NotebookLM integration, status tracking, persistence, and result delivery.

---

## 3. Architecture

```
User
  |
  v
Telegram Chat
  |
  v
OpenClaw (Mac mini)
  |
  v
Researcher Agent
  |
  +---> File Intake Layer (download, normalize, convert)
  |
  +---> NotebookLM Adapter (notebooklm-py)
  |       |
  |       +---> Create notebook/project
  |       +---> Upload sources
  |       +---> Trigger result generation
  |       +---> Retrieve results
  |
  +---> Supabase Layer
  |       |
  |       +---> Projects registry
  |       +---> Materials registry
  |       +---> Status tracking
  |       +---> Search index
  |       +---> Result metadata
  |
  +---> Result Delivery (Telegram messages, files, attachments)
```

### Suggested Code Layout

```
researcher/
  src/
    telegram/
      handlers/
      bot.py
    core/
      projects.py
      orchestration.py
      statuses.py
      grouping.py
    integrations/
      notebooklm/
        client.py
        adapter.py
      supabase/
        client.py
        repositories.py
    workers/
      tasks.py
      retries.py
    models/
      project.py
      material.py
      result.py
    utils/
      files.py
      converters.py
      logging.py
  .env
  requirements.txt
```

---

## 4. User Stories and Acceptance Criteria

### US-1: Create a New Research Task

**As a user**, I want to send a text request and materials in Telegram so the bot creates a research task.

- [ ] User can send a free-form text request
- [ ] User can attach one or more materials (files, links)
- [ ] Bot creates a new research project
- [ ] Messages within a 2-minute window are grouped into the same project
- [ ] Bot sends confirmation that the task has been accepted

### US-2: Support Mixed Input Types

**As a user**, I want to send various material types so I can use one interface for all research.

- [ ] Bot accepts plain links
- [ ] Bot accepts YouTube URLs
- [ ] Bot accepts PDF files
- [ ] Bot accepts audio files
- [ ] Bot accepts images
- [ ] Bot accepts video files
- [ ] Bot accepts other files when format is supported by NotebookLM/notebooklm-py

### US-3: Execute Research Through NotebookLM

**As a user**, I want the bot to use NotebookLM as the main engine for generating research outputs.

- [ ] Bot uses `notebooklm-py` as the integration layer
- [ ] Materials are uploaded to NotebookLM
- [ ] Requested output type is triggered within NotebookLM capabilities
- [ ] Final result is returned to Telegram

### US-4: Free-Form Intent (No Menu)

**As a user**, I want to write what I need in natural language without menus or fixed commands.

- [ ] User can write requests naturally (e.g., "make an audio review", "prepare a summary")
- [ ] Bot interprets requests within NotebookLM-supported capabilities
- [ ] No menu-first UX required

### US-5: Progress Status Updates

**As a user**, I want to see processing progress so I understand what is happening.

- [ ] Status 1: Task accepted
- [ ] Status 2: Materials uploaded / preparing
- [ ] Status 3: Sent to NotebookLM
- [ ] Status 4: Result generating
- [ ] Status 5: Result ready

### US-6: Return Results to Telegram

**As a user**, I want to receive the research result in the same Telegram chat.

- [ ] Result is sent to the same chat
- [ ] Bot can send a single message
- [ ] Bot can send multiple messages if the result format requires it
- [ ] Bot can attach files (audio, images, documents) when needed

### US-7: Context Memory (24 Hours)

**As a user**, I want to continue discussing the current research project for a limited time.

- [ ] Bot remembers current task context for 24 hours
- [ ] User can send follow-up requests within that window (e.g., "make it shorter", "translate it", "now make an audio version")
- [ ] After 24 hours the task is no longer the active context by default

### US-8: Parallel Projects

**As a user**, I want to run multiple research tasks simultaneously.

- [ ] User can have multiple active projects at the same time
- [ ] Each project maps to a separate NotebookLM notebook/project
- [ ] Bot tracks status of each project independently

### US-9: Persist Project Metadata in Supabase

**As a user**, I want my research projects and metadata to be saved.

- [ ] Each project stores: project_id, user_id, project_name, original_user_request, attached_materials, status, notebooklm_project_id, created_at, updated_at, result_type
- [ ] Result metadata also stored: telegram_message_refs, supabase_result_summary, notebooklm_result_ref

### US-10: Auto-Name Projects with Rename

**As a user**, I want projects to be named automatically but allow me to rename them later.

- [ ] Project name is auto-generated from the user request
- [ ] User can rename the project later
- [ ] Updated name is saved in Supabase and used in search

### US-11: Search Past Projects

**As a user**, I want to find old research projects.

- [ ] Search by project name
- [ ] Search by original user request text
- [ ] Search by attached links/files
- [ ] Search by status
- [ ] Search by date

### US-12: List All Projects

**As a user**, I want to see a list of all my research projects.

- [ ] Bot shows a paginated list of projects
- [ ] Each entry shows: project name, creation date, current status, material count, short snippet of original request

### US-13: Add Material to Existing Project

**As a user**, I want to add new material to an already existing project later.

- [ ] User can write "add this file to project X"
- [ ] Bot resolves the target project
- [ ] Material is attached to the project
- [ ] Project is updated in both Supabase and NotebookLM

### US-14: View Project Contents

**As a user**, I want to see which materials belong to a project.

- [ ] Bot shows a list of materials for the project
- [ ] Each material shows: type/name, date added, processing status
- [ ] Material statuses: received, uploading, added_to_notebooklm, error, used_in_result

### US-15: Handle Unsupported Formats Gracefully

**As a user**, I want the bot to not fail entirely because of one unsupported file.

- [ ] Bot attempts automatic conversion for unsupported formats
- [ ] If conversion fails, bot notifies the user about the specific file
- [ ] Remaining materials continue processing normally
- [ ] Failed file gets "error" status

### US-16: Retry on Generation Errors

**As a user**, I want the bot to be resilient to temporary integration failures.

- [ ] Bot automatically retries on temporary NotebookLM failures
- [ ] If retries fail, bot shows the final error reason to the user
- [ ] Error is tracked at project level

### US-17: Cancel and Delete Projects

**As a user**, I want to manage my research projects lifecycle.

- [ ] User can cancel an active task
- [ ] User can delete an old project
- [ ] Deletion removes data from both Supabase and NotebookLM
- [ ] Both cancel and delete require confirmation via inline buttons (Confirm / Cancel)

### US-18: File Size Limit Handling

**As a user**, I want clear error messages when files exceed size limits.

- [ ] Bot relies on Telegram and NotebookLM native limits (no custom internal limits in MVP)
- [ ] When a limit is exceeded, bot explains the error clearly to the user

---

## 5. Technical Design

### 5.1 Data Model

#### Table: `research_projects`

| Column | Type | Description |
|--------|------|-------------|
| `project_id` | uuid, PK | Unique project identifier |
| `user_id` | text | Telegram user ID |
| `project_name` | text | Auto-generated or user-renamed name |
| `original_user_request` | text | First user message that created the project |
| `status` | text | Current project status |
| `notebooklm_project_id` | text | Reference to NotebookLM notebook/project |
| `result_type` | text | Type of last generated result |
| `result_ref` | text | NotebookLM result reference/link/ID |
| `result_summary` | text | Basic result metadata saved in Supabase |
| `created_at` | timestamptz | Creation time |
| `updated_at` | timestamptz | Last update time |

#### Table: `research_materials`

| Column | Type | Description |
|--------|------|-------------|
| `material_id` | uuid, PK | Unique material identifier |
| `project_id` | uuid, FK | Reference to research_projects |
| `material_type` | text | Type: link, youtube, pdf, audio, image, video, file |
| `source_value` | text | URL, file path, or Telegram file_id |
| `display_name` | text | Human-readable name |
| `added_at` | timestamptz | When the material was added |
| `status` | text | received, uploading, added_to_notebooklm, error, used_in_result |

### 5.2 Project Status Flow

```
new -> materials_preparing -> sent_to_notebooklm -> generating -> completed
                                                               -> error
                                                   -> cancelled
```

### 5.3 Message Grouping Logic

- When a new message arrives, check if there is an active project created within the last 2 minutes
- If yes, attach the new message/material to that project
- If no, create a new project
- User can bypass grouping by explicitly referencing an existing project ("add to project X")

### 5.4 Components

| Component | Responsibility |
|-----------|---------------|
| `telegram/bot.py` | Telegram polling, message routing |
| `telegram/handlers/` | Per-action handlers (new task, search, list, cancel, delete, rename, add-to-project) |
| `core/projects.py` | Project CRUD, naming, state machine |
| `core/orchestration.py` | End-to-end workflow: intake -> upload -> generate -> deliver |
| `core/grouping.py` | 2-minute window grouping logic |
| `core/statuses.py` | Status update pipeline, Telegram status messages |
| `integrations/notebooklm/adapter.py` | Wrapper around notebooklm-py: create notebook, upload, generate, delete |
| `integrations/supabase/repositories.py` | CRUD + search queries for projects and materials |
| `workers/tasks.py` | Background/async task execution |
| `workers/retries.py` | Retry logic with exponential backoff |
| `utils/files.py` | File download from Telegram, temp storage |
| `utils/converters.py` | Format conversion for unsupported types |

### 5.5 Confirmation Flow (Inline Buttons)

For dangerous actions (cancel task, delete project):
1. User requests action
2. Bot sends message with inline keyboard: `[Confirm] [Cancel]`
3. On Confirm: execute the action
4. On Cancel: dismiss and notify

---

## 6. Implementation Phases

### Phase 1 -- Core Ingestion
**Goal**: Accept messages and files in Telegram, create research projects.

Tasks:
- Set up Telegram bot with polling
- Accept text messages, files, links
- Implement 2-minute message grouping into projects
- Auto-generate project names
- Send "task accepted" status
- Store project in Supabase

**Verification**: User sends a message + file -> project is created in Supabase -> user sees "task accepted" status.

### Phase 2 -- NotebookLM Integration
**Goal**: Connect to NotebookLM, upload materials, trigger generation, return results.

Tasks:
- Integrate notebooklm-py library
- Create NotebookLM notebook per project
- Upload supported materials
- Trigger result generation based on user request
- Return results to Telegram (text, files, multiple messages)
- Implement retry logic (automatic retries with exponential backoff)
- Show all 5 status stages during processing

**Verification**: User sends "make an audio review" + PDF -> bot processes through NotebookLM -> result returned in Telegram.

### Phase 3 -- Persistence and Search
**Goal**: Full Supabase persistence, search, and project listing.

Tasks:
- Finalize Supabase schema (projects + materials tables)
- Persist all project metadata and material statuses
- Implement search across 5 fields (name, request, files, status, date)
- Implement project listing (name, date, status, material count, request snippet)
- Implement project contents view (material + date + status)
- Implement project rename

**Verification**: User searches "find my research about X" -> bot returns matching projects. User lists projects -> sees all with metadata.

### Phase 4 -- Project Lifecycle
**Goal**: Full project management -- add files, cancel, delete, follow-up.

Tasks:
- Add material to existing project by name
- Cancel active task (with button confirmation)
- Delete project from both Supabase and NotebookLM (with button confirmation)
- 24-hour context memory for follow-up requests
- Handle unsupported formats (auto-convert, then skip with error)
- Handle file size limit errors with clear messages

**Verification**: User adds a file to an existing project -> material appears in project. User deletes a project -> removed from both Supabase and NotebookLM. User sends follow-up within 24h -> bot understands context.

---

## 7. Definition of Done

MVP is considered complete when:

1. User can send a free-form research request in Telegram
2. User can attach one or more supported materials (links, YouTube, PDF, audio, images, video)
3. Bot creates a research project and shows step-by-step progress
4. Materials are sent to NotebookLM via notebooklm-py
5. Bot returns generated results to Telegram
6. Result metadata is stored in Supabase + NotebookLM result ref
7. Projects are searchable by name, request, files, status, and date
8. User can list all projects with summary info
9. User can add files to an existing project
10. User can view project contents (materials + dates + statuses)
11. User can rename a project
12. User can cancel active tasks and delete old projects (with button confirmation)
13. Errors are retried automatically and reported clearly to the user
14. Unsupported formats are auto-converted or gracefully skipped

---

## 8. Open Questions

1. **notebooklm-py capabilities**: What is the actual feature coverage of the library? Need to verify supported input types, output types, and limitations before Phase 2.
2. **NotebookLM rate limits**: Are there API/automation limits that affect parallel project processing?
3. **OpenClaw integration details**: How exactly does the agent plug into OpenClaw routing? What interface does the agent need to expose?
4. **Authentication**: How does the bot authenticate with NotebookLM through notebooklm-py (Google account, API key, cookies)?
5. **Result format mapping**: How does the bot map free-form user requests (e.g., "audio review") to specific NotebookLM generation actions?
6. **VPS migration**: When moving from Mac mini to VPS, what changes are needed (webhook setup, process management)?
7. **Supabase project**: Is this the same Supabase instance used by MaxAiDeveloper or a new project?

---

## 9. Out of Scope (MVP)

- Multi-user mode
- Shared/team projects
- Web UI
- Advanced analytics over research history
- Complex multi-agent orchestration beyond base NotebookLM integration
- Native voice control
- Full artifact history within a project (only basic result metadata stored)

---

## 10. Technical Constraints

- **Single-user personal bot** -- no auth/permissions model needed in MVP
- **Telegram is the only interface** -- no web dashboard
- **NotebookLM is the main execution engine** -- results limited by NotebookLM capabilities
- **Supabase is the metadata/index layer** -- not the content engine
- **Free-form natural language input** -- hard requirement, no menu-first UX
- **Parallel projects** -- hard requirement
- **Search across old projects** -- hard requirement
- **Python** -- recommended stack (notebooklm-py is Python-native)
- **Mac mini first** -- local development, deploy-ready architecture for later VPS migration
