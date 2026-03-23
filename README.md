# Review Cycle — Claude Code Plugin

> [Русская версия](README.ru.md)

Automated PR review cycle for Claude Code. Requests a code review, intelligently triages reviewer comments, applies fixes, and repeats until approval — then squash-merges.

## Installation

```bash
claude plugin add /path/to/claude-code-review-cycle-skill
```

Or for a single session:
```bash
claude --plugin-dir /path/to/claude-code-review-cycle-skill
```

### Prerequisites

- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated with your account

## Usage

```
/review-cycle [pr-number]
```

If no PR number is provided, the skill auto-detects it from the current branch.

## How It Works

The skill runs an automated loop:

1. **Request review** — posts a comment asking the reviewer to focus on critical issues (bugs, security, logic errors)
2. **Wait for response** — polls for 2–3 minutes
3. **Triage comments** — a subagent reads the actual code and classifies each comment:

| Verdict | Action |
|---|---|
| `FIX` | Apply the fix (only after verifying the claim against actual code) |
| `ALREADY_FIXED` | Reply with the commit that addressed it |
| `SKIP` | Reply explaining it's cosmetic |
| `IRRELEVANT` | Reply noting it doesn't relate to the PR |
| `CONFLICTING` | Quote the contradicting comment, ask for clarification |
| `HALLUCINATION` | Reply with evidence from the codebase disproving the claim |

4. **Fix issues** — applies fixes, runs linter and tests
5. **Commit & push** — conventional commit message, return to step 1
6. **Finalize** — when approved, squash-merge and clean up the branch

## Key Features

- Verifies every reviewer claim against the actual codebase before fixing
- Detects hallucinated function names, wrong line numbers, false conventions
- Handles contradictions between review cycles
- Skips cosmetic nitpicks with polite explanations
- Runs lint (`ruff`) and tests (`pytest`) before every push

## License

MIT
