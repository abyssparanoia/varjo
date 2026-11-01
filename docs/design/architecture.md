# varjo architecture: coverage map and minimal components

Status: Draft, pending Phase 0 results; last updated 2026-11-01.

Spec: [docs/requirements.md](../requirements.md) (unchanged; corrections are listed below). Research inputs: `docs/research/claude-code.md`, `remote-access.md`, `max-plan.md`, `auto-memory.md`.

## 1. Summary

- varjo is a thin orchestrator on botPC. It detects GitHub PR review events by polling, keeps state in SQLite, runs one Claude Code session per PR through the `claude` CLI (never the Agent SDK), and notifies the user.
- Claude Code itself provides execution, sessions, worktrees, hooks, headless mode, background sessions, Remote Control (RC), auth tokens and cleanup commands. varjo reuses all of them.
- Users reach botPC sessions through RC (claude.ai/code, Claude Desktop, phone). Desktop-over-SSH is an interactive fallback only. Dispatch and cloud sessions/routines are out of scope.
- varjo builds only: the poller, event store with dedup, session launcher, worktree manager, notifier, usage-limit guard, launchd supervision with reboot recovery, a memory sync script, a PR-handoff hook, and cleanup.
- Implementation: a single TypeScript/Node CLI `varjo` (`poll`, `status`, `open <pr>`, `cleanup`) plus SQLite. Shell is used only for launchd plists and the hook.
- Three decisions remain open for the user (section 9): handoff storage, notification channel, worktree ownership.

## 2. Corrections to the requirements document

| Requirements section | Assumption | What the research found | Consequence |
|---|---|---|---|
| §4.3, §9 Phase 0 | CLI-created sessions on botPC can show up in Claude Desktop on mainPC | Desktop over SSH lists only sessions Desktop starts; no session-ID deep link; `claude --desktop --resume` is local-only (remote-access.md §A). RC is the documented path to claude.ai/code, phone and Desktop (remote-access.md §B, claude-code.md §17.1) | RC is primary (D1). Desktop SSH is an interactive fallback. Experiment #11 decides launch mode M1 vs M2 |
| §2.3, §9 Phase 0 | Dispatch may run tasks in botPC repos | Dispatch cannot target a machine or repo; it needs Desktop running on the paired computer; limited beta, closed to new users (remote-access.md §C) | Dispatch is out of scope; #13 is confirmation only |
| §8 | Agent SDK / Dispatch are options for AI execution and phone control | Anthropic docs steer SDK users to API keys and bar claude.ai login for third-party products; non-bare `claude -p` uses the subscription (claude-code.md §7, max-plan.md §2) | Drive the CLI (`claude -p`, `--bg`, `--worktree`) from launchd-managed code; no `@anthropic-ai/claude-agent-sdk` |
| §5.4 | A Skill/Hook can fire "at PR creation" | No commit/push/PR hook event exists (claude-code.md §3, §17.4) | `PostToolUse` hook with `if: "Bash(gh pr create *)"` (D8); unconfirmed in `-p` runs |
| §1, §8 (Max), max-plan.md §1 | Plan tiers are "Max 5x/20x" | The 20x tier is now called Ultra; docs say "Pro, Max, Ultra" (max-plan.md §1) | Use "Max/Ultra" in docs and config; limits are per account, shared by botPC and mainPC (max-plan.md §3) |
| §6 | Execution is local and only inference leaves the machine | While RC is connected the transcript is stored on Anthropic servers; RC is unavailable under ZDR/HIPAA (remote-access.md §B) | Security model (section 8) states this explicitly; RC use is a conscious trade-off against §6 confidentiality |
| §2.1, §10 | botPC recovers after reboot | `--bg` sessions stop on reboot; no launchd integration is provided (claude-code.md §10) | varjo ships a launchd recovery job (flow e) |
| §9 Phase 0 (Max auth), §6 | One login can serve CLI, RC and unattended jobs | `setup-token` tokens cannot establish RC; `CLAUDE_CODE_OAUTH_TOKEN` overrides `/login` in new sessions; `ANTHROPIC_API_KEY` always wins in `-p`; `--bare` ignores OAuth (claude-code.md §12, max-plan.md §1) | Two credentials on botPC (D2); the token env var is injected only into headless child processes, never into RC-bearing launches |
| §5.3 | Auto Memory sharing is an open question | Memory is per repo, shared across worktrees, machine-local, keyed by absolute path (auto-memory.md) | Identical repo paths on both Macs, fallback `autoMemoryDirectory` (D3) |
| §3.2 | Events can be detected with existing tooling | No local review-event source; Routines GitHub trigger runs in the cloud only (claude-code.md §15, §16) | varjo must build the poller |
| §10 | "Stop gracefully at the usage limit" is straightforward | No documented exit code for limit hits in `-p`; observed behavior differs per version and is unstable (max-plan.md §4) | Detection by OR over several signals, key on `is_error`/`api_error`, not `subtype` (D7) |
| §3.3 step 4 | A worktree is created or updated per PR | `--worktree "#<n>"` keeps the old tip for PR-reference names; `-p` runs never clean up their worktrees (claude-code.md §5) | varjo must refresh the tip itself and sweep stale worktrees |

## 3. Coverage map

| Capability | Existing tool? | Build? | Notes |
|---|---|---|---|
| Run Claude in a repo | Claude Code CLI | no | `claude -p`, `--bg` (claude-code.md §2, §10) |
| Remote GUI (mainPC) | RC via Claude Desktop / claude.ai/code; Desktop SSH as fallback | no | Desktop lists RC sessions; cross-Mac UI path unconfirmed (remote-access.md §B; #11) |
| Smartphone control | Remote Control (Claude app Code tab) | no | Send messages, approve prompts, push notifications (remote-access.md §B, Mobile). Dispatch out of scope |
| PR review event detection | none | **yes** (poller) | `gh` CLI/API; no local event source exists (claude-code.md §16) |
| Dedup + state | none | **yes** (SQLite) | GitHub review/comment IDs as keys (requirements §3.2) |
| Worktree per PR | git; `claude --worktree "#<n>"` | thin wrapper | Ownership is open decision O3 (claude-code.md §5) |
| PR ↔ session mapping | `--from-pr` (interactive helper only) | **yes** | `--from-pr` storage and `-p` coverage unconfirmed (claude-code.md §11) |
| Session create/resume | `--session-id`, `--resume`, `-p --resume` | no | varjo supplies the UUID and stores it (claude-code.md §1) |
| Headless review summary | `claude -p --json-schema` | prompt + schema only | claude-code.md §2, §14 |
| Background per-PR worker | `--bg`, `attach`, `logs`, `stop`, `respawn`, `agents` | no (mode M1) | claude-code.md §10 |
| Notification to mainPC/phone | Stop/Notification hooks, RC push notifications | **yes** (adapter) | Channel is open decision O2; RC push covers only RC-connected sessions |
| Usage-limit handling | none | **yes** (guard) | max-plan.md §4, "Proposed limit-handling behavior" |
| Unattended auth | `claude auth login`, `claude setup-token` | config only | Secure token storage only (claude-code.md §12) |
| PR handoff doc | `PostToolUse` hook, Skill | small | §5.4 template; storage is O1 (claude-code.md §3, §4) |
| Memory sync | rsync over SSH (Git/Syncthing rejected for now) | config only | One-way, `projects/<key>/memory/` only, no `--delete` (auto-memory.md) |
| Scheduling | launchd | config only | Poll interval and recovery jobs |
| Reboot recovery | none for `--bg` | thin | launchd job runs `claude agents` then `claude respawn` (claude-code.md §10) |
| Cleanup of sessions/worktrees | `claude rm`, `claude purge`, `git worktree remove`, `cleanupPeriodDays` | policy only | Which PRs are closed is varjo's job (claude-code.md §13, §16) |
| Logs and traceability | `claude logs`, transcripts, stream-json | **yes** (runs table) | claude-code.md §16 |
| Approval gate for pushes (Phase 2) | RC permission prompts; `--permission-prompt-tool` | no (first iteration) | Human approves through RC; custom relay deferred |
| Jira/CI sources | none | later | Same `EventSource` interface |

## 4. Components varjo builds

All components live in one Node process per invocation (`varjo poll` from launchd) except hooks and scripts. Concurrency control is a per-session lease plus one global concurrent-run slot (max-plan.md §6: single concurrent run).

### 4.1 Poller (GitHub adapter)

- Interfaces: `EventSource { poll(since): Event[] }`, `EventHandler { handle(event): void }`. GitHub is the first `EventSource`; Jira/CI later (requirements §7).
- Responsibility: for each enabled repo, list the user's open PRs, fetch reviews, review comments and replies newer than the stored cursor, and normalize them to `Event`. Also report PR state changes (merged/closed) for cleanup.
- Inputs: `repos` table, GitHub auth already present for `gh` on botPC (fine-grained token or `gh auth`, scoped to PR read + contents read for Phase 1). Outputs: rows in `events`, updated `prs`.
- Flags/commands: `gh` CLI or `gh api`; no `claude` flags.
- Failure handling: network/HTTP errors abort the cycle without advancing the cursor; the next launchd interval retries. GitHub rate limits back off the same way. Never calls `claude`.

### 4.2 Event store and dedup (SQLite)

- Responsibility: idempotency (requirements §10). `UNIQUE(source, kind, external_id)` on `events` makes re-polling safe; event processing status is advanced in the same transaction that claims a lease on the session.
- Inputs: normalized events. Outputs: the work queue (events with status `new`) grouped by PR. Multiple new events on one PR in one cycle are batched into one run.
- Failure handling: crash mid-run leaves the lease to expire (`lease_until`); events stay `queued` and are retried subject to the limit guard. Schema: section 5.

### 4.3 Session launcher (one interface, two implementations)

```ts
interface SessionLauncher {
  launch(job: ReviewJob): Promise<LaunchResult>;   // first review of a PR
  resume(session: Session, job: FollowUpJob): Promise<RunResult>; // later reviews
}
```

- Shared: varjo generates the UUID and stores it before launching; permission and model settings come from per-repo config; PR comment text is passed as delimited data (section 8).
- **M1 (`BgRcLauncher`)**: `claude --bg --remote-control --session-id <uuid> --worktree "#<n>" "<prompt>"`. The notification carries the claude.ai/code URL. Follow-ups: `claude --bg --resume <uuid>` (continues under the same ID; a copy with a `note:` line if already running). Summary text is collected by a `Stop` hook (`last_assistant_message`) or `claude logs`. Unconfirmed: whether `--bg` + `--remote-control` combine, how the URL is obtained programmatically, whether Remote Control starts without an interactive confirmation (claude-code.md §10, remote-access.md §B; #11).
- **M2 (`HeadlessTmuxLauncher`)**: `claude -p --session-id <uuid> --output-format json --json-schema <schema> --permission-mode dontAsk --permission-prompts none --allowedTools <list> --max-turns <n> --model sonnet --effort low`, cwd = worktree. The structured summary is sent in the notification together with the command `varjo open <pr>`, which on botPC runs `claude --resume <uuid> --remote-control` in a tmux session. Follow-ups: `claude -p --resume <uuid> ...`. Unconfirmed: `--resume ... --remote-control` re-attaching a `-p` session to claude.ai/code (remote-access.md §B, Phase 0 order step 3).
- Credentials: M1 and the interactive half of M2 run with the `claude auth login` credential (RC requires it), so `CLAUDE_CODE_OAUTH_TOKEN` must not be in their environment. The token is injected only into headless `-p` children. `ANTHROPIC_API_KEY` is stripped from every child environment; `--bare` is never passed.
- Safety flags (Phase 1, D6): `--permission-mode dontAsk`, allowlist `Read`, `Grep`, `Glob`, `Bash(gh pr view *)`, `Bash(gh pr diff *)`, `Bash(git log *)`, `Bash(git diff *)`; `--max-turns`; wall-clock timeout; default `--model sonnet --effort low`, overridable per repo. Phase 2 fix runs: `--permission-mode acceptEdits` inside the worktree; pushes stay human-approved.
- Failure handling: each invocation is recorded in `runs`. Results are classified as ok / limit (guard, 4.6) / auth (alert, stop) / transient (bounded retry: at most 3, at least 1 min apart) / unknown (inspect transcript for `isApiErrorMessage`, max-plan.md §4). A missing `structured_output` counts as failure. Worktree-resume refusals (`worktree_resume_refused`, `worktree_unverified`, claude-code.md §5) mark the session `failed` for human attention.

### 4.4 Worktree manager

- Responsibility: `ensure(pr) -> path`. Two strategies behind one interface, chosen by O3:
  - `ClaudeManaged`: pass `--worktree "#<n>"`; read back the path (`.claude/worktrees/pr-<n>`). Because PR-reference names keep the old tip (claude-code.md §5), varjo must still `git fetch origin pull/<n>/head` and fast-forward the worktree when `head_sha` changes.
  - `VarjoManaged`: `git worktree add` under a configurable root, fetch `pull/<n>/head`, run `claude` with cwd = worktree. `claude rm` leaves such worktrees alone, so varjo removes them itself.
- Inputs: PR number, head SHA. Outputs: `worktree_path`, verified `head_sha`. Failure handling: dirty worktree is never reset; the session is marked `failed` with the reason. `.worktreeinclude` handles gitignored files such as `.env`.

### 4.5 Notifier

- Interface: `Notifier { send(notification): Promise<void> }`. Content: PR title/number/URL, reviewer, severity, findings summary, files to change, proposed approach, open questions (requirements §3.4), plus the access target (M1 URL or M2 command). Output is truncated and rendered as plain text.
- Dedup via `notifications.dedup_key` (for example `review:<session>:<event_id>`, `limit:<window>`). Channel adapters (Discord, ntfy, macOS-over-SSH, email) depend on O2. Failure handling: send failures are retried next cycle; a failed notification never re-runs Claude.

### 4.6 Usage-limit guard

- Responsibility (max-plan.md "Proposed limit-handling behavior", D7): preflight check of `limit_state.blocked_until` before any `claude` spawn; classify run results; persist the block; notify once per window; send "resumed" after the first success.
- Detection (OR): JSON `api_error == "usage_limit_reached"`; `api_error_status == 429`; `is_error` with text matching `/hit your (session|weekly|Opus|Sonnet) limit/`; stream-json `rate_limit_event` with `status == "rejected"`. `subtype` is ignored.
- Reset: epoch `resetsAt` if present, else parse "resets 3:45pm" in the botPC time zone (unverified), plus 2-5 minutes jitter. Unparsable: backoff 30 min, 1 h, 2 h (capped at 5 h; weekly: 6 h steps).
- Pre-emptive: stop starting jobs when five-hour utilization >= ~0.9 (from `rate_limit_event` `unifiedWindows`, undocumented fields). Never retry within a poll cycle; at most one probe per window. `CLAUDE_CODE_RETRY_WATCHDOG` is never set. Auth errors alert and stop; they are not treated as limits.
- Credits: usage credits stay OFF so no spend is possible; the guard asserts a subscription login at start-up (`claude auth status`, output shape unconfirmed, max-plan.md §5).

### 4.7 launchd supervisor and reboot recovery

- `com.varjo.poll`: `StartInterval` job running `varjo poll`. `com.varjo.recover`: `RunAtLoad` job that waits for network, runs `claude agents` (starts the supervisor) then `claude respawn --all`, reconciles `sessions` against `claude agents --json`, and for sessions it cannot restore marks them `stopped` so the next event respawns or re-launches them. Unconfirmed: that this restores sessions after reboot (claude-code.md §18.7), and `agents --json` stability.
- Sleep/lid handling and keeping botPC awake are validated in #7; not designed here. Expired logins stop `--bg` sessions and RC (claude-code.md §10, §12), so `varjo status` reports login health and the poller alerts on auth failures.
- M2 tmux sessions are not restored on reboot; they are on demand by design.

### 4.8 Memory sync script

- `scripts/memsync.sh`: `rsync -a --update` over SSH of `~/.claude/projects/<key>/memory/` only, one direction (default mainPC to botPC, direction is a variable to confirm in #14), no `--delete`. Key per repo is computed from the absolute path; identical paths on both Macs (D3). Fallback: `autoMemoryDirectory`.
- Nothing else under `~/.claude` or `~/.claude.json` is synced. Optional pre-sync secret scan of the memory files. Runs from launchd on a timer or manually. Failure handling: failed rsync leaves both sides unchanged; logged by `varjo status`.

### 4.9 Handoff hook (Phase 3)

- `hooks/pr-handoff.sh`, registered as `PostToolUse` with `if: "Bash(gh pr create *)"` (D8). Reads the hook stdin JSON (`session_id`, `cwd`, `transcript_path`), extracts the PR URL from the tool output, records the PR to session link in SQLite, and asynchronously asks the same session for the §5.4 template (`claude -p --resume <session_id>` with a repo skill and `--json-schema`).
- Output goes through a `HandoffStore` interface keyed by repo + PR number + head SHA; backend per O1. Unconfirmed: `if` filters firing in `-p`, hook stdin fields (claude-code.md §18.9-10). Failure handling: hook exits 0 always (never blocks PR creation); failures are logged and surfaced by `varjo status`.

### 4.10 Cleanup (`varjo cleanup`)

- Trigger: PR seen merged/closed, plus a grace period (configurable) so the transcript stays available. Steps: `claude stop <id>` if running; `claude rm <id>` for `--bg` sessions (refuses on unpushed commits, keeps dirty worktrees); `git worktree remove` (after `git worktree unlock` if locked) for `-p`-created and VarjoManaged worktrees; `claude purge <worktree path> --dry-run` then `-y` only for worktree paths, never the main checkout (whether purge touches shared memory is unconfirmed).
- Raise `cleanupPeriodDays` on botPC so transcripts of long-lived PRs survive (claude-code.md §13). Failure handling: anything refused (unpushed, dirty) is reported, not forced.

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

CREATE TABLE sessions (
  id INTEGER PRIMARY KEY,
  pr_id INTEGER NOT NULL REFERENCES prs(id),
  claude_session_id TEXT NOT NULL UNIQUE,  -- UUID passed via --session-id
  launch_mode TEXT NOT NULL CHECK (launch_mode IN ('M1','M2')),
  bg_id TEXT,                              -- M1: id printed by claude --bg
  rc_url TEXT,                             -- claude.ai/code/<id> when known
  worktree_path TEXT, worktree_owner TEXT CHECK (worktree_owner IN ('claude','varjo')),
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
  launch_mode TEXT NOT NULL,
  args_json TEXT NOT NULL,                 -- argv, secrets redacted
  started_at TEXT NOT NULL, finished_at TEXT, duration_ms INTEGER,
  exit_code INTEGER, is_error INTEGER,
  api_error TEXT, api_error_status INTEGER, terminal_reason TEXT,
  limit_kind TEXT,                         -- session | weekly | opus | sonnet | NULL
  num_turns INTEGER, total_cost_usd REAL,  -- trend signal only, not billing
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
1. launchd runs `varjo poll`; the guard checks `limit_state.blocked_until` (skip claude work if blocked).
2. The GitHub adapter inserts new review/comment events; duplicates are rejected by the unique key.
3. The handler groups new events by PR, upserts `prs`, and claims the session lease (creating `sessions` with a fresh UUID if none exists).
4. The worktree manager ensures the worktree at the PR head.
5. The launcher starts the session: M1 `--bg --remote-control ...`, or M2 headless `-p` with `--json-schema`. Data is passed as delimited input.
6. The result is recorded in `runs`; events are marked `processed`; `sessions.state` becomes `awaiting_user`.
7. The notifier sends the summary with the access target; `notifications.dedup_key` prevents duplicates.

**(b) Follow-up review on the same PR**
1. The poller finds new events for a PR with an open session (unique index guarantees one).
2. The worktree manager refreshes the tip if `head_sha` changed.
3. M2: `claude -p --resume <uuid> ...`; M1: `claude --bg --resume <uuid> ...`. If the session is still live in another process the launch is refused or yields a copy; varjo then waits for the lease or defers to the next cycle rather than forking.
4. `last_event_id`, `last_run_at` and the run are recorded; the notifier sends the new summary.

**(c) User continues the session from the phone**
- M1: the notification link opens `claude.ai/code/<id>` in the Claude app (or browser, or Desktop's session list). The session is already RC-connected; the user sends instructions and answers permission prompts. RC push notifications cover "Claude finished / needs a decision".
- M2: the notification includes the summary and the command `varjo open <pr>`; `varjo open` runs `claude --resume <uuid> --remote-control` in tmux on botPC, after which the session appears in the Code tab like M1. Gap: a phone cannot run `varjo open` without an SSH client over a private path. Options: an SSH app on the phone, or an "arm ahead" setting that starts the tmux RC session right after the summary (an idle interactive session). Pick after #11/#12.
- Both: the user must not resume the same session on two machines at once (auto-memory.md). Phase 2 pushes require human approval in the RC prompt.

**(d) Usage limit hit**
1. A run returns `is_error`, `api_error`, 429, the limit text, or `rate_limit_event` rejected.
2. The guard parses the reset time, sets `blocked_until = reset + jitter`, records `limit_kind`.
3. If `last_notified_window` differs, a `limit_blocked` notification is sent once; otherwise silent.
4. The affected events stay `queued` (no retry in this cycle). Later polls skip claude work until `blocked_until`; polling GitHub may continue.
5. After the reset, one probe run is allowed; on success send "resumed" and drain the queue serially. At about 90% utilization new jobs are held until `resetsAt`.

**(e) botPC reboot**
1. launchd starts `com.varjo.recover` at load (and `com.varjo.poll` on schedule).
2. Recovery waits for network, runs `claude agents` to start the supervisor, then `claude respawn --all`.
3. It reconciles `sessions` with `claude agents --json`: restored sessions go back to their prior state; unrestored ones become `stopped`; expired leases are cleared.
4. A "botPC restarted" notification is sent. In-flight runs found without `finished_at` are marked failed and their events requeued (idempotent). M2 tmux sessions are not restored.

**(f) PR merged/closed, cleanup**
1. The poller sees `state` change to `merged`/`closed`; `prs.closed_at` is set; the session moves to `closed` after the grace period.
2. `varjo cleanup` stops a running session, runs `claude rm <id>` (M1) and `git worktree remove` for worktrees `claude rm` will not touch.
3. Refusals (unpushed commits, dirty tree) are reported via notification and left in place.
4. Optionally `claude purge <worktree path>` for the session's own project key; memory files are never touched by varjo.

## 7. Phase mapping

Phase 1: 4.1-4.7 (M1 or M2), cleanup minimal. Phase 2: `acceptEdits` runs, approvals through RC. Phase 3: 4.8, 4.9. Phase 4: phone-initiated tasks (RC server mode `--spawn worktree` is a candidate, claude-code.md §8), Jira adapter.

## 8. Security model

| Requirement | Mechanism |
|---|---|
| No inbound ports on botPC (§6) | RC and Claude use outbound HTTPS only; poller is outbound `gh`; no varjo server or webhook. Caveat: Desktop-SSH fallback and the rsync memory sync need sshd on a private path (Tailscale with explicit ACLs, or LAN); never public (§6 candidate 3) |
| No credentials on custom cloud (§6) | Notifier webhooks/tokens are outbound-only secrets stored in 0600 file or Keychain; no varjo-owned relay server |
| Least-privilege GitHub/AWS (§6, §10) | GitHub token scoped to PR read (+ push to non-default branches in Phase 2); no AWS access in Phase 1; Claude gets only the tool allowlist |
| No direct pushes to `main` (§6) | Phase 1 has no push tool; Phase 2 pushes need human approval via RC prompt; branch protection on GitHub as the backstop |
| Human approval for production (§6) | No production credentials on botPC; `dontAsk` denies any unlisted tool; `acceptEdits` limited to the worktree |
| No arbitrary external command execution (§10) | Events only select a PR and a fixed prompt template; no command text from events reaches a shell; `--permission-prompts none` denies rather than asks |
| PR comments are untrusted (§10) | Poller fetches text; it is placed in the prompt inside a clearly delimited data block (via stdin, 10 MB cap) with an instruction to treat it as data; read-only allowlist means an injected instruction cannot edit, push or exfiltrate via `Bash`; optional per-repo reviewer allowlist (proposed config); notification text is truncated plain text |
| Minimum privileges for automation (§10) | Separate OS credentials for botPC (D2); never copy mainPC credentials; `ANTHROPIC_API_KEY` never set; `--bare` never used; usage credits OFF; only memory directories synced from `~/.claude` |
| Idempotency, no concurrent PR writes (§10) | Unique event key; session lease; unique open-session index; global single-run slot |
| Traceability (§10) | `runs` and `notifications` tables, `claude logs`, per-run `log_path` |
| What is sent to Anthropic (§6) | Prompts, file contents and command output read by tools, PR diffs and review text, and, while RC is connected, the session transcript (stored on Anthropic servers, retention per its data-usage policy). Inference is always in Anthropic's cloud (requirements §6). Not available under ZDR/HIPAA organizations |
| Credential isolation | `claude auth login` credential used for RC/`--bg`; `setup-token` token only in headless child env (never plist-global, never logged); both single-user, no sharing (max-plan.md §2) |

## 9. Open decisions

### O1. PR handoff storage (needs user decision)

| Option | Pros | Cons |
|---|---|---|
| (a) Hidden `<details>` in the GitHub PR body | Travels with the PR, visible from any device | Leaks design reasoning into a shared repo and to reviewers; size limits; edit churn |
| (b) Git-tracked file in the repo | Versioned, reviewable | Pollutes history; shared repo exposure; needs a commit and push per update |
| (c) Private store on botPC, `~/.varjo/handoff/<owner>/<repo>/<pr>.md`, keyed by PR number + head SHA, synced with the same rsync as memory | Matches §6 confidentiality; no repo noise; reuses the sync mechanism | Only on the user's machines; needs sync discipline; not visible to teammates |

Recommendation: (c). `HandoffStore` is an interface, so (a) or (b) can be added as an opt-in export later.

### O2. Notification channel to mainPC and phone (needs user decision)

| Option | Pros | Cons |
|---|---|---|
| (a) Discord webhook | Already used with Claude Code's Discord plugin; replies could later flow back; phone push | Third-party service sees summaries; webhook URL is a secret |
| (b) ntfy (self-hosted or ntfy.sh private topic) | Simple push to phone and Mac; no account coupling | ntfy.sh topic is a shared secret; self-hosting needs a reachable host |
| (c) macOS Notification Center via SSH | No third party | mainPC only, no phone; needs sshd/private path |
| (d) Email | Universal | Slow, noisy, summaries stored in a mailbox |

Recommendation: (a) Discord first, (b) ntfy as the second adapter; keep summaries free of file contents beyond what the template needs.

### O3. Worktree ownership (needs user decision, decided by experiment #15)

| Option | Pros | Cons |
|---|---|---|
| Claude's `--worktree "#<n>"` | Less code; `claude rm` cleanup for `--bg` sessions | Tip is not refreshed on reuse; `-p` runs leave worktrees; reuse across runs unconfirmed (claude-code.md §5) |
| varjo-managed `git worktree` under a configurable root | Full control of tip, location and cleanup | More code; `claude rm` will not remove it |

Recommendation: default to `ClaudeManaged` if Phase 0 shows a second `--worktree "#<n>"` run reuses the worktree and varjo can fast-forward it; otherwise `VarjoManaged`. Note that #15 as written covers CLAUDE.md/rules in worktrees, so the reuse check needs adding to it (or to #10).

## 10. What Phase 0 must confirm before Phase 1

- #7: lid-closed stability, sleep prevention (RC needs the Mac awake and the process running, remote-access.md §B).
- #8: dedicated login on botPC works; `setup-token` + `CLAUDE_CODE_OAUTH_TOKEN` works in launchd with `ANTHROPIC_API_KEY` unset and no `--bare` (claude-code.md §18.8); `claude auth status` output shape for the login assertion; the two credentials do not clobber each other (max-plan.md open questions).
- #9: Desktop SSH fallback connects over the private path; sign-in expiry when mainPC sleeps.
- #10: `claude -p --resume` keeps context across cwds (§18.2); exit codes and JSON for usage limit, expired login, bad token, `--max-turns` (§18.3); `rate_limit_event` in plain `json` mode; time zone of "resets".
- #11 (top priority): decides M1 vs M2. Does `--bg --remote-control` work; how to read the claude.ai/code URL; does `--resume <id> --remote-control` re-attach a `-p` session; is a `--bg` session visible via RC (claude-code.md §18.6); does Desktop on a second Mac list it.
- #12: phone drives an RC session (messages, permission prompts, push); resolves the M2 phone gap in flow (c).
- #13: Dispatch confirmation only (out of scope by decision).
- #14: memory sync viability: identical-path keys, `autoMemoryDirectory` per-repo values, symlink behavior, external edits during a live session, sync direction; whether `claude purge` of a worktree path leaves memory intact.
- #15 (plus worktree reuse): CLAUDE.md and rules in worktrees; second `--worktree "#<n>"` run reuses the worktree; `claude rm` on `-p` worktrees; pushing fixes from a `#<n>` worktree (decides O3).
- Hooks: `PostToolUse` with `if: "Bash(gh pr create *)"` firing in `-p` with `session_id`/`cwd`; Stop/SessionEnd/Notification stdin JSON (claude-code.md §18.9-10); `--from-pr` and `-p` sessions (§18.5).
- Policy: re-check whether the paused programmatic-usage split for `claude -p` is revived (max-plan.md §1); whether "ordinary, individual usage" covers a persistent poller (max-plan.md §2).
- #16 (security baseline): confirm the allowlist is sufficient for a useful summary without `gh api`; reviewer allowlist default.

## 11. Repository layout (proposed)

```
varjo/
  src/
    adapters/github/   # EventSource: gh/API polling, normalization, PR state
    store/             # SQLite schema, migrations, leases, dedup queries
    launcher/          # SessionLauncher, BgRcLauncher (M1), HeadlessTmuxLauncher (M2)
    worktree/          # ClaudeManaged and VarjoManaged strategies
    notifier/          # Notifier interface, discord/, ntfy/, ...
    limits/            # usage-limit detection, reset parsing, blocked_until
    cleanup/           # PR-closed policy: claude rm/purge, git worktree remove
    cli/               # varjo poll | status | open <pr> | cleanup
  hooks/               # pr-handoff.sh (PostToolUse), stop hook shim
  launchd/             # com.varjo.poll.plist, com.varjo.recover.plist, com.varjo.memsync.plist
  scripts/             # memsync.sh, install helpers
  docs/                # requirements.md, research/, design/
```
