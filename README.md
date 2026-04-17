# AI Coding Workflow: Three Files + Beads

I use this workflow to solve the usual problems with agents: they forget everything between sessions and they need supervision on long features. It adds two new MD files (PRD and plan) to capture high level project scope, plus an Architecture Decision Record (ARD). It uses Beads to manage task state. Any agent can pick up cold. Well-specified work hands off to an autonomous executor with built-in review.

Two modes run in parallel: conversational for tasks that need real-time judgment, autonomous for everything else.

## The Workflow

```mermaid
flowchart LR
    SS[/start-session] --> W[work one task]
    W --> ES[/end-session]

    B[brainstorm in chat] --> P[/plan-to-epic]
    P --> X[/epic-executor]
    X --> ES

    style SS fill:#e1f5ff
    style ES fill:#e1f5ff
    style P fill:#fff4e1
    style X fill:#fff4e1
```

Top path: conversational. Bottom path: autonomous. Both end at `/end-session`, which closes tasks, updates docs, and pushes everything.

## The Pieces

**CLAUDE.md** — Project rules, stack, conventions. Changes rarely.

**PRD.md** — Product requirements. Aggressively pruned: when a feature is built and tested, its detailed spec gets replaced with a one-line pointer to the code.

**plan.MD** — Session state. Three sections: Overall Plan (which epic is active), Current Status (last 3 sessions), Known Issues. No "what remains" list — that's beads now.

**`docs/adr/`** — Architecture Decision Records. One file per decision. Captures what was chosen, what alternatives were considered, and why. The agent creates these automatically when it detects ADR-worthy decisions during a session — no manual bookkeeping.

**[Beads](https://github.com/steveyegge/beads)** — Git-backed task tracker built for AI agents. Tasks have `description`, `design`, `acceptance`, and `notes` fields. Epics group tasks. Dependencies are tracked, so `bd ready` returns only what's actually unblocked.

`bd setup claude` installs hooks that auto-inject task state at session start. Commits include the task ID (`git commit -m "Add login (bd-42)"`) so `bd doctor` can flag work that got committed but never closed.

## The Skills

| Skill | When |
|---|---|
| `/start-session` | Beginning of a session. Reads docs and beads, flags missing ADRs, proposes a plan. |
| `/end-session` | End of a session. Closes tasks, checks for ADR-worthy decisions, updates docs, pushes everything. |
| `/plan-to-epic` | Convert a design or plan into a beads epic with structured tasks. |
| `/epic-executor` | Autonomously execute all tasks in an epic. Two-stage review per task. |
| `/adr` | Create an Architecture Decision Record. Usually invoked by other skills, not directly. |
| `/migration` | One-time setup for any project. Constructs all docs from scratch. |

## Setup

1. Install [beads](https://github.com/steveyegge/beads).
2. Drop the five skill files into `~/.claude/skills/<name>/SKILL.md` (personal) or `.claude/skills/<name>/SKILL.md` (per-project).
3. Run `/migrate-to-beads` in your project. It handles `bd init`, `bd setup claude`, and converting plan.MD's task list into epics and tasks.

---

`/plan-to-epic` and `/epic-executor` are adapted from [Jarred Kenny's beads workflow](https://jx0.ca/solving-agent-context-loss/).