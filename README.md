# pi-review-skill

A structured code-review skill for coding agents — a port of [`earendil-works/pi-review`](https://github.com/earendil-works/pi-review) (the Pi `/review` + `/end-review` extension) to the portable `SKILL.md` (Agent Skills) format.

The review rubric, mode-specific focus prompts, and output contracts are unchanged from the original (the rubric itself originates from Codex's review feature, adapted by Earendil for pi). Git preparation verifies the requested review target. The parts tied to Pi's TUI — interactive selector, session-tree branching, review widget — are replaced by conversation-driven behavior, so it works in **any agent that loads Agent Skills**: [DSH](https://github.com/deepseek-ai/deepseek-harness), Claude Code, pi, and others.

## What it does

- Review **uncommitted changes** (staged, unstaged, and untracked)
- Review changes against a **base branch** (via merge-base diff)
- Review a specific **commit**
- Review a GitHub **pull request** (via `gh`)
- Review one or more **folders/files** as a snapshot (not a diff)
- Produce prioritized findings ([P0]–[P3]) with a clear verdict (`correct` / `needs attention`) and actionable follow-ups
- Separate agent-facing findings from human callouts (migrations, dependency churn, auth changes, …)
- Load shared project instructions from `REVIEW_GUIDELINES.md`
- End a review with a structured handoff summary, then optionally implement the findings

## Install

Clone and copy the `pi-review/` directory into your agent's skills directory:

```bash
git clone https://github.com/Panmax/pi-review-skill.git
```

| Agent | Skills dir | Copy command |
|---|---|---|
| DSH | `~/.dsh/skills/` | `mkdir -p ~/.dsh/skills/pi-review && cp -R pi-review-skill/pi-review/. ~/.dsh/skills/pi-review/` |
| DSH (project) | `<repo>/.dsh/skills/` | `mkdir -p "<repo>/.dsh/skills/pi-review" && cp -R pi-review-skill/pi-review/. "<repo>/.dsh/skills/pi-review/"` |
| Claude Code | `~/.claude/skills/` | `mkdir -p ~/.claude/skills/pi-review && cp -R pi-review-skill/pi-review/. ~/.claude/skills/pi-review/` |
| Claude Code (project) | `<repo>/.claude/skills/` | `mkdir -p "<repo>/.claude/skills/pi-review" && cp -R pi-review-skill/pi-review/. "<repo>/.claude/skills/pi-review/"` |
| pi | `~/.pi/agent/skills/` | `mkdir -p ~/.pi/agent/skills/pi-review && cp -R pi-review-skill/pi-review/. ~/.pi/agent/skills/pi-review/` |
| pi (project) | `<repo>/.pi/skills/` | `mkdir -p "<repo>/.pi/skills/pi-review" && cp -R pi-review-skill/pi-review/. "<repo>/.pi/skills/pi-review/"` |

Replace `<repo>` with your repository path. These commands create the destination on a fresh installation and copy into the explicit `pi-review/` directory on subsequent updates, without nesting another `pi-review/` directory inside it.

`~/.agents/skills/` is a cross-harness location read by both DSH (`user-agents` root) and pi. Install there once and symlink into `~/.claude/skills/` — which does not read it — to maintain a single copy for all three.

To try it without installing, pi takes a path directly: `pi --skill pi-review-skill/pi-review`. This is a per-run flag, not an install; it must be repeated every session.

DSH and Claude Code pick up new skills without a restart (DSH watches its skill roots; Claude Code re-scans). pi scans skill locations at startup, so restart the pi session.

## Usage

Invoke the skill with its own name as the command (DSH, Claude Code):

```
/pi-review uncommitted
/pi-review branch main
/pi-review commit abc123
/pi-review commit abc123 "Add retry to the upload path"
/pi-review pr 123
/pi-review pr https://github.com/owner/repo/pull/123
/pi-review folder src docs
/pi-review branch main --extra "focus on performance and error handling"
/pi-review branch main --extra="focus on performance and error handling"
```

In pi the same arguments go to `/skill:pi-review` (pi registers skills under the `skill:` prefix), e.g. `/skill:pi-review uncommitted`.

Natural language works the same way ("review uncommitted", "评审一下未提交的改动").

After a review, ask for the handoff in plain phrasing (this port has no separate command):

```
end-review            # structured handoff: scope, verdict, findings, fix queue
fix review findings   # implement the findings in priority order
```

### Review targets

| Request | Behavior |
|---|---|
| `uncommitted` | `git status --porcelain` + `git diff HEAD` + untracked files |
| `branch <name>` | merge-base diff against the branch (upstream-aware) |
| `commit <sha>` | full diff introduced by the commit |
| `pr <n \| url>` | Get current PR head with `gh pr view`, checkout/update when `HEAD` differs, verify the commit, then merge-base diff (requires `gh`; tracked files must be clean) |
| `folder <paths…>` | snapshot review — files read directly, no diff |

### Project guidelines

Walk up from the current directory to the nearest directory containing `.dsh` or `.pi`. Stop at the Git repository root (a `.git` file or directory) if no closer anchor exists, using that root as the fallback. Look for `REVIEW_GUIDELINES.md` in the anchor only; do not continue into parent projects if the file is absent. This preserves the original `.pi` behavior for nested projects while supporting DSH. Its contents are appended to the review as project-specific instructions that override the default rubric where more specific.

## Triggering

All of the following route to the same workflow:

- **Skill gesture (DSH, Claude Code)** — `/pi-review` plus the target, e.g. `/pi-review uncommitted`, `/pi-review branch main`, `/pi-review pr 123`. Both hosts resolve a bare `/<skill-name>` against the skill registry, so the command is the skill's own name.
- **Skill gesture (pi)** — `/skill:pi-review` plus the target, e.g. `/skill:pi-review uncommitted`. pi registers skills under a `skill:` prefix rather than as bare names (controlled by its `enableSkillCommands` setting, on by default).
- **Natural language** — "review uncommitted", "评审一下未提交的改动", "帮我看看这个 PR"
- **Phrasings inherited from the original extension** — `review uncommitted`, and the original's `/review` / `/end-review` command words, are accepted as *request phrasing* only. They are not commands here: `/review` never reaches this skill (in Claude Code it is the built-in `code-review` alias; elsewhere it resolves to nothing), so always use `/pi-review` or `/skill:pi-review`.

### Naming note (Claude Code)

Claude Code bundles a `code-review` skill registered with `review` as an alias, so both `/code-review` and `/review` reach the built-in one. Do **not** rename this skill to `code-review`: it would collide with the bundled command while `/review` still points at the bundled one. The `pi-review` name sidesteps the conflict.

### Automation

For hooks and CI, skip the chat layer and invoke your agent headlessly — the request text becomes the API argument:

```bash
dsh --profile headless "review uncommitted"                        # DSH
claude -p "Use the pi-review skill to review uncommitted changes"  # Claude Code
pi -p "Use the pi-review skill to review uncommitted changes"      # pi
```

## Output format

Each review ends with:

1. **Findings** tagged `[P0]`–`[P3]`, each with file location, why it matters, and what should change
2. A verdict: `correct` (no blocking issues) or `needs attention`
3. A mandatory **Human Reviewer Callouts (Non-Blocking)** section (migrations, new dependencies, lockfile changes, auth/permission changes, breaking contract changes, destructive operations, feature flags, config defaults)

## Differences from the original

| Original (pi extension) | This port |
|---|---|
| `/review` TUI selector | Natural-language / chat target selection |
| Commands `/review` + `/end-review` (Pi extension) | One skill, invoked as `/pi-review` (`/skill:pi-review` in pi); ending a review is a workflow step inside it, not a separate command |
| Fresh review session on a session-tree branch | Review runs in the current conversation |
| Review widget + `/end-review` navigation | `end-review` produces the handoff summary in-chat |
| Smart default target (`uncommitted` → feature branch → commit) | Defaults to `uncommitted` when the tree is dirty, otherwise asks and suggests the base-branch diff |
| Session-scoped shared custom instructions (`Add`/`Remove` in the selector) | Not ported — use project-level `REVIEW_GUIDELINES.md` for durable instructions, or one-off `--extra` |
| Always runs `gh pr checkout` for PR review | Checks out/updates whenever `HEAD` differs from the current PR head SHA; skips checkout only on an exact match, and requires clean tracked files for review |
| Interactive only (`Review requires interactive mode`) | Runs headless too, e.g. in CI or scheduled jobs |

## Credits & license

- Original workflow and rubric: [earendil-works/pi-review](https://github.com/earendil-works/pi-review) (rubric adapted from Codex's review feature) — © Earendil Inc., MIT
- Agent Skills port: © Panmax, MIT

See [LICENSE](./LICENSE).
