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
3. Updates `fb-042` status to **"in-progress"**

### Day 6 + minutes: Claude Code picks up the issue

The GitHub Action (`.github/workflows/claude-agent.yml`) in `org/usernode-dapp-starter` triggers on the new issue with the `ai-mayor` label.

Claude Code runs on a GitHub Actions runner:
1. Reads `CLAUDE.md` for project conventions
2. Reads GitHub Issue #42 for full context (summary, feedback items, scope hint: `dapps/him/`)
3. Examines `dapps/him/him.html`, finds the sell handler doesn't call `refreshLoopOnce()` after the transaction completes
4. Adds the missing `await refreshLoopOnce();` call
5. Runs linting (eslint, prettier) — passes
6. Commits: `fix: refresh balance display after share sale`
7. Pushes branch to GitHub
8. Opens a draft PR:
   - Title: `Fix: Balance doesn't update after selling shares`
   - Body: `Fixes #42` + summary + link to Hub issue
   - Labels: `ai-generated`, `him`

Hub Server detects the PR via GitHub API polling → status updated to **"pr-open"**

### Day 6 + hours: Maintainer reviews

A maintainer sees the PR on GitHub. The code change looks right but they want a tweak:

> `@claude also call refreshLoopOnce() after the buy handler for consistency`

The GitHub Action re-triggers on the comment. Claude Code reads the review feedback, makes the additional change, pushes new commits. The PR auto-updates.

### Day 7: Merge

The maintainer approves and merges the PR. GitHub auto-closes Issue #42 (because the PR body contained `Fixes #42`).

The Hub Server detects the merge via GitHub API polling (or webhook) → `fb-042` status updated to **"merged"**.

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
| Agent dispatch | Hub Server → Octokit → GitHub Issue (with `ai-mayor` label) |
| Agent execution | GitHub Action triggers → Claude Code → reads CLAUDE.md → writes code → tests → PR |
| PR creation | Claude Code → git push → `gh pr create` → GitHub API |
| Review loop | Maintainer `@claude` comment → GitHub Action re-triggers → Claude pushes new commits |
| Merge detection | Hub Server polls GitHub API (or receives webhook) → status update |

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

Users leave feedback from any dapp via an embedded widget. An AI service suggests groupings of similar feedback into issues. Users control whether their feedback is grouped with others. The community votes on which issues to tackle. Issues that pass voting thresholds automatically become GitHub Issues in the target repo, where Claude Code picks them up, writes code, and opens pull requests. Repo maintainers review, interact with Claude in the PR, and merge.

The long-term vision is fully community-governed: no maintainer bottleneck. The initial version keeps maintainers in the loop for PR review while automating everything upstream.

## How It Works (User's Perspective)

1. You're using a dapp. You notice a bug, have an idea, or something feels off.
2. You tap the feedback button (floating in the corner of every dapp). A small form appears: pick a category, type your feedback.
3. Your feedback is submitted as an on-chain transaction — transparent, permanent, attributed to your address.
4. The Feedback Hub (a separate dapp) shows your feedback. The AI has suggested grouping your "the sell button doesn't update the balance" with three other people's reports about the same bug. You can accept the suggested grouping, or ungroup your feedback if the AI got it wrong.
5. You vote yes/no on issues you care about. You can filter by app.
6. Issues that get enough votes automatically become GitHub Issues in the target repo. Claude Code picks them up, writes a fix, and opens a PR.
7. Maintainers review, discuss with Claude in the PR using `@claude`, request changes, and merge.

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

### 3. Hub Server (backend service — 1 container)

A Node.js server that:
- Serves the Hub dapp UI on the same origin
- Polls the chain for new feedback transactions (submissions, grouping actions, votes)
- Runs AI collation (suggesting groupings, deduplication, priority scoring)
- Tracks votes and computes thresholds
- Creates GitHub Issues in target repos when community-voted issues pass voting thresholds
- Polls the GitHub API for PR creation, merge, and close events to update pipeline status

### 4. AI Agent (GitHub Action)

Each target repo contains a GitHub Action workflow (`.github/workflows/claude-agent.yml`) that triggers Anthropic's `claude-code-action` when an issue is created with the `ai-mayor` label or when `@claude` is mentioned in a PR comment. The Action runs Claude Code headlessly on GitHub's runners — reading the repo's CLAUDE.md for context, writing code, running tests, and opening a PR. The review feedback loop is handled natively: maintainer comments with `@claude` re-trigger the Action, and Claude pushes follow-up commits.

No custom agent service is needed. The Hub Server's only responsibility is creating a well-formatted GitHub Issue in the target repo when a community-voted issue reaches "ready" status, and polling for PR status updates afterward.

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
| App Registry (`config.json`) | Configuration for issue dispatch: which GitHub repo to create issues in for each dapp, plus app identity (pubkey, displayName, url) for the Hub UI |

The server exposes issue data via a REST API that the Hub dapp consumes. On-chain transactions (feedback submissions, grouping actions, votes) are the source of truth; the server is a read-derived index with AI enrichment.

## App Registry

The app registry (`config.json`) is the single source of truth for all known apps in the ecosystem and system-wide settings. The hub server reads it for UI, voting thresholds, and GitHub Issue creation.

```json
{
  "settings": {
    "voting": {
      "quorumFraction": 0.33,
      "approvalThreshold": 0.667,
      "waitDays": 3
    },
    "collation": {
      "intervalMinutes": 5
    },
    "polling": {
      "chainIntervalMs": 3000,
      "activeParticipantRefreshMinutes": 60
    }
  },
  "apps": {
    "him": {
      "displayName": "Human Input Market",
      "pubkey": "ut1...",
      "url": "https://dapps.usernodelabs.org/him",
      "repo": "org/usernode-dapp-starter",
      "path": "dapps/him/"
    },
    "lastwin": {
      "displayName": "Last One Wins",
      "pubkey": "ut1...",
      "url": "https://dapps.usernodelabs.org/last-one-wins",
      "repo": "org/usernode-dapp-starter",
      "path": "dapps/last-one-wins/"
    },
    "falling-sands": {
      "displayName": "Falling Sands",
      "pubkey": "ut1...",
      "url": "https://dapps.usernodelabs.org/falling-sands",
      "repo": "org/falling-sands",
      "path": "/"
    },
    "feedback-hub": {
      "displayName": "Feedback Hub",
      "pubkey": "ut1...",
      "url": "https://hub.usernodelabs.org",
      "repo": "org/usernode-feedback-hub",
      "path": "/"
    }
  }
}
```

### Settings

System-wide tunables that govern the Hub's behavior. These affect governance rules, so changes should be deliberate.

**Voting** (used by the hub server's threshold checker):
- `quorumFraction`: Minimum fraction of active participants that must vote for a quorum (default `0.33` = 1/3)
- `approvalThreshold`: Minimum fraction of yes votes to pass (default `0.667` = 2/3)
- `waitDays`: Consecutive days an issue must stay at passing threshold before auto-promotion (default `3`)

**Collation** (used by the hub server's AI collation scheduler):
- `intervalMinutes`: How often the collation pass runs (default `5`)

**Polling** (used by the hub server's chain poller and participant cache):
- `chainIntervalMs`: Chain polling interval in milliseconds (default `3000`)
- `activeParticipantRefreshMinutes`: How often to refresh the per-app active participant set from the explorer API (default `60`)

### App entries

Each key under `apps` is a `target_app` identifier (matches the `target_app` field in feedback submission memos).

**App identity fields** (used by the hub server):
- `displayName`: Human-readable name shown in the hub UI (filter bar, issue cards, leaderboard)
- `pubkey`: The app's on-chain public key / address. Used to query active participants for voting thresholds, and to match `target_app` in feedback submissions to real chain data.
- `url`: Where the dapp is deployed. Shown as a link in the hub UI.

**Repo fields** (used by hub server for issue dispatch):
- `repo`: GitHub `owner/repo` string. Used for GitHub Issue creation when issues reach "ready."
- `path`: Scopes the agent's work to a subdirectory (useful for monorepos). Included in the GitHub Issue body as a hint.

Agent configuration (model, budget, allowed tools) lives in each repo's `CLAUDE.md` and the GitHub Action workflow YAML — not in the Hub Server's config.

App entries are exposed via `/api/apps`.

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
| Issue has been at passing threshold (quorum + approval) for **3 consecutive days** | Promoted to **"ready"** — GitHub Issue created in target repo, agent picks it up |
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
| **ready** | Issue | Passed voting thresholds; GitHub Issue created in target repo with `ai-mayor` label | Auto-promotion rules (see above) |
| **in-progress** | Issue | Claude Code is working on the task (GitHub Action running) | Automatic when GitHub Action triggers on the dispatched issue |
| **pr-open** | Issue | PR created on GitHub, awaiting review | Claude Code (via GitHub Action) |
| **needs-help** | Issue | No PR created within timeout, or Claude commented that it's stuck | Hub Server timeout detection or Claude's own comment |
| **manual** | Issue | 3 failed agent attempts; flagged for human implementation | Hub Server (automatic) |
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

The GitHub Action (`claude-code-action`) triggers on the GitHub Issue directly — Claude Code has full access to the issue body as its prompt. The PR it creates references `Fixes #N` so GitHub handles auto-close on merge.

### Why Mirror (Not Replace)

GitHub Issues are a convenience layer, not the source of truth:
- The Hub still tracks the full pipeline (feedback → grouping → voting → ready → PR → merged)
- On-chain data (feedback, votes, groupings) remains the authoritative record
- GitHub Issues are created late in the pipeline (at "ready"), so they don't clutter repos with unvetted feedback
- Maintainers who prefer GitHub get a native workflow; community members who prefer the Hub see the full picture

---

## AI Agent Pipeline

### How It Works

The agent pipeline is deliberately simple. When a community-voted issue passes its voting threshold and is promoted to "ready," the Hub Server creates a GitHub Issue in the target repo. A GitHub Action in that repo triggers, runs Claude Code, and produces a PR. The Hub Server monitors the result.

```
Hub Server                          Target Repo (GitHub)

issue promoted ──Octokit──────────► GitHub Issue created
to "ready"       creates issue      (label: "ai-mayor")
                 in target repo              │
                                             ▼
                                    GitHub Action triggers
                                    (claude-code-action)
                                             │
                                    Claude Code runs:
                                    • reads CLAUDE.md
                                    • plans approach
                                    • writes code
                                    • runs tests/lint
                                    • commits + pushes
                                    • opens PR
                                             │
Hub Server ◄──polls GitHub API──── PR created
updates pipeline                             │
status to "pr-open"                          │
                                    Maintainer reviews
                                    comments @claude
                                             │
                                    Action re-triggers
                                    Claude pushes new commits
                                             │
Hub Server ◄──polls GitHub API──── PR merged
updates to "merged"
```

### GitHub Action Workflow

Each target repo includes this workflow file at `.github/workflows/claude-agent.yml`:

```yaml
name: AI Mayor
on:
  issues:
    types: [opened, labeled]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  claude:
    if: |
      contains(github.event.issue.labels.*.name, 'ai-mayor') ||
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude'))
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          label_trigger: "ai-mayor"
```

This single file handles: issue→PR creation (triggered by the `ai-mayor` label), PR review feedback (triggered by `@claude` in comments), and follow-up commits.

### Hub Server Integration

The Hub Server adds two small pieces of functionality:

**1. Issue dispatch** (~50 lines): When an issue is promoted to "ready," the Hub Server uses Octokit to create a GitHub Issue in the target repo:

```javascript
async function dispatchToAgent(issue) {
  const { repo } = appRegistry[issue.targetApp];
  const [owner, repoName] = repo.split('/');

  await octokit.issues.create({
    owner, repo: repoName,
    title: `[Feedback Hub] ${issue.title}`,
    body: formatIssueBody(issue),
    labels: ['ai-mayor', issue.category],
  });
}
```

The issue body is the task spec — it includes the synthesized summary, all grouped feedback items, the target directory path (from the App Registry), and any scope hints. Claude Code reads this as its prompt.

**2. Status polling** (~30 lines): A periodic job (every 5 minutes) checks the GitHub API for PRs linked to dispatched issues:

```javascript
async function pollAgentStatus() {
  for (const task of activeTasks) {
    const { data: prs } = await octokit.pulls.list({
      owner, repo, state: 'all',
      head: `claude/issue-${task.githubIssueNumber}`,
    });
    if (prs.length > 0) updatePipelineStatus(task, prs[0]);
  }
}
```

Alternatively, the Hub Server can receive GitHub webhooks (`pull_request` events) for real-time status updates. Either approach works; polling is simpler to start with.

### CLAUDE.md for Repo Context

Each target repo should include a `CLAUDE.md` file at its root. This is the single highest-leverage file for agent output quality. It should contain:

- Repo architecture overview (what the dapp does, how it's structured)
- Coding conventions (style, naming, patterns used)
- Test and lint commands (`npm test`, `npx eslint .`, etc.)
- Scoping rules for monorepos (e.g., "When working on HIM, only modify files in `dapps/him/`")
- Branch naming convention: `claude/issue-{N}-{slug}`
- PR conventions: draft PRs, link back to the originating Feedback Hub issue
- Any known gotchas or architectural constraints

### Task Lifecycle: Happy Path

1. Issue reaches "ready" in the Hub (passed voting thresholds).
2. Hub Server creates a GitHub Issue in the target repo with the `ai-mayor` label, containing the task spec as the issue body.
3. The GitHub Action triggers on issue creation.
4. `claude-code-action` runs Claude Code headlessly on a GitHub runner.
5. Claude reads CLAUDE.md, understands the repo, plans the approach.
6. For large tasks, Claude spawns sub-agents (Explore for codebase analysis, Plan for design) each with their own 200K context window.
7. Claude writes code, scoped to the target directory from the issue body.
8. Claude runs tests and lint checks.
9. Claude creates a branch, commits, pushes, and opens a draft PR.
10. PR body includes: issue summary, what changed and why, link back to Feedback Hub.
11. Hub Server detects the PR via polling/webhook, updates issue to "pr-open".
12. Maintainer reviews the PR.

### Task Lifecycle: Review Feedback

The review loop is native to claude-code-action:

1. Maintainer comments on the PR: `@claude use the existing refreshBalance() helper instead of writing a new one`
2. The GitHub Action re-triggers on the `issue_comment` event.
3. Claude Code resumes with the PR context + the reviewer's comment.
4. Claude pushes new commits to the same branch.
5. PR auto-updates.

No custom webhook relay, no queue, no re-dispatch logic.

### Task Lifecycle: Failure

If Claude can't complete the task:

1. Claude may open a PR with partial work and a comment explaining what it couldn't resolve.
2. Or Claude may comment on the GitHub Issue explaining the blocker.
3. Hub Server detects no PR after a timeout period (configurable, e.g., 30 minutes) and marks the issue as "needs-help".
4. Maintainer can comment `@claude` with additional guidance, or manually implement.
5. After 3 Action runs with no successful PR, Hub Server marks the issue as "manual".

### Agent Commands (GitHub PR/Issue Comments)

| Original Command | Trigger | Behavior |
|---|---|---|
| `@mayor revise: <instruction>` | `@claude <instruction>` | Claude pushes new commits addressing the instruction |
| `@mayor retry` | Close and re-open the GitHub Issue | Action re-triggers from scratch |
| `@mayor explain: <question>` | `@claude explain: <question>` | Claude responds with a comment |
| `@mayor scope: <narrower instruction>` | `@claude scope this to: <instruction>` | Claude narrows scope and pushes |
| `@mayor abort` | Close the GitHub Issue + PR | Hub Server moves issue back to "ready" |

The exact trigger phrases are flexible. Claude Code responds to any `@claude` mention; the instruction following it is natural language.

### Why This Approach

- **No custom agent infrastructure.** The entire agent layer is a YAML file per repo. No services to deploy, monitor, or maintain.
- **Claude Code's sub-agents handle large tasks.** Independent 200K context windows per sub-agent provide built-in task decomposition without external orchestration.
- **The switching cost is ~10 minutes.** If a better agent or model emerges, swap the YAML to `openhands-agent`, `openai/codex-action`, or `google-github-actions/run-gemini-cli`. The Hub Server's issue-creation code doesn't change.
- **GitHub Actions handles sandboxing, auth, and runner management.** No Docker socket mounting, no container lifecycle management, no token refresh logic.
- **The tradeoff is Claude lock-in** — but only at the Action layer, which is trivially swappable. The Hub Server, on-chain data, voting system, and dapp UI are completely agent-agnostic.

### Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Agent execution | **claude-code-action** (GitHub Action) | Official Anthropic Action. Handles issue→PR→review loop. Sub-agents for large tasks. Zero infrastructure. |
| Agent context | **CLAUDE.md** per repo | Auto-loaded by Claude Code. Repo conventions, architecture, scoping rules. |
| Issue dispatch | **Octokit** (`@octokit/rest`) in Hub Server | Creates GitHub Issues in target repos when community-voted issues reach "ready" |
| Status tracking | **GitHub API polling** or **webhooks** | Hub Server monitors PR creation, merge, and close events |
| GitHub auth | **GitHub App** (`@octokit/auth-app`) | For Hub Server's Octokit calls (issue creation, status polling). The Action uses its own `ANTHROPIC_API_KEY` secret. |
| Status DB | **SQLite** (Hub Server) | Maps Feedback Hub issues → GitHub Issue numbers → PR URLs → pipeline status |

### GitHub App Setup

**Required permissions**: Issues (Read & Write), Pull Requests (Read), Metadata (Read).

**Webhook events** (optional — only if using webhooks instead of polling for status): `pull_request` (detect merges/closes), `issues` (detect manual close/reopen of mirrored issues).

The `ANTHROPIC_API_KEY` for the coding agent is a separate GitHub Actions secret in each target repo. The Hub Server never touches the Anthropic API for code generation — clean credential separation.

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

### `usernode-feedback-hub` (new repo — hub server + UI)

```
usernode-feedback-hub/
├── hub/
│   └── index.html                # Feedback Hub dapp (grouping, voting, browsing, status)
├── server/
│   ├── Dockerfile                # Hub server container image
│   ├── server.js                 # main server: API, chain poller, scheduler
│   ├── collator.js               # AI grouping suggestions and deduplication
│   ├── prioritizer.js            # AI priority scoring
│   ├── thresholds.js             # vote counting, quorum checks, auto-promotion logic
│   ├── dispatch.js               # creates GitHub Issues in target repos via Octokit
│   ├── status.js                 # polls GitHub API for PR status updates
│   ├── github-app.js             # GitHub App auth + installation token management
│   └── db.js                     # SQLite persistence for issues, groupings, pipeline state
├── lib/
│   └── dapp-server.js            # shared utilities (copied from dapp-starter)
├── config.json                   # App Registry + settings (see "App Registry")
├── .env                          # secrets (see Environment Variables)
├── .github/
│   └── workflows/
│       └── deploy.yml            # GitHub Actions deploy workflow
├── AGENTS.md
└── README.md
```

No `docker-compose.yml` is needed for the agent layer. If the Hub Server itself needs Docker for deployment, that's separate and simpler (just the server + SQLite).

### Why the split

- **Widget** is tiny client-side infrastructure (like the bridge). Anyone cloning dapp-starter to build a dapp gets feedback support for free.
- **Hub** has operational complexity (AI services, GitHub integration, SQLite, LLM API keys) that doesn't belong in a template repo.
- They communicate through the chain, not through imports — the widget writes feedback transactions, the hub reads them. Clean boundary.

## Environment Variables

| Variable | Description |
|---|---|
| `HUB_PUBKEY` | Shared feedback address (all feedback/votes sent here) |
| `HUB_SECRET_KEY` | Not needed initially (no server-side sends) |
| `LLM_API_KEY` | API key for the LLM used in AI collation and prioritization |
| `GITHUB_APP_ID` | GitHub App ID (for Hub Server's Octokit calls) |
| `GITHUB_APP_PRIVATE_KEY` | GitHub App private key (PEM format) |
| `GITHUB_WEBHOOK_SECRET` | For verifying incoming GitHub webhook payloads (optional, if using webhooks for status) |

Note: `ANTHROPIC_API_KEY` for the coding agent lives as a GitHub Actions secret in each target repo, not in the Hub Server's environment. This separation is clean — the Hub Server never touches the Anthropic API for code generation.

## Deployment

The Hub Server is a single Node.js container. No agent containers, no Redis, no LiteLLM proxy.

```yaml
version: "3.8"
services:
  hub-server:
    build: ./server
    ports: ["3000:3000"]
    env_file: .env
    volumes:
      - hub-data:/app/data

volumes:
  hub-data:
```

Adding a new app to the ecosystem: edit `config.json` (add the app entry), add `.github/workflows/claude-agent.yml` + `CLAUDE.md` to the target repo, set `ANTHROPIC_API_KEY` as a GitHub Actions secret in that repo, redeploy the Hub Server.

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
- If PR exists: link to GitHub PR, current status (open/merged/closed), agent activity log
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
- Agent performance: success rate, average time from ready → merged, retry rate

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

2. **Cross-repo PRs**: Deferred to v2. Each issue targets a single repo via the App Registry. Issues that span repos need manual coordination for now.

3. **Feedback on the Hub itself**: Yes — the Hub includes its own feedback widget with `target_app: "feedback-hub"`, which maps to `org/usernode-feedback-hub` in the App Registry. The same GitHub Action flow handles its own PRs.

4. **Identity / Usernames**: Yes, with a global username system shared across all dapps. See "Global Usernames" below.

5. **Notification**: Manual for v1 — users check the Hub to see status of their feedback. Widget-based notification badges are a Phase 4 enhancement.

6. **Rate limiting**: Yes — max 5 feedback submissions per address per day, enforced on read (same pattern as HIM's survey creation rate limit).

7. **LLM choice**: The AI collation layer (Hub Server) uses `LLM_API_KEY` and is model-agnostic. The coding agent uses `claude-code-action` (Claude lock-in at the Action layer, but trivially swappable — see #14).

8. **Sybil resistance**: Handled at the L1 protocol layer as a core feature of Usernode. Not addressed at the application level.

9. **Feedback grouping authority**: Users control their own feedback. The AI suggests groupings; only the original submitter can confirm (group) or remove (ungroup) their feedback from an issue. See "User-Controlled Grouping."

10. **Agent repo targeting**: Each app entry in the App Registry (`config.json`) includes identity fields (pubkey, displayName, url) and repo targeting (owner/repo, path scope). The hub server uses identity fields for UI and voting thresholds; repo fields for GitHub Issue creation. Agent configuration lives in each repo's CLAUDE.md and GitHub Action YAML.

11. **Agent architecture**: The agent layer is Anthropic's official `claude-code-action` GitHub Action, triggered by the Hub Server creating labeled GitHub Issues in target repos. No custom agent service, no task queues, no sandbox management. Claude Code handles task decomposition via sub-agents, code writing, testing, branch management, and PR creation natively. The tradeoff is Claude model lock-in, but the switching cost to OpenHands, Codex, or Gemini Actions is ~10 minutes of YAML editing since the Hub Server's issue-creation interface is agent-agnostic.

12. **GitHub auth**: GitHub App for the Hub Server's Octokit calls (creating issues, polling status). The `ANTHROPIC_API_KEY` for the coding agent is a separate GitHub Actions secret in each target repo. This clean separation means the Hub Server never handles Anthropic credentials.

13. **Large task handling**: Claude Code's sub-agent system (Explore, Plan, general-purpose) gives each sub-agent an independent 200K context window, providing built-in task decomposition without external orchestration. For tasks Claude can't complete, the Hub Server detects a timeout (no PR within 30 minutes) and flags the issue as "needs-help" for maintainer guidance. After 3 failed attempts, the issue moves to "manual" status.

14. **Agent swappability**: The agent layer is deliberately decoupled from the Hub. The Hub Server creates a GitHub Issue with a structured body (the task spec) and a label. Any GitHub Action that triggers on that label and produces a PR is a valid agent. Switching from claude-code-action to openhands, codex-action, or run-gemini-cli requires only changing the YAML workflow file in each target repo — no Hub Server code changes.

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

### Phase 3: AI Agent Pipeline

**Repo: `usernode-feedback-hub`**
- Create a GitHub App for Hub Server (issue creation, status polling permissions)
- Add `dispatch.js` to Hub Server: creates GitHub Issues via Octokit when issues reach "ready"
- Add `status.js` to Hub Server: polls GitHub API for PR creation/merge events

**In each target repo:**
- Write `CLAUDE.md` (conventions, scoping rules, test commands)
- Add `.github/workflows/claude-agent.yml`
- Set `ANTHROPIC_API_KEY` as a GitHub Actions secret

**Testing:**
- Test end-to-end: manually promote an issue → GitHub Issue created → Action runs → PR opened
- Test review loop: comment `@claude` on the PR → Action re-triggers → new commits
- Test failure path: timeout detection, "needs-help" status
- Wire up Hub dapp to show GitHub Issue + PR links in the pipeline status UI

### Phase 4: Polish + Scale

**Both repos**
- Screenshot support in widget (`dapp-starter`)
- Notification badges in widget (`dapp-starter`)
- Multi-repo agent support (`feedback-hub`)
- Pipeline analytics and contributor leaderboard (`feedback-hub`)
- Agent performance dashboard (`feedback-hub`)
- Evaluate community governance (`feedback-hub`)

## Future: Decentralized Agent Execution

Today the Usernode team manages the GitHub Action workflow files in each target repo and the `ANTHROPIC_API_KEY` secrets. Because the agent layer is just a YAML file + API key per repo, decentralization is straightforward: dapp developers add the workflow to their own repo, use their own API key, and the Hub Server's issue-creation interface works identically — it just creates a GitHub Issue, and whatever Action is configured in that repo handles the rest.

Not planned — just a direction the architecture naturally supports.

## Future: Community-Governed Merges

In the long run, the maintainer merge step can be replaced by community governance:
- PRs that pass automated tests get a community review period
- Community members vote to approve/reject the PR
- PRs with sufficient approval are auto-merged

This requires high confidence in the test suite and the voting mechanism. It's a natural extension of this system once trust is established, but not part of the initial implementation.
