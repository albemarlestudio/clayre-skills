# Connect an assistant to Clayre — setup & troubleshooting

This is the public setup and troubleshooting guide for the Clayre connector.
It covers signing in, choosing a key, what the connector can do, priced tasks,
and the errors you can hit.

- MCP endpoint: `https://www.clayre.app/mcp`
- REST/OpenAPI: `https://www.clayre.app/api/v1/openapi.json` (same bearer key)
- Website: `https://www.clayre.app`

## What the connector does

Clayre brings your marketing workspace into your assistant: saved brand
knowledge, the content calendar, ideas, inbox items and post results. You can
review what is planned and what awaits approval, approve rows and replies,
schedule approved posts through your connected provider, and run priced tasks —
each shown with an exact quote a person confirms in the browser before it
starts. You decide what is ready to go out.

## Sign in or use a key

**OAuth sign-in (recommended).** Add `https://www.clayre.app/mcp` as a custom
connector in your assistant and sign in. The connector discovers the OAuth door
from `/.well-known/oauth-protected-resource`, opens the Clayre consent page,
and gets a scoped token. Tokens renew silently; revoke access in the Clayre
app at **Settings → Connections → Assistants** under "Signed-in assistants".

**Scoped key.** Mint a key in the Clayre app at **Settings → Connections →
Assistants → Create key**. It is shown **once** — copy it immediately; it looks
like `clyr_…`. Send it as a `Bearer` header, never in a URL. A key only sees
actions its scopes include; the owner or an admin can revoke it with one click.

### Per-assistant setup

- **Claude** — Settings → Connectors → Add custom connector → URL
  `https://www.clayre.app/mcp` → **Sign in** (OAuth, no key needed), or pass
  the key as a bearer token.
- **Claude Code** — `claude mcp add --transport http clayre
  https://www.clayre.app/mcp`, then sign in when prompted. A `Bearer <key>`
  header works too.
- **ChatGPT (OpenAI Responses API)** — use the `mcp` tool with the key in
  `headers.Authorization`:
  ```json
  {
    "type": "mcp",
    "server_label": "clayre",
    "server_url": "https://www.clayre.app/mcp",
    "headers": { "Authorization": "Bearer <key>" },
    "allowed_tools": ["list_calendar", "list_ideas", "list_reviews", "read_brand_profile"],
    "require_approval": "always"
  }
  ```
  The key must travel in `headers.Authorization` — a top-level `authorization`
  field is ignored and the server answers 401.
- **Anything REST** — OpenAPI at
  `https://www.clayre.app/api/v1/openapi.json`; bearer key on `/api/v1/*`.

## Scopes

| Scope | What it unlocks |
|---|---|
| `read` | Every read/search tool (brand profile, calendar, insights, ideas, reviews, previews, search). |
| `capture` | Create ideas and draft rows, steer the calendar, edit editable rows. |
| `gates:approve` | Approval gates: approve rows, release sourcing, approve assets, send back. |
| `assets:write` | Upload and attach your own assets to rows. |
| `publish` | `publish_row` — schedule an approved row to its connected channel. |
| `runs:start` | Named tasks: `list_jobs`, `quote_job`, `start_job`, `get_job`. |

## Priced tasks

`quote_job` prices a task exactly and returns a browser confirmation link; a
person opens the link and approves the price; `start_job` then runs it. Even a
zero-credit task needs the confirmation click. `get_job`/`list_jobs` report
status and results.

Separately, `publish_row` has a pass-through cost on X: an X post debits 8
credits per linked X entry plus 1 credit per extra thread item, automatically
at scheduling (released if the schedule is cancelled). Other channels are
included in the plan.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401` on `/mcp` | No or bad bearer | Sign in via OAuth, or send the key in `headers.Authorization` (not a top-level field). |
| `403 insufficient_scope` | Key lacks the scope | Re-mint with the scope, or use OAuth sign-in scoped to your role. |
| `429` / `rate_limited` | Bucket exceeded | Respect `Retry-After`: 60 requests/min overall, 10 writes/min, 1 gate/publish call per row per minute. |
| `503 rate_limiter_unavailable` | Limiter unreachable | Retry after 5s; the surface fails closed. |
| `preview_required` | Stale or missing preview receipt | Call `get_row_preview` on the current revision, then approve within 15 minutes with the same key. |
| `asset_warnings` | Validation findings on the asset | Review the findings; approve with `acceptWarnings: true` if acceptable. |
| `locked` / `field_refused` | Row past the editable window or field not whitelisted | Check `list_calendar` for the row's state; only editable rows take edits. |
| `channel_not_connected` / `publishing_unavailable` | Channel or provider not connected | Connect the channel/provider in the Clayre app first. |
| `gate_refused` | A gate named blockers | Read `findings`/`missingFacts` in the result and resolve them. |
| `browser_consent_required` on `start_job` | The quote was never confirmed | Open the confirmation link `quote_job` returned and approve the price. |
| `job_not_found` | Unknown task name | Call `list_jobs` — it returns the available task definitions. |

If none of these resolve it, open **Settings → Connections → Assistants** in
the Clayre app to check the key or signed-in assistant's scopes, or remove and
re-add the connector.
