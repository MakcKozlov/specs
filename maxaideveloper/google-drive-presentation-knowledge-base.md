# Google Drive Presentation Knowledge Base

## Overview
Add a new feature to **MaxAiDeveloper** that lets team members search the company presentation library stored in Google Drive via Telegram. The bot should know what presentations exist, search by filename and content, return matching links, and support lightweight metadata in the form of one text comment per presentation.

The feature also includes a content-ingestion workflow: team members can upload presentation files into a dedicated Telegram topic, and the bot will assist with selecting an existing destination folder or creating a new one before uploading the file into the Google Drive knowledge base.

## Problem
The team has a large presentation library stored in a single Google Drive root folder with nested subfolders. People need a fast way to find the right deck without manually browsing Drive. They also need a simple workflow to keep the library fresh by uploading new files and organizing them into folders.

Today this knowledge is fragmented across Drive structure and individual team memory. The bot should become the access layer for this presentation base.

## Goals
- Let team members find presentation links from Telegram using natural-language queries
- Index the full Google Drive presentation tree under one root folder
- Search by both filename and presentation content
- Return the best link or clarify when several presentations match
- Show similar presentations when no exact match is found
- Allow one internal text comment per presentation
- Allow the team to upload new files through a dedicated Telegram topic and sort them into folders with bot assistance
- Restrict access to authorized Telegram users only

## Non-Goals
- Fine-grained multi-role permissions inside the allowed team
- Multiple comments, threaded discussion, or comment history per presentation
- Public access outside the authorized Telegram group
- Support for multiple unrelated Google Drive roots in MVP
- Complex document editing inside Telegram

## Users
### Primary users
- Team members who are part of the Telegram group **стройка** (`3784470567`)

### Unauthorized users
- Any Telegram user who is not a member of the allowed group
- Such users must not receive access to search results in private chat

## User Stories

### US1 — Search for a presentation
As a team member,
I want to ask the bot for a presentation in natural language,
So that I can quickly get the right Google Drive link.

#### Acceptance Criteria
- User can ask in Telegram for a presentation using a natural-language request
- Bot searches indexed presentations by filename and content
- If one strong match is found, bot returns the presentation link
- Response includes enough context to understand why this file is relevant (for example title/path/snippet)
- Only authorized users can use this flow

### US2 — Clarify ambiguous search results
As a team member,
I want the bot to ask a clarifying question when several files may match,
So that I can choose the correct presentation.

#### Acceptance Criteria
- If multiple strong matches are found, bot does not guess immediately
- Bot asks a clarifying question and/or presents several candidate files
- User can select or clarify which file they mean
- Bot then returns the correct Google Drive link

### US3 — Show similar presentations when exact match is not found
As a team member,
I want to see similar presentations if the exact one is not found,
So that I can still discover useful materials.

#### Acceptance Criteria
- If no exact match is found, bot responds with similar presentations
- Similarity can be based on filename, content, tags derived from indexing, or embeddings/search relevance
- Returned results include links to the suggested files
- Response clearly indicates that these are similar suggestions, not exact matches

### US4 — Keep the library indexed
As a system owner,
I want the Google Drive tree to be re-indexed automatically once per day,
So that new and changed presentations become searchable.

#### Acceptance Criteria
- Bot indexes one Google Drive root folder and all nested subfolders
- Scheduled sync runs once per day
- Sync detects newly added files and updates the index
- Index stores enough metadata for search and link retrieval
- Failures are logged and visible to maintainers

### US5 — Attach one internal comment to a presentation
As a team member,
I want to add one text comment to a presentation,
So that the team can store lightweight internal context about the file.

#### Acceptance Criteria
- Each indexed presentation can have one text comment
- Team members can create or update the comment
- Comment is visible in search context or in a dedicated details flow
- Latest saved comment replaces the previous one
- Only authorized users can edit comments

### US6 — Upload a new presentation from Telegram
As a team member,
I want to send a file into a dedicated Telegram topic,
So that the bot can help me add it into the presentation library.

#### Acceptance Criteria
- There is a dedicated Telegram topic for presentation intake
- Authorized team member can upload a presentation file into that topic
- Bot analyzes the file name/content enough to suggest relevant destination folders
- Bot presents several matching folders and an option to create a new folder
- After user confirmation, bot uploads the file to Google Drive into the selected folder
- Uploaded file becomes searchable after indexing or immediate post-upload refresh

### US7 — Create new folders during intake
As a team member,
I want to create a new destination folder during upload,
So that new categories can be added without leaving Telegram.

#### Acceptance Criteria
- During intake, bot can offer “create new folder”
- User can specify the new folder name
- Bot creates the folder under the configured Google Drive root or selected parent folder
- Bot uploads the file into the new folder
- New folder is included in future indexing

### US8 — Enforce access by Telegram group membership
As a system owner,
I want the feature to work only for members of the authorized Telegram group,
So that the presentation base is not exposed outside the team.

#### Acceptance Criteria
- Authorized group id is configurable and set to `3784470567`
- Before responding in private chat, bot checks whether the user is a member of the authorized group
- If user is not a member, bot does not provide search results or access to upload/comment flows
- Unauthorized access attempts are handled safely and do not leak file names or links

## Functional Requirements

### Search
- Support natural-language queries in Telegram
- Search over:
  - file name
  - file content / extracted text
  - folder path metadata
  - optional internal comment
- Return Google Drive shareable link for matched presentation
- If several strong matches exist, ask for clarification
- If no exact match exists, return similar files

### Indexing
- One configured Google Drive root folder
- Recursive indexing of all nested folders
- Daily sync job
- Track file metadata such as:
  - Google Drive file id
  - file name
  - file link
  - folder path
  - mime type / format
  - extracted text for search
  - last indexed timestamp
  - internal comment

### Comments
- One plain-text comment per presentation
- Comment can be created or updated by authorized team members
- Comment participates in search context if useful

### Intake Topic Workflow
- Dedicated Telegram topic for uploads
- Bot detects uploaded file
- Bot suggests several target folders
- Bot offers create-new-folder path
- User confirms destination
- Bot uploads the file to Google Drive
- Bot confirms success and resulting path/link

### Authorization
- Access limited to members of Telegram group `3784470567`
- Private-chat responses must be blocked for non-members
- Same authorization applies to search, commenting, and upload flows

## UX / Conversation Flow

### Search flow
1. User sends a message such as “скинь презентацию про X”
2. Bot checks authorization
3. Bot runs search across indexed presentation base
4. Outcomes:
   - single clear result → return link
   - several likely results → ask clarifying question
   - no exact result → show similar presentations

### Upload flow
1. User uploads file in the dedicated intake topic
2. Bot checks authorization
3. Bot analyzes file name/content
4. Bot suggests several matching folders
5. User either:
   - selects an existing folder, or
   - chooses to create a new folder
6. Bot uploads file to Google Drive
7. Bot confirms storage location and link
8. File becomes searchable

### Comment flow
1. Authorized user opens a file detail flow or uses a comment command/action
2. Bot shows current comment if present
3. User adds or replaces the text comment
4. Bot saves the latest comment

## Technical Design Notes
- Integration with Google Drive API for:
  - recursive folder traversal
  - file metadata retrieval
  - link storage
  - file upload
  - folder creation
- Integration with Telegram bot workflows for:
  - chat query handling
  - private chat auth gating
  - topic-based intake flow
  - clarification prompts and selection UX
- Search implementation may use:
  - metadata search
  - full-text extraction/indexing from presentations
  - semantic search / embeddings for “similar presentations” and content understanding
- Authorization should validate Telegram user membership in the allowed group before any sensitive response
- Index may be stored in the existing application database or a dedicated search store
- Consider event-driven partial refresh after successful upload to avoid waiting until next daily sync

## Edge Cases
- User requests a very broad topic and many files match
- Presentation file exists in Drive but text extraction fails
- File is uploaded in unsupported format
- User tries to upload/search in private chat but is not in the authorized group
- Folder suggestion confidence is low
- Folder creation name conflicts with an existing folder
- Drive permissions prevent the bot from reading or uploading a file
- Daily sync fails or times out
- File moved between folders after indexing

## Dependencies
- Google Drive API credentials and access to the root folder
- Telegram bot access to group membership checks and topic handling
- Storage/index layer for metadata and extracted content
- Content extraction pipeline for presentation formats (e.g. Google Slides, PPTX, PDF exports if relevant)
- Scheduler/cron for daily reindex

## MVP Scope
The MVP includes the full requested flow:
- Authorized team-only access
- Search by filename and content
- Clarification on ambiguous matches
- Similar result suggestions when exact match is absent
- Daily indexing of one Google Drive root tree
- One comment per presentation
- Dedicated Telegram topic for file uploads
- Folder suggestions during upload
- Create-new-folder option during upload
- Upload into Google Drive and make files searchable

## Out of Scope for MVP
- Multiple comments per file
- Role-based moderation/admin workflows
- Advanced taxonomy management beyond folders and one comment
- Multi-drive / multi-root support
- Rich approval workflow for uploads

## Phases

### Phase 1 — Searchable index
- Connect Google Drive root folder
- Build recursive indexer
- Extract searchable text/content
- Implement Telegram search flow
- Return links for exact matches

### Phase 2 — Ambiguity and recommendation UX
- Add clarification flow for multiple matches
- Add similar-results fallback when no exact match exists
- Surface folder path and short context in results

### Phase 3 — Comments and intake workflow
- Add one-comment-per-file support
- Create dedicated upload topic workflow
- Suggest folders for uploaded files
- Support create-new-folder flow
- Upload files to Google Drive and refresh index

### Phase 4 — Hardening and operations
- Daily scheduled reindex
- Auth enforcement by group membership
- Logging, monitoring, and failure handling
- Performance tuning for larger presentation bases

## Definition of Done
- Authorized team members can ask the bot for presentations and receive relevant Google Drive links
- Search works across file names and presentation content
- Bot asks clarifying questions for ambiguous matches
- Bot suggests similar files when exact match is unavailable
- Daily indexing keeps the searchable base fresh
- Each presentation supports one editable internal comment
- Team members can upload files in a dedicated Telegram topic and place them into existing or new folders
- Non-members of the authorized Telegram group cannot access the feature in private chat
- Specified root folder structure in Google Drive is indexed recursively
