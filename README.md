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

Ask your agent in natural language — the skill triggers on review requests:

```
review uncommitted
review branch main
review commit abc123
review pr 123
review pr https://github.com/owner/repo/pull/123
review folder src docs
review branch main --extra "focus on performance and error handling"
```

After a review, finish with:

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

If a `REVIEW_GUIDELINES.md` exists at the project root (walking up parent directories), its contents are appended to the review as project-specific instructions that override the default rubric where more specific.

## Triggering

All of the following route to the same workflow:

- **Natural language** — "review uncommitted", "评审一下未提交的改动", "帮我看看这个 PR"
- **Slash-style text** — type `/review uncommitted` as a chat message; the token matches the skill's description triggers in hosts without a custom command system (e.g. DSH)
- **Slash command** (Claude Code) — Claude Code merges custom commands into skills, so this installs as `/pi-review` automatically, in addition to description-based auto-triggering

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
| Fresh review session on a session-tree branch | Review runs in the current conversation |
| Review widget + `/end-review` navigation | `end-review` produces the handoff summary in-chat |

## Credits & license

- Original workflow and rubric: [earendil-works/pi-review](https://github.com/earendil-works/pi-review) (rubric adapted from Codex's review feature) — © Earendil Inc., MIT
- Agent Skills port: © Panmax, MIT

See [LICENSE](./LICENSE).
