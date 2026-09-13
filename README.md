# pi-review-skill

A structured code-review skill for coding agents — a port of [`earendil-works/pi-review`](https://github.com/earendil-works/pi-review) (the Pi `/review` + `/end-review` extension) to the portable `SKILL.md` (Agent Skills) format.

The review rubric, git recipes, and output contracts are unchanged from the original (the rubric itself originates from Codex's review feature, adapted by Earendil for pi). The parts tied to Pi's TUI — interactive selector, session-tree branching, review widget — are replaced by conversation-driven behavior, so it works in **any agent that loads Agent Skills**: [DSH](https://github.com/deepseek-ai/deepseek-harness), Claude Code, pi, and others.

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

| Agent | User-level skills dir | Copy command |
|---|---|---|
| DSH | `~/.dsh/skills/` or `~/.agents/skills/` | `cp -r pi-review-skill/pi-review ~/.dsh/skills/` |
| DSH (project) | `<repo>/.dsh/skills/` | `cp -r pi-review-skill/pi-review <repo>/.dsh/skills/` |
| Claude Code | `~/.claude/skills/` | `cp -r pi-review-skill/pi-review ~/.claude/skills/` |
| Claude Code (project) | `<repo>/.claude/skills/` | `cp -r pi-review-skill/pi-review <repo>/.claude/skills/` |
| pi | any path | `pi --skill pi-review-skill/pi-review` |

Restart your agent session so the new skill is discovered.

## Usage

Invoke the skill with its own name as the command:

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
| `pr <n \| url>` | `gh pr view` for base/title, then merge-base diff (requires `gh`; clean tree for checkout) |
| `folder <paths…>` | snapshot review — files read directly, no diff |

### Project guidelines

Walk up from the current directory to the anchor — the first directory containing `.dsh`, falling back to the first containing `.git` — and look for `REVIEW_GUIDELINES.md` there only, stopping at that level (the original anchored on the `.pi` directory the same way). Its contents are appended to the review as project-specific instructions that override the default rubric where more specific.

## Triggering

All of the following route to the same workflow:

- **Skill gesture (DSH, Claude Code)** — `/pi-review` plus the target, e.g. `/pi-review uncommitted`, `/pi-review branch main`, `/pi-review pr 123`. Hosts resolve `/<skill-name>` against the skill registry, so the command is the skill's own name.
- **Natural language** — "review uncommitted", "评审一下未提交的改动", "帮我看看这个 PR"
- **Phrasings inherited from the original extension** — `review uncommitted`, and the original's `/review` / `/end-review` command words, are accepted as *request phrasing* only. They are not commands here: `/review` resolves to no skill and injects nothing, so always use `/pi-review`.

### Naming note (Claude Code)

Claude Code bundles a `/code-review` skill and reserves `/review` as its alias. Do **not** rename this skill to `code-review`: it would shadow the bundled command while `/review` still points at the bundled one. The `pi-review` name sidesteps the conflict.

### Automation

For hooks and CI, skip the chat layer and invoke your agent headlessly — the request text becomes the API argument:

```bash
dsh --profile headless "review uncommitted"                        # DSH
claude -p "Use the pi-review skill to review uncommitted changes"  # Claude Code
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
| Commands `/review` + `/end-review` (Pi extension) | One skill, invoked as `/pi-review`; ending a review is a workflow step inside it, not a separate command |
| Fresh review session on a session-tree branch | Review runs in the current conversation |
| Review widget + `/end-review` navigation | `end-review` produces the handoff summary in-chat |
| Smart default target (`uncommitted` → feature branch → commit) | Defaults to `uncommitted` when the tree is dirty, otherwise asks and suggests the base-branch diff |
| Session-scoped shared custom instructions (`Add`/`Remove` in the selector) | Not ported — use project-level `REVIEW_GUIDELINES.md` for durable instructions, or one-off `--extra` |
| Always runs `gh pr checkout` for PR review | Checks out only when the PR head is not available locally, so it never moves your working tree unnecessarily |
| Interactive only (`Review requires interactive mode`) | Runs headless too, e.g. in CI or scheduled jobs |

## Credits & license

- Original workflow and rubric: [earendil-works/pi-review](https://github.com/earendil-works/pi-review) (rubric adapted from Codex's review feature) — © Earendil Inc., MIT
- Agent Skills port: © Panmax, MIT

See [LICENSE](./LICENSE).
