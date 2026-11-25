---
name: stories
description: "Manage user stories: create, activate (with story branch), plan, implement, review against acceptance criteria, finish with learnings, glean their learnings into a curated project learnings file, and generate kanban board. Works across all projects."
argument-hint: "<command> [story-name | --all]"
allowed-tools: Read Write Edit Bash Glob Grep Agent
user-invocable: true
---

# User Story Manager

You manage user stories in the `user-stories/` directory of the current project. Stories move through three columns: `backlog/`, `current/`, and `done/`. A kanban board at `user-stories/index.html` reflects the current state.

Parse `$ARGUMENTS` for the command and story name. Match a story by filename, with or without `.md`. If `user-stories/` doesn't exist and the command isn't `init`, suggest `/stories init`. If the command isn't recognized, list the available commands.

## Commands

### `init`

Create `user-stories/` with `backlog/`, `current/`, `done/`, a `_template.md`, and the board. The template:

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

Generate the board with the `board` logic and an empty data set.

### `new <story-name>`

Create `user-stories/backlog/<kebab-case-name>.md` from `_template.md` (or the template above), with `[Title]` replaced by the title-cased name. Set `created_at`.

### `activate <story-name>` (alias: `start`)

Move a story from `backlog/` to `current/` on a new branch:

1. In a git repo, create `story/<story-name>` from the default branch. If the working tree is dirty, you're not on the default branch, or the branch already exists, stop and describe the situation so the user can decide.
2. After switching branches, move the file to `current/` (`git mv` in a repo, otherwise `mv`).
3. Set `activated_at`.

### `plan <story-name>`

Write an implementation plan for a story in `current/`:

1. Read the story, the project's CLAUDE.md, and the code the story touches. Find the closest existing feature and plan to follow its pattern.
2. Write the plan into `## Implementation Plan`, replacing what's there: the approach in a sentence or two, ordered steps with real file paths, data model changes (or "none"), the tests to add, and risks or open questions. For user-facing UI, include the states (empty, loading, error, overflow) and accessibility needs the step must handle, given the project's context.
3. Gaps in the story itself (missing acceptance criteria, unhandled edge cases, scope worth cutting) are not plan steps. List them separately, ask the user which to adopt, and apply the accepted ones to the story.
4. Set `planned_at`, then show the user the plan.

### `implement <story-name>`

1. Read the story. If `## Implementation Plan` is missing or still the placeholder, tell the user to run `/stories plan <story-name>` and stop.
2. Implement the plan, following the project's conventions. Use parallel agents only when steps are independent and doing them at once is meaningfully faster.
3. Run the project's tests (see CLAUDE.md) and check each acceptance criterion.
4. Report what changed, how the tests went, and which criteria still need manual verification.

### `review <story-name>`

Review the story branch against its acceptance criteria:

1. Read the story in `current/`. If it isn't there or has no acceptance criteria, say so and stop. If you're not on `story/<story-name>`, ask before continuing.
2. Collect the diff: `git diff $(git merge-base main HEAD)` (use the actual default branch), which includes uncommitted changes, plus any untracked files from `git status`. Note anything uncommitted. If there's no diff, say so and stop.
3. Spawn one general-purpose agent for an independent read; it hasn't seen the implementation, so it isn't anchored on it. Give it the full story, the diff command, and CLAUDE.md's location, but not your own conclusions. Ask it for:
   - a verdict on each acceptance criterion (✅ met, ⚠️ partial or needs manual check, ❌ not met) with `file:line` evidence, where a criterion with no evidence is ⚠️;
   - correctness bugs and convention problems in the diff, most severe first, confirmed in the source;
   - for UI changes, accessibility and usability problems (keyboard and touch operability, focus, contrast, target size, states).
4. Check its findings against the code, drop any you can't confirm, and write a `## Review` section into the story (replacing any earlier one) with the date, `git rev-parse --short HEAD`, the criteria table, and the findings.
5. Show the user the review, leading with the criteria verdicts and anything that blocks `finish`.

### `finish <story-name>`

1. Add a `## Learnings` section to the story: what went well, what was surprising, and what to do differently next time. Write only what is specific to this project or not generally known to Claude: a quirk of the codebase, domain, or tools, a preference the user showed, or an approach that failed for a reason you couldn't have predicted. Leave out general engineering practice, and write "None" when the story taught nothing of that kind. Don't ask the user to approve it; they can edit the file later.
2. Set `finished_at` and move the file to `done/` (`git mv` in a repo, otherwise `mv`).
3. If you're on a `story/` branch, bring the work onto the default branch (shown here as `main`; use the actual one):
   1. If the working tree has uncommitted changes other than the story file, stop and describe them so the user can decide. Otherwise commit the learnings and the move.
   2. Run `git pull --rebase origin main`, then `git switch main`, then `git rebase -` to fast-forward `main` onto the story branch.
   3. If any step hits a conflict, stop and describe it so the user can resolve it. Don't push.

### `glean [--all]` (alias: `gather-learnings`)

Distill the learnings of finished stories into the project's learnings file, and edit that file down to a short list worth reading before the next story:

1. Find the learnings file: `learnings.md` in any letter case, at the project root or in `docs/`. If there is none, the new file goes in `docs/` when the project has that directory, and at the root otherwise. Read the file if it exists, and note the `gleaned_at` timestamp in its opening comment.
2. Read the `## Learnings` sections of the stories in `done/` that are new since the last glean. Don't invent learnings for a story whose section is missing or says "None". If there is no file and no story has learnings, say so and stop.
   - A story is new when its `finished_at` or `updated_at` is later than `gleaned_at`, or, in a git repo, when `git log --since=<gleaned_at> --name-only -- user-stories/done/` lists it. The git check catches edits made by hand, which don't refresh `updated_at`.
   - Read every story when the file is missing, has no `gleaned_at`, or the command has `--all`.
   - The older stories still count toward recurrence. When a new learning doesn't qualify on its own in step 3, search their `## Learnings` sections for the same lesson before leaving it out.
3. Decide what belongs in the file. One bar applies to new learnings and to the entries already there. An entry belongs when it is both:
   - worth the space: the lesson recurs in two or more stories, or it appears once but ignoring it would be costly or would change how another story is planned, built, or reviewed; and
   - news to its reader: a capable agent new to this project would act differently for having read it. That rules out general engineering practice (read the code first, run the tests, keep commits small), Claude Code's default behavior, what CLAUDE.md already says, and what the code, tests, or linter enforce.
4. Edit the file so it reads as one document, not a log with a batch appended. Do this on every glean, even when no story adds anything:
   - Keep one entry per lesson. When a story supports an existing entry, add the story to its sources and sharpen the entry if the story adds something. Combine existing entries that teach the same lesson, keeping the sources of each.
   - Put each new entry under the theme heading it fits, and add a heading only when none fits. Move misplaced entries, merge overlapping headings, and drop empty ones.
   - Remove entries that fail the bar, and entries about code or process that has since changed. Hand-written entries get no exemption, but if you can't tell whether an entry still applies, keep it and ask the user.
   - If the stories contradict an existing entry, leave it alone and tell the user.
   - Tighten long or vague wording without changing what an entry says.
   - Follow the file's existing entry format. In a new file, start with the `gleaned_at` comment, then a title and a sentence on what the file is for, group entries under theme headings (planning, testing, and so on), and write each entry as a bullet: the lesson in bold, stated as guidance that applies beyond the story it came from; a sentence or two of concrete evidence; then the source stories as plain paths (`user-stories/done/<name>.md`).
5. Keep the file within 30 entries and 1,500 words (check with `wc -w`). If it is still over after step 4, cut the entries that matter least: those with one source story, narrow reach, or a low cost if ignored. A new learning goes in only if it is worth more than the entry it displaces.
6. Make sure the project's CLAUDE.md (at the root or in `.claude/`) points to the learnings file, giving its path and when to read it, such as before planning a story. If CLAUDE.md already names the path, leave it. Otherwise add the line to an existing references or docs section, or to a new `## References` section. Use a plain path, not an `@` import, so the file is read when needed instead of loaded into every session. If there is no CLAUDE.md, create one holding just this reference.
7. Set `gleaned_at` to the current time, in the timestamp format of the story metadata. It goes in a comment that opens the file, followed by a blank line; add the comment if the file lacks it:

   ```markdown
   <!--
   metadata:
     gleaned_at: 2026-06-14T11:45:33-07:00
   -->
   ```
8. Report the stories read, the entries added, the entries that gained sources, the entries removed or combined and why, the stories with no learnings, and any contradictions. Don't commit.

### `open`

Open `user-stories/index.html` in the default browser (`open` on macOS, `xdg-open` on Linux). If it doesn't exist, suggest `/stories init`.

### `board`

Regenerate `user-stories/index.html` by copying `board-template.html` from this skill's directory and replacing its `/*STORIES_DATA*/` placeholder with an object literal. Don't hand-write the board.

```js
{
  backlog: [ { title, path, story? }, ... ],
  active:  [ ... ],  // from current/
  done:    [ ... ],
}
```

- `title`: the story's first `# ` heading, without a leading `Story: `.
- `path`: relative to `user-stories/`, e.g. `backlog/foo.md`. It is both the card link and the modal's fetch target, so it must stay relative.
- `story`: an optional one-line blurb, written by hand. It can't be derived from the file, so carry it over from the existing `index.html` by matching `path`. If a regeneration would lose blurbs you can't recover, stop and tell the user.

Scan `backlog/`, `current/`, and `done/` recursively for `.md` files, excluding `_template.md`, and drop entries whose files are gone. Sort each column newest first: `backlog` by `updated_at`, `active` by `activated_at`, `done` by `finished_at`. Break ties by title, and sort missing timestamps last.

Regenerate the board after every command that creates, moves, or retitles a story.

## Story Metadata

Every story file starts with this block, followed by a blank line:

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

- All five keys are always present. An event that hasn't happened yet keeps an empty value.
- Timestamps are ISO 8601 with a colon in the offset, matching git's `%cI`. Get one with `date +%Y-%m-%dT%H:%M:%S%z` and change `-0700` to `-07:00`.
- Values are aligned one space after `activated_at:`, the longest key.
- When a command sets an event's timestamp, keep the other values.
- Refresh `updated_at` on every edit to a story file, whether or not the command says so. If a file lacks the block, add it.
