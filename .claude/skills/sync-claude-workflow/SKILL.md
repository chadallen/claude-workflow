---
name: sync-claude-workflow
description: Syncs selected skills, agents, and docs from ~/.claude to the public claude-workflow GitHub repo.
when_to_use: Use when you've updated any of the skills or files that belong in the public claude-workflow repo and want to publish them. Trigger phrases: "sync claude-workflow", "push to public repo", "/sync-claude-workflow".
allowed-tools: Bash
---

# Sync Claude Workflow

## Files to sync

Edit this section to add or remove files from the public repo.

**Skills** (directories under `~/.claude/skills/`):
- adr
- build-tasks
- create-tasks
- end-session
- init-project
- migrate-project
- start-session

**Agents** (files under `~/.claude/agents/`):
- code-reviewer.MD
- implementer.MD

**Docs** (files copied to the root of the public repo):
- `~/.claude/docs/claude-workflow/README.MD` → `README.MD`

**Templates** (files copied to the root of the public repo):
- `~/.claude/docs/claude-workflow/templates/PRD.md` → `PRD.md`

---

## Steps

Run each block in order.

**1. Reverse-sync: copy project repo changes back to ~/.claude**

Skills and agents are sometimes edited directly in the project repo rather than in `~/.claude/`. Copy them back first so `~/.claude/` is authoritative before the forward sync runs.

```bash
for skill in adr build-tasks create-tasks end-session init-project migrate-project start-session; do
  src=~/projects/claude-workflow/.claude/skills/$skill
  dst=~/.claude/skills/$skill
  if [ -d "$src" ]; then
    rsync -a --checksum "$src/" "$dst/" && echo "← $skill"
  fi
done

for agent in code-reviewer.MD implementer.MD; do
  src=~/projects/claude-workflow/.claude/agents/$agent
  dst=~/.claude/agents/$agent
  if [ -f "$src" ]; then
    rsync -a --checksum "$src" "$dst" && echo "← $agent"
  fi
done
```

**2. Clear destination directories**

This ensures anything removed from the lists above is also removed from the public repo.

```bash
rm -rf ~/projects/claude-workflow/.claude ~/projects/claude-workflow/skills ~/projects/claude-workflow/agents
mkdir -p ~/projects/claude-workflow/.claude/skills ~/projects/claude-workflow/.claude/agents ~/projects/claude-workflow/docs/adr
```

**3. Sync skills**

For each skill listed above, run:
```bash
rsync -a ~/.claude/skills/<skill>/ ~/projects/claude-workflow/.claude/skills/<skill>/
```

**4. Sync agents**

```bash
cp ~/.claude/agents/code-reviewer.MD ~/projects/claude-workflow/.claude/agents/
cp ~/.claude/agents/implementer.MD ~/projects/claude-workflow/.claude/agents/
```

**5. Sync docs**

```bash
cp ~/.claude/docs/claude-workflow/README.MD ~/projects/claude-workflow/README.MD
cp ~/.claude/docs/claude-workflow/templates/PRD.md ~/projects/claude-workflow/PRD.md
```

**6. Commit and push**

```bash
cd ~/projects/claude-workflow && git add -A && git diff --staged --quiet && echo "Nothing changed — public repo already up to date." || (git commit -m "sync from ~/.claude $(date '+%Y-%m-%d %H:%M')" && git push origin main && echo "Done.")
```

If any step fails, show the error and do not retry without asking the user first.
Report which files were synced and link to https://github.com/chadallen/claude-workflow.
