---
name: Feedback Hub Full Build
overview: Implement the full Feedback Hub system across both repos (usernode-dapp-starter for the widget, usernode-feedback-hub for the Hub dapp/server/Mayor), following the 4-phase roadmap in FEEDBACK_HUB_SPEC.md.
todos:
  - id: phase1a-widget
    content: "Phase 1A: Build usernode-feedback.js widget in dapp-starter (floating button, Shadow DOM, submit memo, progress bar, auto-detect target_app). Add serving routes. Embed in all example dapps."
    status: pending
  - id: phase1b-hub-minimal
    content: "Phase 1B: Generate HUB_PUBKEY. Build minimal hub/index.html (Feed + My Feedback screens, polls chain, displays raw feedback). Build server/server.js with chain poller + basic /api/feedback. Clean up example dapps from repo."
    status: pending
  - id: phase2a-sqlite
    content: "Phase 2A: Add SQLite persistence (server/db.js) with tables for issues, feedback_items, votes, active_participants. Wire chain poller to write to DB."
    status: pending
  - id: phase2b-ai-collation
    content: "Phase 2B: Build server/collator.js (AI grouping suggestions via LiteLLM) and server/prioritizer.js (priority scoring). Run on schedule + on new feedback."
    status: pending
  - id: phase2c-hub-grouping-voting
    content: "Phase 2C: Expand hub/index.html with Issue List, Issue Detail, and enhanced My Feedback screens. Add group/ungroup/vote transaction sending. State derivation with conflict resolution."
    status: pending
  - id: phase2d-thresholds
    content: "Phase 2D: Build server/thresholds.js (quorum/approval checks, 3-day wait, immediate promotion, auto-archive). Active participant caching."
    status: pending
  - id: phase2e-api
    content: "Phase 2E: Expand Hub Server REST API (/api/issues, /api/issues/:id, /api/feedback/ungrouped, /api/stats, /api/apps)."
    status: pending
  - id: phase3a-infra
    content: "Phase 3a: Set up docker-compose with Redis + LiteLLM + Hub Server + Mayor. Create config.json, litellm-config.yaml, mayor-sandbox/Dockerfile. Build mayor/orchestrator.js, registry.js, worker.js, sandbox.js, github-app.js."
    status: pending
  - id: phase3b-pr-pipeline
    content: "Phase 3b: Build GitHub Issue mirroring, PR creation via Octokit, @mayor command parser, webhook handler for review loop + merge detection. Failure handling (needs-help, manual)."
    status: pending
  - id: phase3c-polish
    content: "Phase 3c: CI auto-retry, cost tracking + budget enforcement, task timeouts, concurrency limits, mayor performance metrics."
    status: pending
  - id: phase4-scale
    content: "Phase 4: Screenshot support in widget, notification badges, stats/leaderboard screen, mayor dashboard, multi-repo issues, community-governed merges evaluation."
    status: pending
isProject: false
---

# Feedback Hub — Full Implementation Plan

## Current State

- `**usernode-feedback-hub**`: Clone of dapp-starter with `FEEDBACK_HUB_SPEC.md` added. No hub-specific code exists yet. Has the full dapp-starter infrastructure (bridge, usernames, server, examples, Docker, deploy workflow).
- `**usernode-dapp-starter**`: Production repo with bridge, usernames, server, and example dapps (HIM, Last One Wins, Falling Sands). No widget yet.

## Repo Boundary

Per the spec, the **widget** (`usernode-feedback.js`) lives in `usernode-dapp-starter` alongside the bridge and usernames module. The **Hub dapp, Hub Server, and Mayor Orchestrator** live in `usernode-feedback-hub`. They communicate only through on-chain transactions (widget writes, hub reads).

---

## Phase 1: Widget + On-Chain Feedback

### 1A — Feedback Widget (in `usernode-dapp-starter`)

Build `usernode-feedback.js` — a self-contained script that adds a floating feedback button to any dapp.

**Key decisions:**

- Shadow DOM for CSS isolation (avoids conflicts with host dapp styles)
- Auto-detects `target_app` from `data-app` attribute, then `location.pathname`, then `"unknown"`
- Configurable via `data-hub-url` and `data-hub-api` attributes on the script tag
- Uses the bridge's `sendTransaction` to submit feedback on-chain (no server dependency)
- Progress bar during submission (same pattern as HIM/Last One Wins)

**Files to create/modify:**

- Create `usernode-feedback.js` at repo root — floating button + form, Shadow DOM, memo format `{ app: "feedback", type: "submit", target_app, category, text }`
- Modify `server.js` — add serving route for `/usernode-feedback.js` (same pattern as bridge/usernames: `no-store` cache header)
- Modify `examples/server.js` — same serving route
- Modify each dapp HTML to include: `<script src="/usernode-feedback.js" data-app="him"></script>` (and equivalent for lastwin, falling-sands, starter)

**Memo schema:**

```json
{ "app": "feedback", "type": "submit", "target_app": "him", "category": "bug", "text": "..." }
```

Categories: `bug`, `feature`, `improvement`, `other`. Max text ~900 chars (JSON overhead ~75 chars within 1024 limit).

### 1B — Minimal Hub Dapp + Server (in `usernode-feedback-hub`)

A minimal version that reads raw feedback from the chain and displays it. No AI, no voting yet.

**Generate a unique keypair:**

```bash
node scripts/generate-keypair.js --env
```

This `APP_PUBKEY` becomes `HUB_PUBKEY` — the shared address for all feedback/vote transactions. The widget in dapp-starter needs this same value.

**Files to create:**

- `hub/index.html` — single-file dapp (follows HIM pattern: IIFE, dark/light theme, mobile-first layout, hash routing)
  - Screens: **Feed** (all feedback, filterable by app/category), **My Feedback** (user's submissions)
  - Includes bridge + usernames scripts
  - Polls `getTransactions({ account: HUB_PUBKEY })` every 4 seconds
  - Rebuilds state by parsing `submit` memos, sorting by timestamp
  - Rate-limit enforcement on read: max 5 per sender per 24h rolling window
  - Displays feedback items with username, app badge, category badge, timestamp
- `server/server.js` — Node.js server that:
  - Serves `hub/index.html` at `/` (or `/feedback`)
  - Serves bridge + usernames scripts
  - Includes explorer proxy (`/explorer-api/`* via `handleExplorerProxy` from `lib/dapp-server.js`)
  - Includes mock API (`/__mock/*` via `createMockApi` when `--local-dev`)
  - Runs a chain poller (`createChainPoller`) to index feedback transactions into memory (for Phase 2's API)
  - Exposes `GET /api/feedback` — returns indexed feedback (simple JSON, no DB yet)
- Copy `lib/dapp-server.js` from examples (shared server utilities)
- Update `Dockerfile` and `docker-compose.yml` for the hub server

**Cleanup:** Remove the example dapps (`examples/him`, `examples/last-one-wins`, `examples/falling-sands`, `examples/server.js`) from the feedback-hub repo — they belong in dapp-starter. Keep `lib/dapp-server.js`.

---

## Phase 2: AI Collation + User Grouping + Voting

### 2A — SQLite Persistence

- Create `server/db.js` — SQLite via `better-sqlite3`
  - Tables: `issues` (id, title, summary, target_app, category, priority, status, created_at, github_issue_number, github_issue_url, pr_url), `feedback_items` (tx_hash, sender, target_app, category, text, timestamp, issue_id nullable, status: submit/suggested/grouped), `votes` (tx_hash, sender, issue_id, value, timestamp), `active_participants` (app, address, last_seen, cached_at)
  - The chain poller writes to this DB; the API reads from it

### 2B — AI Collation

- Create `server/collator.js` — AI grouping suggestions
  - Runs every 5 minutes (or on new feedback)
  - Fetches ungrouped feedback from DB
  - Calls LLM (via LiteLLM proxy at `LITELLM_PROXY_URL`) to compare against existing issues
  - Creates new suggested issues or matches to existing ones
  - Stores suggestions in DB (not on-chain — these are AI suggestions)
- Create `server/prioritizer.js` — AI priority scoring (1-5)
  - Called by collator after grouping
  - Based on frequency, severity, recency, user impact

### 2C — Hub Dapp: Grouping + Voting UI

Expand `hub/index.html` with new screens and transaction types:

**New screens:**

- **Issue List** — filterable by app/category/status, sortable by votes/priority/recency. Each card shows: title, app badge, category badge, report count, priority stars, vote bar, quorum progress, pipeline status
- **Issue Detail** — AI-generated title/summary, confirmed feedback list, pending suggestions (accept/reject for submitters), vote bar with yes/no + quorum, your vote, pipeline status timeline
- **My Feedback** — badge for ungrouped count, AI suggestions with accept/ignore CTAs

**New on-chain transaction types:**

- `group`: `{ app: "feedback", type: "group", feedback_tx: "<hash>", issue: "<id>" }`
- `ungroup`: `{ app: "feedback", type: "ungroup", feedback_tx: "<hash>" }`
- `vote`: `{ app: "feedback", type: "vote", issue: "<id>", value: "yes"|"no" }`

**State derivation:**

- Chain poller picks up `group`, `ungroup`, `vote` transactions
- `group`/`ungroup` verified on read: sender must match original `feedback_tx` sender
- Votes: latest per user per issue wins
- Issue enters "voting" when it has >= 1 confirmed grouping

### 2D — Voting Thresholds + Auto-Promotion

- Create `server/thresholds.js` — vote counting and promotion logic
  - **Quorum**: unique voters > 1/3 of active participants for target app
  - **Approval**: > 2/3 of votes are "yes"
  - **Standard promotion**: at passing threshold for 3 consecutive days
  - **Immediate promotion**: > 2/3 of total app users voting with > 2/3 yes
  - **Auto-archive**: only 1 user activity for 3 days
  - Active participant cache: refreshed hourly from explorer API
- Server scheduler runs threshold checks periodically (every hour or on new votes)

### 2E — Hub Server REST API

Expand `server/server.js` with the full API surface:

- `GET /api/issues` — list with filters/sort
- `GET /api/issues/:id` — detail with feedback, suggestions, votes, status
- `GET /api/issues/:id/suggestions` — ungrouped feedback AI thinks belongs here
- `GET /api/feedback/ungrouped?address=...` — user's ungrouped feedback with AI matches
- `GET /api/stats` — aggregate stats (active participants, issue counts, throughput)
- `GET /api/apps` — known apps with participant counts

---

## Phase 3a: Mayor Infrastructure

### Docker + External Services

- Update `docker-compose.yml` to match the spec:
  - **Redis** (redis:7-alpine) — for BullMQ task queues
  - **LiteLLM proxy** — model routing
  - **Hub Server** — existing, add depends_on
  - **Mayor Orchestrator** — new service, mounts Docker socket
- Create `litellm-config.yaml` — LLM provider routing config
- Create `config.json` — Mayor Registry mapping target_app to repo/path/model/limits
- Create `mayor-sandbox/Dockerfile` — sandbox image (node:20-bookworm + git + gh CLI + eslint/prettier)

### Mayor Orchestrator

- Create `mayor/orchestrator.js` — Express API:
  - `POST /api/tasks` — accept task spec from Hub Server
  - `GET /api/tasks/:id` — task status
  - `POST /api/tasks/:id/cancel` — cancel running task
- Create `mayor/registry.js` — reads `config.json`, provides `getConfig(targetApp)`
- Create `mayor/worker.js` — BullMQ worker:
  - Pulls tasks from per-repo queues (`mayor:him`, `mayor:lastwin`, etc.)
  - Clones repo, creates branch `feedback/<issueId>-<slug>`
  - Spawns Docker sandbox container
  - Runs OpenHands headless inside container
  - Pushes branch, reports status
- Create `mayor/sandbox.js` — Docker container lifecycle (create, start, wait, destroy with resource limits)
- Create `mayor/github-app.js` — GitHub App auth via `@octokit/auth-app`, installation token generation

### Hub Server Integration

- Wire `server/server.js` to POST task specs to Mayor Orchestrator when issues reach "ready" status
- Add `GET /api/issues/:id/spec` — AI-generated task spec endpoint

---

## Phase 3b: PR Pipeline

### GitHub Issue Mirroring

- Create `mayor/github-issues.js` — create GitHub Issues when issues reach "ready":
  - Title, summary, vote results, grouped feedback items, labels (`feedback-hub`, app, category)
  - Store GitHub Issue number/URL in DB

### PR Creation + Review Loop

- Extend `mayor/worker.js`:
  - Create draft PRs via Octokit with `Fixes #N`, labels `ai-generated` + app name
  - Report PR URL back to Hub Server
- Create `mayor/commands.js` — parse `@mayor` commands from PR comments:
  - `revise: <instruction>` — re-run on existing branch
  - `retry` — fresh branch
  - `explain: <question>` — comment-only response
  - `scope: <narrower>` — re-run with narrowed scope
  - `abort` — close PR, move issue back to "ready"
- Add webhook handler in `mayor/orchestrator.js`:
  - `issue_comment` — detect `@mayor` commands
  - `pull_request` — detect merges, update Hub status to "merged"
  - `pull_request_review` — detect "changes requested"
  - `issues` — detect manual close/reopen of mirrored issues

### Failure Handling

- On agent failure: create PR anyway with `needs-help` label + explanation
- Track retry count per issue; after 3 failures, move to "manual" status
- Hub dapp shows "needs-help" and "manual" states with explanation

---

## Phase 3c: Agent Polish

- CI failure auto-retry via `check_suite` webhook
- Per-repo model configuration tested across providers (Anthropic, OpenAI, local)
- Cost tracking: log LLM token usage per task, enforce `maxBudgetUsd`
- Task timeout handling (kill sandbox after configurable limit)
- Concurrency limits per mayor enforced via BullMQ `concurrency` option
- Mayor performance metrics: success rate, time-to-merge, cost-per-PR, retry rate

---

## Phase 4: Polish + Scale

### In `usernode-dapp-starter`:

- Screenshot support in widget (image upload, IPFS/blob storage, reference in memo)
- Notification badges in widget (dot indicator for ungrouped feedback via `data-hub-api`)

### In `usernode-feedback-hub`:

- Stats/Leaderboard screen: top contributors, pipeline throughput, per-app breakdown
- Mayor performance dashboard: success rate, avg time ready-to-merged, cost metrics
- Multi-repo issue support (issues spanning multiple repos)
- Evaluate community-governed merges (auto-merge after community vote on PR)

---

## Architecture Diagram

```mermaid
flowchart TB
  subgraph dappStarter ["usernode-dapp-starter"]
    Widget["usernode-feedback.js"]
    Bridge["usernode-bridge.js"]
    Usernames["usernode-usernames.js"]
    HIM["HIM dapp"]
    LOW["Last One Wins"]
    FS["Falling Sands"]
  end

  subgraph chain ["On-Chain (UTXO Memos)"]
    SubmitTx["submit txs"]
    GroupTx["group/ungroup txs"]
    VoteTx["vote txs"]
  end

  subgraph feedbackHub ["usernode-feedback-hub"]
    HubDapp["hub/index.html"]
    HubServer["server/server.js"]
    Collator["server/collator.js"]
    Thresholds["server/thresholds.js"]
    DB["SQLite"]
    Mayor["mayor/orchestrator.js"]
    Worker["mayor/worker.js"]
    Sandbox["Docker Sandbox"]
  end

  subgraph external ["External Services"]
    Redis["Redis + BullMQ"]
    LiteLLM["LiteLLM Proxy"]
    GitHub["GitHub API"]
    LLM["LLM Providers"]
  end

  Widget -->|sendTransaction| SubmitTx
  HubDapp -->|sendTransaction| GroupTx
  HubDapp -->|sendTransaction| VoteTx
  HubDapp -->|"REST /api/*"| HubServer
  HubServer -->|chain poller| SubmitTx
  HubServer -->|chain poller| GroupTx
  HubServer -->|chain poller| VoteTx
  HubServer --> DB
  HubServer -->|"POST /api/tasks"| Mayor
  Collator --> LiteLLM
  LiteLLM --> LLM
  Mayor --> Redis
  Mayor --> Worker
  Worker --> Sandbox
  Worker --> GitHub
  Thresholds -->|"issue ready"| GitHub
```



## Key Technical Decisions

- **Hub dapp is a single HTML file** following the same pattern as HIM (IIFE, inline CSS/JS, bridge + usernames, hash routing, dark/light theme, tx progress bar)
- **Hub Server and Hub dapp are co-deployed** on the same origin (relative API paths like `/api/issues`)
- **Chain poller is the source of truth bridge** — all on-chain data (feedback, groups, votes) flows through the poller into SQLite; the REST API reads from SQLite
- **AI suggestions are off-chain** (stored in SQLite); user confirmations are on-chain (`group`/`ungroup` txs)
- **Mayor Orchestrator is deterministic Node.js** — no AI reasoning in the orchestrator itself; AI runs only inside sandboxed OpenHands agents
- **GitHub App for auth** — short-lived installation tokens, bot identity, webhook support
- **LiteLLM for model routing** — swap LLM providers per-repo via config, no code changes

## Dependencies to Add

- `**usernode-feedback-hub`**: `better-sqlite3`, `bullmq`, `@octokit/rest`, `@octokit/auth-app`, `@octokit/webhooks`, `ioredis` (or use bullmq's built-in), `express` (or keep raw http like server.js)
- `**usernode-dapp-starter**`: No new dependencies (widget is zero-dependency like the bridge)

