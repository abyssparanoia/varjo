# varjo architecture: coverage map and minimal components

Status: Draft, revised for T3 Code on 2026-11-23 (#30); pending Phase 0 round 2.

Spec: [docs/requirements.md](../requirements.md) (unchanged; corrections are listed below). Research inputs: `docs/research/claude-code.md`, `remote-access.md`, `max-plan.md`, `auto-memory.md`.

T3 Code inputs (repo `pingdotgg/t3code`, branch `main`): `docs/user/providers-claude.md`, `docs/user/faq.md`, `docs/user/remote-access.md`, `docs/user/background-service.md`, `docs/user/mobile-notifications.md`, `docs/user/permission-modes.md`, `docs/user/outside-agents.md`, `docs/user/source-control.md`, `docs/user/managed-policy.md`, `docs/internals/overview.md`, `docs/internals/t3-connect.md`, `apps/server/src/scheduledTasks/*`, `packages/contracts/src/scheduledTask.ts`. Scheduled tasks and webhooks have no user-facing doc page yet; their semantics here come from source and release notes v0.1.7-v0.1.10.

## 1. Summary

- T3 Code (t3.codes, MIT, v0.1.10 released 2026-11-17, pre-1.0) is the runtime, UI and remote layer on botPC. varjo is reduced to the GitHub event adapter plus glue (#30).
- T3 Code owns: Claude execution (Agent SDK `@anthropic-ai/claude-agent-sdk`, reusing the `claude auth login` subscription credential), thread persistence and native resume, per-thread worktrees, permission modes, remote access (T3 Connect, LAN pairing, Tailscale, SSH), clients (desktop, web, iOS, Android), push notifications, the always-on service (`t3 up`), scheduled tasks and webhook ingestion, and an MCP server through which outside agents start and read threads.
- varjo builds only: the GitHub poller, the event store with dedup, the thread launcher (an MCP client of T3 Code), the review-summary schema and prompt, the usage-limit guard, a memory sync script, a PR-handoff step (pending #34), cleanup policy, the power manager, and the launchd jobs `com.varjo.poll` and `com.varjo.memsync`.
- varjo drops: the `claude --bg` / `--remote-control` arm-ahead launcher, `claude logs` URL scraping, `claude respawn`, the `com.varjo.recover` job, notifier channel adapters as a required component (T3 push replaces them; a Discord/ntfy adapter stays optional), and `setup-token` / `CLAUDE_CODE_OAUTH_TOKEN` handling (needed only for the CLI fallback).
- Users reach botPC threads from the T3 Code desktop, web or mobile app. The `SessionLauncher` interface stays with `T3McpLauncher` as its only implementation; the former CLI launcher (M2 + arm-ahead, #11) is recorded as the fallback design and is built only if #32 or #34 fails.
- Implementation: a single TypeScript/Node CLI `varjo` (`poll`, `status`, `open <pr>`, `cleanup`) plus SQLite. Shell is used only for launchd plists and scripts.
- Five decisions remain open for the user (section 9): handoff storage, notification channel, worktree ownership, T3 Connect vs Tailscale-only, trigger route.

## 2. Corrections to the requirements document

The first five columns are the record of the `claude` CLI round and are kept as written. The last column states what #30 (T3 Code) did to each row.

| Requirements section | Assumption | What the research found | Consequence | Status after #30 |
|---|---|---|---|---|
| §4.3, §9 Phase 0 | CLI-created sessions on botPC can show up in Claude Desktop on mainPC | Desktop over SSH lists only sessions Desktop starts; no session-ID deep link; `claude --desktop --resume` is local-only (remote-access.md §A). RC is the documented path to claude.ai/code, phone and Desktop (remote-access.md §B, claude-code.md §17.1) | RC is primary (D1). Desktop SSH is an interactive fallback. Experiment #11 passed and selected M2 + arm-ahead (headless `-p` summary, then `--bg --resume --remote-control`; recorded as the CLI fallback in section 4.3) | Superseded by T3 Code (#30): the T3 Code apps are the clients; RC and Desktop SSH are not used |
| §2.3, §9 Phase 0 | Dispatch may run tasks in botPC repos | Dispatch cannot target a machine or repo; it needs Desktop running on the paired computer; limited beta, closed to new users (remote-access.md §C) | Dispatch is out of scope; #13 is confirmation only | Unchanged (#13 closed) |
| §8 | Agent SDK / Dispatch are options for AI execution and phone control | Anthropic docs steer SDK users to API keys and bar claude.ai login for third-party products; non-bare `claude -p` uses the subscription (claude-code.md §7, max-plan.md §2) | Drive the CLI (`claude -p`, `--bg`, `--worktree`) from launchd-managed code; no `@anthropic-ai/claude-agent-sdk` | Superseded by T3 Code (#30); terms re-check in the Agent SDK row below |
| §5.4 | A Skill/Hook can fire "at PR creation" | No commit/push/PR hook event exists (claude-code.md §3, §17.4) | `PostToolUse` hook with `if: "Bash(gh pr create *)"` (D8); unconfirmed in `-p` runs | Re-check (#34): whether the Agent SDK session under T3 Code loads and fires the hook; fallback in section 4.9 |
| §1, §8 (Max), max-plan.md §1 | Plan tiers are "Max 5x/20x" | The 20x tier is now called Ultra; docs say "Pro, Max, Ultra" (max-plan.md §1) | Use "Max/Ultra" in docs and config; limits are per account, shared by botPC and mainPC (max-plan.md §3) | Unchanged |
| §6 | Execution is local and only inference leaves the machine | While RC is connected the transcript is stored on Anthropic servers; RC is unavailable under ZDR/HIPAA (remote-access.md §B) | Security model (section 8) states this explicitly; RC use is a conscious trade-off against §6 confidentiality | Superseded by T3 Code (#30): the trade-off is now the T3 Connect relay (O4, section 8); RC is not used |
| §2.1, §10 | botPC recovers after reboot | `--bg` sessions stop on reboot; no launchd integration is provided (claude-code.md §10) | varjo ships a launchd recovery job (flow e) | Superseded by T3 Code (#30): the `t3 up` service restarts itself; varjo reconciles at poll start; no recovery job |
| §9 Phase 0 (Max auth), §6 | One login can serve CLI, RC and unattended jobs | `setup-token` tokens cannot establish RC; `CLAUDE_CODE_OAUTH_TOKEN` overrides `/login` in new sessions; `ANTHROPIC_API_KEY` always wins in `-p`; `--bare` ignores OAuth (claude-code.md §12, max-plan.md §1) | Two credentials on botPC (D2); the token env var is injected only into headless child processes, never into RC-bearing launches | Superseded by T3 Code (#30): one `claude auth login` credential used by T3 Code; token handling kept only for the CLI fallback; re-check (#30 decision 3) |
| §5.3 | Auto Memory sharing is an open question | Memory is per repo, shared across worktrees, machine-local, keyed by absolute path (auto-memory.md) | Identical repo paths on both Macs, fallback `autoMemoryDirectory` (D3) | Unchanged; memory key cross-check added in #34 |
| §3.2 | Events can be detected with existing tooling | No local review-event source; Routines GitHub trigger runs in the cloud only (claude-code.md §15, §16) | varjo must build the poller | Unchanged; T3 webhooks add an optional route (O5) |
| §10 | "Stop gracefully at the usage limit" is straightforward | No documented exit code for limit hits in `-p`; observed behavior differs per version and is unstable (max-plan.md §4) | Detection by OR over several signals, key on `is_error`/`api_error`, not `subtype` (D7) | Re-check (#8): detection moves to thread state and `t3_thread_read` output; signals pending |
| §3.3 step 4 | A worktree is created or updated per PR | `--worktree "#<n>"` keeps the old tip for PR-reference names; `-p` runs never clean up their worktrees (claude-code.md §5) | varjo must refresh the tip itself and sweep stale worktrees | Re-check (#15): T3 workspace modes replace `--worktree`; varjo still fetches and refreshes the tip (O3) |
| §8 (Agent SDK) | The Agent SDK is not usable with a subscription login (CLI-round finding, ToS) | T3 Code runs Claude through the Agent SDK, reusing the `claude auth login` credential; the T3 FAQ says the Max subscription works and, since v0.1.7, draws down the same 5-hour limit (faq.md, providers-claude.md) | varjo does not call the SDK itself; T3 Code does, and varjo drives T3 Code over MCP | Re-check (#30 decision 3): terms for subscription use through the SDK are re-checked before Phase 1 |
| §4.3, §2.3 (remote access) | RC and Claude Desktop over SSH are the remote paths | T3 Code offers T3 Connect (Clerk-authenticated relay plus outbound tunnel, no inbound ports), LAN pairing, Tailscale (including Tailscale HTTPS) and SSH (remote-access.md, t3-connect.md) | RC and Desktop SSH are replaced by T3 Connect or Tailscale; the choice is O4 | Superseded by T3 Code (#30); decided by #31 |

## 3. Coverage map

| Capability | T3 Code provides? | varjo builds? | Notes |
|---|---|---|---|
| Run Claude in a repo | Yes: Agent SDK with the `claude auth login` credential | no | providers-claude.md, faq.md |
| Remote GUI (mainPC) | Yes: desktop app (macOS/Win/Linux), app.t3.codes | no | Reached over T3 Connect, LAN pairing, Tailscale or SSH (remote-access.md); O4 |
| Smartphone control | Yes: iOS and Android apps | no | Send messages, answer approvals in the thread UI; #12 confirms |
| PR review event detection | No | **yes** (poller) | `gh` CLI/API; T3 webhooks are an optional route (O5) |
| Dedup + state | Partly: dedupe for T3's own webhook ingestion only | **yes** (SQLite) | GitHub review/comment IDs as keys (requirements §3.2) |
| Worktree per PR | Yes: workspace modes root / new worktree / existing worktree | thin wrapper | Ownership is open decision O3; varjo fetches `pull/<n>/head` either way |
| PR ↔ session mapping | No | **yes** | `sessions.t3_thread_id`, `t3_thread_url` |
| Session create/resume | Yes: event-sourced thread persistence with native resume | no | varjo stores the thread id returned by `t3_thread_launch`; follow-ups are messages into the thread |
| Headless review summary | Partly: threads run headless; no `--json-schema` equivalent is exposed | prompt + schema + validation | Prompt asks for a fenced JSON block; varjo extracts it from the final assistant message and validates it |
| Background per-PR worker | Yes: the `t3 up` launchd service | no | background-service.md |
| Notification to mainPC/phone | Yes: T3 push | optional adapter | Adapter only for Tailscale-only setups and for limit/reboot alerts without a thread (O2) |
| Push notifications | Yes: finish / fail / approval / question, requires T3 Connect | no | mobile-notifications.md; O4 |
| Usage-limit handling | No | **yes** (guard) | Detection via thread state and `t3_thread_read` output; signals pending #8 |
| Unattended auth | Yes: picks up the `claude auth login` credential | config only | varjo's own MCP OAuth credential, 0600 on botPC |
| MCP server | Yes: per environment at `https://<env>/mcp`, OAuth, access levels Read only / Supervised / Auto-accept edits / Auto / Full access | MCP client only | outside-agents.md; tools `t3_thread_launch`, `t3_thread_wait`, `t3_thread_read`, `t3_thread_list`, `t3_thread_search`, plus message and stop |
| Permission modes | Yes: Supervised / Auto-accept edits / Auto / Full access; managed policy can cap | mapping only | permission-modes.md, managed-policy.md; exact read-only mapping recorded by #32 |
| Scheduled tasks / webhooks | Yes: `fixed_time` / `once` / `interval` (min 60 s) / `webhook` with HMAC-SHA256, JSON-path filters, `{{path}}` templates, dedupe; the T3 Connect relay can hold hooks up to 24 h | no (optional adapter) | No user-facing doc yet (source and release notes); trigger route is O5, decided by #33 |
| Source-control UI | Yes (source-control.md) | no | Not evaluated yet; no varjo dependency in Phase 1 |
| PR handoff doc | Unconfirmed: depends on whether hooks and Skills load in the SDK session (#34) | small | §5.4 template; storage is O1; fallback in 4.9 |
| Memory sync | No | config only | One-way, `projects/<key>/memory/` only, no `--delete` (auto-memory.md); key cross-check in #34 |
| Scheduling | Partly: scheduled tasks exist | config only | launchd runs the poll interval and memsync; T3 interval tasks are expected unsuitable (O5) |
| Reboot recovery | Yes: the `t3 up` service restarts itself | thin | `varjo poll` reconciles `sessions` against `t3_thread_list` at its start |
| Cleanup of sessions/worktrees | Partly: T3 worktree cleanup setting | policy only | Which PRs are closed is varjo's job; interaction with T3's setting by #15 |
| Logs and traceability | Yes: event-sourced thread history | **yes** (runs table) | `runs` keeps the MCP request, status and result path |
| Approval gate for pushes (Phase 2) | Yes: approvals in T3's UI | no (first iteration) | Human approves in the thread; varjo adds no relay |
| Jira/CI sources | No | later | Same `EventSource` interface |

## 4. Components varjo builds

All components live in one Node process per invocation (`varjo poll` from launchd) except scripts. Concurrency control is a per-session lease plus one global concurrent-run slot (max-plan.md §6: single concurrent run).

### 4.1 Poller (GitHub adapter)

- Interfaces: `EventSource { poll(since): Event[] }`, `EventHandler { handle(event): void }`. GitHub is the first `EventSource`; Jira/CI later (requirements §7).
- Responsibility: for each enabled repo, list the user's open PRs, fetch reviews, review comments and replies newer than the stored cursor, and normalize them to `Event`. Also report PR state changes (merged/closed) for cleanup.
- Inputs: `repos` table, GitHub auth already present for `gh` on botPC (fine-grained token or `gh auth`, scoped to PR read + contents read for Phase 1). Outputs: rows in `events`, updated `prs`.
- Flags/commands: `gh` CLI or `gh api`; no T3 Code calls.
- Failure handling: network/HTTP errors abort the cycle without advancing the cursor; the next launchd interval retries. GitHub rate limits back off the same way. Never launches a thread.

### 4.2 Event store and dedup (SQLite)

- Responsibility: idempotency (requirements §10). `UNIQUE(source, kind, external_id)` on `events` makes re-polling safe; event processing status is advanced in the same transaction that claims a lease on the session.
- Inputs: normalized events. Outputs: the work queue (events with status `new`) grouped by PR. Multiple new events on one PR in one cycle are batched into one run.
- Failure handling: crash mid-run leaves the lease to expire (`lease_until`); events stay `queued` and are retried subject to the limit guard. Schema: section 5.

### 4.3 Thread launcher (`SessionLauncher`, `T3McpLauncher`)

```ts
interface SessionLauncher {
  launch(job: ReviewJob): Promise<LaunchResult>;   // first review of a PR
  resume(session: Session, job: FollowUpJob): Promise<RunResult>; // later reviews
}
```

- Shared: varjo stores the thread id returned by the launch before it waits; permission mode and model come from per-repo config; PR comment text is passed as delimited data (section 8).
- **`T3McpLauncher` (the only implementation)**: an MCP client of the T3 Code environment at `https://<env>/mcp`, authenticated by OAuth.
  1. First review: `t3_thread_launch` with the project (the repo root registered in T3 Code), workspace mode per O3, the read-only summary permission mode, the model per repo config (default Sonnet, low effort if T3 Code exposes it), and the prompt (fixed template plus review text in a delimited data block).
  2. `t3_thread_wait` (bounded), then `t3_thread_read` to extract the structured summary: a fenced JSON block in the final assistant message, validated by varjo against the schema.
  3. Follow-up on the same PR: send the new review as a message into the same thread (MCP message tool), then wait and read as above. Stop uses the MCP stop tool; reconciliation uses `t3_thread_list`.
  4. varjo stores `t3_thread_id`, `t3_thread_url` and `t3_environment`. The summary is visible on the phone as the thread's last assistant turn, and T3 push notifies "finished".
- Credentials: Claude credentials stay with T3 Code (it picks up `claude auth login`); varjo never handles them. varjo's own MCP OAuth credential is stored 0600 on botPC, never synced, approved at the lowest access level that allows `t3_thread_launch`; a thread varjo starts cannot exceed that level. `ANTHROPIC_API_KEY` is never set and usage credits stay OFF; the `t3 up` service environment is checked in #8.
- Safety (Phase 1, D6): read-only summary mode (target: Supervised with a read-only allowlist `Read`, `Grep`, `Glob`, `Bash(gh pr view *)`, `Bash(gh pr diff *)`, `Bash(git log *)`, `Bash(git diff *)`; the exact T3 mode mapping is recorded by #32); wall-clock timeout on the bounded wait; default Sonnet at low effort, overridable per repo. Phase 2 fix runs: auto-accept edits inside the worktree; pushes stay human-approved in T3's UI. T3 managed policy can cap the maximum mode.
- Failure handling: each launch, message and read is recorded in `runs`. Results are classified as ok / limit (guard, 4.6) / auth (alert, stop) / transient (bounded retry: at most 3, at least 1 min apart) / unknown (inspect the thread output). A missing or invalid JSON summary counts as failure. Worktree problems (4.4) mark the session `failed` for human attention.
- **CLI fallback (not built)**: `HeadlessBgLauncher` (M2 + arm-ahead, validated by #11 on Claude Code 2.1.306): a headless `claude -p --session-id` summary, then `claude --bg --resume --remote-control`, URL from `claude logs`, restarts via `claude respawn`. Recorded in git history (commit 8bdc33d) and issue #11; built only if #32 or #34 fails.

### 4.4 Worktree manager

- Responsibility: `ensure(pr) -> path`. Two strategies behind one interface, chosen by O3:
  - `T3Managed`: launch with workspace mode "new worktree"; T3 Code names the branch and picks the location. varjo must still `git fetch origin pull/<n>/head` into it and fast-forward the worktree when `head_sha` changes (the thread must be idle).
  - `VarjoManaged`: `git worktree add` under a configurable root at `pull/<n>/head`, passed to T3 Code as workspace mode "existing worktree". varjo refreshes and removes these worktrees itself.
- Inputs: PR number, head SHA. Outputs: `worktree_path`, verified `head_sha`. Failure handling: dirty worktree is never reset; the session is marked `failed` with the reason. Gitignored files such as `.env` are handled in #15.

### 4.5 Notifications (T3 push; optional adapter)

- Default: T3 push to the phone on finish / fail / approval / question (mobile-notifications.md). It requires T3 Connect (O4). varjo sends nothing for "summary ready" or "needs approval".
- Optional: `Notifier { send(notification): Promise<void> }` with a Discord adapter first and ntfy second, opt-in for Tailscale-only setups where T3 push is unavailable and for limit/reboot alerts that have no thread. Content: PR title/number/URL, reviewer, severity, findings summary, files to change, proposed approach, open questions (requirements §3.4), plus the `t3_thread_url`. Output is truncated and rendered as plain text.
- Dedup via `notifications.dedup_key` (for example `review:<session>:<event_id>`, `limit:<window>`). Failure handling: send failures are retried next cycle; a failed notification never re-runs Claude.

### 4.6 Usage-limit guard

- Responsibility (max-plan.md "Proposed limit-handling behavior", D7): preflight check of `limit_state.blocked_until` before any `t3_thread_launch` or message; classify run results; persist the block; surface it once per window; note "resumed" after the first success.
- Detection: from thread state (failed or stalled) and the limit text in `t3_thread_read` output. Signals and parsing are pending #8; the CLI-round signals (max-plan.md §4: `api_error`, 429, the limit text) are the starting hypothesis, and `subtype`-style fields are not relied on.
- Reset: parse the reset time from the limit text in the botPC time zone (unverified), plus 2-5 minutes jitter. Unparsable: backoff 30 min, 1 h, 2 h (capped at 5 h; weekly: 6 h steps).
- Pre-emptive: if T3 Code exposes utilization (unconfirmed), stop starting jobs at five-hour utilization >= ~0.9; otherwise this is dropped. Never retry within a poll cycle; at most one probe per window. Auth errors alert and stop; they are not treated as limits.
- Alerts: no "limit" thread is started. The block is recorded, shown in `varjo status`, and sent through the optional adapter if configured. Usage credits stay OFF so no spend is possible.

### 4.7 launchd jobs

- `com.varjo.poll`: `StartInterval` job running `varjo poll`. `com.varjo.memsync`: timer job running `scripts/memsync.sh` (4.8). There is no recovery job: the `t3 up` service restarts itself (background-service.md).
- Reconciliation at the start of each poll: compare `sessions` with `t3_thread_list`; threads that are missing become `stopped` and are re-launched on the next event; in-flight runs without `finished_at` are marked failed and their events requeued; expired leases are cleared.
- Sleep/lid handling and keeping botPC awake are validated in #7; not designed here. `varjo status` reports login health and the poller alerts on auth failures seen in runs.

### 4.8 Memory sync script

- `scripts/memsync.sh`: `rsync -a --update` over SSH of `~/.claude/projects/<key>/memory/` only, one direction (default mainPC to botPC, direction is a variable to confirm in #14), no `--delete`. Key per repo is computed from the absolute path; identical paths on both Macs (D3). Fallback: `autoMemoryDirectory`.
- Nothing else under `~/.claude` or `~/.claude.json` is synced. Optional pre-sync secret scan of the memory files. Runs from launchd on a timer or manually. Failure handling: failed rsync leaves both sides unchanged; logged by `varjo status`.
- Memory key (#34): confirm that T3 Code's Agent SDK sessions write memory under the same `projects/<key>/memory/` key (including for worktree paths) and honour `CLAUDE_CONFIG_DIR`, so the sync target is the directory T3 Code actually uses.

### 4.9 Handoff hook (Phase 3, pending #34)

- `hooks/pr-handoff.sh`, registered as `PostToolUse` with `if: "Bash(gh pr create *)"` (D8). Reads the hook stdin JSON (`session_id`, `cwd`, `transcript_path`), extracts the PR URL from the tool output, records the PR to session link in SQLite, and asynchronously asks for the §5.4 template. Under T3 Code the request cannot use `claude -p --resume`; it goes into the originating thread as a message via MCP (thread looked up from `session_id`; mapping confirmed in #34).
- Pending #34: whether the Agent SDK session under T3 Code loads project settings and hooks (`settingSources`) and fires this hook with `session_id`/`cwd`.
- Fallback if the hook does not fire: varjo detects PR creation through the poller (a new PR for a branch tied to a tracked thread) and runs the handoff prompt as a message into the originating thread via MCP.
- Output goes through a `HandoffStore` interface keyed by repo + PR number + head SHA; backend per O1. Failure handling: the hook exits 0 always (never blocks PR creation); failures are logged and surfaced by `varjo status`.

### 4.10 Cleanup (`varjo cleanup`)

- Trigger: PR seen merged/closed, plus a grace period (configurable) so the thread stays available. Steps: stop the thread via MCP if running; remove varjo-managed worktrees with `git worktree remove` (after `git worktree unlock` if locked); T3-created worktrees are removed per the O3 outcome.
- Interaction with T3 Code's worktree cleanup setting: it must not delete worktrees varjo still tracks. The exact setting and behavior are documented by #15. Never remove the main checkout.
- Failure handling: anything refused (unpushed, dirty) is reported, not forced.

### 4.11 Power manager

- Requirement: botPC must stay awake only while varjo runs. `varjo start` turns sleep prevention on; `varjo stop` (and the launchd job's exit path) restores the machine's previous settings, not macOS defaults. Validated on hardware by #7.
- Mechanism: on start, snapshot `pmset -g custom` values (`sleep`, `disksleep`, `displaysleep` for AC and battery, plus `SleepDisabled`) to `~/.varjo/power-state.json`, then run `pmset -a sleep 0 disksleep 0 displaysleep 0` and `pmset -a disablesleep 1`. On stop, write the snapshot back and delete it. Idempotent: a present snapshot means "already on" and is never overwritten. `varjo status` reports the current state.
- Why `pmset` and not only `caffeinate`: a `caffeinate -imsu` child prevents idle sleep while the lid is open, but closing the lid is a forced sleep that only `disablesleep 1` (root) overrides on a MacBook without an external display. varjo holds the `caffeinate` child as a belt-and-braces measure and uses `pmset` for the lid-closed case.
- Privilege: `pmset` needs root, so the setup script installs `/etc/sudoers.d/varjo` allowing the varjo user to run `/usr/bin/pmset` with the fixed argument sets above and nothing else, `NOPASSWD`; varjo calls `sudo -n pmset ...` and treats a failure as "sleep prevention unavailable" (alert, keep running). The restore commands take values from the snapshot, so the sudoers wildcards are limited to the `-c`/`-b` restore lines; a later hardening option is a tiny root helper that only accepts the snapshot file.
- Prototype used during Phase 0: `~/varjo-phase0/awake.sh on|off|status` with the matching sudoers rule (`~/varjo-phase0/sudoers-varjo`), to be ported into `varjo start|stop`.

## 5. Data model

Timestamps are ISO 8601 UTC text. The `sessions` table holds the §4.2 fields.

```sql
CREATE TABLE repos (
  id INTEGER PRIMARY KEY,
  owner TEXT NOT NULL, name TEXT NOT NULL,
  local_path TEXT NOT NULL,                -- identical absolute path on both Macs (D3)
  default_branch TEXT NOT NULL,
  model TEXT NOT NULL DEFAULT 'sonnet', effort TEXT NOT NULL DEFAULT 'low',
  allowed_tools_json TEXT,                 -- overrides the Phase 1 default allowlist
  enabled INTEGER NOT NULL DEFAULT 1,
  poll_cursor TEXT,                        -- last successful poll time
  UNIQUE (owner, name)
);

CREATE TABLE prs (
  id INTEGER PRIMARY KEY,
  repo_id INTEGER NOT NULL REFERENCES repos(id),
  number INTEGER NOT NULL,
  title TEXT, url TEXT NOT NULL, author TEXT,
  base_ref TEXT NOT NULL, head_ref TEXT NOT NULL, head_sha TEXT NOT NULL,
  state TEXT NOT NULL CHECK (state IN ('open','merged','closed')),
  closed_at TEXT, updated_at TEXT NOT NULL,
  UNIQUE (repo_id, number)
);

CREATE TABLE sessions (                    -- one row per PR; maps to one T3 Code thread
  id INTEGER PRIMARY KEY,
  pr_id INTEGER NOT NULL REFERENCES prs(id),
  t3_thread_id TEXT NOT NULL UNIQUE,       -- thread id returned by t3_thread_launch
  launcher TEXT NOT NULL CHECK (launcher IN ('t3')),  -- column kept so the CLI fallback can be added
  t3_thread_url TEXT,                      -- link to open the thread in the T3 Code app
  t3_environment TEXT,                     -- T3 Code environment id or name
  worktree_path TEXT, worktree_owner TEXT CHECK (worktree_owner IN ('t3','varjo')),
  branch TEXT, head_sha TEXT,              -- commit the session last saw
  last_event_id INTEGER REFERENCES events(id),
  last_run_at TEXT,
  state TEXT NOT NULL CHECK (state IN
    ('idle','queued','running','awaiting_user','blocked','failed','closed')),
  error_code TEXT, error_message TEXT,
  lease_owner TEXT, lease_until TEXT,      -- prevents concurrent writes to one PR
  created_at TEXT NOT NULL, closed_at TEXT
);
CREATE UNIQUE INDEX one_open_session_per_pr ON sessions(pr_id) WHERE state <> 'closed';

CREATE TABLE events (
  id INTEGER PRIMARY KEY,
  repo_id INTEGER NOT NULL REFERENCES repos(id),
  pr_id INTEGER REFERENCES prs(id),
  source TEXT NOT NULL,                    -- 'github' now; 'jira', 'ci' later
  kind TEXT NOT NULL,                      -- review | review_comment | issue_comment | reply | pr_state
  external_id TEXT NOT NULL,               -- GitHub review id / comment id / node id
  actor TEXT, state_detail TEXT,           -- e.g. APPROVED / CHANGES_REQUESTED
  payload_json TEXT NOT NULL,
  occurred_at TEXT NOT NULL, detected_at TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'new' CHECK (status IN ('new','queued','processed','skipped','failed')),
  run_id INTEGER REFERENCES runs(id),
  UNIQUE (source, kind, external_id)       -- dedup key
);

CREATE TABLE runs (
  id INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL REFERENCES sessions(id),
  purpose TEXT NOT NULL CHECK (purpose IN ('summarize','followup','fix','handoff','recover')),
  launcher TEXT NOT NULL,
  request_json TEXT NOT NULL,              -- the MCP request, secrets redacted
  started_at TEXT NOT NULL, finished_at TEXT, duration_ms INTEGER,
  is_error INTEGER,
  api_error TEXT, api_error_status INTEGER, terminal_reason TEXT,
  limit_kind TEXT,                         -- session | weekly | opus | sonnet | NULL
  num_turns INTEGER, total_cost_usd REAL,  -- trend signal only, not billing; NULL if T3 Code does not expose it
  t3_run_status TEXT,                      -- thread/turn status as reported by T3 Code
  result_path TEXT, log_path TEXT
);

CREATE TABLE notifications (
  id INTEGER PRIMARY KEY,
  kind TEXT NOT NULL,                      -- review_summary | limit_blocked | limit_resumed | auth_failed | error
  session_id INTEGER REFERENCES sessions(id), run_id INTEGER REFERENCES runs(id),
  channel TEXT NOT NULL,
  dedup_key TEXT NOT NULL UNIQUE,
  body TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending','sent','failed')),
  error TEXT, created_at TEXT NOT NULL, sent_at TEXT
);

CREATE TABLE limit_state (                 -- single row: limits are per account
  id INTEGER PRIMARY KEY CHECK (id = 1),
  blocked_until TEXT, limit_kind TEXT, reset_source TEXT,  -- 'event' | 'parsed' | 'fallback'
  backoff_step INTEGER NOT NULL DEFAULT 0,
  utilization_five_hour REAL, utilization_seven_day REAL,
  last_notified_at TEXT, last_notified_window TEXT,        -- one notification per window
  updated_at TEXT NOT NULL
);
```

## 6. Flows

**(a) New review detected, summary, notification**
1. launchd runs `varjo poll`; the guard checks `limit_state.blocked_until` (skip T3 work if blocked).
2. The GitHub adapter inserts new review/comment events; duplicates are rejected by the unique key.
3. The handler groups new events by PR, upserts `prs`, and claims the session lease.
4. The worktree manager ensures the worktree at the PR head per O3.
5. The launcher calls `t3_thread_launch` with the project, the workspace mode per O3, the read-only summary permission mode (exact T3 mapping recorded by #32), the model per repo config, and the prompt (fixed template plus review text in a delimited data block). It waits with `t3_thread_wait` (bounded), then `t3_thread_read` to extract and validate the JSON summary.
6. The result is recorded in `runs`; events are marked `processed`; `sessions.state` becomes `awaiting_user`; varjo stores `t3_thread_id` and `t3_thread_url`.
7. The summary is visible on the phone as the thread's last assistant turn; T3 push notifies "finished". No varjo notification is sent unless the optional adapter is configured (`notifications.dedup_key` prevents duplicates).

**(b) Follow-up review on the same PR**
1. The poller finds new events for a PR with an open session (unique index guarantees one).
2. The worktree manager refreshes the tip if `head_sha` changed and the thread is idle.
3. The launcher sends the new review as a message into the same thread (MCP message tool), then waits and reads as in (a).
4. `last_event_id`, `last_run_at` and the run are recorded.

**(c) User continues from the phone or mainPC**
- The user opens the thread in the T3 Code mobile, desktop or web app. Messages and approvals happen in T3's UI; nothing in varjo is needed.
- `varjo open <pr>` prints the stored `t3_thread_url`.
- Phase 2 pushes require human approval in the thread. Phone-initiated work that no event triggered is the T3 mobile app starting a thread (Phase 4).

**(d) Usage limit hit**
1. A thread fails or stalls with the limit text.
2. The guard classifies it from `t3_thread_read` output (signals and parsing pending #8), sets `blocked_until = reset + jitter`, and records `limit_kind`.
3. The block is surfaced once per window: recorded, shown in `varjo status`, and sent through the optional adapter if configured. No "limit" thread is started.
4. The affected events stay `queued` (no retry in this cycle). Later polls skip T3 work until `blocked_until`; polling GitHub may continue.
5. After the reset, one probe run is allowed; on success the queue drains serially.

**(e) botPC reboot**
1. The T3 Code launchd service (`t3 up`) restarts itself; launchd starts `com.varjo.poll` on schedule.
2. At the start of the next poll, varjo reconciles `sessions` with `t3_thread_list`: threads that are missing become `stopped` and are re-launched on the next event; expired leases are cleared.
3. In-flight runs found without `finished_at` are marked failed and their events requeued (idempotent).

**(f) PR merged/closed, cleanup**
1. The poller sees `state` change to `merged`/`closed`; `prs.closed_at` is set; the session moves to `closed` after the grace period.
2. `varjo cleanup` stops the thread via MCP and removes the worktree per O3 (`git worktree remove` for varjo-managed ones).
3. T3 Code's worktree cleanup setting must not delete worktrees varjo still tracks; the interaction is documented by #15.
4. Refusals (unpushed commits, uncommitted changes) are reported and left in place.

## 7. Phase mapping

Phase 1: poller, store, T3 launcher, summary schema, guard, `com.varjo.poll`, power manager (4.11), minimal cleanup. Phase 2: auto-accept-edits runs; pushes approved in T3's UI. Phase 3: memory sync (4.8), handoff (4.9). Phase 4: phone-initiated tasks are simply the T3 mobile app starting a thread (no varjo code); Jira adapter via the same `EventSource` interface; optional GitHub webhook adapter (O5 route 3).

## 8. Security model

| Requirement | Mechanism |
|---|---|
| No inbound ports on botPC (§6) | T3 Connect is an outbound tunnel; LAN and Tailscale pairing stay private; the poller is outbound `gh`; no varjo server or webhook. sshd is needed only for memsync, over a private path (Tailscale with explicit ACLs, or LAN); never public (§6 candidate 3) |
| No credentials on custom cloud (§6) | T3 Connect holds device-auth tokens for the relay and, only if enabled, held webhook bodies (up to 24 h); this is the §6 trade-off recorded as O4. Claude credentials never leave botPC. varjo's MCP OAuth credential is stored 0600 on botPC and never synced; optional notifier webhooks/tokens are outbound-only secrets in a 0600 file or Keychain; no varjo-owned relay server |
| Least-privilege GitHub/AWS (§6, §10) | GitHub token scoped to PR read (+ push to non-default branches in Phase 2); no AWS access in Phase 1; varjo's MCP sign-in is approved at the lowest level that allows `t3_thread_launch`; varjo-launched threads use a read-only mode in Phase 1 and auto-accept edits inside the worktree in Phase 2; T3 managed policy caps the maximum mode |
| No direct pushes to `main` (§6) | Phase 1 has no push tool; Phase 2 pushes need human approval in T3's UI; branch protection on GitHub as the backstop |
| Human approval for production (§6) | No production credentials on botPC; read-only mode denies unlisted tools; auto-accept edits is limited to the worktree |
| No arbitrary external command execution (§10) | Events only select a PR and a fixed prompt template; no command text from events reaches a shell; a thread started over MCP cannot exceed the level its sign-in was approved with |
| PR comments are untrusted (§10) | Review text goes into a delimited data block in the prompt (poller route) or through T3's `{{path}}` template (webhook route, which has injection hardening) with an instruction to treat it as data; read-only mode means an injected instruction cannot edit, push or exfiltrate via Bash; optional per-repo reviewer allowlist (proposed config); optional notification text is truncated plain text |
| Minimum privileges for automation (§10) | Separate OS credentials for botPC (D2); never copy mainPC credentials; `ANTHROPIC_API_KEY` never set; usage credits OFF; only memory directories synced from `~/.claude` |
| Idempotency, no concurrent PR writes (§10) | Unique event key; session lease; unique open-session index; global single-run slot |
| Traceability (§10) | `runs` and `notifications` tables, T3 Code thread history, per-run `log_path` |
| What is sent where (§6) | Prompts, file contents and command output read by tools, PR diffs and review text go to Anthropic (inference is always in Anthropic's cloud, requirements §6). While T3 Connect is used, thread traffic transits the relay (end-to-end over the tunnel; the relay docs say bodies and tokens are kept out of traces). T3 telemetry per `docs/user/telemetry.md` (opt-out to record) |
| Credential isolation | One `claude auth login` credential, used by T3 Code only; varjo holds no Claude credential; single-user, no sharing (max-plan.md §2) |
| Update pinning | T3 Code is pre-1.0 with 1-2 stable releases a week plus nightlies; botPC pins a stable version and upgrades deliberately (policy item in #16) |
| Root privilege | Only `pmset` with fixed argument sets via `/etc/sudoers.d/varjo` (4.11); no other sudo; varjo never runs as root |

## 9. Open decisions

### O1. PR handoff storage (needs user decision)

| Option | Pros | Cons |
|---|---|---|
| (a) Hidden `<details>` in the GitHub PR body | Travels with the PR, visible from any device | Leaks design reasoning into a shared repo and to reviewers; size limits; edit churn |
| (b) Git-tracked file in the repo | Versioned, reviewable | Pollutes history; shared repo exposure; needs a commit and push per update |
| (c) Private store on botPC, `~/.varjo/handoff/<owner>/<repo>/<pr>.md`, keyed by PR number + head SHA, synced with the same rsync as memory | Matches §6 confidentiality; no repo noise; reuses the sync mechanism | Only on the user's machines; needs sync discipline; not visible to teammates |

Recommendation: (c). `HandoffStore` is an interface, so (a) or (b) can be added as an opt-in export later.

### O2. Notification channel to mainPC and phone (needs user decision; depends on O4)

| Option | Pros | Cons |
|---|---|---|
| (a) T3 push (default) | No varjo code; covers finish / fail / approval / question; works on the phone | Requires T3 Connect (O4); only for events that have a thread, so no limit or reboot alerts |
| (b) Discord webhook (optional adapter) | Works without T3 Connect; can carry limit/reboot alerts; phone push | Third-party service sees summaries; webhook URL is a secret |
| (c) ntfy (optional adapter; self-hosted or ntfy.sh private topic) | Simple push to phone and Mac; no account coupling | ntfy.sh topic is a shared secret; self-hosting needs a reachable host |
| (d) macOS Notification Center via SSH | No third party | mainPC only, no phone; needs sshd/private path |
| (e) Email | Universal | Slow, noisy, summaries stored in a mailbox |

Recommendation: (a) as the default with T3 Connect. Add an adapter only for Tailscale-only setups or for limit/reboot alerts: Discord first, ntfy second; keep summaries free of file contents beyond what the template needs.

### O3. Worktree ownership (needs user decision, decided by experiment #15)

| Option | Pros | Cons |
|---|---|---|
| (a) T3 Code `new worktree` per thread | Less code; T3 Code handles creation | T3 names the branch and picks the location; varjo must still fetch `pull/<n>/head` into it and refresh the tip |
| (b) varjo-managed `git worktree` at `pull/<n>/head`, passed as `existing worktree` | Exact, refreshable PR head; full control of tip, location and cleanup | More code; varjo removes the worktree itself and must coexist with T3's cleanup setting |

Recommendation: (b), because the PR head must be exact and refreshable; decided by #15.

### O4. T3 Connect vs Tailscale-only (needs user decision, decided after #31)

| Option | Pros | Cons |
|---|---|---|
| (a) T3 Connect | Push notifications; phone access over mobile data; held webhooks | Third-party relay (Clerk auth, device-auth tokens on the relay, held webhook bodies if enabled); 3-machine free cap; pricing may change |
| (b) Tailscale-only | Everything first-party; no relay | No push; the phone needs the tailnet |

Recommendation: (a) with "Require approval for new devices" on and held webhooks off; record in #30 decision 1 after #31.

### O5. Trigger route (needs user decision, decided by #33)

| Option | Pros | Cons |
|---|---|---|
| (1) varjo poller to MCP (default) | No webhook rights needed; dedup in SQLite; already designed | Polling latency; needs the poller and an MCP credential |
| (2) T3 interval task only | No varjo trigger code | Expected unsuitable: fixed workspace per task, dedup in prompts |
| (3) GitHub webhook to T3 Connect relay to T3 webhook task (optional adapter) | Immediate; relay can hold hooks up to 24 h while botPC is offline | Needs webhook admin on the repo and T3 Connect; held bodies on the relay |

Recommendation: (1) as the default, (3) as an optional adapter for repos with webhook admin.

## 10. What Phase 0 must confirm before Phase 1

- #7: lid-closed stability, sleep prevention (the `t3 up` service and the Claude process need the Mac awake).
- #8 (re-scoped): T3 Code picks up the dedicated `claude auth login` on botPC; the `t3 up` service environment (including `ANTHROPIC_API_KEY` unset); limit observability via MCP (thread state and `t3_thread_read` output at a limit, reset text); Max terms via the SDK (= #30 decision 3, including whether the paused programmatic-usage split is revived, max-plan.md §1).
- #31 (replaces #9): T3 Connect vs Tailscale-only (O4): phone over mobile data, push, device approval, held webhooks off, what the relay stores.
- #32 (top priority, replaces #11): end-to-end MCP launch: OAuth sign-in and access level for varjo; `t3_thread_launch` with project, workspace mode, permission mode and model; `t3_thread_wait` and `t3_thread_read` returning the fenced JSON summary; a follow-up message into the same thread; exact T3 mode for the read-only summary; thread visible on the phone with a "finished" push.
- #34: the Agent SDK session under T3 Code loads CLAUDE.md, rules, settings, hooks and memory (`settingSources`, `CLAUDE_CONFIG_DIR`); the `PostToolUse` handoff hook fires with `session_id`/`cwd`; memory key cross-check for #14.
- #10 (re-scoped): toolchains and `gh` are available in the `t3 up` service environment; the read-only summary is useful with the allowlist.
- #33: trigger routes (O5): whether a T3 interval task can serve the poller's job; the webhook route via the T3 Connect relay; decides O5.
- #12 (re-scoped): the T3 mobile app drives a thread: messages, approvals, push.
- #15 (re-scoped): worktree lifecycle for T3-created and varjo-created worktrees: `pull/<n>/head` fetch and refresh, pushing fixes from the worktree, T3's worktree cleanup setting against tracked worktrees (decides O3).
- #14: memory sync viability: identical-path keys, `autoMemoryDirectory` per-repo values, symlink behavior, external edits during a live thread, sync direction.
- #16 (security baseline, extended): the read-only allowlist is sufficient for a useful summary without `gh api`; reviewer allowlist default; T3 managed policy cap; MCP access levels; update pinning policy; telemetry opt-out.
- Closed: #9, #11 (result kept as the CLI fallback), #13.

Order: #7 #8, then #31, then #32, then #34 #10, then #33, then #12 #15, then #14, then the #30 decision, then the docs revision.

Fallback rule: if #32 or #34 fails, return to the CLI launcher recorded in git history (commit 8bdc33d, issue #11) and reopen the Remote Control items.

## 11. Repository layout (proposed)

```
varjo/
  src/
    adapters/github/   # EventSource: gh/API polling, normalization, PR state
    store/             # SQLite schema, migrations, leases, dedup queries
    t3/                # SessionLauncher, T3McpLauncher: MCP client, OAuth credential handling, thread launch/message/read/wait
    worktree/          # T3Managed and VarjoManaged strategies
    notifier/          # optional adapters: Notifier interface, discord/, ntfy/
    limits/            # usage-limit detection, reset parsing, blocked_until
    cleanup/           # PR-closed policy: stop thread, git worktree remove
    cli/               # varjo poll | status | open <pr> | cleanup
  hooks/               # pr-handoff.sh (PostToolUse; pending #34)
  launchd/             # com.varjo.poll.plist, com.varjo.memsync.plist
  scripts/             # memsync.sh, install helpers
  docs/                # requirements.md, research/, design/
```
