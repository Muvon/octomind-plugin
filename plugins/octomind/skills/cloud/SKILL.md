---
name: cloud
description: Drive Octomind Cloud from outside over REST - create and manage octos (cloud agents) and their machines, run conversations headlessly, schedule routines, read usage (Developer API, api.octomind.run, om_ key), and call models through the OpenAI-compatible hub (hub.octomind.run/v1). Use when the user mentions Octomind Cloud, octos, cloud agents or machines, the developer API, an om_ key, hub.octomind.run, calling models through the hub, or automating/scheduling octos from code or CI.
license: Apache-2.0
---

# Octomind Cloud from outside

An octo is an agent for one workstream: its own workspace directory on a machine, plus its sessions (conversations) and routines. Two separate HTTP surfaces, two separate credentials.

## Credentials

| | Developer key | Hub key |
|---|---|---|
| Starts with | `om_` | (not documented) |
| Base URL | `https://api.octomind.run/api/v1` | `https://hub.octomind.run/v1` |
| Does | control plane: octos, machines, conversations, routines, usage | model calls only (OpenAI-compatible) |
| Env var | `OCTOMIND_API_KEY` | `OCTOHUB_API_KEY` |

- Both go in `Authorization: Bearer <key>`. `OCTOMIND_API_KEY` is this skill's own naming convention, not a variable the API defines.
- They are not interchangeable: the control plane rejects hub keys, and a developer key can never call the gateway.
- Create them in the panel Keys page (`https://octomind.run/app/keys`). No developer key exists by default. With an existing developer key, `POST /account/keys` mints either: body `{"name":"ci","kind":"api"}` or `{"kind":"hub"}` (default `hub`). The plaintext is shown once.
- A key is bound to its context: personal key = personal account, team-owned key = the team.
- NEVER ask the user to paste a key into chat and never echo one. Test presence with `[ -n "$OCTOMIND_API_KEY" ]`. If unset, tell the user to create the key in the panel and `export` it in their own shell.
- Do not try the cloud MCP server (`/api/v1/mcp`). It needs a hub key plus an `X-Octomind-Machine` header and is for agents running inside cloud machines, not for an external client.

## Step 0: read the contract

Operations change; the live contract wins over this file. It is public, no auth, a bare OpenAPI 3.1 JSON document:

```bash
curl -s https://api.octomind.run/api/v1/openapi > /tmp/octomind-openapi.json
jq '.paths | keys' /tmp/octomind-openapi.json
jq '.paths["/sessions/{id}/message"]' /tmp/octomind-openapi.json   # one operation
```

Read only the operations you need, including their error codes. Changes within v1 are additive-only.

## Response shape

Success is `[null, data]`; failure is `["e_code", details]` with an honest HTTP status. Parse with `jq '.[1]'` and check `.[0]` first. Every response carries `X-Request-Id`; quote it to support.

## Core loop: octo, conversation, message, poll

```bash
API=https://api.octomind.run/api/v1
api() { curl -sS -H "Authorization: Bearer $OCTOMIND_API_KEY" -H "Content-Type: application/json" "$@"; }

# 1. create an octo (placed on a machine with room, or a new machine). Keep .[1].id and .[1].machine_id
api -X POST $API/octos -H "Idempotency-Key: $(uuidgen)" -d '{"name":"CI","about":"keeps the build green"}'
# or reuse one:  api $API/octos

# 2. start a conversation with its first turn in one call; keep .[1].id (session id)
api -X POST $API/machines/$MACHINE_ID/sessions -H "Idempotency-Key: $(uuidgen)" \
  -d '{"octo_id":"'$OCTO_ID'","title":"fix build","message":"clone repo X and make the tests pass"}'
# .[1].run is accepted | session_busy | session_runtime_unavailable

# 3. poll raw frames from a cursor until working=false; pass back .[1].next as the new cursor
api "$API/sessions/$SESSION_ID/history?after=0"

# 4. follow-up turn (async by default)
api -X POST $API/sessions/$SESSION_ID/message -d '{"text":"now add a regression test"}'
```

Details that matter:
- `POST /machines/{machine_id}/sessions`: `octo_id` must be an octo of yours on THIS machine; without it the session belongs to the machine's first octo. `message` is optional (max 65536 chars). `role` defaults to the cloud orchestrator.
- `POST /sessions/{id}/message` body: `text` (required), `wait_s` (1-120) long-polls and returns `{events, working, next}` when the turn completes or after `wait_s` seconds, whichever is first (on timeout `working` is `true`: keep polling from `next`); absent/0 returns `{accepted, working}` at once. Also `model` (run this turn on a hub model), `media_ids` (max 4), `request_id`.
- History frames: `{events, working, next}`. Each event is `{id, at, dir, type, frame}`; `id` is the opaque cursor. The octo's reply text is in events with `type: "assistant"`, at `.frame.content`. A turn ends with a `cost`, `status` or `error` frame. `history` without `after` returns a reduced transcript page (`turns` 1-100, default 20; page older with `before`) that also carries `delegating`.
- Delegation: the orchestrator may hand work to a specialist. That turn then ends with a `cost` frame whose payload has `pending_work: true`: a promise, not the answer, and `history?after=` already reads `working: false`. Keep polling (or stay on `events`) until a later `cost`, `status` or `error` frame arrives without `pending_work`; that one is final. With `wait_s`, `working` stays `true` while the delegated result is still being collected, so `false` there means final. The server stops waiting for a delegated result after 6 h.
- Push instead of poll (works with `curl -N`, no WebSocket client needed):
  `api -N "$API/sessions/$SESSION_ID/events"` sends `id: <cursor>` + `data: <frame JSON>` events and a heartbeat every 25 s; resume with `?after=` or `Last-Event-ID`.
- `GET /octos/{id}` shows `state`: `working`, `idle`, `asleep` (machine suspended), `blocked` (usage limit reached). `GET /octos/{id}/activity` lists its sessions and routine runs, 30 per page.
- Slash commands (`/model`, `/effort`, `/info`, `/done`, `/plan`, `/schedule`, `/agents`) work as message text.
- Clean up when the job is over: `api -X DELETE $API/octos/$OCTO_ID` removes the octo, its workspace, sessions and routines; its machine stays.

## Busy sessions, retries, rate limits

- One driver per session: a message while a turn is open returns `409 session_busy` (also for turns started from the panel). Do not blind-retry: poll `history` until `working` is false, or queue the message with `POST /sessions/{id}/queue` (`request_id` required, idempotent on it; it goes out when the session is free; max 20 queued/failed per session, else `409 queue_full`). Inspect with `GET /sessions/{id}/queue`.
- Session create and message wake a suspended machine automatically; you do not manage machine state for them. `POST /machines/{id}/start` and `/stop` exist if you want to. Files endpoints do not wake it (below).
- Send an `Idempotency-Key` header on every POST you may retry. A 2xx response is replayed for 24 h (same method, path and key; key up to 200 chars, stored response up to 1 MiB); errors are not stored, so they re-execute.
- Rate limits are per developer key per minute, by plan (numbers: `.info.description` of the OpenAPI document). Every authenticated response has `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. Over the limit: `429` with `Retry-After` (seconds) and body `["rate_limited", {"retry_after": n}]` - sleep that long, then retry.
- Concurrent SSE streams are capped per plan; over the cap is also `429 rate_limited`.
- Other named errors to handle: `402` (`subscription_required`, `account_blocked`, `member_capped`, `insufficient_usage`), `403 cloud_unavailable`, `409 limit_reached` (octo, machine or routine caps; on session create `limit` is `concurrent_sessions` with `retry_after_s`, so retry later), `409 machine_restoring` (archived machine being restored; retry later), `422 model_not_available`, `422 invalid_octo`, `503 session_runtime_unavailable` / `provision_failed` / `no_capacity`. Do not retry 402/403/422; report them to the user.

## Routines (scheduled or webhook runs)

`POST /routines` required: `name`, `prompt`. Useful fields: `octo_id` (or `machine_id`), `trigger` (`schedule` default, or `webhook`), `cron_expr` (five fields, required for a schedule), `tz` (default `UTC`), `session_mode` (`new` | `continue`), `on_overlap` (`skip` | `queue_one` | `cancel_previous`), `notify` (`always` | `on_change` | `on_failure` | `never`), `model`, `check_cmd`, `max_minutes`, `max_cost_usd`, `enabled`.
- `GET /routines/{id}` returns the routine with its recent `runs` (`status`, `result_text`, `error_text`, `cost_usd`); `PATCH` changes only the fields sent (plus `rotate_webhook`); `DELETE` keeps the run ledger.
- Possible errors: `422 invalid_cron | invalid_timezone | interval_too_short`, `409 limit_reached` (routines), `403 plan_required` (webhook).
- Set `max_cost_usd` and `max_minutes` on unattended routines.

## Other control-plane groups (read the OpenAPI for fields)

Machines (`/machines`, patch to rename/retime/grow disk; size is immutable); files (`/machines/{id}/files`: list, download up to 256 MB with `download=1`, upload up to 8 MB, delete; a suspended machine answers `409 not_running`, and only a download with `wake=1` wakes it); env vars (`/account/vars`, `/machines/{id}/vars`: values are masked on read, a `null` value deletes, machine vars apply on next start); `/images`; `/sessions` (all machines); account and spend (`/account`, `/account/usage`, `/account/usage/events|machines|models`, `/account/storage`); `/models` (the whole live roster of every kind, with health `status`; not filtered by plan or key); `/plans`; `/teams*`.

Among others, routes that exist in the API code and the cloud docs page but are NOT in the OpenAPI document: `POST /routines/{id}/run` (run now), `GET /routines/{id}/runs`, `POST /machines/{id}/media` (upload attachments, then pass the ids as `media_ids`). Treat them as unstable and tell the user.

## Hub: OpenAI-compatible model gateway

Base URL `https://hub.octomind.run/v1`, `Authorization: Bearer $OCTOHUB_API_KEY`. Always set `model` explicitly (the id below is only an example; check it is still listed).

```bash
curl -sS https://hub.octomind.run/v1/models -H "Authorization: Bearer $OCTOHUB_API_KEY"   # ids this key may call
curl -sS https://hub.octomind.run/v1/chat/completions \
  -H "Authorization: Bearer $OCTOHUB_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4.1-flash","messages":[{"role":"user","content":"Explain Rust in one sentence."}]}'
```

- Pick model ids from the public catalog, never from memory: `curl -s https://api.octomind.run/api/v1/catalog | jq '.[1] | keys'`. It is an envelope `[null, {models, embeddings, evaluations, media, services, included_paid, included_free}]`. Prices (`input_per_m`, `output_per_m`) and ids change; read them there. The catalog keeps retired models; skip any whose `deprecated_at` date has passed. Hub `GET /v1/models` lists only the ids the key's allow-list permits; control-plane `GET /models` lists the whole live roster (all kinds), so a listed id can still be refused with `422 model_not_available`.
- `POST /v1/chat/completions` is classic OpenAI (point any OpenAI SDK at the base URL). `stream: true` is accepted but emulated: the upstream call finishes first, then it is re-framed as SSE, so first-token latency does not improve. `tool_choice` is accepted and ignored.
- `POST /v1/completions` is octohub's native Responses API: `model`, `input` (string or items), `instructions`, `previous_completion_id` (pass the prior response `id` to continue a chain), `max_output_tokens`, `reasoning_effort`, `text.format` (`json_object` | `json_schema`), flat `tools`. Tool results go back as `function_call_output` input items.
- Approved WebSocket contract (pending final implementation review; not a deployed-availability guarantee): authenticated `GET /v1/completions` uses an HTTP/1.1 WebSocket upgrade. Send one existing native completion JSON text message per call; receive one buffered final JSON text reply `{status:<HTTP numeric status>,headers:<lowercase string-valued map>,body:<existing JSON response/error>}` (`headers` has lowercase header names and string values). Requests are serial per socket: wait for the reply before sending the next request; no multiplexing or token streaming. `POST /v1/completions` and chat SSE are unchanged; non-completion routes remain HTTP.
  - Use native WebSocket ping/pong. Bearer authentication is required at the handshake and revalidated on every request. The server enforces the existing 16 MiB input cap; oversized input may close the socket with code `1009`. Error envelopes preserve the existing HTTP status and JSON body discriminators, including `error.type`, and carry `Retry-After` as `headers["retry-after"]` when present.
  - The octolib `octohub` provider transport is configurable: HTTP (library default) or WebSocket (requires the `websockets` feature). Octomind's new root setting defaults to `octohub_transport = "websocket"`; set `octohub_transport = "http"` to override. `OCTOHUB_API_URL` remains an `http(s)` URL; the library derives `ws(s)` while preserving private/local host and port. External proxies must be checked separately for upgrade support.
- `POST /v1/embeddings` `{model, input}` (string or array): the response body is the raw vector(s) with no envelope or usage. Embedding ids come from the catalog `embeddings` list.
- Media (same key): `POST /v1/images/generations`, `/v1/videos`, `/v1/audio/speech`, `/v1/audio/transcriptions`, then `GET /v1/media/{id}`; `GET /v1/media/models` lists capabilities and price. `"wait": false` or a long job returns `202` with a job id to poll.
- Errors: `401` missing/revoked key, `403` model not allowed for this key, `400` unknown model or bad request, `413` request body over 16 MiB (octohub 0.14.5+, rejected before any model call), `429` concurrency budget full, or every provider rate window exhausted (then with `Retry-After`), `503` no provider slot in time, `504` upstream timed out, `500` upstream failure. Providers may fail over to another lane server-side, but once that is exhausted the error reaches you: handle it and retry with backoff.
- Hub concurrency is a per-account in-flight budget shared by all your keys. At the limit a request queues about 15 s, then returns `429`: lower parallelism and retry.

## What costs money

State this to the user before launching large or unattended work.
- Model tokens are metered per call at provider list price, whether made inside octos or through the hub with a key. Paid plans draw their allowance first, then prepaid credits; Free has no allowance and draws prepaid credits only.
- Also metered, one ledger `meter` each: `media_gen`, `web_search`, `browser_start`, `browser_minute`, `byok_request`, `voice_in`, `voice_out`, `cpu` and `ram` (machine compute while it runs), `activation` (each new conversation), `disk` (working disk above the included amount), `storage` (shared storage above the plan allowance), `network` (egress above the included amount). Hub media calls bill per unit produced, not per token. There is no per-request fee for the Developer API itself.
- When neither allowance nor credits remain, model calls pause (octo `state` is `blocked`, calls return `402`); machines, files and history stay intact.
- Check spend with `GET /account/usage` and `GET /account/usage/events`. Prices and plan limits drift: read `/catalog`, `/plans` and the OpenAPI text, do not quote numbers from memory.

## Pitfalls

- Using a hub key on `api.octomind.run` (or the reverse) fails by design. Use the right env var for each host.
- Polling `history` without advancing the cursor re-reads old frames; always pass back `next`.
- Two clients on one session collide (`409 session_busy`). Give each job its own session.
- `DELETE /octos/{id}` and `DELETE /machines/{id}` are destructive (a machine's volume is destroyed). Confirm with the user first.
- `PATCH /machines/{id}`: `suspend_timeout_min` 0 = always on; disk only grows.
- Routine `model`, session `model` and octo `model` must be ids the plan allows, else `422 model_not_available`.

## Read more

- Humans: `https://octomind.run/cloud/docs#devapi` (Developer API), `#api` (hub), `#octos`, `#sessions`, `#machines`, `#routines`, `#models`, `#billing`.
- Contract: `https://api.octomind.run/api/v1/openapi`. Public catalog: `https://api.octomind.run/api/v1/catalog`.
