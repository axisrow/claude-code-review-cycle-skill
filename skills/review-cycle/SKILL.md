---
name: review-cycle
description: Automated PR review cycle — request review, fix issues, repeat until approved, then merge
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Agent
argument-hint: "[pr-number]"
---

# Review Cycle

Automated PR review cycle until full approval.

PR number: $ARGUMENTS (if not provided — detect from the current branch via `gh pr view --json number -q .number`).

## Cycle

Repeat the following steps until the reviewer approves the PR or no comments remain:

### 1. Request review
Leave a comment on the PR instructing the reviewer to focus on significant issues:
```
gh pr comment <PR> --body "@claude review. Focus on critical issues: bugs, security vulnerabilities, logical errors, data loss risks, performance problems. Do NOT nitpick style, naming conventions, minor formatting, or subjective preferences — only flag issues that could break functionality or cause real harm in production."
```

### 2. Wait for reviewer response
Wait 2 minutes, then check for new comments and review status:
```
gh pr view <PR> --json reviews,comments
```
If no response yet — wait another 1 minute and check again (maximum 1 additional attempt).

### 3. Analyze comments
Read all comments and review comments. If the status is "APPROVED" and there are no new comments — go to step 6.

### 3.1. Triage comments (subagent)
Launch a subagent (Agent tool) to triage each comment. The subagent must:
- Read the current code of files referenced in the comments
- Check whether the issue was already fixed in previous commits (compare with what the reviewer is requesting)
- Assess severity: critical (bug, security, logical error) vs cosmetic (style, naming, formatting)
- Check relevance: does the comment actually relate to this PR's code? The reviewer may be mistaken — referencing non-existent files, confusing function names, or providing feedback that clearly belongs to a different project/PR. Mark such comments as `IRRELEVANT`
- Check consistency: does the comment contradict previous comments from the same or another reviewer? If the reviewer asks for X now but asked for not-X in the previous cycle — mark as `CONFLICTING`
- **Verify every claim before assigning `FIX`**: for each comment, read the actual files and grep the codebase to confirm that every statement the reviewer makes is true. Do not take any claim at face value — LLM reviewers routinely hallucinate: non-existent functions, wrong line numbers, incorrect patterns, false project-wide conventions (e.g. "only English comments", "this method doesn't exist", "this pattern is used everywhere"). A `FIX` verdict is only valid when the underlying claim is confirmed by reading the actual code. If any claim is false — mark as `HALLUCINATION`
- Return a list of comments with a verdict: `FIX` (needs fixing), `ALREADY_FIXED` (already resolved), `SKIP` (cosmetic), `IRRELEVANT` (unrelated to this PR), `CONFLICTING` (contradicts previous comments), `HALLUCINATION` (reviewer's factual claim about the codebase is verifiably false)

Only fix comments with the `FIX` verdict. For other verdicts — leave a reply comment on the PR with an explanation:
- `ALREADY_FIXED` — specify which commit already addressed the issue
- `SKIP` — explain why the comment is cosmetic and does not affect functionality
- `IRRELEVANT` — politely note that the comment does not relate to this PR's code
- `CONFLICTING` — quote the contradicting previous comment and ask the reviewer to clarify
- `HALLUCINATION` — show concrete evidence from the codebase (grep results, file contents) that disproves the reviewer's claim

### 4. Fix issues
- Only fix comments with the `FIX` verdict from step 3.1
- Read the files referenced in the comments
- Apply fixes
- Run linter: `ruff check src/ tests/`
- Run tests: `pytest tests/ -v`

### 5. Commit and push
- Commit fixes with a meaningful message
- Push to remote
- Return to step 1

### 6. Finalization
When the PR is approved and no comments remain:
```bash
gh pr merge <PR> --squash --delete-branch
git checkout main
git pull
```

## Important
- All `gh` commands (and any other GitHub API calls) must be run via Bash with `dangerouslyDisableSandbox: true`, as the sandbox blocks TLS connections to api.github.com
- Do not skip critical comments — fix all with the `FIX` verdict. Cosmetic comments (`SKIP`) can be skipped with a reply
- Every commit must have a meaningful message following conventional commits style
- Run lint and tests before every commit
- If tests fail after fixes — fix them before pushing
- If the reviewer does not respond after 2 checks in a single cycle — notify the user and stop: the reviewer bot may be misconfigured or unavailable
