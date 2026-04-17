---
name: full-migration
description: Convert any project to the three-file plus beads workflow from scratch. Reads existing docs and code to construct CLAUDE.md, PRD.md, plan.MD, and a beads task structure. Use when the user says "set up the workflow on this project", "do a full migration", "convert this project to beads", or types /full-migration. Use when the project has no existing CLAUDE.md/PRD.md/plan.MD structure — for projects already on the three-file system, use /migrate-to-beads instead.
---

# Full Migration

Convert a project that isn't using the three-file system into one that is, with beads task tracking on top. This is more expensive than `/migrate-to-beads` because it has to construct documentation from scratch, often by reading code.

Do NOT write any application code during this migration.

## Step 1: Cost warning

Tell the user up front:

> This migration may consume significant tokens depending on what's already documented. If your project has existing markdown docs (README, design notes, specs, etc.), the migration is cheaper because I can extract structure from them. If documentation is sparse, I'll need to read code to infer the project's architecture, conventions, and current state — which is much more expensive.

Then ask:

1. **Do you have any existing planning or spec documents?** (README, design docs, spec files, planning notes — any markdown or text files describing what the project is or what's planned.) List paths if yes.
2. **Are there any unbuilt features or known next steps you can describe in chat?** Even a few sentences helps avoid expensive code-reading to figure out what's next.
3. **Roughly how large is the codebase?** (Number of source files, or "small/medium/large".) This affects how much I'll need to read to construct CLAUDE.md.
4. **Any specific files I should focus on?** (Key entry points, main modules, etc.)

Wait for answers before proceeding. The answers determine how aggressive the code-reading needs to be.

## Step 2: Verify prerequisites

Check:

- `beads` is installed: `bd version`. If not, stop — user installs it first.
- The project is in a git repo: `git status`. If not, stop — initialize git first.
- Working tree is clean. If not, stop — user resolves uncommitted work first.

## Step 3: Inventory existing docs

Read every doc the user listed in Step 1. Also check common locations:

- `README.md`, `README.MD`, `README.txt`
- `docs/` directory (list contents, read top-level files)
- `CONTRIBUTING.md`, `ARCHITECTURE.md`, `DESIGN.md`
- Any `.md` files at the repo root

For each doc, note:

- What it describes (project overview, architecture, contribution rules, deployment, etc.)
- Whether content should migrate to CLAUDE.md, PRD.md, or stay where it is.
- Whether it's stale (references things that no longer exist).

## Step 4: Inventory the codebase (lightweight)

Even with good docs, do a minimal pass to confirm what's actually there:

- `git log --oneline -20` — recent activity, what's been worked on
- Top-level directory listing
- `package.json` / `Cargo.toml` / `go.mod` / `pyproject.toml` / etc. — language, dependencies, scripts
- Test directory existence
- CI config existence

If the user said docs are sparse and the codebase is medium/large, ask before doing a deep read:

> To construct an accurate CLAUDE.md, I'll need to read the main source files to infer architecture and conventions. Estimated files to read: [N]. Proceed, or do you want to provide a quick verbal description first?

A verbal description from the user is almost always cheaper than reading code.

## Step 5: Propose document structure

Based on what was found, propose:

**For CLAUDE.md:**
- Stack and dependencies (extract from package files).
- Project structure overview.
- Build/test/run commands (extract from package.json scripts, Makefile, etc.).
- Conventions inferred from existing docs or code (naming, testing patterns, file organization).
- Standing instructions (suggest the defaults: wait for approval, frequent commits, task IDs in commit messages, don't read scratch.md).

**For PRD.md:**
- Project purpose and high-level goals (from README or user description).
- Features that exist today (from README features section, or inferred).
- Features planned but not built (from user input in Step 1, or design docs).
- Acceptance criteria where known.

**For plan.MD:**
- Overall Plan: high-level roadmap. If the user has phases or milestones in mind, capture them. Otherwise leave it as a single phase.
- Current Status: one entry — "[date] Migrated to three-file plus beads workflow. Project state at migration: [brief summary of what's built]."
- Known Issues / Blockers: anything the user mentioned in Step 1 that's blocking or unresolved.

Show the proposed structure to the user as outlines (section headings + a sentence or two each), not full content yet. Get approval before writing the files.

## Step 6: Write the three docs

Once the structure is approved, write the actual files. Be conservative — better to have a sparse, accurate doc than a thorough, hallucinated one. If something is uncertain, write it with a `TODO:` marker and ask the user.

For CLAUDE.md specifically, include this section:

```
## Task tracking

This project uses beads for task management.
- Next ready task: `bd ready --json`
- Show task details: `bd show <task-id> --json`
- Claim a task: `bd update <task-id> --claim --json`
- Close a task: `bd close <task-id> --reason="..." --json`
- Sync to git: `bd sync` (run at session end)
- Commit with task ID: `git commit -m "<message> (<task-id>)"` — enables orphan detection via `bd doctor`

Workflow skills: /start-session, /end-session, /plan-to-epic, /epic-executor
```

## Step 7: Initialize beads

```
bd init --quiet
bd setup claude
```

This installs the SessionStart hook (`bd prime`) and PreCompact hook (`bd sync`).

Verify: `bd setup claude --check`

## Step 8: Create initial epic and tasks

Based on the unbuilt features identified in Step 1 and Step 5, propose an initial epic structure. Use the same logic as `/migrate-to-beads`:

- One epic per phase or major feature area.
- Tasks within each epic, with inferred dependencies.

Show the user the proposed structure before creating anything in beads. Wait for approval.

If the user has nothing planned yet (greenfield project, or all caught up), skip this step. They can use `/plan-to-epic` later when they're ready to plan a feature.

For each task, populate the four fields:

- `--description`: What to do (mechanics).
- `--design`: Why and how it fits (context from PRD.md).
- `--acceptance`: Verifiable outcomes.
- `--notes`: "Created during initial migration."

```
bd create "<Title>" -t epic -p 1 --description="<scope>" --json
bd create "<Title>" -t task -p 1 \
  --parent <epic-id> \
  --description="..." \
  --design="..." \
  --acceptance="..." \
  --notes="Created during initial migration." \
  --json
```

Add dependencies:

```
bd dep add <dependent-task-id> <dependency-task-id>
```

## Step 9: Generate ADRs for existing decisions

Scan the project docs and code for past architectural decisions that meet the ADR criteria. Common sources:

- README sections explaining "why we chose X"
- Config files with non-obvious choices (e.g., specific framework versions, unusual build setups)
- Existing architecture or design docs
- Comments in code that explain design rationale

Create `docs/adr/` and propose a batch of ADR titles. Show the list, let the user approve/reject/modify, then invoke `/adr` for each approved one.

If no ADR-worthy decisions are found, skip this step. For greenfield projects, there may not be any yet.

## Step 10: Handle existing docs

For each existing doc that overlapped with the new three files:

- If it's now redundant (everything migrated to CLAUDE.md/PRD.md/plan.MD): ask the user before deleting. Some users want to keep old docs as historical reference.
- If it has unique content (e.g., contributor guide, deployment runbook): leave it alone.
- If it's stale and the user agrees: delete it.

For docs that don't fit the three-file system but are still useful (architecture deep-dives, runbooks), suggest moving to `docs/` so they don't clutter the repo root. Don't auto-move — ask first.

## Step 11: Update .gitignore

Add `scratch.md` to `.gitignore` if not already there. This is the standard "don't read me, don't commit me" file the workflow assumes.

## Step 12: Commit and push

```
git add .beads/ .claude/ docs/adr/ CLAUDE.md PRD.md plan.MD .gitignore
git commit -m "chore: migrate to three-file + beads workflow"
git push
bd sync
```

## Step 13: Summary

Print for the user:

- Three docs created (with line counts).
- Beads initialized, hooks installed.
- Epics created (count and IDs), tasks created (count).
- ADRs created (count and titles).
- Existing docs: what was kept, moved, or deleted.
- Items flagged with `TODO:` markers in the new docs that need user input.
- Next steps: `/start-session` to begin working with the new system.

## Key principles

- **Ask early, ask often.** This skill makes a lot of decisions about how to structure a project. Get user input upfront so you don't waste tokens guessing wrong.
- **Verbal beats code-reading.** Always prefer asking the user to describe something than reading code to infer it.
- **Better sparse than wrong.** A short, accurate CLAUDE.md is more useful than a long hallucinated one. Use `TODO:` markers liberally.
- **Don't delete without asking.** Existing docs may have meaning to the user even if they're redundant.
- **Single commit at the end.** The migration lands as one logical change.