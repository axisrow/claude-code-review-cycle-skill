# CLAUDE.md

## Project Overview

Claude Code plugin providing the `/review-cycle` skill — an automated PR review loop. Requests a code review, triages reviewer comments (FIX / SKIP / HALLUCINATION / IRRELEVANT / CONFLICTING / ALREADY_FIXED), applies fixes, and repeats until approval, then squash-merges.

## Structure

```
.claude-plugin/plugin.json        # plugin manifest
skills/review-cycle/SKILL.md      # skill definition
```

## Key Details

- Single skill, no scripts or dependencies
- Requires `gh` CLI authenticated with GitHub
- All `gh` commands need `dangerouslyDisableSandbox: true` (sandbox blocks TLS to api.github.com)
- Skill uses Agent tool for comment triage (subagent)
- Language: skill prompts in English, responds in user's language
