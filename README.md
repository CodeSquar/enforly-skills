# Enforly skills

Agent skills for integrating [Enforly](https://enforly.com): an API that evaluates content, data, or an operation against a natural-language policy and returns `allow`, `deny`, or `review` before your code acts on it.

With the skill installed, ask your coding agent things like:

- "Guard the tools of my agent with Enforly so it can't drop tables without confirmation."
- "Check user posts with Enforly before publishing them."
- "Add a Claude Code hook that blocks force-pushes and deleting files outside the repo."

## Install

```bash
npx skills add CodeSquar/enforly-skills
```

Or copy [`skills/enforly`](skills/enforly) into your agent's skills directory (for Claude Code: `.claude/skills/enforly`).

## What it teaches the agent

- Install `@enforly/sdk` and keep `ENFORLY_API_KEY` server-side.
- Use inline `policy`, a saved `policyId`, or up to the plan limit of `policyIds`.
- Pin a saved policy version, or rely on the tenant's active version (rollback via dashboard or `policies.activate`).
- Deactivate saved policies (`policies.delete` returns `{ id, deactivated: true }`); checks using a deactivated ID fail with `POLICY_NOT_FOUND`.
- Proceed only on `allow`; stop on `deny`, `review`, and every error.
- Put the exact operation and its evidence in `data`.
- Write policies that say who approves, and resolve a `review` by checking again with the approval in `data`.
- Recipes for agent tool calls, a Claude Code PreToolUse hook, and non-AI checks.

Get a free API key at [enforly.com](https://enforly.com).

## License

MIT
