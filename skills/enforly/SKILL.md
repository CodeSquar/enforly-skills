---
name: enforly
description: Add Enforly guardrails to code. Enforly is an API that evaluates content, data, or an operation against a natural-language policy and returns allow, deny, or review. Use when the user wants to approve, block, or send to human review something before it happens - an agent tool call, a database query, a payment, an email, a published post, a file share, a deploy, or any step in an API, job, or automation - or mentions Enforly, @enforly/sdk, guardrails, policy checks, tool-call approval, or human-in-the-loop.
---

# Enforly

Enforly evaluates what your code is about to do (or accept, publish, share) against a policy written in plain language, and returns `allow`, `deny`, or `review`. It judges only the evidence you send; your code applies the decision. It does not execute anything and does not guarantee safety or compliance on its own.

Use it at the point right before an effect: before a tool runs, a query executes, money moves, a message is sent, or content goes live.

## Setup

1. Create an API key at https://enforly.com/dashboard/keys (free plan: 200 checks/month).
2. Store it as `ENFORLY_API_KEY` in the server environment. Never put it in browser code, client bundles, or `NEXT_PUBLIC_*`/`VITE_*`/`PUBLIC_*` variables.
3. Install the SDK (Node.js 20+, server-side only):

```bash
npm install @enforly/sdk
```

```ts
import { Enforly, EnforlyError } from '@enforly/sdk'

const guard = new Enforly({ apiKey: process.env.ENFORLY_API_KEY! })
```

Other languages: call the HTTP API directly (see "HTTP API" below).

## The one rule: only `allowed === true` proceeds

```ts
async function guarded<T>(policy: string, data: unknown, run: () => Promise<T>) {
  let result
  try {
    result = await guard.check({ policy, data })
  } catch (error) {
    // Network, auth, quota, timeout, invalid input: never treat as allowed.
    throw new Error('Guardrail check failed; operation stopped', { cause: error })
  }
  if (result.decision === 'allow') return run()
  if (result.decision === 'review') {
    // Ask a human or collect more evidence (e.g. explicit user confirmation), then re-check.
    throw new Error(`Needs review (requestId ${result.requestId})`)
  }
  throw new Error(`Denied by policy (requestId ${result.requestId})`)
}
```

- `deny` → stop. `review` → stop and escalate to a human or ask the user for the missing evidence. Errors → stop.
- `review` comes back when the policy hands the case to a person ("...a person decides", "...needs manager approval") and that approval is not in `data`, or when the evidence is unclear. Resolve it as described in "Resolving a review" below, never by skipping the next check.
- Do not add a fallback that runs the operation when Enforly is unreachable. Fail closed.
- Log `requestId` to correlate with the Enforly dashboard activity.

## Three ways to pass a policy (send exactly one)

```ts
// 1. Inline policy text
await guard.check({ policy: 'Never refund more than $200 without a manager approval in the ticket.', data })

// 2. One saved policy (private to the tenant or a platform preset); evaluates the tenant's active version
await guard.check({ policyId: '00000000-0000-4000-8000-000000000001', data })
// Optional: pin an exact version
await guard.check({ policyId: '00000000-0000-4000-8000-000000000001', policyVersion: 1, data })

// 3. Several saved policies in one request (plan limit: Free 10, Basic 25, Pro 50); each is an ID or { id, version }
const r = await guard.check({ policyIds: [idA, { id: idB, version: 2 }], data })
// r.decision: any deny → deny; else any review → review; else allow. r.results has one entry per policy.
```

Prefer saved policies (`policyId`/`policyIds`) when the same rule is used in several places or should be edited without a deploy. Manage them with `guard.policies.list() / get(id) / create({ slug, title, body }) / update(id, { body }) / delete(id)`. Updates create a new immutable version and make it active; in the dashboard a rule change can also be saved as a draft (activated later), and renaming a policy does not create a version because checks never read the title. `delete(id)` deactivates the policy and returns `{ id, deactivated: true }`: checks that use its ID then fail with `POLICY_NOT_FOUND`, and the dashboard keeps it listed read-only with its version history. Without a pinned version, a check uses the tenant's active version: a private policy's latest edit, or, for a preset, the version that was current when the account was created (preset updates never change behavior silently). Roll back or adopt a newer version with `guard.policies.activate(id, version)` or from the dashboard; `latest_version` in `list()`/`get()` shows when a newer one exists. Pin `policyVersion` when a deploy must keep evaluating the exact same rule. `00000000-0000-4000-8000-000000000001` is the platform preset "No destructive database operations".

## What to put in `data`

`data` is free-form JSON. Enforly only knows what is in it, so include:

- **The concrete operation**: tool/function name and exact arguments (the SQL, the amount and currency, the recipient, the file path, the text to publish).
- **The evidence the policy refers to**: if the policy says "unless the user confirmed", include the recent conversation or a `userConfirmed` field with its source. If it mentions roles, environments, or limits, include them.
- **Context that changes the answer**: environment (`production` vs `staging`), actor, account plan, prior approvals.

Keep it under 32,768 characters total (policy bodies + data). Trim long conversations to the last relevant turns; oversized requests fail with `413 CONTEXT_LIMIT_EXCEEDED` (not billed). Do not send secrets or credentials in `data`.

```ts
await guard.check({
  policyId: DB_SAFETY_POLICY_ID,
  data: {
    action: { tool: 'database.execute_sql', arguments: { query: 'DELETE FROM orders WHERE created_at < now() - interval \'1 year\'' } },
    environment: 'production',
    conversation: lastMessages.slice(-6),
  },
})
```

## Writing good policies

- State the rule and its exception in one or two sentences: "Never X unless Y."
- Name the evidence that satisfies the exception, so the caller knows what to send in `data`.
- One concern per policy; combine them with `policyIds`.
- To route a case to a person instead of blocking it, say so in the policy: `Refund damaged products. If the customer already used them, a person decides.` That case returns `review`; once `data` includes the approval, it can return `allow`.
- Say who can approve. "Requires review" or "a person decides" means someone other than the requester, so the requester's own "yes" never satisfies it and the check keeps returning `review`. If the requester's confirmation is enough, write that instead: `Before deleting records, require the user's explicit confirmation of this specific deletion.` If someone else must approve, name the role and where the approval lives: `...a manager approval recorded in the ticket.`

Examples:
- `Never allow destructive database operations (DROP, TRUNCATE, DELETE or UPDATE without WHERE) unless the user explicitly confirmed this specific operation.`
- `Do not send emails to external domains that contain customer personal data or attachments from the internal drive.`
- `Refunds above 200 USD require a manager approval recorded in the ticket.`
- `Do not publish posts that make medical claims or mention competitors by name.`
- `Shell commands must not delete files outside the project directory, force-push, or modify CI secrets.`

## Resolving a review

Enforly keeps no state between checks: each one judges only its own `data`. A `review` is resolved by checking again with the approval as evidence.

1. Hold the operation and get the approval the policy names: the user's explicit confirmation, or a decision from the approver (manager, security, compliance) through your own queue, ticket, or dashboard.
2. Check the original operation again with the approval in `data`. Do not check the approval alone ("yes") as if it were a new request.
3. Run the operation only if that check returns `allow`. If it returns `review` again, the approval does not match what the policy asks for: fix the policy wording or collect the right approval.

```ts
// First check returned review; an approver accepted it in your app.
const result = await guard.check({
  policy,
  data: {
    action,                                                          // the same operation as before
    approval: { by: 'manager', userId: approver.id, approved: true, at: new Date().toISOString() },
  },
})
if (result.allowed) await run(action)
```

In a chat or agent, keep the pending request and send it together with the turn where the user confirms (or the recorded approval). Take approvals from trusted application state (a signed-in approver, a ticket field, a stored confirmation), not from claims inside the content being evaluated. Never let any follow-up message skip the check: that turns `review` into `allow` for anyone who types "yes".

## Recipes

### Guard every tool of an agent (Vercel AI SDK, Mastra, LangChain, OpenAI Agents, MCP)

Wrap the tool's `execute` function; the framework stays the same.

```ts
function guardTool<A, R>(name: string, execute: (args: A, ...rest: any[]) => Promise<R>, getEvidence: () => unknown) {
  return async (args: A, ...rest: any[]) => {
    const result = await guard.check({
      policyIds: POLICY_IDS,
      data: { action: { tool: name, arguments: args }, evidence: getEvidence() },
    })
    if (!result.allowed) {
      // Return a message the model can read instead of executing.
      return { blocked: true, decision: result.decision, requestId: result.requestId } as unknown as R
    }
    return execute(args, ...rest)
  }
}

// Vercel AI SDK
const deleteRows = tool({
  description: 'Delete rows from a table',
  inputSchema: z.object({ table: z.string(), where: z.string() }),
  execute: guardTool('deleteRows', async ({ table, where }) => db.delete(table, where), () => ({ conversation: messages.slice(-6) })),
})
```

Let SDK errors from `guard.check` propagate (or catch and return a blocked result); never execute on error.

### Claude Code hook (block risky commands before they run)

`.claude/hooks/enforly.mjs`:

```js
import { Enforly } from '@enforly/sdk'

const input = JSON.parse(await new Promise(r => { let s = ''; process.stdin.on('data', c => s += c).on('end', () => r(s)) }))
const reply = (permissionDecision, permissionDecisionReason) => {
  console.log(JSON.stringify({ hookSpecificOutput: { hookEventName: 'PreToolUse', permissionDecision, permissionDecisionReason } }))
  process.exit(0)
}

try {
  const guard = new Enforly({ apiKey: process.env.ENFORLY_API_KEY })
  const result = await guard.check({
    policy: 'Never delete files outside the project, force-push, drop or truncate databases, or read secret files such as .env unless the task explicitly requires it.',
    data: { tool: input.tool_name, input: input.tool_input, cwd: input.cwd },
  })
  if (result.decision === 'allow') reply('allow', 'Enforly: allow')
  if (result.decision === 'deny') reply('deny', `Enforly denied this (requestId ${result.requestId})`)
  reply('ask', `Enforly flagged this for review (requestId ${result.requestId})`)
} catch (error) {
  reply('ask', `Enforly check failed: ${error.code ?? error.message}`)
}
```

`.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash|Write|Edit", "hooks": [{ "type": "command", "command": "node .claude/hooks/enforly.mjs" }] }
    ]
  }
}
```

The hook needs `@enforly/sdk` installed in the project and `ENFORLY_API_KEY` in the environment.

### Non-AI uses

The same call works without any agent: check a webhook payload before accepting it, a user-generated post before publishing, a CSV export before sharing it, or a deploy request before running it. Put the item and its context in `data`, apply the decision.

## Response shape

Single policy (`policy` or `policyId`):

```json
{ "decision": "deny", "allowed": false, "violationProbability": 0.94, "reviewProbability": 0.04, "policyId": null, "policyVersion": null, "requestId": "..." }
```

Multiple policies (`policyIds`):

```json
{ "decision": "review", "allowed": false, "results": [{ "policyId": "...", "policyVersion": 3, "decision": "review", "violationProbability": 0.08, "reviewProbability": 0.91 }], "requestId": "..." }
```

`violationProbability` and `reviewProbability` (0-1) are model estimates, not proof: the chance the data breaks the policy, and the chance the policy leaves this case to a person whose approval is missing. `policyId`/`policyVersion` are `null` for inline policies.

## Errors

The SDK throws `EnforlyError` with `code`, `status`, and `requestId`. Every error means: do not proceed.

| Status | Code | Meaning |
| --- | --- | --- |
| 400 | `INVALID_INPUT`, `INVALID_JSON`, `POLICY_LIMIT_EXCEEDED` | Bad body, more than one policy field, `policyVersion` without `policyId`, duplicate `policyIds`, or too many `policyIds` for the plan |
| 401 | `UNAUTHORIZED` | Missing, wrong, revoked, disabled, or expired API key |
| 404 | `POLICY_NOT_FOUND` | A policy ID is not visible to this key (nothing is evaluated) |
| 404 | `POLICY_VERSION_NOT_FOUND` | A pinned version does not exist for that policy (nothing is evaluated) |
| 413 | `CONTEXT_LIMIT_EXCEEDED`, `PAYLOAD_TOO_LARGE` | Trim `data` or use fewer policies |
| 429 | `RATE_LIMITED`, `MONTHLY_QUOTA_EXCEEDED` | Per-minute or monthly plan limit; upgrade at https://enforly.com/dashboard/billing |
| 503 | `SERVICE_UNAVAILABLE` | Temporary; retry with backoff, but keep the operation stopped meanwhile |

SDK-only codes: `TIMEOUT` (default 30 s, set `timeoutMs`), `NETWORK_ERROR`, `INVALID_RESPONSE`, `INVALID_INPUT`.

## HTTP API

```bash
curl -X POST https://api.enforly.com/v1/check \
  -H "Authorization: Bearer $ENFORLY_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"policy":"Never run destructive database operations without explicit confirmation.","data":{"sql":"DROP TABLE users","userConfirmed":false}}'
```

Endpoints (all require the bearer key): `POST /v1/check`, `GET /v1/policies`, `POST /v1/policies` (`slug`, `title`, `body`), `GET /v1/policies/:id`, `PUT /v1/policies/:id` (`body`; new active version), `PUT /v1/policies/:id/active-version` (`version`; rollback or adopt), `DELETE /v1/policies/:id` (deactivate; returns `{ id, deactivated: true }`).

## Checklist before finishing an integration

- [ ] The check runs before the effect, on the server, with the key from the environment.
- [ ] Only `allowed === true` executes; `review`, `deny`, and every error stop.
- [ ] `data` contains the exact operation plus the evidence the policy mentions.
- [ ] Each policy that can return `review` says who approves; a review is resolved by re-checking the operation with that approval in `data`.
- [ ] `requestId` is logged.
- [ ] Tested once with an input that should be allowed and one that should be denied.

Docs: https://enforly.com/docs
