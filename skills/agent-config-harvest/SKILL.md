---
name: agent-config-harvest
description: Harvest existing AI agent configs scattered across a project repo (Claude Code, Codex, Cursor, Kimi, etc.), deduplicate conflicting skill/guidance/MCP copies, and migrate them into the personal agent-config repo as a new managed scope. Use this the first time a repo with pre-existing .claude/, .cursor/, .agents/, .kimi/, CLAUDE.md, AGENTS.md, or .mcp.json content is brought under agent-config/Ruler management — when the user says "onboard this repo", "import this project's skills", "my tools have different skill copies here, clean them up", or "migrate this repo's agent config".
---

# Agent Config: Harvest an Existing Repo

Take a project repo with accumulated, possibly conflicting agent configs from multiple tools, and consolidate them into a clean `~/agent-config/projects/<name>/` scope. After this runs once, Ruler owns all generated configs and per-tool drift can no longer occur.

Work non-destructively until the final cutover step: inventory and merge first, delete only after the user confirms the regenerated output matches.

## 0. Pull first

```bash
cd ~/agent-config && git pull --ff-only
```

Every reconciliation decision in steps 3–5 compares harvested content against the library — comparing against a stale library re-imports skills that already exist and creates duplicates the next pull turns into conflicts. If `~/agent-config` doesn't exist or lacks the expected structure, run `agent-config-init` first. If a scope for this project already exists in `projects/` (harvested from another machine), switch to reconciling *into* it rather than creating it fresh.

## 1. Inventory

Scan the project repo for every agent artifact and show the user the full list before touching anything:

- **Guidance:** `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursor/rules/`, `.github/copilot-instructions.md`, tool-specific instruction files
- **Skills:** `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.kimi/skills/`, `.opencode/skills/`
- **MCP:** `.mcp.json`, `.cursor/mcp.json`, `.vscode/mcp.json`

## 2. Deduplicate skills across tool stores

Different tools often hold divergent copies of the same skill. Resolve by content, not by assumption:

1. For each skill name, hash the directory in every store where it appears — hash **content only**, never archive metadata: `(cd <dir> && find . -type f -exec shasum {} \; | sort | shasum)`. (Do not use `tar | shasum`: tar embeds mtimes, so byte-identical copies hash differently, and GNU-only flags like `--sort=name` fail on macOS BSD tar.)
2. Same name + same hash everywhere → keep one copy, no user input needed.
3. Same name + different hashes → show the user `diff -r` output between the copies and ask which to keep (or merge by hand). File modification times hint at which copy is newest, but content wins over recency — a tool may have auto-rewritten a file without improving it.

## 3. Reconcile against the library

For each surviving skill, check `~/agent-config/library/skills/<name>`:

- Doesn't exist → move the skill into the library. If it needs a runtime (script shebangs are the tell), declare it in a `deps.toml` beside its SKILL.md (`runtimes = ["python3"]`).
- Exists with identical content → discard the harvested copy; the scope will symlink the library version.
- Exists with different content → diff and ask the user: merge into the library version (preferred — one skill, improved), or keep both under distinct names (`<name>` and `<name>-<repo>`) if they've genuinely forked.

## 4. Merge guidance

`CLAUDE.md`, `AGENTS.md`, and per-tool rules files are typically 80% overlapping. Consolidate into a single `AGENTS.md` for the scope dir, keeping the union of unique content and dropping repeats. Sections that are reusable beyond this project (style guides, git conventions) go into `library/guidance/` as separate `.md` files, symlinked from the scope — reusable content left inline in a project's `AGENTS.md` is invisible to future projects.

## 5. Convert MCP configs

1. Union the server definitions across all discovered JSON configs; dedupe by server name. If two definitions of the same name differ (usually env vars or args), show both and let the user pick.
2. Translate each JSON definition into a `[mcp_servers.<name>]` TOML block in the scope's `ruler.toml` (stdio: `command` + `args` + `env`; remote: `url` + headers).
3. Add any server not already in `library/mcp/servers.toml` to the catalog. Runtimes need no separate record — `bootstrap/doctor.sh` derives them from the `command` values — but run `doctor.sh` before cutover: a harvested server can need a runtime this machine has never had (a docker-based server on a machine without Docker, say).

## 6. Assemble the scope dir

`~/agent-config/projects/<name>/` ends up with exactly: `ruler.toml`, the merged `AGENTS.md`, relative symlinks into `library/` for shared guidance, and `skills/` containing relative symlinks into `library/skills/`. Nothing in the scope dir should be a real file that could exist in the library instead.

## 7. Cut over

Only after the user has approved the merged results:

```bash
echo ".ruler" >> <project>/.git/info/exclude
~/agent-config/bootstrap/apply.sh <project-path>
```

`apply.sh` materializes the scope into `<project>/.ruler` (`rsync -aL --delete`, dereferencing the library symlinks — Ruler does not follow symlinked skill dirs) and runs `ruler apply --skills`. Note: unless the scope's ruler.toml sets `[backup] enabled = false` (the template default), Ruler leaves `.bak` copies beside every file it regenerated — delete those strays as part of cutover. Then compare the regenerated dirs against the originals (`diff -r` per store) and show the user anything that differs beyond the intended dedup. Once confirmed, delete the old native config stores that Ruler does not regenerate (e.g. a leftover `.kimi/skills/`) — stale stores silently diverge from the managed config, which is the exact disease this harvest cures. If any of the old configs were **committed** to the project repo, surface it and let the user decide — untracking is the repo owner's call, not this skill's. Explain the consequence: exclude entries never untrack already-tracked files, so the tracked generated files (`CLAUDE.md`, `.mcp.json`, `.claude/skills/`, …) will show as modified after every apply, and a careless `git commit -a` would push personal generated config upstream. Offer `git rm --cached` on those paths as the fix; apply it only with approval, and otherwise leave the repo's tracking exactly as it is. Whatever was approved (store deletions, optional untracking) forms one commit in the project repo; write a message explaining the move to external management.

## 8. Commit the harvest

```bash
cd ~/agent-config
git add projects/<name> library bootstrap
git commit -m "Harvest <name>: <k> skills (<n> new to library), <m> MCP servers"
git push
```

The commit message should record what was imported and what was merged, since this is the moment config history transfers from scattered tool dirs into the tracked repo.
