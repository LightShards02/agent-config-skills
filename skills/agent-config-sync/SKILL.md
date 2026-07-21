---
name: agent-config-sync
description: Sync this machine's AI agent configuration (skills, guidance, MCP servers) from the personal agent-config git repo, install any missing runtime dependencies, and re-apply Ruler to all wired project repos. Use this whenever the user asks to sync/update/pull their agent config, skills, or MCP setup on a machine, says "sync my skills", "update my agent setup", "pull my config", or after they mention editing their agent-config repo on another machine. Also use the push half when the user says they changed a skill/guidance/MCP locally and wants other machines to get it.
---

# Agent Config Sync

Keep this machine's agent configuration in sync with the personal `~/agent-config` git repo, which is the single source of truth for all skills, guidance, and MCP server definitions.

## Layout assumptions

- `~/agent-config` — personal config repo: `library/` (canonical storage), `global/` (the machine-level scope), `projects/<name>/` (per-project scopes), `bootstrap/` (Brewfile, doctor + apply helpers).
- Scope dirs compose from the library via relative symlinks. Ruler does **not** follow symlinked skill directories, so scopes are never consumed in place: `bootstrap/apply.sh` materializes them (`rsync -aL --delete`, which dereferences the links into real files) into their consumption points — `~/.config/ruler` for the global scope, `<project>/.ruler` for project scopes — and then runs `ruler apply --skills`.
- The global scope additionally fans out to **user-level tool configs**: `apply.sh --global` regenerates `~/.claude/CLAUDE.md` and `~/.codex/AGENTS.md` from `global/AGENTS.md` (marker-gated — it refuses files it doesn't own) and keeps `~/.claude/skills` / `~/.agents/skills` symlinked to the materialized skills. Those user-level files are build outputs, same as everything else downstream.
- A project is "wired" when it has a materialized `.ruler` directory matching a scope in `~/agent-config/projects/` (scope name = project dir basename unless overridden).
- Everything downstream of the scope dirs — `~/.config/ruler`, `<project>/.ruler`, `.claude/skills/`, `.mcp.json`, `CLAUDE.md`, etc. — is disposable build output. Never edit it; never treat its contents as authoritative.

If `~/agent-config` does not exist, or exists but is missing expected structure, run the `agent-config-init` skill instead — it is pull-first and idempotent, so it clones/pulls whatever exists and creates only what's missing. Don't hand-bootstrap here; a partial setup that skips init's import and dedup steps can fork the user's config.

## Pull workflow (bring this machine up to date)

### 1. Pull the repo

```bash
cd ~/agent-config && git pull --ff-only
```

If the pull fails because of local uncommitted changes, show the user `git status` and ask whether to commit/stash — do not discard changes silently; they may be unsynced skill edits. If no remote is configured at all (local-only setup), note it and skip the pull — the rest of the workflow still converges local state; don't invent a remote, just remind the user that cross-machine sync needs one.

### 2. Check and install dependencies (doctor step)

The config declares what it needs; verify the machine can actually run it. Run `~/agent-config/bootstrap/doctor.sh` (report mode), then — with user approval — `doctor.sh --install` for anything missing. It unions two declaration sources:

1. Each skill's `deps.toml` (`runtimes = [...]`) in `library/skills/*/`.
2. Runtimes derived from `[mcp_servers.*]` `command` values in every ruler.toml (`npx`/`node` → Node, `uvx`/`uv` → uv, `docker` → Docker; any other command is itself the requirement).

If doctor.sh doesn't exist (repo predates it), do the same two collections by hand — and seed the script from the init skill while you're there. Two follow-ups keep the system truthful:

- If a runtime was missing because a skill never *declared* it, add the declaration to that skill's `deps.toml` and include it in the next push — otherwise every other machine rediscovers the same gap.
- Tell the user what was installed and which skill or server needed it.

### 3. Re-materialize and re-apply everything

Global scope first (its consumption points are `~/.config/ruler` plus the user-level tool configs):

```bash
~/agent-config/bootstrap/apply.sh --global
```

If the script warns about an unmarked user-level file, that file predates management — hand it to `agent-config-init` for import rather than overwriting. Then reconcile user-scope MCP: every `[mcp_servers.*]` in `global/ruler.toml` should exist user-scope for Claude (`claude mcp add <name> --scope user -- <command> <args…>`) and Codex (`[mcp_servers]` blocks in `~/.codex/config.toml`, patching only those sections). Additive-only — never remove user-scope servers the repo doesn't list.

Then every wired project:

```bash
~/agent-config/bootstrap/apply.sh <project-path>
```

Find wired projects rather than guessing: look for `.ruler` directories in the user's code roots (`find ~/code -maxdepth 2 -type d -name .ruler` — adjust to where this user keeps repos) whose basename-matched scope exists in `~/agent-config/projects/`.

### 4. Wire any unwired projects

If a project exists on this machine and has a scope dir in `~/agent-config/projects/` but no `.ruler` yet, wiring *is* the first apply:

```bash
echo ".ruler" >> <project>/.git/info/exclude
~/agent-config/bootstrap/apply.sh <project-path>
```

`.git/info/exclude` (not the shared `.gitignore`) because this is personal config that must not leak into the team repo. For the same reason every scope's `ruler.toml` should carry `[gitignore] local = true`, which makes Ruler write its own generated-file ignores there too.

### 5. Clean stale generated skills

Ruler copies skills into tool dirs but never deletes orphans: a skill removed from a scope lingers in `.claude/skills/` forever and silently diverges from the managed set. After applying, compare each project's generated skill dirs against its materialized `.ruler/skills/` and remove entries that no longer exist in the source (same for `~/.claude/skills` vs `~/.config/ruler/skills` if the user wires global skills there). Show the user what gets removed.

### 6. Report

Summarize: what changed in the pull, what dependencies were installed, which projects were re-applied or newly wired, and any stale skills cleaned.

## Push workflow (publish local edits)

When the user has edited skills/guidance/MCP on this machine:

1. Confirm the edits live in `~/agent-config` (library or scope dirs), not in build output. If a new skill was created inside a generated dir (e.g. `.claude/skills/` or a project's `.ruler/skills/`), move it into `library/skills/` and symlink it from the relevant scope dir first — otherwise the next apply clobbers it. Same rescue rule for guidance: if the user edited `~/.claude/CLAUDE.md` or `~/.codex/AGENTS.md` directly (old habit — they're generated now), merge that edit into `global/AGENTS.md` before re-running the fan-out.
2. If a new skill introduced a runtime requirement, declare it in that skill's `deps.toml` in the same commit, so the doctor step on other machines stays truthful (MCP server runtimes need no declaration — doctor derives them from ruler.toml).
3. `cd ~/agent-config && git add -A && git commit && git push`. Write a commit message naming what changed (which skill/scope), since other machines' pull logs are how the user tracks config history.
4. Re-run the apply step (3 above) locally so this machine's build output reflects what was just committed.
