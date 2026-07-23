---
name: agent-config-new-project
description: Scaffold agent configuration for a new project by creating a scope directory in the personal agent-config repo and cherry-picking skills, guidance, and MCP servers from the library. Use this whenever the user starts a new repo/project and wants their AI tooling set up, says "set up agent config for this project", "add my skills to this repo", "wire up ruler here", or asks which of their existing skills/MCPs to reuse in a new codebase.
---

# Agent Config: New Project Setup

Create the agent configuration for a new project as a scope directory in `~/agent-config`, composing from the library rather than copying — so every skill and guidance file continues to exist exactly once.

## Layout assumptions

- `~/agent-config/library/` — canonical storage: `skills/`, `guidance/`, `mcp/servers.toml` (catalog of `[mcp_servers.*]` blocks).
- `~/agent-config/projects/<name>/` — one scope dir per project, containing only: `ruler.toml`, `AGENTS.md`, and symlinks into the library.
- The project's `.ruler` is a **materialized copy** of the scope, built by `~/agent-config/bootstrap/apply.sh <project>` (`rsync -aL --delete` + `ruler apply --skills`). Ruler does not follow symlinked skill dirs, so scopes are never consumed in place. Everything under `.ruler/`, `.claude/`, `.mcp.json`, `CLAUDE.md` etc. is disposable build output.

## Workflow

### 0. Pull first

```bash
cd ~/agent-config && git pull --ff-only
```

Compose against the latest library — another machine may have added or renamed skills since this one last synced, and a scope built on a stale library links to things that moved. If `~/agent-config` doesn't exist or lacks the expected structure, run `agent-config-init` first. If a scope for this project already exists in `projects/` (created on another machine), don't recreate it — skip straight to wiring (step 6).

### 1. Create the scope directory

```bash
mkdir -p ~/agent-config/projects/<name>/skills
cd ~/agent-config/projects/<name>
```

Start `ruler.toml` from `~/agent-config/projects/_template/ruler.toml` if it exists; otherwise generate one with `ruler init` in a scratch dir and trim it.

### 2. Help the user shop from the library

This is the core value of the skill — make selection informed, not a blind copy:

1. List what's available: skill names + their descriptions (from each `SKILL.md` frontmatter), guidance files, and the server names in `library/mcp/servers.toml`.
2. Recommend a starting set based on the project's nature (language, framework, domain) — e.g. a Python service probably wants the Python style guidance and any DB/testing skills, not the frontend ones. Note that skills *and MCP servers* also flow in from the `global` scope — don't re-link skills or re-copy server blocks the global config already provides everywhere (a duplicate server is harmless at load time since project scope wins, but two copies drift independently).
3. Confirm the selection with the user before linking.

### 3. Link the selections

Symlinks, not copies — a copy would drift from the library the first time either side is edited:

```bash
ln -s ../../library/guidance/<file>.md .
ln -s ../../../library/skills/<skill-name> skills/
```

Use relative link targets (as above) so the links survive the repo being cloned to a different home directory on another machine. Mind the depths: guidance links live in the scope root (two levels up to the repo root), skill links live inside `skills/` (three levels up) — verify each link resolves (`ls -L`) before moving on.

### 4. Add MCP servers

MCP definitions are TOML sections inside `ruler.toml` and cannot be symlinked. Copy the chosen `[mcp_servers.*]` blocks from `library/mcp/servers.toml` into the scope's `ruler.toml` verbatim. If a needed server isn't in the catalog yet, add it to the catalog first, then copy — the catalog must stay the superset.

Runtimes for copied servers need no separate record — `bootstrap/doctor.sh` derives them from the `command` values in ruler.toml.

### 5. Write project-specific guidance

Put anything unique to this project in the scope dir's `AGENTS.md`. If while writing it you produce a section that would be useful elsewhere (a style rule, a workflow convention), move that section to `library/guidance/` and symlink it instead — the scope dir should hold only what is truly project-specific.

### 6. Wire the project repo and apply

```bash
echo ".ruler" >> <project>/.git/info/exclude
~/agent-config/bootstrap/apply.sh <project-path>
```

`.git/info/exclude` rather than `.gitignore` because this is personal config that should not appear in the team repo's history (the scope's `ruler.toml` should carry `[gitignore] local = true` for the same reason). `apply.sh` materializes the scope into `<project>/.ruler` — dereferencing the library symlinks into real files, which Ruler requires — and runs `ruler apply --skills`. Verify the output: the generated `.claude/skills/`, instruction files, and MCP configs should reflect exactly the selections made above.

### 7. Run the dependency check, then commit

Adding catalog servers to a new scope can bring launch-runtime needs this machine hasn't met yet — run `~/agent-config/bootstrap/doctor.sh` and install (with approval) anything missing. (Skill dependencies resolve at use time, not here.)

```bash
cd ~/agent-config
git add projects/<name>          # plus library/ and bootstrap/ ONLY if this setup
                                 # actually changed them (e.g. a new catalog entry)
git commit -m "Add <name> project config"
git push
```

Pushing is what makes the new project's config reach the user's other machines via their sync workflow.
