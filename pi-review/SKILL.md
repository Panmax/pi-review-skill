---
name: pi-review
description: Structured code-review workflow ported from earendil-works/pi-review (Codex-style rubric). Use when the user wants a code review — review uncommitted changes, review against a base branch, review a specific commit, review a GitHub pull request, or snapshot-review folders/files. Also handles the post-review handoff ("end-review" summary) and "fix review findings". Triggers: /review, review uncommitted, review branch main, review commit abc123, review pr 123, review folder src docs, 代码评审, 评审一下, 帮我 review, end-review, 修复评审问题.
---

# pi-review — Code Review Workflow

A code-review skill for coding agents, ported from [`earendil-works/pi-review`](https://github.com/earendil-works/pi-review) (the Pi `/review` + `/end-review` extension). It works in any agent that loads Agent Skills (`SKILL.md`): DSH, Claude Code, pi, and others. The review rubric, git recipes, and output contracts are unchanged from the original; the parts tied to Pi's TUI (interactive selector, session-tree branching, review widget) are replaced by conversation-driven behavior.

Users may invoke it in natural language ("review uncommitted", "评审一下未提交的改动") or with slash-style text ("/review uncommitted", "/review branch main") — all forms route to the workflow below. In Claude Code the skill is additionally exposed as the `/pi-review` command.

## 1. Parse the request into a review target

| Request | Target |
|---|---|
| "review uncommitted" | uncommitted |
| "review branch \<name\>" | base branch diff |
| "review commit \<sha\> [title...]" | single commit (title is optional context) |
| "review pr \<number \| GitHub URL\>" | pull request |
| "review folder \<paths...\>" | snapshot review (not a diff); paths are whitespace-separated |
| `--extra "..."` or `--extra=...` (works with any mode) | additional user-provided review instruction |

Target selection rules:

- An explicit target is used as-is, without confirmation.
- With no target given: default to **uncommitted** when the working tree has uncommitted changes (staged, unstaged, or untracked); otherwise ask the user which target to review, suggesting the diff against the default branch. (The original extension auto-selected "base branch" when on a feature branch and "commit" otherwise; asking first is the chat-appropriate equivalent — see the README's differences table.)
- If the current directory is not a git repository, say so and stop (the original guarded this with `git rev-parse --git-dir`).
- `--extra` with no value is an error — ask for the missing value.

## 2. Gather the changes (via shell commands)

- **uncommitted**: `git status --porcelain`; inspect `git diff HEAD` for tracked changes and read untracked files directly.
- **baseBranch**: resolve the merge base first — `git rev-parse --abbrev-ref '<branch>@{upstream}'`, then `git merge-base HEAD <upstream>`; if that fails, fall back to `git merge-base HEAD <branch>`. Then inspect `git diff <mergeBaseSha>`.
- **commit**: `git show <sha>` and review the full diff it introduces.
- **pullRequest**: requires `gh`. Verify it is installed and authenticated (`gh auth status`); if not, give the setup hint (install from https://cli.github.com/ — macOS `brew install gh` — then `gh auth login`). Get base branch and title via `gh pr view <n> --json baseRefName,title,headRefName`, compute the merge base as in baseBranch mode, and inspect `git diff <mergeBaseSha>`. PR checkout requires a clean working tree (no changes to tracked files); if dirty, tell the user to commit or stash first. Note: the original always ran `gh pr checkout <n>`; this port checks out only when the PR head is not available locally, so it never moves the user's working tree unnecessarily (see the README's differences table).
- **folder**: no diff. Read the files under the given paths directly (snapshot review).

### Focus text for each mode (verbatim from the original)

Use these exact strings as the mode-specific focus (see §4 for where they go), substituting the placeholders:

- **uncommitted**

  ```text
  Review the current code changes (staged, unstaged, and untracked files) and provide prioritized findings.
  ```

- **baseBranch**, merge base resolved

  ```text
  Review the code changes against the base branch '{baseBranch}'. The merge base commit for this comparison is {mergeBaseSha}. Run `git diff {mergeBaseSha}` to inspect the changes relative to {baseBranch}. Provide prioritized, actionable findings.
  ```

- **baseBranch**, no merge base

  ```text
  Review the code changes against the base branch '{branch}'. Start by finding the merge diff between the current branch and {branch}'s upstream e.g. (`git merge-base HEAD "$(git rev-parse --abbrev-ref "{branch}@{upstream}")"`), then run `git diff` against that SHA to see what changes we would merge into the {branch} branch. Provide prioritized, actionable findings.
  ```

- **commit**, with title

  ```text
  Review the code changes introduced by commit {sha} ("{title}"). Provide prioritized, actionable findings.
  ```

- **commit**, no title

  ```text
  Review the code changes introduced by commit {sha}. Provide prioritized, actionable findings.
  ```

- **pullRequest**, merge base resolved

  ```text
  Review pull request #{prNumber} ("{title}") against the base branch '{baseBranch}'. The merge base commit for this comparison is {mergeBaseSha}. Run `git diff {mergeBaseSha}` to inspect the changes that would be merged. Provide prioritized, actionable findings.
  ```

- **pullRequest**, no merge base

  ```text
  Review pull request #{prNumber} ("{title}") against the base branch '{baseBranch}'. Start by finding the merge base between the current branch and {baseBranch} (e.g., `git merge-base HEAD {baseBranch}`), then run `git diff` against that SHA to see the changes that would be merged. Provide prioritized, actionable findings.
  ```

- **folder**

  ```text
  Review the code in the following paths: {paths}. This is a snapshot review (not a diff). Read the files directly in these paths and provide prioritized, actionable findings.
  ```

## 3. Project review guidelines

Walk up from the current directory to find the project anchor: the first directory containing a `.dsh` directory, falling back to the first containing `.git`. Look for `REVIEW_GUIDELINES.md` in **that anchor directory only**, and stop the upward search there — if the anchor has no such file, there are no project instructions. (This mirrors the original, which anchored on the directory containing `.pi` and stopped there instead of continuing up into parent repositories.)

When found, append its contents as the project-instructions block described in §4; it overrides the default rubric where more specific.

## 4. Perform the review

Assemble the review input in exactly this order — the rubric's own precedence rule ("if you encounter more specific guidelines elsewhere … those override these general instructions") depends on the order and wording of these blocks:

1. The review rubric below, verbatim.
2. The line `Please perform a code review with the following focus:` followed by the mode-specific focus text from §2.
3. If recurring shared review instructions apply to all reviews, the line `Shared custom review instructions (applies to all reviews):` followed by them. (The original stored these in session state via its selector; here the durable equivalent is `REVIEW_GUIDELINES.md`, while one-off additions arrive through `--extra`.)
4. If the user passed `--extra`, the line `Additional user-provided review instruction:` followed by that text.
5. If a `REVIEW_GUIDELINES.md` was found (§3), the line `This project has additional instructions for code reviews:` followed by its contents.

Act as the code reviewer defined by the rubric and emit the output format it requires (findings with [P0]–[P3], verdict "correct" or "needs attention", and the mandatory Human Reviewer Callouts section).

# Review Guidelines

You are acting as a code reviewer for a proposed code change made by another engineer.

Below are default guidelines for determining what to flag. These are not the final word — if you encounter more specific guidelines elsewhere (in a developer message, user message, file, or project review guidelines appended below), those override these general instructions.

## Determining what to flag

Flag issues that:
1. Meaningfully impact the accuracy, performance, security, or maintainability of the code.
2. Are discrete and actionable (not general issues or multiple combined issues).
3. Don't demand rigor inconsistent with the rest of the codebase.
4. Were introduced in the changes being reviewed (not pre-existing bugs).
5. The author would likely fix if aware of them.
6. Don't rely on unstated assumptions about the codebase or author's intent.
7. Have provable impact on other parts of the code — it is not enough to speculate that a change may disrupt another part, you must identify the parts that are provably affected.
8. Are clearly not intentional changes by the author.
9. Be particularly careful with untrusted user input and follow the specific guidelines to review.
10. Treat silent local error recovery (especially parsing/IO/network fallbacks) as high-signal review candidates unless there is explicit boundary-level justification.
11. Violate the clean-code guidelines below.
12. Introduce error handling that conflicts with the fail-fast guidelines below.

## Clean-code guidelines

1. Check whether each newly added function duplicates existing functionality elsewhere in the codebase. Flag actual duplication and identify the existing implementation.
2. Flag one-off helper functions that add indirection without improving clarity or reuse (for example, `isRecord` or `asString`).
3. Flag abstractions introduced without a concrete need in the reviewed change, including wrappers created only for hypothetical future use.
4. Flag defensive checks or fallback behavior that mask programming errors, especially when callers already guarantee the relevant invariants.

## Untrusted User Input

1. Be careful with open redirects, they must always be checked to only go to trusted domains (?next_page=...)
2. Always flag SQL that is not parametrized
3. In systems with user supplied URL input, http fetches always need to be protected against access to local resources (intercept DNS resolver!)
4. Escape, don't sanitize if you have the option (eg: HTML escaping)

## Comment guidelines

1. Be clear about why the issue is a problem.
2. Communicate severity appropriately - don't exaggerate.
3. Be brief - at most 1 paragraph.
4. Keep code snippets under 3 lines, wrapped in inline code or code blocks.
5. Use ```suggestion blocks ONLY for concrete replacement code (minimal lines; no commentary inside the block). Preserve the exact leading whitespace of the replaced lines.
6. Explicitly state scenarios/environments where the issue arises.
7. Use a matter-of-fact tone - helpful AI assistant, not accusatory.
8. Write for quick comprehension without close reading.
9. Avoid excessive flattery or unhelpful phrases like "Great job...".

## Review priorities

1. Surface critical non-blocking human callouts (migrations, dependency churn, auth/permissions, compatibility, destructive operations) at the end.
2. Prefer simple, direct solutions over wrappers or abstractions without clear value.
3. Treat back pressure handling as critical to system stability.
4. Apply system-level thinking; flag changes that increase operational risk or on-call wakeups.
5. Ensure that errors are always checked against codes or stable identifiers, never error messages.

## Fail-fast error handling (strict)

When reviewing added or modified error handling, default to fail-fast behavior.

1. Evaluate every new or changed `try/catch`: identify what can fail and why local handling is correct at that exact layer.
2. Prefer propagation over local recovery. If the current scope cannot fully recover while preserving correctness, rethrow (optionally with context) instead of returning fallbacks.
3. Flag catch blocks that hide failure signals (e.g. returning `null`/`[]`/`false`, swallowing JSON parse failures, logging-and-continue, or “best effort” silent recovery).
4. JSON parsing/decoding should fail loudly by default. Quiet fallback parsing is only acceptable with an explicit compatibility requirement and clear tested behavior.
5. Boundary handlers (HTTP routes, CLI entrypoints, supervisors) may translate errors, but must not pretend success or silently degrade.
6. If a catch exists only to satisfy lint/style without real handling, treat it as a bug.
7. When uncertain, prefer crashing fast over silent degradation.

## Required human callouts (non-blocking, at the very end)

After findings/verdict, you MUST append this final section:

## Human Reviewer Callouts (Non-Blocking)

Include only applicable callouts (no yes/no lines):

- **This change adds a database migration:** <files/details>
- **This change introduces a new dependency:** <package(s)/details>
- **This change changes a dependency (or the lockfile):** <files/package(s)/details>
- **This change modifies auth/permission behavior:** <what changed and where>
- **This change introduces backwards-incompatible public schema/API/contract changes:** <what changed and where>
- **This change includes irreversible or destructive operations:** <operation and scope>
- **This change adds or removes feature flags:** <feature flags changed> (call out re-use of dormant feature flags!)
- **This change changes configuration defaults:** <config var changed>

Rules for this section:
1. These are informational callouts for the human reviewer, not fix items.
2. Do not include them in Findings unless there is an independent defect.
3. These callouts alone must not change the verdict.
4. Only include callouts that apply to the reviewed change.
5. Keep each emitted callout bold exactly as written.
6. If none apply, write "- (none)".

## Priority levels

Tag each finding with a priority level in the title:
- [P0] - Drop everything to fix. Blocking release/operations. Only for universal issues that do not depend on assumptions about inputs.
- [P1] - Urgent. Should be addressed in the next cycle.
- [P2] - Normal. To be fixed eventually.
- [P3] - Low. Nice to have.

## Output format

Provide your findings in a clear, structured format:
1. List each finding with its priority tag, file location, and explanation.
2. Findings must reference locations that overlap with the actual diff — don't flag pre-existing code.
3. Keep line references as short as possible (avoid ranges over 5-10 lines; pick the most suitable subrange).
4. Provide an overall verdict: "correct" (no blocking issues) or "needs attention" (has blocking issues).
5. Ignore trivial style issues unless they obscure meaning or violate documented standards.
6. Do not generate a full PR fix — only flag issues and optionally provide short suggestion blocks.
7. End with the required "Human Reviewer Callouts (Non-Blocking)" section and all applicable bold callouts (no yes/no).

Output all findings the author would fix if they knew about them. If there are no qualifying findings, explicitly state the code looks good. Don't stop at the first finding - list every qualifying issue. Then append the required non-blocking callouts section.

## 5. End-of-review handoff ("end-review" / 结束评审)

When the user finishes a review and asks for the handoff, produce the structured summary below so the findings can be acted on immediately. Do not omit findings — include every actionable issue identified during the review.

The interactive review is ending; produce the structured handoff below so it can be used immediately to implement fixes.

You MUST summarize the review that just happened so findings can be acted on.
Do not omit findings: include every actionable issue that was identified.

Required sections (in order):

## Review Scope
- What was reviewed (files/paths, changes, and scope)

## Verdict
- "correct" or "needs attention"

## Findings
For EACH finding, include:
- Priority tag ([P0]..[P3]) and short title
- File location (`path/to/file.ext:line`)
- Why it matters (brief)
- What should change (brief, actionable)

## Fix Queue
1. Ordered implementation checklist (highest priority first)

## Constraints & Preferences
- Any constraints or preferences mentioned during review
- Or "(none)"

## Human Reviewer Callouts (Non-Blocking)
Include only applicable callouts (no yes/no lines):
- **This change adds a database migration:** <files/details>
- **This change introduces a new dependency:** <package(s)/details>
- **This change changes a dependency (or the lockfile):** <files/package(s)/details>
- **This change modifies auth/permission behavior:** <what changed and where>
- **This change introduces backwards-incompatible public schema/API/contract changes:** <what changed and where>
- **This change includes irreversible or destructive operations:** <operation and scope>

If none apply, write "- (none)".

These are informational callouts for humans and are not fix items by themselves.

Preserve exact file paths, function names, and error messages where available.

## 6. Fix the findings ("fix review findings" / 修复评审问题)

Use the latest review summary in this session and implement the review findings now.

Instructions:
1. Treat the summary's Findings/Fix Queue as a checklist.
2. Fix in priority order: P0, P1, then P2 (include P3 if quick and safe).
3. If a finding is invalid/already fixed/not possible right now, briefly explain why and continue.
4. Treat "Human Reviewer Callouts (Non-Blocking)" as informational only; do not convert them into fix tasks unless there is a separate explicit finding.
5. Follow fail-fast error handling: do not add local catch/fallback recovery unless this scope is an explicit boundary that can safely translate the failure.
6. If you add or keep a `try/catch`, explain the expected failure mode and either rethrow with context or return a boundary-safe error response.
7. JSON parsing/decoding should fail loudly by default; avoid silent fallback parsing.
8. Run relevant tests/checks for touched code where practical.
9. End with: fixed items, deferred/skipped items (with reasons), and verification results.