# Feature Specifications

Catalog of feature specs gathered by **Lisa** (SpecAgent) through structured PDR interviews.

## How It Works

1. **Lisa** conducts an interview in Telegram — gathers requirements, user stories, acceptance criteria
2. Spec files committed here, organized by project
3. **Dev agent** picks up specs, implements, updates progress

## Projects

| Project | Path | Description |
|---------|------|-------------|
| MaxAiDeveloper | [`maxaideveloper/`](maxaideveloper/) | Telegram knowledge bot |
| meeting-recorder | [`meeting-recorder/`](meeting-recorder/) | iPad meeting recorder app |
| PostRepost | [`postrepost/`](postrepost/) | Telegram media reposter |
| Researcher | [`researcher/`](researcher/) | Personal Telegram research agent (NotebookLM + Supabase) |
| SpecAgent | [`specagent/`](specagent/) | Lisa interview bot itself |

## Spec Format

Each feature produces 3 files:

- `{feature-slug}.md` — full specification (user stories, acceptance criteria, technical design, phases)
- `{feature-slug}.json` — structured data for programmatic access
- `{feature-slug}-progress.txt` — phase checklist with `[x]`/`[ ]` markers

## Statuses

- **new** — spec created, not started
- **in-progress** — dev agent working on it
- **done** — all phases complete
