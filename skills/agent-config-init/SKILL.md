---
name: agent-config-init
description: Set up or join the personal agent-config system on a machine — pull/clone the ~/agent-config repo if it exists anywhere, create whatever structure is missing, import machine-level (global) skills, push to a remote, and interactively harvest existing project repos. Idempotent and safe to re-run. Use this when the user wants to start managing their agent configuration centrally, says "init my agent config", "set up my skills repo", "bootstrap agent-config on this machine", or invokes any agent-config workflow on a machine where ~/agent-config does not exist yet.
---

# Agent Config: Init

Bootstrap the central agent-config system on this machine. This skill is the entry point; after it completes, the other agent-config skills (`agent-config-sync`, `agent-config-new-project`, `agent-config-harvest`) take over day-to-day work.

The skill is **pull-first and idempotent**: always fetch existing state before creating anything, and create only what is missing. Never scaffold blindly — a fresh scaffold on a machine that should have joined an existing repo forks the user's config, which is the worst failure mode this system has. Re-running the skill on an already-set-up machine should change nothing and report "already in order."

## 1. Pull first

Establish the latest existing state, in this order:

1. **Local repo exists** (`~/agent-config/.git`): `cd ~/agent-config && git pull --ff-only` if a remote is configured. If the pull fails on local uncommitted changes, show `git status` and resolve with the user before continuing — those may be unsynced edits from an interrupted session.
2. **No local repo**: ask the user whether an agent-config repo already exists on a remote (they may have initialized on another machine). If yes, `git clone <url> ~/agent-config`. If the remote exists but is **empty** (created ahead of time), the clone still works — you get an empty worktree with `origin` configured and an unborn HEAD; continue as a fresh init from step 2 onward, *including the branch question below, which still applies* because the first push will define the repo. If unsure, help them check their GitHub/GitLab account before creating anything new.
3. **Nothing exists anywhere**: `mkdir -p ~/agent-config && cd ~/agent-config && git init`, and ask for a remote to create (private is the sensible default, since guidance and MCP configs may reference internal systems). Also ask which branch name to use, defaulting to `main` only if the user has no preference — on an empty **forge** remote (GitHub/GitLab), the first branch pushed becomes the repo's default branch, so this choice sticks. A plain bare remote doesn't update its HEAD by itself; set it explicitly after the first push: `git --git-dir=<remote-path> symbolic-ref HEAD refs/heads/<branch>`. Use the chosen branch consistently in every later push. If they don't have a remote yet, continue without — everything works locally — but remind them at the end that cross-machine sync needs the remote.

Everything after this point operates on the freshest available state, so "create if missing" decisions are made against reality, not against an empty directory that was about to be cloned over.

## 2. Ensure structure (create only what's missing)

Check each expected path and create it only if absent — never overwrite a file that exists, since it may carry another machine's content:

```bash
mkdir -p ~/agent-config/{library/{skills,guidance,mcp},global/skills,projects/_template,bootstrap}
```

Standing files, each seeded **only if it does not exist**:

- `library/mcp/servers.toml` — empty catalog with a comment explaining it holds every known `[mcp_servers.*]` block.
- `bootstrap/doctor.sh` — the dependency executor (see below). Dependencies are **declared, not scripted**: a skill that needs a runtime carries a small `deps.toml` beside its SKILL.md (`runtimes = ["rg"]` — the binary names to check with `command -v`; plus optional `brew = ["ripgrep"]` when the Homebrew formula name differs from the binary). MCP server runtimes are derived from their `command` values in ruler.toml. doctor.sh unions both sources, reports what's missing, and installs only when invoked with `--install`. Never store executable install commands in the repo — a pulled repo whose sync auto-runs shell is an attack surface the day a skill is shared.
- `bootstrap/apply.sh` — the materialize-and-apply helper (see below). Scope dirs compose from the library via symlinks, but Ruler does **not** follow symlinked skill directories, so scopes must be materialized (`rsync -aL --delete`, dereferencing links into real files) into their consumption points before `ruler apply` runs.
- `projects/_template/ruler.toml` — minimal ruler.toml that new project scopes copy from; generate via `ruler init` in a scratch dir, trim, and set `[gitignore] local = true` so Ruler writes its generated-file ignores to `.git/info/exclude` instead of the team-visible `.gitignore`. Also set `[backup] enabled = false` — Ruler otherwise drops a `.bak` beside every file it regenerates (`CLAUDE.md.bak`, `.mcp.json.bak`, …), which is pure litter in a system where everything is rebuildable from the scope dirs.
- `global/ruler.toml` and `global/AGENTS.md` — the global scope.
- Both tomls set `default_agents = ["claude", "cursor", "codex"]`. "Agents" here is Ruler's term for the **target tools it generates config files for** (Claude Code → `CLAUDE.md`/`.claude/`, Cursor → `.cursor/`, Codex → root `AGENTS.md`/`.agents/`) — not the AI running this skill. This trio is the user's standing tool set (ChatGPT desktop has no file-based config, so Codex is the OpenAI-side target); adjust only if the user says their tools differ.
- Root `README.md` — describes the layout and the two rules: everything lives once in `library/` (scope dirs hold only symlinks, `ruler.toml`, `AGENTS.md`), and everything outside `~/agent-config` is disposable build output.

`bootstrap/apply.sh` (seed only if missing):

```sh
#!/bin/sh
# Materialize a scope (dereferencing library symlinks) and run ruler apply.
# --global additionally fans out USER-LEVEL configs: the files each tool reads
# machine-wide are build outputs regenerated from global/, never edited directly.
# Usage: apply.sh --global | apply.sh <project-path> [scope-name]
set -e
if [ "$1" = "--global" ]; then
  AC="$HOME/agent-config"
  mkdir -p "$HOME/.config/ruler"
  rsync -aL --delete "$AC/global/" "$HOME/.config/ruler/"
  # user-level guidance: regenerate ONLY files that are ours (marker) or absent —
  # an unmarked file predates this system and must be imported by init first
  for f in "$HOME/.claude/CLAUDE.md" "$HOME/.codex/AGENTS.md"; do
    if [ ! -e "$f" ] || head -1 "$f" | grep -q "generated by agent-config"; then
      mkdir -p "$(dirname "$f")"
      { echo "<!-- generated by agent-config; edit global/AGENTS.md instead -->"
        cat "$AC/global/AGENTS.md"; } > "$f"
    else
      echo "warn: $f not agent-config-managed; run agent-config-init to import it" >&2
    fi
  done
  # user-level skills: point each tool's user skill dir at the materialized copy
  for d in "$HOME/.claude/skills" "$HOME/.agents/skills"; do
    if [ ! -e "$d" ]; then mkdir -p "$(dirname "$d")"; ln -s "$HOME/.config/ruler/skills" "$d"
    elif [ ! -L "$d" ]; then echo "warn: $d is a real dir; run agent-config-init to import it" >&2; fi
  done
  exit 0
fi
proj="$1"; scope="${2:-$(basename "$proj")}"
src="$HOME/agent-config/projects/$scope"
[ -d "$src" ] || { echo "no scope '$scope' in ~/agent-config/projects" >&2; exit 1; }
rsync -aL --delete "$src/" "$proj/.ruler/"
cd "$proj" && ruler apply --skills
```

`bootstrap/doctor.sh` (seed only if missing):

```sh
#!/bin/sh
# Dependency executor. Report by default; install missing runtimes with --install.
# deps.toml keys: runtimes = ["rg"] (binaries to check); brew = ["ripgrep"] (install
# names, when the formula differs from the binary — defaults to the runtime name).
CFG="$HOME/agent-config"; missing=""; inst=""
for f in "$CFG"/library/skills/*/deps.toml; do
  [ -f "$f" ] || continue
  rts=$(sed -n 's/^ *runtimes *= *\[\(.*\)\]/\1/p' "$f" | tr -d '",')
  brews=$(sed -n 's/^ *brew *= *\[\(.*\)\]/\1/p' "$f" | tr -d '",')
  for r in $rts; do
    command -v "${r%%[<>=]*}" >/dev/null 2>&1 || { missing="$missing $r"; inst="$inst ${brews:-$r}"; }
  done
done
cmds=$(grep -rh '^command' "$CFG"/global/ruler.toml "$CFG"/projects/*/ruler.toml 2>/dev/null | sed 's/.*= *"\(.*\)"/\1/')
for c in $cmds; do
  case "$c" in npx|node) r=node;; uvx|uv) r=uv;; python*) r=python3;; *) r=$c;; esac
  command -v "$r" >/dev/null 2>&1 && continue
  case " $missing " in *" $r "*) ;; *) missing="$missing $r"; inst="$inst $r";; esac
done
[ -z "$missing" ] && { echo "doctor: all dependencies present"; exit 0; }
echo "doctor: missing:$missing"
if [ "$1" = "--install" ]; then
  for m in $(echo $inst | tr ' ' '\n' | sort -u); do brew install "$m"; done
fi
```

Machine wiring, likewise convergent:

- Install Ruler if missing (`npm install -g @intellectronica/ruler`, Node ≥ 20.19).
- Materialize the global scope: `bootstrap/apply.sh --global`. If `~/.config/ruler` already exists as a hand-managed directory from before this system, import its contents **first** (merge into `global/`/`library/`, dedup against what pulled down in step 1) — the first materialization overwrites it.
- Run `bootstrap/doctor.sh` — on every init, fresh or join. Even a fresh init can import skills or harvest MCP servers whose runtimes this machine lacks (a docker-based server on a machine without Docker, say). Install what's missing with `doctor.sh --install` after user approval.

## 3. Import and push the global (machine-level) skills

Gather skills and guidance that live at the user level on *this machine* and fold them into the repo:

1. Look in the user-level tool dirs: `~/.claude/skills/`, `~/.claude/CLAUDE.md`, `~/.codex/`, `~/.agents/skills/`, `~/.kimi/skills/`, and user-level MCP configs (`~/.claude.json` MCP entries, `~/.cursor/mcp.json`). Show the user what was found.
2. Deduplicate — both across this machine's tools *and against what already exists in the pulled library* — the way `agent-config-harvest` does: hash-compare same-named skills, auto-keep identical copies, diff divergent ones for the user. On a joining machine most content will already be in the library; only genuinely new or diverged items need attention.
3. Move surviving new skills into `library/skills/`, symlink each from `global/skills/` (relative targets). If an imported skill needs a runtime (script shebangs are the tell), declare it in that skill's `deps.toml` (`runtimes = ["python3"]`). User-level instruction content merges into `global/AGENTS.md`, with reusable sections split into `library/guidance/`. User-level MCP servers become `[mcp_servers.*]` blocks in `global/ruler.toml` plus catalog entries in `library/mcp/servers.toml` — their runtimes need no separate record; doctor.sh derives them from the `command` values.
4. Re-run `bootstrap/apply.sh --global`. This is the **user-level fan-out**: after it, `~/.claude/CLAUDE.md` and `~/.codex/AGENTS.md` are regenerated from `global/AGENTS.md` (with a "generated by agent-config" marker), and `~/.claude/skills` / `~/.agents/skills` are symlinks to the materialized skills. From this point those files are build outputs — all editing happens in `global/`. The script refuses to overwrite unmarked pre-existing files, which is why import (steps 1–3 above) must happen first; confirm with the user before the first replacement of any file they hand-maintained.
5. Reconcile user-scope MCP the same way: every `[mcp_servers.*]` block in `global/ruler.toml` should be registered user-scope for Claude (`claude mcp add <name> --scope user -- <command> <args…>`) and for Codex (add the block to `[mcp_servers]` in `~/.codex/config.toml` — patch only those sections, never rewrite that file wholesale, it holds unrelated settings; or use `codex mcp add` if available). **Additive-only**: never delete user-scope servers the repo doesn't list — they may be personal or app-managed. ChatGPT desktop has no file-based config; its connectors remain manual.

Commit and push whatever this machine contributed:

```bash
cd ~/agent-config && git add -A
git commit -m "Init on <machine>: <what was added, if anything>"
git push -u origin <branch>   # if a remote is configured; <branch> = the name chosen in step 1
```

If nothing changed (machine join with no new local content), there is nothing to commit — say so and move on.

## 4. Install the agent-config skills themselves

The four agent-config skills (init, sync, new-project, harvest) belong in `library/skills/`, symlinked from `global/skills/`, so every machine that joins gets the workflows automatically. If the pull already brought them in, done; otherwise **copy** them in from wherever they exist on this machine — copy, not move: one of them is the skill currently executing, and its original must survive until the fan-out delivers the library copy. If any of the four exist nowhere on this machine, don't silently proceed with a partial set — tell the user which are missing and where to fetch them (their skills source repo).

## 5. Interactive harvest of existing project repos

Ask the user which project repos on this machine should come under management. Do the legwork for them first: scan their likely code roots (`~/code`, `~/projects`, `~/dev`, `~/Desktop` — ask which apply) for git repos containing agent artifacts (`.claude/`, `.cursor/`, `.agents/`, `.kimi/`, `CLAUDE.md`, `AGENTS.md`, `.mcp.json`), and present the candidates with what was found in each. Skip repos whose scope dirs already exist in `projects/` — those just need wiring (exclude entry + `bootstrap/apply.sh <project>`, as in the sync skill), not harvesting.

For each repo the user selects, run the `agent-config-harvest` skill end-to-end (inventory → dedup → reconcile → merge → convert → cutover → commit). One repo at a time — harvest has interactive conflict-resolution steps, and interleaving two repos' conflicts is confusing.

Repos the user declines stay untouched and can be harvested later by invoking `agent-config-harvest` directly.

## 6. Final report

Summarize the converged state: repo location and remote, whether this was a fresh init or a machine join, what this machine contributed vs received, which projects are now managed or newly wired, and the two habits that keep the system healthy — edit only in `~/agent-config` (never build output), and run the sync skill's push half after editing so other machines pick changes up.
