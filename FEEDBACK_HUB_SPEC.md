# Feedback Hub — Product Spec

## Context

### What is Usernode?

Usernode is a Layer 1 blockchain designed to run on mobile phones. Users run full nodes from the Usernode Flutter mobile app. The chain uses a UTXO model with zero-knowledge proofs, VRF-based block production, and a peer-to-peer network.

### What are Usernode dapps?

Dapps are lightweight web apps (single HTML files) that run inside the mobile app's WebView. They interact with the blockchain through a JavaScript bridge (`usernode-bridge.js`) that provides three primitives: `getNodeAddress()`, `sendTransaction()`, and `getTransactions()`. All dapp state lives on-chain as JSON-encoded transaction memos — no backend database required.

### The dapp ecosystem today

The dapp-starter repo (`usernode-dapp-starter`) contains the bridge, a dev server with mock endpoints, and several example dapps:

- **Human Input Market (HIM)** — A prediction market where users vote on questions and bet credits on outcomes using a Dynamic Parimutuel Market mechanism. Votes are hidden until reveal checkpoints. Markets settle based on vote results. Features a credit system, leaderboard, and voter dividends.
- **Last One Wins** — A token game where players send tokens to a shared pot. If nobody sends for 24 hours, the last sender wins everything. Has server-side automated payouts.
- **Falling Sands** — A collaborative pixel sandbox where users draw with physics-simulated elements. Strokes are committed on-chain and a server runs a WASM physics engine streamed to all clients via WebSocket.

All dapps share infrastructure: the bridge for chain interaction, `usernode-usernames.js` for a global username system, an explorer API proxy, and common UI patterns (dark/light themes, transaction progress bars, mobile-first layouts).

### Shared username system

A global username module (`usernode-usernames.js`) provides cross-dapp usernames. Users set their name once via a transaction to a shared usernames address (`app: "usernames", type: "set_username"`), and every dapp resolves it. Legacy per-app usernames are supported as fallback.

### How dapp data works

Dapps store all data as transaction memos (max 1024 characters JSON). Each dapp has a shared "app address" that all users transact with. State is derived client-side by replaying the full transaction history — no server database for core state. The memo always includes an `app` field for filtering and a `type` field for action discrimination.

### Where this product fits

The Feedback Hub closes the loop between users and developers. Today, if a user finds a bug or wants a feature, there's no structured path from feedback to shipped code. This product creates that path — feedback collection embedded in every dapp, AI-powered triage, community-driven prioritization, and AI-generated pull requests — all transparent and on-chain where possible.

---

## End-to-End Example

A concrete walkthrough of a bug report going from feedback to merged PR. This traces every system interaction to show how the pieces fit together.

### Day 0: Three users notice the same bug

**Alice** is using HIM on her phone. She sells shares in a market but the balance display still shows the old number. She taps the feedback button (bottom-right corner), selects "Bug", types "After selling shares, my balance doesn't update until I refresh the page", and hits Submit.

Behind the scenes:
- The widget calls `sendTransaction(HUB_PUBKEY, 1, '{"app":"feedback","type":"submit","target_app":"him","category":"bug","text":"After selling shares, my balance doesn't update until I refresh the page"}')` via the bridge
- The transaction lands on-chain within ~30–90 seconds (progress bar shows status)
- The widget shows "Thanks! Your feedback is on-chain." and closes

**Bob** and **Carol** independently submit similar feedback the same day — Bob writes "sell button doesn't update the balance", Carol writes "balance is wrong after selling, have to reload".

### Day 0 + 5 minutes: AI collation runs

The Hub Server's chain poller picks up all three new `submit` transactions. The collation scheduler fires (runs every 5 minutes or on new feedback).

The AI (via LiteLLM proxy) compares the three items against existing issues in SQLite. No match found — these are new. The AI clusters them together and creates a new suggested issue:
- **Issue ID**: `fb-042`
- **Title**: "Balance doesn't update after selling shares"
- **Summary**: "Three users report that selling shares in HIM does not refresh the displayed balance. Users must manually reload the page to see the updated amount."
- **Target app**: `him`
- **Category**: `bug`
- **Priority**: 4 (high — blocks core functionality, multiple reports)

The AI also records a suggested grouping: `fb-042` ← {Alice's tx, Bob's tx, Carol's tx}. This is stored in SQLite — not on-chain, because it's an AI suggestion that users haven't confirmed yet.

### Day 0 + 10 minutes: Users confirm the grouping

Alice opens the Feedback Hub dapp. The Hub dapp calls `GET /api/feedback/ungrouped?address=ut1_alice...` to check her feedback status.

The UI shows:
> **Your feedback**: "After selling shares, my balance doesn't update until I refresh the page"
> **AI suggests**: This matches issue "Balance doesn't update after selling shares" (2 other reports)
> **[Accept grouping]** · [Ignore]

Alice taps "Accept grouping". The Hub dapp sends an on-chain transaction:
`sendTransaction(HUB_PUBKEY, 1, '{"app":"feedback","type":"group","feedback_tx":"tx_alice_abc...","issue":"fb-042"}')`

This is Alice's on-chain confirmation that she agrees with the AI's grouping. Bob does the same later that day. Carol doesn't visit the Hub.

**Issue `fb-042` now has 2 confirmed groupings** (Alice and Bob). It enters the **voting** stage — it's now visible in the Hub's issue list and votable by the community.

### Days 1–3: Community votes

The Hub dapp shows `fb-042` in the issue list with its current vote tally. HIM has 90 active users (unique senders in the last 30 days).

Over the next few days, community members browse the Hub, filter by "HIM" and "Bug", and vote. Each vote is an on-chain transaction:
`sendTransaction(HUB_PUBKEY, 1, '{"app":"feedback","type":"vote","issue":"fb-042","value":"yes"}')`

By Day 3: 35 votes — 31 yes, 4 no.
- Quorum check: 35 voters > 30 (1/3 of 90 active users) — **passes**
- Approval check: 31/35 = 89% > 66.7% — **passes**
- The issue is now at passing threshold. The 3-day wait timer starts.

### Day 6: Auto-promotion

The threshold check scheduler runs. `fb-042` has been at passing threshold for 3 consecutive days. No one has changed their vote to push it below threshold.

The Hub Server:
1. Updates `fb-042` status to **"ready"**
2. Creates a GitHub Issue in `org/usernode-dapp-starter` via the GitHub App:
   - Title: "Balance doesn't update after selling shares"
   - Body: summary, vote results ("31 yes / 4 no — 89% approval"), grouped feedback items with usernames, link to Hub detail page
   - Labels: `feedback-hub`, `him`, `bug`
   - → GitHub Issue **#42** is created
3. POSTs a task spec to the Gastown adapter:
   ```json
   {
     "issueId": "fb-042",
     "targetApp": "him",
     "title": "Balance doesn't update after selling shares",
     "summary": "Three users report that selling shares...",
     "feedbackItems": ["tx_alice_abc...", "tx_bob_def..."],
     "category": "bug",
     "priority": 4,
     "githubIssue": { "repo": "org/usernode-dapp-starter", "number": 42 }
   }
   ```
4. Updates `fb-042` status to **"in-progress"**

### Day 6 + minutes: Gastown Mayor dispatches agents

The Gastown adapter:
1. Looks up the config for `"him"` in the Mayor Registry → rig `him`, repo `org/usernode-dapp-starter`, path `dapps/him/`
2. Creates a bead from the task spec and a convoy for `fb-042`
3. Slings the bead to the `him` rig

The Gastown Mayor in the `him` rig:
4. Picks up the convoy, determines this is a small bug fix (single bead, no decomposition needed)
5. Slings the bead to a polecat
6. The polecat works in an isolated Git worktree:
   - Reads `CLAUDE.md` for project conventions
   - Reads GitHub Issue #42 via `gh issue view 42` for full context
   - Examines `dapps/him/him.html`, finds the sell handler doesn't call `refreshLoopOnce()` after the transaction completes
   - Adds the missing `await refreshLoopOnce();` call
   - Runs linting (eslint, prettier) — passes
   - Commits: `fix: refresh balance display after share sale`
7. Pushes branch to GitHub
8. Creates a draft PR:
   - Title: `[AI Mayor] Fix: Balance doesn't update after selling shares`
   - Body: `Fixes #42` + summary + link to Hub issue
   - Labels: `ai-generated`, `him`
9. Adapter detects convoy completion, reports PR URL to Hub Server → status updated to **"pr-open"**

### Day 6 + hours: Maintainer reviews

A maintainer sees the PR on GitHub. The code change looks right but they want a tweak:

> `@mayor revise: also call refreshLoopOnce() after the buy handler for consistency`

GitHub fires an `issue_comment` webhook → Gastown adapter relays via `gt mail --stdin` → the polecat receives the instruction, makes the additional change, pushes → PR auto-updates.

The agent comments on the PR: "Pushed revision addressing your feedback."

### Day 7: Merge

The maintainer approves and merges the PR. GitHub auto-closes Issue #42 (because the PR body contained `Fixes #42`).

The `pull_request` webhook fires → Gastown adapter reports to Hub Server → `fb-042` status updated to **"merged"**.

Alice opens the Hub and sees her feedback item now shows: "Status: Merged — [PR #43](https://github.com/org/usernode-dapp-starter/pull/43)". The fix ships in the next deploy.

### What this walkthrough exercises

| Step | Components involved |
|---|---|
| Feedback submission | Widget → Bridge → Chain |
| AI collation | Hub Server chain poller → LLM via LiteLLM → SQLite |
| Grouping confirmation | Hub dapp UI → Bridge → Chain → Hub Server chain poller |
| Voting | Hub dapp UI → Bridge → Chain → Hub Server threshold checker |
| Auto-promotion | Hub Server scheduler → SQLite |
| GitHub Issue creation | Hub Server → GitHub App → GitHub API |
| Task dispatch | Hub Server → Gastown adapter → Beads → Convoy → Rig |
| Agent execution | Gastown Mayor → Polecat → Git worktree → git → GitHub |
| PR creation | Polecat → `gh pr create` → GitHub API |
| Review loop | GitHub webhook → Gastown adapter → `gt mail` → Mayor/Polecat |
| Merge detection | GitHub webhook → Gastown adapter → Hub Server |

### Design notes surfaced by this walkthrough

1. **Hub dapp identity**: The Hub dapp calls `getNodeAddress()` via the bridge — same as any other dapp. It must include `usernode-bridge.js` and `usernode-usernames.js`.

2. **Hub dapp ↔ Hub Server**: Co-deployed on the same origin. The Hub dapp uses relative API paths (`/api/issues`). The Hub Server serves the Hub dapp HTML alongside the API — same pattern as other dapp servers.

3. **Issue IDs are server-assigned**: The `issue` field in `group` memos (e.g., `"fb-042"`) comes from the Hub Server. Users discover these IDs via the Hub dapp UI (`/api/feedback/ungrouped` or `/api/issues`). Grouping requires visiting the Hub — it can't be done from on-chain data alone. This is acceptable since grouping is inherently a Hub UI action.

4. **Ungrouped feedback handling**: Feedback that a user never confirms stays in the "suggested" state. The AI collation re-evaluates ungrouped items on every pass (every 5 minutes), potentially re-suggesting them for new or different issues. The Hub dapp's "My Feedback" screen shows a persistent badge for any ungrouped items. For the widget: if the user has ungrouped feedback, the widget's feedback button shows a small dot indicator as a nudge to visit the Hub.

5. **Active participant caching**: The server maintains a cached set of active participants per app (refreshed hourly via explorer API) to avoid expensive per-vote lookups during threshold checks.

6. **Graceful degradation**: If the Hub Server is unavailable, the Hub dapp falls back to showing raw on-chain feedback and votes (derived client-side from chain data) with a "server unavailable" banner. AI-enriched data (issue titles, suggestions, pipeline status, PR links) requires the server.

---

## Core Concept

A feedback-to-pull-request pipeline that turns user feedback from any Usernode dapp into shipped code changes, with AI collation, community voting, and automated PR generation.

Users leave feedback from any dapp via an embedded widget. An AI service suggests groupings of similar feedback into issues. Users control whether their feedback is grouped with others. The community votes on which issues to tackle. Issues that pass voting thresholds are dispatched to per-repo Gastown "mayors" — AI coordinators that decompose tasks, dispatch coding agents (polecats), and create pull requests. Repo maintainers review, interact with the mayor in the PR, and merge.

The long-term vision is fully community-governed: no maintainer bottleneck. The initial version keeps maintainers in the loop for PR review while automating everything upstream.

## How It Works (User's Perspective)

1. You're using a dapp. You notice a bug, have an idea, or something feels off.
2. You tap the feedback button (floating in the corner of every dapp). A small form appears: pick a category, type your feedback.
3. Your feedback is submitted as an on-chain transaction — transparent, permanent, attributed to your address.
4. The Feedback Hub (a separate dapp) shows your feedback. The AI has suggested grouping your "the sell button doesn't update the balance" with three other people's reports about the same bug. You can accept the suggested grouping, or ungroup your feedback if the AI got it wrong.
5. You vote yes/no on issues you care about. You can filter by app.
6. Issues that get enough votes are automatically picked up by the repo's AI mayor, which decomposes the work if needed, dispatches coding agents, writes a fix, and opens a PR on GitHub.
7. Maintainers review, discuss with the mayor in the PR using `@mayor` commands, request changes, and merge.

## Architecture Overview

Four components, each independently deployable:

### 1. Feedback Widget (`usernode-feedback.js`)

A lightweight script that any dapp includes alongside the bridge. Adds a floating feedback button to the page. Submits feedback as a transaction memo to the shared Hub address.

**Integration**: One `<script>` tag per dapp. No configuration beyond including it.

```html
<script src="/usernode-feedback.js"></script>
```

The widget auto-detects which dapp it's running in via `document.title` or a `data-app` attribute on the script tag. It uses the existing bridge (`sendTransaction`) to submit feedback on-chain. The `HUB_PUBKEY` (destination address for feedback transactions) will be configured when multi-repo deployment is set up.

### 2. Feedback Hub (dapp UI)

A new dapp (single HTML file, same pattern as HIM / Last One Wins) where users browse AI-suggested issue groupings, accept or reject grouping of their own feedback, vote on issues, and track pipeline status. Served by the Hub Server on the same origin, so REST API calls use relative paths (`/api/issues`, etc.). Must include `usernode-bridge.js` and `usernode-usernames.js` for chain interaction and identity.

If the Hub Server is unavailable, the dapp should degrade gracefully — show raw on-chain feedback and votes with a "server unavailable" banner, since AI-enriched data (issue titles, suggestions, pipeline status) requires the server.

### 3. Hub Server (backend service)

A Node.js server that:
- Polls the chain for new feedback transactions (submissions, grouping actions, votes)
- Runs AI collation (suggesting groupings, deduplication, priority scoring)
- Tracks votes and computes thresholds
- Dispatches promoted issues to the Gastown adapter for multi-agent PR generation
- Receives status callbacks from the Gastown adapter to update pipeline state

### 4. Mayor Orchestrator (Gastown + adapter)

Each repo gets a **Gastown rig** — an independent multi-agent workspace with its own Mayor (AI coordinator), polecats (worker agents), Refinery (merge queue), Witness (health monitor), and Deacon (patrol loop). A thin Node.js adapter service sits between the Hub Server and Gastown, translating task specs into Gastown CLI commands and polling for status updates.

When the Hub Server dispatches a task:
- The adapter creates beads (`bd create`) from the task spec
- Groups them into a convoy (`gt convoy create`) with dependency ordering
- The Gastown Mayor decomposes large tasks further if needed, then slings beads to polecats
- Polecats work in isolated Git worktrees, making changes and committing
- The Refinery manages the merge queue, resolving conflicts between parallel workers
- The adapter polls convoy status (`gt convoy list --json`) and reports back to the Hub Server
- PRs are created by polecats or the Refinery via `gh` CLI

See "Mayor Architecture" for full details.

## On-Chain Data

All feedback, grouping actions, and votes are on-chain transactions sent to a single shared Hub address (`HUB_PUBKEY`). The `app` field in every memo is `"feedback"`.

### Transaction Types (Memo Schemas)

| Type | Memo | Notes |
|---|---|---|
| `submit` | `{ app: "feedback", type: "submit", target_app: "him", category: "bug", text: "..." }` | Raw feedback submission |
| `group` | `{ app: "feedback", type: "group", feedback_tx: "tx_hash", issue: "issue_id" }` | User assigns their own feedback to a suggested issue. Only the original submitter of `feedback_tx` can do this. |
| `ungroup` | `{ app: "feedback", type: "ungroup", feedback_tx: "tx_hash" }` | User removes their feedback from its current issue grouping. Only the original submitter can do this. |
| `vote` | `{ app: "feedback", type: "vote", issue: "issue_id", value: "yes" }` | Vote yes or no on an issue. Latest per user per issue wins. |

Notes:
- `target_app` identifies which dapp the feedback is about (e.g., `"him"`, `"lastwin"`, `"falling-sands"`, `"starter"`)
- `category` is one of: `"bug"`, `"feature"`, `"improvement"`, `"other"`
- `value` in votes is `"yes"` or `"no"`
- `issue` references a server-assigned issue ID (AI grouping suggestions create these). Users discover issue IDs via the Hub dapp UI — grouping requires visiting the Hub.
- `group` and `ungroup` are verified on read: the server checks that the sender of the `group`/`ungroup` tx matches the sender of the referenced `feedback_tx`

### What Lives Off-Chain (Server DB)

| Data | Why Off-Chain |
|---|---|
| AI-suggested issue groupings (which feedback items the AI thinks belong together) | AI output; mutable suggestions that users can accept/reject via on-chain `group`/`ungroup` txs |
| AI priority scores | Mutable AI assessment |
| Issue titles and summaries (AI-generated) | Derived content, editable by maintainers |
| Pipeline status (voting → ready → in-progress → PR-open → merged) | Workflow state tied to GitHub |
| PR URLs, merge status, agent conversation history | Lives in GitHub |
| GitHub Issue mirror (number, URL, repo) | Created in target repo when issue reaches "ready"; synced back to Hub |
| Mayor registry (per-rig Gastown configuration) | Operational config for the Gastown adapter: rig mapping, model selection, cost limits |

The server exposes issue data via a REST API that the Hub dapp consumes. On-chain transactions (feedback submissions, grouping actions, votes) are the source of truth; the server is a read-derived index with AI enrichment.

## Mayor Registry

The Gastown adapter maintains a registry mapping each `target_app` to its Gastown rig, agent configuration, and resource limits. This is the single source of truth for both repo routing and agent behavior.

```json
{
  "him": {
    "repo": "org/usernode-dapp-starter",
    "path": "dapps/him/",
    "rig": "him",
    "mayorModel": "claude-opus-4-6",
    "workerAgent": "glm5",
    "maxPolecats": 3,
    "maxBudgetUsd": 10.00
  },
  "lastwin": {
    "repo": "org/usernode-dapp-starter",
    "path": "dapps/last-one-wins/",
    "rig": "lastwin",
    "mayorModel": "claude-opus-4-6",
    "workerAgent": "glm5",
    "maxPolecats": 3,
    "maxBudgetUsd": 10.00
  },
  "falling-sands": {
    "repo": "org/falling-sands",
    "path": "/",
    "rig": "falling-sands",
    "mayorModel": "claude-sonnet-4-20250514",
    "workerAgent": "kimi",
    "maxPolecats": 2,
    "maxBudgetUsd": 5.00
  },
  "feedback-hub": {
    "repo": "org/usernode-feedback-hub",
    "path": "/",
    "rig": "feedback-hub",
    "mayorModel": "claude-sonnet-4-20250514",
    "workerAgent": "glm5",
    "maxPolecats": 2,
    "maxBudgetUsd": 5.00
  }
}
```

- `repo`: GitHub `owner/repo` string
- `path`: Scopes the agent's changes to a subdirectory (useful for monorepos)
- `rig`: Name of the Gastown rig (maps to `~/gt/<rig>/`)
- `mayorModel`: LLM used for the Mayor (task decomposition, coordination). Use a strong model.
- `workerAgent`: Gastown agent preset for polecats. Configured via `gt config agent set <name> "<command>"`. Use cheaper models for cost efficiency.
- `maxPolecats`: Maximum concurrent worker agents for this rig
- `maxBudgetUsd`: Cost ceiling per convoy (adapter tracks via Gastown's token metrics)

This mapping is stored in `config.json` and exposed via `/api/apps`.

## Feedback Widget

### UI

A small floating button in the bottom-right corner of every dapp. Tapping it opens a compact form:

```
┌──────────────────────────────┐
│  Leave Feedback              │
│                              │
│  Category: [Bug ▾]           │
│                              │
│  ┌──────────────────────────┐│
│  │ What's on your mind?     ││
│  │                          ││
│  │                          ││
│  └──────────────────────────┘│
│                              │
│           [Submit]            │
│                              │
│  ── or ──                    │
│  [View all feedback →]       │
└──────────────────────────────┘
```

- Category dropdown: Bug, Feature idea, Could be better, Other
- Free-text area (maps to `text` in memo; max ~900 chars to stay within 1024 memo limit with JSON overhead)
- "View all feedback" link opens the Feedback Hub dapp (configurable URL, set via `data-hub-url` attribute on the script tag or defaults to `/feedback`)
- Shows the standard transaction progress bar during submission
- After submission: brief "Thanks! Your feedback is on-chain." confirmation, then auto-closes
- If the Hub Server is reachable, the widget checks for ungrouped feedback (via `GET /api/feedback/ungrouped?address=...`) and shows a small dot indicator on the floating button as a nudge to visit the Hub. This check is best-effort — if the Hub Server is unavailable, the dot is simply not shown.

### Auto-Detection

The widget determines `target_app` automatically:
1. If the `<script>` tag has `data-app="him"`, use that
2. Otherwise, infer from `location.pathname` (e.g., `/him` → `"him"`, `/last-one-wins` → `"lastwin"`)
3. Fallback: `"unknown"`

The widget accepts optional configuration via `data-` attributes on the script tag:
- `data-app` — override `target_app` detection
- `data-hub-url` — Hub dapp URL for "View all feedback" link (default: `/feedback`)
- `data-hub-api` — Hub Server API base URL for ungrouped-feedback nudge (default: none — dot indicator disabled if not set)

### Technical

- Zero dependencies beyond the bridge
- Self-contained: injects its own CSS (scoped via a Shadow DOM or unique prefix to avoid conflicts)
- Works in both local-dev (mock) and production (real chain) modes via the bridge
- Does not interfere with the host dapp's layout, scroll, or event handling

## AI Collation

The server periodically (every 5 minutes, or on-demand when new feedback arrives) runs an AI collation pass over all unprocessed feedback. The AI produces **suggested groupings** — it does not assign feedback to issues. Users decide whether to accept the suggested grouping via on-chain `group` transactions.

### Suggestion Algorithm

1. **Fetch ungrouped feedback** — all `submit` transactions not yet confirmed into an issue (no `group` tx from the submitter). This includes both newly arrived feedback and previously suggested items the submitter hasn't acted on — the AI re-evaluates all ungrouped feedback on every pass, potentially suggesting different or better matches as new issues emerge.
2. **Compare against existing issues** — for each new feedback item, use an LLM to determine if it semantically matches an existing open issue
3. **Create new suggested issues** — feedback that doesn't match any existing issue triggers creation of a new suggested issue. If multiple new items cluster together, they form a single new suggested issue.
4. **Generate issue metadata** — for each new or updated issue, the LLM generates:
   - A concise title (e.g., "Sell button doesn't update balance immediately")
   - A summary that synthesizes all grouped feedback items
   - A `target_app` (inherited from the majority of grouped items)
   - A `category` (inherited or synthesized)
   - An AI priority score (1-5, based on severity, frequency, and user impact)
5. **Notify ungrouped feedback owners** — the Hub UI shows each user their ungrouped feedback alongside the AI's suggested issue match, with an "Accept grouping" action that sends a `group` tx on-chain

### User-Controlled Grouping

The core invariant: **only the submitter of a piece of feedback controls which issue it belongs to.**

- When the AI suggests a grouping, the Hub UI presents it to the feedback submitter as a suggestion. The submitter can accept (sends a `group` tx) or ignore it.
- A user can also browse existing issues and manually group their feedback into any open issue (sends a `group` tx with the chosen `issue_id`).
- A user can ungroup their feedback at any time (sends an `ungroup` tx), removing it from the current issue. The feedback returns to the "ungrouped" pool and may receive new AI suggestions.
- If all feedback items are ungrouped from an issue, the issue becomes empty and is auto-archived.

This design means:
- **No AI mis-grouping is permanent** — users always have recourse
- **Grouping is transparent** — all `group`/`ungroup` actions are on-chain and auditable
- **AI saves effort, doesn't dictate** — the AI's role is to suggest, not assign

### Priority Scoring

The AI assigns a priority score (1-5) based on:
- **Frequency**: How many independent users have grouped feedback into this issue
- **Severity**: Bug > improvement > feature (for equal frequency)
- **Recency**: Recent feedback weighted higher
- **User impact**: Does it block core functionality or is it cosmetic?

Priority scores are advisory — they affect default sort order in the Hub UI but do not affect voting thresholds.

## Voting

### Who Can Vote

Any user who has submitted at least one on-chain transaction to any dapp (not just feedback — any `sendTransaction` in the ecosystem). Sybil resistance is handled at the L1 layer as a core protocol feature and does not need to be addressed at the application level.

### Voting Mechanics

- Each user gets one vote per issue: yes or no
- Latest vote per user per issue wins (you can change your mind)
- Votes are on-chain transactions (transparent, verifiable)

### Thresholds

An issue is considered **passing** when:

1. **Quorum**: The number of unique voters on the issue is > 1/3 of eligible participants for that issue's `target_app`
2. **Approval**: > 2/3 of votes are "yes"

**Eligible participants** are determined at vote time: a user's vote counts toward quorum if they were an active participant in the target app (had at least one on-chain transaction to that app's address) within the 30 days preceding their vote submission. The quorum denominator for threshold checks is the count of active participants at the time the threshold is evaluated (i.e., the active user count at the moment the server runs its promotion check).

**Example**: At threshold-check time, HIM has 90 active users (unique senders in the last 30 days). An issue about HIM needs > 30 voters who were each active at the time they voted, with > 2/3 yes votes, to pass.

For issues tagged with multiple apps, the quorum is based on the union of active participants across all tagged apps.

The server maintains a cached set of active participants per app, refreshed hourly by querying the explorer API. This avoids expensive per-vote lookups.

### Auto-Promotion

Issues move through the pipeline automatically based on these rules:

| Condition | Action |
|---|---|
| Issue has been at passing threshold (quorum + approval) for **3 consecutive days** | Promoted to **"ready for task"** — dispatched to the repo's mayor |
| Issue has **> 2/3 of total app users** voting, with **> 2/3 yes** | **Immediately promoted** — no 3-day wait (clear community mandate) |
| Issue has had activity from **only 1 user** for **3 days** | **Archived** — not enough community interest. Can be un-archived if new feedback/votes arrive. |

"Activity" means any new feedback item grouped into the issue, or any new vote on the issue.

The 3-day wait for standard promotion prevents premature action on issues that briefly spike above the threshold. The immediate promotion path exists for issues with overwhelming support.

### Voting UI

The Hub dapp shows issues in a list:

```
┌─────────────────────────────────────────┐
│ ★ Sell button doesn't update balance    │
│ HIM · Bug · 3 reports · Score: 4        │
│                                         │
│ 23 yes / 4 no  ████████████░░  85%      │
│ Quorum: 27/30 (90%)                     │
│ Status: Voting (2 days at threshold)    │
│                                         │
│ [👍 Yes]  [👎 No]                       │
└─────────────────────────────────────────┘
```

Filterable by: app, category, status (voting / ready / in-progress / done / archived), and sort by: votes, priority, recency.

## Pipeline Stages

The pipeline tracks two entity types. The first three stages are **per feedback item** (each submission has its own status). Once feedback is grouped into an issue, the remaining stages are **per issue** (the issue moves through the pipeline as a unit).

```
Per feedback item:   submit → suggested → grouped ─┐
                                                    ├─► Per issue:   voting → ready → in-progress → pr-open → merged
                                                    │                                              → needs-help
                                                    │                                              → manual
                                                    └─►                                            → archived
```

| Stage | Entity | Description | Who/What Advances It |
|---|---|---|
| **submit** | Feedback | Raw feedback on-chain | User via widget |
| **suggested** | Feedback | AI has suggested an issue grouping for this feedback | AI collation (automatic) |
| **grouped** | Feedback | User has accepted grouping (sent `group` tx) | User via Hub dapp |
| **voting** | Issue | Has at least one confirmed feedback item; open for community votes | Automatic once first `group` tx lands |
| **ready** | Issue | Passed voting thresholds; GitHub Issue created in target repo; queued for the repo's mayor | Auto-promotion rules (see above) |
| **in-progress** | Issue | Gastown Mayor has dispatched polecats in isolated Git worktrees | Gastown adapter (automatic) |
| **pr-open** | Issue | PR created on GitHub, awaiting review | Gastown polecat / Refinery |
| **needs-help** | Issue | Agent failed; PR opened with `needs-help` label for maintainer guidance | Gastown adapter (automatic) |
| **manual** | Issue | 3 failed agent attempts; flagged for human implementation | Gastown adapter (automatic) |
| **merged** | Issue | PR merged by maintainer | Maintainer on GitHub |
| **archived** | Issue | Insufficient interest, all feedback ungrouped, or resolved by other means | Auto-archive rule or maintainer action |

Note: An issue enters **voting** as soon as it has at least one user-confirmed feedback item (a `group` tx). AI-suggested issues with zero confirmed groupings are visible in the Hub but not yet votable — they are in the **suggested** state, waiting for at least one submitter to confirm.

## GitHub Issue Mirroring

When an issue reaches **"ready"** status (passed voting thresholds), the Hub Server creates a GitHub Issue in the target repo via the GitHub App. This gives maintainers a familiar interface and gives the coding agent native context to reference.

### What Gets Created

The GitHub Issue includes:
- **Title**: The AI-generated issue title (e.g., "Sell button doesn't update balance immediately")
- **Body**:
  - Summary synthesized from grouped feedback
  - Vote results (e.g., "23 yes / 4 no — 85% approval, 90% quorum")
  - List of grouped feedback items (submitter username, text, timestamp)
  - AI priority score
  - Link to the Hub dapp issue detail page (e.g., `https://hub.usernodelabs.org/#issue/fb-042`)
- **Labels**: `feedback-hub`, app label (e.g., `him`), category label (e.g., `bug`)

### Lifecycle Sync

| GitHub Event | Hub Action |
|---|---|
| Issue created (by Hub Server) | Hub stores the GitHub Issue number and URL |
| PR references `Fixes #N` | GitHub auto-closes the issue when PR merges |
| Issue closed (via merge) | Hub detects via webhook, updates issue to "merged" |
| Issue closed manually by maintainer | Hub detects via webhook, updates issue to "archived" |
| Issue reopened | Hub detects via webhook, moves issue back to "ready" |

The Hub Server is the source of truth for issue state. GitHub Issues are a mirror — if the Hub and GitHub disagree (e.g., someone closes the GitHub Issue without merging), the webhook handler reconciles by updating the Hub.

### Agent Access

The coding agent (Gastown polecat) receives the GitHub Issue number in its bead context. Inside the Git worktree, it can read the issue natively via `gh issue view #N` to get full context. The PR it creates references `Fixes #N` so GitHub handles auto-close on merge.

### Why Mirror (Not Replace)

GitHub Issues are a convenience layer, not the source of truth:
- The Hub still tracks the full pipeline (feedback → grouping → voting → ready → PR → merged)
- On-chain data (feedback, votes, groupings) remains the authoritative record
- GitHub Issues are created late in the pipeline (at "ready"), so they don't clutter repos with unvetted feedback
- Maintainers who prefer GitHub get a native workflow; community members who prefer the Hub see the full picture

---

## Mayor Architecture (Gastown)

When an issue reaches "ready" status, the Hub Server dispatches it to the Gastown adapter. Each repo in the ecosystem gets an independent Gastown rig — a multi-agent workspace with its own Mayor (AI coordinator), polecats (worker agents), Refinery (merge queue), Witness (health monitor), and Deacon (patrol loop).

### Why Gastown

Gastown provides task decomposition, merge queue management, worker monitoring, crash recovery, and multi-agent coordination out of the box. Building these capabilities from scratch with BullMQ + OpenHands would require months of engineering for functionality that Gastown already ships. The integration surface is a thin Node.js adapter (~500 lines) that translates HTTP to CLI calls. Gastown is model-agnostic via agent presets — workers (polecats) can use cheap models (GLM-5, Kimi K2.5) while the Mayor uses a stronger model. Git worktrees provide per-agent isolation. Beads (Git-backed work units) persist state across context window resets and agent crashes.

### Why This Architecture

**Why Gastown over a custom orchestrator?** Gastown provides task decomposition (Mayor), merge queue management (Refinery), worker monitoring (Witness + Deacon), crash recovery (GUPP + Beads persistence), and multi-agent parallel execution in isolated Git worktrees — all out of the box. Building this from scratch with BullMQ + OpenHands would require months of engineering for capabilities that Gastown already ships. The integration surface is a thin Node.js adapter (~500 lines) that translates HTTP ↔ CLI calls.

**Why Gastown's Beads over BullMQ?** Beads is Git-backed (versioned, mergeable, crash-recoverable) and designed specifically for agent work tracking. It persists agent identity and work state across context window resets — the core problem with long-running AI agents. BullMQ is a great general-purpose queue but doesn't solve the agent-specific problems of context loss, work resumption, and multi-agent coordination.

**Why Git worktrees over Docker containers?** Gastown isolates agents using Git worktrees — each polecat gets its own working directory branched from the same repo. This is lighter than Docker containers (no image builds, no socket mounting, instant startup) and naturally integrates with Git-based PR workflows. The tradeoff is weaker process isolation, which is acceptable for trusted agent code running in a controlled environment.

**GitHub App over PAT.** Short-lived installation tokens (1–8 hours vs indefinite), 15,000 req/hr rate limits (vs 5,000 for PATs), fine-grained per-repo permissions, and the bot shows up as "AI Mayor[bot]" in the PR author/commenter fields — clearer provenance for the community.

**Why a thin adapter instead of using Gastown directly?** Gastown is CLI-first with no built-in REST API. The Feedback Hub needs an HTTP interface for task dispatch and status polling. The adapter is a ~500-line Node.js service that wraps `gt` and `bd` CLI calls with `--json` and `--stdin` flags. It's thin enough to maintain but necessary for the Hub integration. If Gastown ships a native API in the future, the adapter can be replaced.

**Why split Mayor model / worker model?** The Mayor (task decomposition, coordination) benefits from a strong reasoning model (Opus, Sonnet). Polecats (code execution) work well with cheaper models (GLM-5 at 1/6th cost, Kimi K2.5 at 1/10th cost). This split reduces per-convoy costs from ~$100 (all-Opus) to ~$10-20 for typical tasks while maintaining decomposition quality.

### Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Agent orchestration | **Gastown** (`gt` CLI) | Task decomposition, merge queue, worker monitoring, crash recovery, multi-agent coordination — all built-in |
| Adapter service | **Node.js** + Express | Thin translation layer: HTTP ↔ `gt` CLI. Webhook handler. Status polling. |
| Worker agents | **Gastown polecats** (model-agnostic) | Configurable per-rig: Claude Code, Codex, Goose, OpenCode + any model via OpenRouter/LiteLLM |
| Task tracking | **Beads** (Git + SQLite) | Gastown's built-in persistent work state. Survives context window resets and agent crashes. |
| Isolation | **Git worktrees** | Gastown creates per-agent worktrees within each rig. No Docker containers needed for isolation. |
| GitHub auth | **GitHub App** (`@octokit/auth-app`) | Short-lived tokens, 15K req/hr, bot identity. Tokens passed to `gh` CLI inside Gastown. |
| GitHub API | **`gh` CLI** (inside Gastown) + **Octokit** (in adapter) | Polecats use `gh pr create`. Adapter uses Octokit for webhook verification and status updates. |
| Repo context | **CLAUDE.md** / agent-specific context files per repo | Auto-loaded by Claude Code and other agents at session start |
| Status DB | **SQLite** (Hub Server) | Pipeline status, PR tracking. Gastown's Beads handles agent-side state separately. |

### System Diagram

```
Hub Server                     Gastown Adapter              Gastown (per rig)
(existing)                     (Node.js service)            (gt CLI + tmux)

issue reaches ──POST /tasks──► Adapter receives             
"ready"                        task spec                    
                                    │                       
                               bd create (beads)            
                               gt convoy create ──────────► Mayor decomposes
                               gt sling ──────────────────► Polecats execute
                                    │                            │
                               polls gt convoy list --json       │
                                    │                       Refinery merges
                                    │                       gh pr create
◄──GET /status/:taskId─────── reports status                     │
                               back to Hub                       │
                                                                 ▼
                                                            GitHub PR

GitHub ──webhooks──► Adapter ──gt mail --stdin──► Mayor/Polecat
  issue_comment                                   (revises, retries)
  pull_request_review
  pull_request (merged)
```

### Gastown Adapter

The adapter is a thin Node.js + Express service with three responsibilities:

**1. Task dispatch.** Receives POSTs from the Hub Server, translates into `bd create` + `gt convoy create` + `gt sling` CLI calls via `child_process.execSync`. Beads are created from the task spec's title, summary, feedback items, category, and target path. For large tasks (priority 4-5 or feature category), the adapter creates a single high-level bead and lets the Mayor decompose it into sub-beads. For small tasks (priority 1-2, bug category), the adapter creates a single bead and slings it directly to a polecat.

**2. Status polling.** A `setInterval` loop (every 30 seconds) runs `gt convoy list --json` and `bd list --json` for each active rig, parses results, and POSTs status updates to the Hub Server. Detects state transitions: convoy complete → report PR URL, convoy stalled → report needs-help, all beads closed → report merged.

**3. Webhook relay.** Receives GitHub webhooks (same events as before: `issue_comment`, `pull_request_review`, `check_suite`, `pull_request`). For `@mayor` commands, translates into `gt mail --stdin` to the relevant rig's Mayor or polecat session. For PR merges, updates the Hub Server status.

### Task Lifecycle: Happy Path

1. Issue reaches "ready" in the Hub.
2. Hub Server creates a GitHub Issue in the target repo (see "GitHub Issue Mirroring") and stores the issue number.
3. Hub Server POSTs a task spec to the Gastown adapter:
   ```json
   {
     "issueId": "fb-042",
     "targetApp": "him",
     "title": "Sell button doesn't update balance",
     "summary": "3 users report that after selling...",
     "feedbackItems": ["tx_abc...", "tx_def...", "tx_ghi..."],
     "category": "bug",
     "priority": 4,
     "githubIssue": { "repo": "org/usernode-dapp-starter", "number": 42 }
   }
   ```
4. Adapter creates bead(s): `bd create --title "Fix: Sell button doesn't update balance" --body "<summary + feedback items>" --label bug --label him`
5. Adapter creates convoy: `gt convoy create "fb-042" <bead-ids> --notify`
6. Adapter slings to rig: `gt sling <bead-id> him`
7. Hub updates the issue to "in-progress".
8. Gastown Mayor picks up the convoy, optionally decomposes into sub-beads if the task is complex.
9. Mayor slings sub-beads to polecats.
10. Polecats work in isolated Git worktrees within the `him` rig, scoped to `dapps/him/`. Each polecat reads `CLAUDE.md` for repo conventions, reads GitHub Issue #42 via `gh issue view` for full context, implements changes, runs linting, and commits.
11. Refinery manages the merge queue, resolving conflicts between parallel workers.
12. Polecats or Refinery create PR via `gh pr create` with title `[AI Mayor] Fix: Sell button doesn't update balance`, body containing `Fixes #42`, the issue summary, feedback items, labels `ai-generated` and `him`, and a link to the Hub issue detail page.
13. Adapter detects convoy completion via polling, reports PR URL to Hub Server. Hub updates the issue to "pr-open".

### Task Lifecycle: Review Feedback Loop

1. Maintainer reviews the PR and leaves a comment: `@mayor revise: use the existing refreshBalance() helper instead`
2. GitHub fires an `issue_comment` webhook to the Gastown adapter.
3. Webhook handler verifies the signature, confirms the comment contains `@mayor`, looks up which rig the PR belongs to, and extracts the instruction.
4. Adapter relays the instruction via `gt mail --stdin` to the relevant rig's Mayor or polecat session.
5. The Mayor or relevant polecat receives the mail, processes the instruction, and makes changes in its worktree.
6. New commits are pushed to the same branch. The PR auto-updates.
7. The agent comments on the PR: "Pushed revision addressing your feedback."

### Task Lifecycle: Failure Path

1. A polecat fails (lint errors it can't fix, scope too large, timeout, budget exceeded).
2. Gastown's Deacon detects stuck agents via patrol loops. The Witness monitors health. If a polecat crashes, GUPP ensures the next session picks up where it left off (work is persisted in beads).
3. If recovery doesn't succeed, the adapter detects a stalled convoy via status polling and creates a PR with a `needs-help` label, including in the body what was attempted, where it got stuck, and relevant error logs.
4. Adapter reports "needs-help" status to the Hub. Hub keeps the issue at "needs-help" and flags it in the UI.
5. Maintainer can comment `@mayor retry` (fresh attempt) or `@mayor scope: just fix X` (narrowed scope).
6. After 3 stalled convoys on the same issue, the adapter reports "manual" status — visible in the Hub as needing human implementation. Can be re-queued if new feedback arrives with better information.

### Mayor Commands (GitHub PR Comments)

| Command | Behavior |
|---|---|
| `@mayor revise: <instruction>` | Push new commits addressing the instruction |
| `@mayor retry` | Re-run the task from scratch on a fresh branch |
| `@mayor explain: <question>` | Respond with a PR comment (no code changes) |
| `@mayor scope: <narrower instruction>` | Re-run with narrowed scope, force-push |
| `@mayor abort` | Close the PR, move issue back to "ready" for future re-attempt |

### Gastown Setup Per Rig

Each target repo is configured as a Gastown rig:

```bash
# Initialize the town (one-time)
gt init ~/gt

# Add a rig for each target app
gt rig add him https://github.com/org/usernode-dapp-starter.git
gt rig add falling-sands https://github.com/org/falling-sands.git
gt rig add feedback-hub https://github.com/org/usernode-feedback-hub.git

# Configure agent presets (cheap workers)
gt config agent set glm5 "opencode -m openrouter/z-ai/glm-5"
gt config agent set kimi "opencode -m openrouter/moonshotai/kimi-k2.5"

# Each rig gets its Mayor, Witness, Refinery, Deacon automatically
```

Repo context: each target repo should include a `CLAUDE.md` (or equivalent for the worker agent) with repo conventions, architecture notes, and coding standards — Gastown loads this automatically.

### Cost Management

The key insight from the Gastown community: use an expensive model (Opus) for the Mayor (coordination/decomposition) and cheap models (GLM-5, Kimi K2.5) for polecats (execution). This brings per-convoy costs from ~$100 (all-Opus) to ~$10-20 for typical tasks. The adapter enforces `maxBudgetUsd` from the Mayor Registry by monitoring Gastown's token metrics dashboard and killing convoys that exceed budget.

### GitHub App Setup

**Required permissions**: Contents (Read & Write), Pull Requests (Read & Write), Issues (Read & Write), Metadata (Read), Checks (Read).

**Webhook events**: `issue_comment` (detect `@mayor` commands on PRs), `pull_request_review` (detect "changes requested"), `check_suite` (detect CI failures for auto-retry), `pull_request` (detect merges to update Hub status), `issues` (detect manual close/reopen of mirrored issues).

The adapter generates a fresh installation token at the start of each task. Tokens are short-lived (1 hour, refreshable to 8 hours). Tokens are passed to `gh` CLI inside Gastown for push access and PR creation.

---

## Hub Server API

The Hub dapp (client) communicates with the Hub server via REST:

| Endpoint | Method | Description |
|---|---|---|
| `/api/issues` | GET | List issues (filterable by app, category, status; sortable by votes, priority, recency) |
| `/api/issues/:id` | GET | Issue detail (grouped feedback items, AI-suggested items pending confirmation, votes, pipeline status, PR link) |
| `/api/issues/:id/spec` | GET | AI-generated task spec (when in ready/in-progress status) |
| `/api/issues/:id/suggestions` | GET | List of ungrouped feedback items the AI suggests belong to this issue (for submitters to accept/reject) |
| `/api/feedback/ungrouped?address=ut1...` | GET | List of the given user's feedback not yet grouped into any issue, with AI-suggested matches |
| `/api/stats` | GET | Aggregate stats: active participants per app (last 30 days), total issues, pipeline throughput |
| `/api/apps` | GET | List of known apps with participant counts, target repos, model configs, and paths |

Votes, feedback submissions, and grouping actions go on-chain via the bridge (not through the server API). The server reads them from the chain.

### Gastown Adapter API (internal)

The Hub Server communicates with the Gastown adapter via internal REST:

| Endpoint | Method | Description |
|---|---|---|
| `/api/tasks` | POST | Submit a task spec for a promoted issue |
| `/api/tasks/:id` | GET | Task status (queued / running / complete / failed) |
| `/api/tasks/:id/cancel` | POST | Cancel a running task |

The Gastown adapter also receives GitHub webhooks directly (configured in the GitHub App settings) and relays `@mayor` commands to Gastown via `gt mail --stdin`.

## Repo Structure

The project is split across two repos. The widget is shared client-side infrastructure and lives in the dapp-starter repo alongside the bridge and usernames module. The hub (server, voting UI, AI pipeline) is a standalone product with its own deployment and secrets.

### `usernode-dapp-starter` (existing repo — widget only)

```
usernode-dapp-starter/
├── usernode-bridge.js            # shared — chain interaction
├── usernode-usernames.js         # shared — global usernames
├── usernode-feedback.js          # shared — feedback widget (NEW)
├── server.js                     # serves all three shared scripts
├── examples/
│   ├── him/him.html              # includes feedback widget
│   ├── last-one-wins/index.html  # includes feedback widget
│   └── falling-sands/index.html  # includes feedback widget
└── ...
```

The widget is a single JS file that sends on-chain transactions. Core functionality (submitting feedback) has no dependency on the hub server — feedback is written to the chain, and the hub reads it from there. The optional ungrouped-feedback nudge requires the Hub Server API but degrades gracefully if unavailable.

### `usernode-feedback-hub` (new repo — hub server + UI + Gastown adapter)

```
usernode-feedback-hub/
├── hub/
│   └── index.html                # Feedback Hub dapp (grouping, voting, browsing, status)
├── server/
│   ├── server.js                 # main server: API, chain poller, scheduler
│   ├── collator.js               # AI grouping suggestions and deduplication
│   ├── prioritizer.js            # AI priority scoring
│   ├── thresholds.js             # vote counting, quorum checks, auto-promotion logic
│   └── db.js                     # SQLite persistence for issues, groupings, pipeline state
├── adapter/
│   ├── adapter.js                # Express API: receives tasks from Hub, relays webhooks
│   ├── gastown-bridge.js         # Wraps gt/bd CLI calls with --json/--stdin
│   ├── status-poller.js          # Polls convoy/bead status, reports to Hub Server
│   ├── webhook-handler.js        # GitHub webhook → gt mail relay
│   ├── github-app.js             # GitHub App auth + installation token management
│   └── commands.js               # @mayor command parser (unchanged)
├── lib/
│   └── dapp-server.js            # shared utilities (copied from dapp-starter)
├── config.json                   # Mayor registry (target_app → rig, models, limits)
├── docker-compose.yml            # Hub Server + Gastown adapter + Gastown
├── .env                          # secrets (see Environment Variables)
├── AGENTS.md
└── README.md
```

### Why the split

- **Widget** is tiny client-side infrastructure (like the bridge). Anyone cloning dapp-starter to build a dapp gets feedback support for free.
- **Hub** has operational complexity (AI services, GitHub integration, SQLite, LLM API keys, webhooks, Gastown orchestration) that doesn't belong in a template repo.
- They communicate through the chain, not through imports — the widget writes feedback transactions, the hub reads them. Clean boundary.

## Environment Variables

| Variable | Description |
|---|---|
| `HUB_PUBKEY` | Shared feedback address (all feedback/votes sent here) |
| `HUB_SECRET_KEY` | Not needed initially (no server-side sends) |
| `LLM_API_KEY` | API key for the LLM used in collation and prioritization (separate from Gastown's agent models) |
| `GITHUB_APP_ID` | GitHub App ID for the AI Mayor bot |
| `GITHUB_APP_PRIVATE_KEY` | GitHub App private key (PEM format) |
| `GITHUB_WEBHOOK_SECRET` | For verifying incoming GitHub webhook payloads |
| `GASTOWN_HOME` | Gastown town directory (default: `~/gt`) |
| `GASTOWN_MAYOR_MODEL` | Default Mayor model (can be overridden per-rig in config.json) |
| `OPENROUTER_API_KEY` | API key for OpenRouter (for cheap worker models like GLM-5, Kimi) |
| `MAYOR_API_URL` | Internal URL of the Gastown adapter (default: `http://localhost:3001`) |

Per-rig configuration lives in `config.json` (Mayor Registry), not environment variables.

## docker-compose.yml

```yaml
version: "3.8"
services:
  hub-server:
    build: ./server
    ports: ["3000:3000"]
    env_file: .env

  gastown-adapter:
    build: ./adapter
    ports: ["3001:3001"]
    env_file: .env
    volumes:
      - gastown-home:/home/gastown/gt

  gastown:
    build:
      context: .
      dockerfile: gastown/Dockerfile
    volumes:
      - gastown-home:/home/gastown/gt
      - /var/run/docker.sock:/var/run/docker.sock
    env_file: .env
    tty: true
    stdin_open: true

volumes:
  gastown-home:
```

Note: Gastown typically runs interactively in tmux. For a headless/daemon deployment, the Gastown container runs `gt` commands via the adapter's `child_process` calls. The container must have `tmux`, `gt`, `bd`, `gh`, and the configured agent CLIs (claude, opencode, etc.) installed.

## Widget Integration Checklist

The widget lives in `usernode-dapp-starter` and is served by the same server that serves the bridge and usernames module. For each dapp that wants the feedback button:

1. Include the script: `<script src="/usernode-feedback.js"></script>`
2. Optionally set the app name: `<script src="/usernode-feedback.js" data-app="him"></script>`
3. Optionally configure Hub links: `<script src="/usernode-feedback.js" data-app="him" data-hub-url="https://hub.usernodelabs.org" data-hub-api="https://hub.usernodelabs.org"></script>`
4. That's it. The widget handles everything else.

The widget needs the bridge to be loaded first (it calls `sendTransaction`). It should be included after `usernode-bridge.js`. Core functionality (submitting feedback) works without the Hub Server — it writes directly to the chain. The `data-hub-api` attribute enables the optional ungrouped-feedback dot indicator.

## Hub Dapp Screens

### Issue List (default view)

- Filter bar: app (all / HIM / Last One Wins / Falling Sands / ...), category (all / bug / feature / improvement), status (suggested / voting / ready / in-progress / done / archived)
- Sort: most votes, highest priority, newest, trending (recent vote velocity)
- Each issue card shows: title, app badge, category badge, report count (confirmed groupings), AI priority stars, vote bar (yes/no), quorum progress, pipeline status
- Issues at or near threshold are highlighted
- Issues in "suggested" status (no confirmed groupings yet) are visually distinguished (e.g., dimmed or in a separate section)

### Issue Detail

- AI-generated title and summary
- List of confirmed (grouped) feedback items (each showing: submitter, text, timestamp, app)
- List of AI-suggested (pending) feedback items, with accept/reject actions visible to each item's submitter
- Vote bar with yes/no counts and quorum progress (only active for issues with at least one confirmed grouping)
- Your vote (with change option)
- Pipeline status with timestamps (when suggested, when first grouped, when voting started, when promoted, PR link)
- If PR exists: link to GitHub PR, current status (open/merged/closed), mayor activity log
- If needs-help or manual: explanation of what the agent tried and where it got stuck

### My Feedback

- Persistent badge on the "My Feedback" tab showing the count of ungrouped items (e.g., "My Feedback (2)")
- List of feedback you've submitted, sorted with ungrouped items first
- For each item: current grouping status (ungrouped / suggested match / grouped into issue X)
- Ungrouped items with AI suggestions are highlighted with a call-to-action ("The AI thinks this matches: [issue title] — [Accept] / [Ignore]")
- Actions: accept AI suggestion, manually group into an issue, ungroup from current issue
- Status of the issues your feedback is grouped into

### Stats / Leaderboard

- Top feedback contributors (most submissions that led to merged PRs)
- Pipeline throughput: issues created / voted on / PRs generated / PRs merged over time
- Per-app breakdown
- Mayor performance: success rate, average time from ready → merged, retry rate

## Memo Size Budget

The 1024-char memo limit constrains feedback length. Budget breakdown for a `submit` transaction:

```json
{"app":"feedback","type":"submit","target_app":"him","category":"bug","text":"..."}
```

JSON overhead: ~75 chars. Leaves ~949 chars for `text`. This is sufficient for most feedback.

Grouping transactions are lightweight:
```json
{"app":"feedback","type":"group","feedback_tx":"<64-char-hash>","issue":"fb-042"}
```
~110 chars — well within budget.

## Resolved Design Questions

1. **Screenshot hosting**: Skipped in v1. Text-only feedback. Add screenshot support in Phase 4.

2. **Cross-repo PRs**: Deferred to v2. Each issue targets a single repo via the Mayor Registry. Issues that span repos need manual coordination for now.

3. **Feedback on the Hub itself**: Yes — the Hub includes its own feedback widget with `target_app: "feedback-hub"`, which maps to `org/usernode-feedback-hub` in the Mayor Registry. The Hub's mayor handles its own PRs.

4. **Identity / Usernames**: Yes, with a global username system shared across all dapps. See "Global Usernames" below.

5. **Notification**: Manual for v1 — users check the Hub to see status of their feedback. Widget-based notification badges are a Phase 4 enhancement.

6. **Rate limiting**: Yes — max 5 feedback submissions per address per day, enforced on read (same pattern as HIM's survey creation rate limit).

7. **LLM choice**: Model-agnostic. Gastown agent presets allow per-rig model selection (Mayor model + worker model configured in the Mayor Registry). The AI collation layer uses `LLM_API_KEY` directly and can use a different model than the coding agents.

8. **Sybil resistance**: Handled at the L1 protocol layer as a core feature of Usernode. Not addressed at the application level.

9. **Feedback grouping authority**: Users control their own feedback. The AI suggests groupings; only the original submitter can confirm (group) or remove (ungroup) their feedback from an issue. See "User-Controlled Grouping."

10. **Agent repo targeting**: Each app has a target repo, path scope, Gastown rig, Mayor model, worker agent preset, polecat limit, and budget defined in the Mayor Registry (`config.json`). See "Mayor Registry."

11. **Agent architecture**: Per-repo independent Gastown rigs, each with a Mayor (AI coordinator), polecats (model-agnostic workers), Refinery (merge queue), Witness (monitor), and Deacon (patrol). A thin Node.js adapter bridges the Feedback Hub and Gastown's CLI interface. Gastown was chosen over a custom BullMQ + OpenHands orchestrator because it provides task decomposition, merge management, worker monitoring, and crash recovery out of the box. The naming naturally aligns with the "mayor" metaphor that inspired the design.

12. **GitHub auth for agents**: GitHub App (not PAT) for short-lived tokens, higher rate limits, per-repo permissions, and clean bot identity ("AI Mayor[bot]").

13. **Large task handling**: Gastown's convoy model handles tasks of any scope. Small tasks (bugs, one-file fixes) are dispatched as single beads directly to polecats. Large tasks (new features, architectural changes) are dispatched as high-level beads that the Mayor decomposes into phased sub-bead sequences, executed by parallel polecats and merged by the Refinery. This tiered approach addresses the known limitation of single-agent systems on complex work (~11% success rate on feature-level benchmarks vs ~70% on bug fixes).

## Global Usernames

Usernames are shared across all dapps via a single **usernames address** (`USERNAMES_PUBKEY`). This has already been implemented in the dapp-starter repo as `usernode-usernames.js`.

- Users send `set_username` once to the usernames address: `{ app: "usernames", type: "set_username", username: "alice_a1b2c3" }`
- All dapps resolve usernames by reading transactions sent to the usernames address
- The shared module (`usernode-usernames.js`) provides `getUsernameSync(pubkey)` and `setUsername(base)` helpers
- The module caches username lookups (30s TTL) to avoid redundant chain reads
- Legacy per-app `set_username` transactions are supported as fallback via `importLegacy()`

### Integration

```html
<script src="/usernode-bridge.js"></script>
<script src="/usernode-usernames.js"></script>
```

```js
await UsernodeUsernames.init();
const name = UsernodeUsernames.getUsernameSync(pubkey);
await UsernodeUsernames.setUsername("alice"); // sends to USERNAMES_PUBKEY
```

The Feedback Hub uses this module for displaying usernames on feedback submissions, issue votes, and contributor leaderboards.

## Implementation Phases

### Phase 1: Widget + On-Chain Feedback

**Repo: `usernode-dapp-starter`**
- Build `usernode-feedback.js` widget
- Add serving route in `server.js` and `examples/server.js`
- Embed in all existing dapps (HIM, Last One Wins, Falling Sands, starter)
- Feedback submissions go on-chain

**Repo: `usernode-feedback-hub` (create)**
- Minimal hub dapp showing raw (ungrouped) feedback, filterable by app
- Simple server with chain poller — no AI yet, just reads and displays feedback

### Phase 2: AI Collation + User Grouping + Voting

**Repo: `usernode-feedback-hub`**
- AI collation suggests groupings of feedback into issues
- Hub dapp presents suggestions to feedback submitters; users confirm/reject via on-chain `group`/`ungroup` txs
- Issues with at least one confirmed grouping enter voting
- On-chain voting with quorum/approval thresholds
- Auto-promotion and auto-archive rules active
- SQLite persistence for issue groupings and pipeline state

### Phase 3a: Gastown Infrastructure

**Repo: `usernode-feedback-hub`**
- Install Gastown, Beads, tmux, and agent CLIs
- Set up GitHub App
- Initialize Gastown town with rigs for each target app
- Configure agent presets (Mayor model + cheap worker models)
- Create `CLAUDE.md` / repo context files in each target repo
- Build the Gastown adapter: Express API, CLI bridge, status poller
- Wire up Hub Server → adapter task dispatch (POST `/api/tasks`)
- Test: manual task submission → beads created → convoy runs → branch pushed

### Phase 3b: PR Pipeline + Feedback Loop

**Repo: `usernode-feedback-hub`**
- Implement PR creation flow (polecats use `gh pr create`, adapter detects and reports to Hub)
- Implement GitHub webhook handler for `@mayor` commands (adapter relays via `gt mail --stdin`)
- Implement review feedback loop (comment → mail to Mayor/polecat → new commits)
- Implement failure detection (stalled convoy → needs-help, repeated failure → manual)
- Wire up status reporting back to Hub Server (convoy completion, PR URLs, merge detection)

### Phase 3c: Agent Polish + Cost Management

**Repo: `usernode-feedback-hub`**
- Test Mayor-driven task decomposition on larger feature requests
- Tune worker agent selection per rig (benchmark GLM-5 vs Kimi vs Sonnet on your codebases)
- Implement cost tracking and budget enforcement via adapter
- Test Refinery merge quality on parallel polecat output
- Gastown Deacon patrol tuning (stuck agent detection thresholds)
- Add convoy-level metrics to Hub's stats dashboard (success rate, cost-per-PR, time-to-merge)

### Phase 4: Polish + Scale

**Both repos**
- Screenshot support in widget (`dapp-starter`)
- Notification badges in widget (`dapp-starter`)
- Multi-repo agent support (`feedback-hub`)
- Pipeline analytics and contributor leaderboard (`feedback-hub`)
- Mayor performance dashboard (`feedback-hub`)
- Evaluate community governance (`feedback-hub`)

## Future: Decentralized Gastown Rigs

Today the Usernode team operates all Gastown instances centrally. This is necessary because centralized ops means running privileged code (GitHub App credentials, LLM API keys) — we can't run Dockerfiles authored by untrusted third parties with those credentials.

The adapter API contract (`POST /api/tasks`, `GET /api/tasks/:id`, webhooks) is designed to be location-agnostic. The hub doesn't care whether it's talking to a centralized adapter or a per-dapp instance. When the time comes, dapp developers could self-host their own Gastown rigs with their own credentials, register their adapter endpoint with the hub, and receive tasks directly. The adapter code could be extracted to `dapp-starter` as shared infrastructure at that point.

Not planned — just a direction the architecture supports.

## Future: Community-Governed Merges

In the long run, the maintainer merge step can be replaced by community governance:
- PRs that pass automated tests get a community review period
- Community members vote to approve/reject the PR
- PRs with sufficient approval are auto-merged

This requires high confidence in the test suite and the voting mechanism. It's a natural extension of this system once trust is established, but not part of the initial implementation.
