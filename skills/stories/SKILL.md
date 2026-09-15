---
name: stories
description: "Manage user stories: create, activate (with story branch), plan, implement (directly or with specialist agents), review against acceptance criteria, finish with learnings, and generate kanban board. Works across all projects."
argument-hint: "<command> [story-name]"
allowed-tools: Read Write Edit Bash Glob Grep Agent
user-invocable: true
---

# User Story Manager

You manage user stories in the `user-stories/` directory of the current project. Stories move through three columns: `backlog/`, `current/`, and `done/`. A kanban board at `user-stories/index.html` reflects the current state.

## Commands

Parse `$ARGUMENTS` to determine which command to run:

### `init`

Create the user-stories directory structure and kanban board:

```
user-stories/
  backlog/
  current/
  done/
  _template.md
  index.html
```

Create `_template.md` with this content (note the leading metadata block — see [Story Metadata](#story-metadata)):

```markdown
<!--
metadata:
  created_at:
  activated_at:
  planned_at:
  finished_at:
  updated_at:
-->

# Story: [Title]

## Summary

AS a [user type]
I WANT [goal]
SO THAT [benefit]

## Acceptance Criteria

- [criterion 1]
- [criterion 2]

## Notes

[any additional context]

## Implementation Plan

[to be filled in by /stories plan]
```

Then generate the kanban board with the `board` logic — this copies the bundled `board-template.html` and injects an empty data set (`{ backlog: [], active: [], done: [] }`), giving a fresh project the full-featured board (markdown modal, collapsible columns, optional story blurbs, responsive light theme) from the start.

### `new <story-name>`

Create a new story file in `user-stories/backlog/`. The story name argument becomes the filename (kebab-cased, with `.md` extension). Copy from `_template.md` if it exists, otherwise use the template above. Replace `[Title]` with a title-cased version of the story name.

Ensure the file starts with the metadata block (add it if the template lacked one) and set `created_at` and `updated_at` to the current timestamp — see [Story Metadata](#story-metadata).

After creating the file, regenerate the board with the `board` logic.

### `activate <story-name>`

Move a story from `backlog/` to `current/` and cut a working branch:

1. Find the `.md` file matching `<story-name>` in `user-stories/backlog/` (match by filename with or without `.md` extension)
2. If in a git repo, create a story branch from `main`:
   - The branch name is `story/<story-name>` (e.g. `story/add-new-dinos`)
   - Before creating it, check the repo state. If the working tree has uncommitted changes, the current branch isn't `main`, or the branch `story/<story-name>` already exists, **stop and describe the situation to the user** and let them decide how to proceed (e.g. stash, commit, branch from the current HEAD instead, or reuse the existing branch). Do not silently work around it.
   - If the default branch is named something other than `main` (e.g. `master`), use that as the base and mention it.
   - Otherwise, run `git checkout -b story/<story-name> main`
3. Move the story file to `user-stories/current/` using `git mv` if in a git repo, otherwise `mv` (do this after switching to the story branch, so the move is committed on the branch)
4. Set `activated_at` (and refresh `updated_at`) in the metadata block — see [Story Metadata](#story-metadata)
5. Regenerate the board

Note `start` works as an alias for `activate`

### `plan <story-name>`

Build an implementation plan for a story in `current/`:

1. Read the story file from `user-stories/current/<story-name>.md`
2. Read the project's CLAUDE.md and explore relevant parts of the codebase to understand the tech stack and architecture
3. Spawn a **story-planner** agent using the Agent tool:
   - `subagent_type: "story-planner"`
   - Pass the full story content, project context, and tech stack in the prompt
   - The planner picks its team from what the story touches (product-manager and developer always; ui-ux-designer and accessibility-specialist only for UI work; best-practices-engineer only for real architectural, performance, or security surface), spawns them in parallel, and returns a consolidated plan
4. If the returned plan has a `### Proposed story changes` section, split it off: it contains acceptance criteria, edge cases, and scope changes for the story itself, not implementation steps. Present them to the user and ask which to adopt; apply the accepted ones to the story's acceptance criteria and scope sections. Leave this section out of the written plan.
5. Write the rest of the plan into the story file's `## Implementation Plan` section (replace the placeholder or existing content)
6. Set `planned_at` (and refresh `updated_at`) in the metadata block — see [Story Metadata](#story-metadata)
7. Present the plan to the user for review

### `implement <story-name>`

Execute the implementation plan for a story in `current/`:

1. Read the story file from `user-stories/current/<story-name>.md`
2. Look for an `## Implementation Plan` section. If it's missing or still contains the placeholder text `[to be filled in by /stories plan]`, tell the user to run `/stories plan <story-name>` first and stop.
3. Read the project's CLAUDE.md and explore the codebase to understand the tech stack, architecture, and conventions.
4. Parse the implementation plan into discrete steps.
5. **Decide how to execute — directly or with specialist agents.** Delegation costs a round trip and a fresh context; it pays off only when a step needs a perspective or an amount of exploration you don't already have. Judge this per step, not for the whole plan.

   **Implement the step yourself** when all of these hold:
   - It is ordinary code work — the kind of change the plan already spells out — with no open design, scope, or accessibility question.
   - You have already read the files it touches (or they are few and small enough to read now).
   - It is confined to a handful of files in one area of the codebase.

   **Spawn a specialist agent** when any of these hold — pick the agent by what the step actually needs:
   - **UI/component creation or visual styling** with real design latitude → **ui-ux-designer**
   - **Accessibility requirements** beyond obvious markup hygiene → **accessibility-specialist**
   - **Design-system or best-practices judgment calls** → **best-practices-engineer**
   - **Unresolved scope or acceptance-criteria questions** → **product-manager**
   - **Code work in an unfamiliar or large area** that would need substantial exploration before you could write it → **developer** (it plans by default; tell it to use implement mode)
   - Several independent steps could be built in parallel, and doing so is meaningfully faster.

   A simple plan often needs no agents at all. A plan may also be mixed — do the routine steps yourself and delegate only the ones above. Say briefly which way you're going and why before starting, so the user can redirect.

6. Execute the plan step by step, in the mode chosen for each step:
   - **Direct**: make the edits yourself, following the plan and the project's conventions.
   - **Delegated**: spawn the specialist agent(s) with the Agent tool. Spawn independent steps in parallel; run a step sequentially when it depends on a prior step's output. Each agent prompt must include the full story content, the specific step to implement, the project context (tech stack, conventions, relevant file contents), and explicit instructions to write code (not just research). Review each agent's work before moving on; if its output needs correction, fix it directly.

7. After all steps are complete:
   - Run the project's test suite if one exists (check CLAUDE.md for the test command)
   - Verify the acceptance criteria from the story are met
   - Refresh `updated_at` in the metadata block — see [Story Metadata](#story-metadata)
   - Report a summary of what was implemented: files created/modified, key decisions made, whether the work was done directly or delegated, and any acceptance criteria that still need manual verification

**Important**: Every specialist agent is advisory by default. Agents doing implementation work must be told explicitly to use their **implement mode** and to **write code, create files, and make edits** — not just provide recommendations. Pass them specific file paths and the current file contents when relevant.

### `review <story-name>`

Review the story branch's changes against the story's acceptance criteria:

1. Read the story file from `user-stories/current/<story-name>.md` and extract the `## Acceptance Criteria` section. If the story isn't in `current/` or has no acceptance criteria, tell the user and stop.
2. Determine the diff to review:
   - Identify the story branch (`story/<story-name>`). If the current branch isn't the story branch, tell the user and ask before proceeding.
   - Diff against the base branch using the merge-base: `git diff $(git merge-base main HEAD)...HEAD` (substitute the actual default branch if it isn't `main`). Include uncommitted changes in the review and note that they are uncommitted.
   - If there are no changes to review, say so and stop.
3. Spawn specialist review agents **in parallel** using the Agent tool. Each prompt must include the full story content (especially the acceptance criteria), the diff (or the list of changed files with instructions to read them), and project context:
   - **product-manager** — tell it to use **acceptance verification mode**: verify each acceptance criterion against the actual changes and return, for each criterion, a verdict of ✅ met, ⚠️ partially met / needs manual verification, or ❌ not met, with evidence (file paths and what was found).
   - **pr-review-toolkit:code-reviewer** — review the changed code for quality, project conventions, and potential issues.
   - If the changes include UI work, also spawn a **ui-ux-designer**; if they touch user-facing markup or interactions, also spawn an **accessibility-specialist** in audit mode. Skip these when not relevant.
4. Consolidate the agents' findings into a review with two parts:
   - **Acceptance criteria**: a per-criterion checklist with verdict and evidence
   - **Code review findings**: notable issues or suggestions, ordered by severity
5. Write the consolidated review into a `## Review` section in the story file (replace any existing `## Review` section — it represents the latest review). Include the date and the commit reviewed (`git rev-parse --short HEAD`). Refresh `updated_at` in the metadata block — see [Story Metadata](#story-metadata).
6. Present the review to the user in the conversation, leading with the acceptance-criteria verdicts and calling out anything that blocks `finish`.

### `finish <story-name>`

Close out a story:

1. Find the story in `user-stories/current/`
2. Summarize what we learned in the planning and implementation flow — what went well, what was surprising, what to do differently next time. Do not ask the user to approve or adjust the learnings; the story file is editable after the fact.
3. Add a `## Learnings` section to the story file with the essential contents of that reflection.
4. Set `finished_at` (and refresh `updated_at`) in the metadata block — see [Story Metadata](#story-metadata)
5. Move the file from `current/` to `done/` using `git mv` if in a git repo, otherwise `mv`
6. Regenerate the board

### `open`

Open the kanban board in the user's default browser:

1. Check that `user-stories/index.html` exists. If not, suggest running `/stories init` first.
2. Run `open user-stories/index.html` (macOS) to open it in the default browser.

### `board`

Regenerate `user-stories/index.html` from the current directory state, using the bundled template at `board-template.html` (in this skill's directory). The template renders a feature-rich board — a click-to-open modal that fetches and renders each story's markdown (including GFM task-list checkboxes), collapsible columns whose state persists in `localStorage`, optional per-story description blurbs, and a responsive light theme. Do **not** hand-write a board from scratch; populate the template.

The board is data-driven: the template contains a single placeholder, `/*STORIES_DATA*/`, where a JavaScript object literal is injected:

```js
{
  backlog: [ { title, path, story? }, ... ],
  active:  [ ... ],  // sourced from the current/ directory
  done:    [ ... ],
}
```

Each entry's fields:
- `title` — from the first `# ` heading in the story file (strip a leading `Story: ` prefix if present).
- `path` — relative path from `user-stories/` to the `.md` file (e.g. `backlog/foo.md`). Used both as the card link and the modal fetch target, so it must stay relative.
- `story` — an optional one-line description shown in italic on the card. **Curated**; not derivable from the file. Omit when unknown.

**Preserve the curated `story` blurbs.** They are hand-maintained and cannot be reconstructed from the files. Before writing, parse the existing `index.html`'s `stories` object (if the file exists) into a map keyed by `path`. When rebuilding:

1. Scan `user-stories/backlog/`, `user-stories/current/`, and `user-stories/done/` recursively for `.md` files (exclude `_template.md`).
2. For each file found, build an entry: refresh `title` from the heading; carry over `story` from the matching existing entry by `path` if present, otherwise leave it absent.
3. Drop entries whose files no longer exist.
4. Place each entry in the column matching its top-level directory — `backlog/` → `backlog`, `current/` → `active`, `done/` → `done`.
5. Sort each column using the timestamps in each story's [metadata block](#story-metadata), newest first:
   - **`done`** — by `finished_at` **descending** (most recently completed at the top).
   - **`backlog`** — by `updated_at` **descending** (most recently touched at the top).
   - **`active`** — by `activated_at` descending.
   - Tie-break on equal timestamps by `title` ascending, so the order stays stable across regenerations. If a timestamp is missing, treat it as the oldest (sorts to the bottom).
6. Inject the resulting object literal at `/*STORIES_DATA*/` in a copy of `board-template.html` and write it to `user-stories/index.html`.

When in doubt about a regeneration that would lose curated `story` values you cannot recover, stop and tell the user rather than overwriting.

## Story Metadata

Every story file begins with an HTML-comment block that records lifecycle timestamps. It sits at the very top of the file, before the `# Story:` heading, followed by a blank line:

```markdown
<!--
metadata:
  created_at:   2026-06-14T11:02:46-07:00
  activated_at: 2026-06-14T11:45:33-07:00
  planned_at:
  finished_at:
  updated_at:   2026-06-14T11:45:33-07:00
-->
```

Rules:

- **All five keys are always present.** Fill a value only when its status event has happened; otherwise leave the value empty (`planned_at:` with nothing after the colon — no placeholder text).
- **Timestamps are ISO 8601 with a timezone offset** (e.g. `2026-06-14T11:45:33-07:00`). Get the current time with `date +%Y-%m-%dT%H:%M:%S%z`, then insert a colon into the offset (`-0700` → `-07:00`) so it matches `git`'s `%cI` format.
- **Align values** one space after the longest key, `activated_at:` (so the other keys get extra padding before their value).
- **Preserve existing values when editing.** Set only the timestamp for the event you're performing, and always refresh `updated_at` to the current time.

Which command sets what:

| Command | Sets |
| --- | --- |
| `new` | `created_at`, `updated_at` |
| `activate` | `activated_at`, `updated_at` |
| `plan` | `planned_at`, `updated_at` |
| `implement` / `review` | `updated_at` |
| `finish` | `finished_at`, `updated_at` |

Any other edit you make to a story file should also refresh `updated_at`.

## General Rules

- Always use relative links in the kanban board (e.g., `backlog/story.md`, not absolute paths)
- Never regenerate the board by hand-writing HTML — always populate `board-template.html`, and always preserve curated `story` blurbs (see the `board` command)
- Story filenames should be kebab-case: "Add New Dinos" becomes `add-new-dinos.md`
- When a command is not recognized, show available commands
- After any mutation (new, activate, implement, finish), always regenerate the board
- When regenerating the board, sort each column by its metadata timestamp descending — `done` by `finished_at`, `backlog` by `updated_at`, `active` by `activated_at` (see the `board` command)
- Keep the metadata block current: set the status event's timestamp and refresh `updated_at` whenever you touch a story file (see [Story Metadata](#story-metadata))
- If `user-stories/` doesn't exist and the command isn't `init`, suggest running `/stories init` first
