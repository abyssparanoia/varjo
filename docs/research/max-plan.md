# Research: Running varjo unattended on a Claude Max subscription

Issue #3. Fetched 2026-11-01 (local CLI 2.1.306). Items marked **unconfirmed** were not verified from an official page.
Note: pages are moving fast; several support articles were edited in Jun/Oct 2026.

## Sources

- Claude Code docs (code.claude.com/docs/en/): `headless`, `authentication`, `legal-and-compliance`, `costs`, `errors`,
  `interactive-mode` (section "Wait for a usage limit to reset"), `model-config`, `statusline`, `routines`,
  `remote-control`, `github-actions`, `agent-sdk/overview`, `agent-sdk/typescript`
- https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan
- https://support.claude.com/en/articles/11049741-what-is-the-max-plan
- https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans (now "Manage usage credits")
- https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
- https://support.claude.com/en/articles/17154008-monthly-api-credits-for-max-ultra-and-team-plans
- https://www.anthropic.com/legal/consumer-terms (effective Oct 8, 2025)
- GitHub issues anthropics/claude-code: #103807, #79500, #103820, #104690, #91857, #88583, #103253 (all OPEN at fetch time)

## 1. What Max covers, by execution mode

Plan naming drift: the "What is the Max plan?" article now describes a single Max tier ($100/mo, 5x Pro) and says
the former 20x tier is now called **Ultra**. Docs elsewhere say "Pro, Max, Ultra". Treat "5x/20x" as "Max/Ultra".

| Mode | Status | Evidence |
|---|---|---|
| Interactive CLI / IDE | Subscription-covered | support 11145838 |
| `claude -p` signed in with plan | Subscription-covered, draws plan limits | support 15036540: "the Claude Agent SDK and `claude -p` still draw from your subscription limits" |
| Agent SDK (signed in) | Same as above, but docs steer developers/products to API keys (see section 2) | legal-and-compliance |
| `claude --bare -p` | NOT subscription: bare mode never reads OAuth/keychain; needs `ANTHROPIC_API_KEY` | headless |
| `ANTHROPIC_API_KEY` set in env | Overrides subscription; in `-p` the key is always used when present, billed at API rates | authentication |
| `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`, 1-year) | Subscription-covered; model requests only (no Remote Control, no claude.ai connectors); ignored by `--bare` | authentication |
| Cloud sessions / scheduled tasks (formerly routines) | Subscription usage, same as interactive; research preview; 100 scheduled runs/hr/account; at limit: rejected unless usage credits on | routines |
| Remote Control | Subscription only, API keys unsupported; needs interactive-style local session | remote-control |
| GitHub Actions (`claude-code-action`) | Subscription via `CLAUDE_CODE_OAUTH_TOKEN` is documented; but the Oct 2026 API-credit article lists Action runs as "Claude Code usage" | github-actions, support 17154008 |
| Dispatch | **unconfirmed** (not researched beyond index; Phase 0 task) | - |

Paused change: on 2026-05-13 Anthropic announced a separate programmatic-usage credit for `claude -p`/Agent SDK/Actions starting
2026-06-15. Support 15036540 now says "We've paused the previously-announced changes to Claude Agent SDK usage" (update dated Jun 15 2026).
Oct 7 2026 update: Max/Ultra/Team get monthly **API credits** (Max $100, Ultra $200), claimed by linking a Console org;
they are used only when `claude -p`/SDK runs with an API key from that org. Plan-login runs do not consume them.
Pause could be lifted later (no date given) -> **re-check this page periodically**; it is the main policy risk.

## 2. Terms

- Claude Code license: Consumer Terms apply to Free/Pro/Max/Ultra (code.claude.com legal-and-compliance).
- Same page: "Advertised usage limits for Pro, Max, and Ultra plans assume ordinary, individual usage of Claude Code and the Agent SDK."
- Same page: OAuth "is intended exclusively for purchasers of ... subscription plans and is designed to support ordinary use of Claude Code
  and other native Anthropic applications." Developers "building products or services" (incl. Agent SDK) "should use API key authentication";
  Anthropic does not permit routing requests "through Free, Pro, Max, or Ultra plan credentials on behalf of their users."
- Consumer Terms s.3: no automated access "Except when you are accessing our Services via an Anthropic API Key or where we otherwise explicitly permit it."
  Anthropic's own docs explicitly document `claude -p`, `setup-token` for "CI pipelines and scripts", scheduled tasks, so unattended
  Claude Code CLI use for your own work is explicitly permitted in practice. **Not a legal opinion; the wording of "ordinary, individual usage" is vague.**
- Account sharing: Consumer Terms s.2 forbids sharing login/credentials with anyone else. Personal bot on own hardware = not sharing.
  varjo must stay single-user (no serving other people's requests with this login).
- Same account on two machines: no prohibition found in the fetched pages (`CLAUDE_CONFIG_DIR` doc even shows multi-account use). Technical risk
  instead: #88583 (open) reports the single-use OAuth refresh token racing between concurrent sessions and blanking the Keychain credential
  (observed with Desktop sessions). Mitigation: botPC uses its own `claude setup-token` token or its own login, not a copied credential.
- No explicit "bots"/"background automation" ban beyond the s.3 clause above was found. Enforcement "without prior notice" is reserved.

## 3. Usage limits

- Max: a 5-hour session limit (window starts at first message, resets 5h later), a weekly all-models limit (fixed per-account reset time,
  visible at claude.ai/settings/usage), and a separate weekly limit for Fable models. Opus/Sonnet-specific limit messages also exist.
  Everything in Claude chat + Claude Code counts against the same limits (support 11145838). Limits are per account, so botPC and mainPC
  share one pool; no per-machine split. Advertised numbers are not published; Anthropic may change limits.
- Where surfaced: `/usage` (plan bars, 24h/7d breakdown; local-history based), `/status`, claude.ai/settings/usage, warnings like
  "You've used 85% of your session limit . resets 3:45pm".
- Programmatic remaining quota:
  - Status line JSON stdin has `rate_limits.five_hour|seven_day.{used_percentage (0-100), resets_at (epoch s)}` for Pro/Max/Ultra only, after the first API
    response in a session; each window may be absent (costs, statusline docs). Usable from a `statusLine` command in a `-p` run: **unconfirmed** whether
    it fires in `-p`.
  - Stream-JSON `rate_limit_event` (SDK type `SDKRateLimitEvent`): fields observed in #91857: `status` (allowed|allowed_warning|rejected),
    `resetsAt` (epoch), `rateLimitType` (five_hour|seven_day...), `overageStatus`, `isUsingOverage`, and `unifiedWindows.{five_hour,seven_day}.{utilization 0-1,resetsAt}`.
    `unifiedWindows` is undocumented/undeclared in types (may change).
  - No documented zero-token API for quota; `getUsage()` in SDK is marked experimental (#91857). `/usage` is text only.

## 4. Behavior at the limit in `-p`

Documented messages (errors page): `You've hit your session limit . resets 3:45pm`, `... weekly limit . resets Mon 12:00am`,
`... Opus limit`, `... Sonnet limit`; time zone not stated (a bug report shows `resets 11am (UTC)`). Session/weekly are shared across models; Opus/Sonnet limits
can be sidestepped by switching model family.

Mechanics (docs):
- 429 throttles are retried up to `CLAUDE_CODE_MAX_RETRIES` (default 10) with backoff. Genuine usage-limit rejection is not a transient throttle.
- Interactive-only auto-wait (`autoContinueAtUsageLimit`, v2.1.234+): "the menu row isn't available" for `-p` runs.
- `CLAUDE_CODE_RETRY_WATCHDOG=1` (for unattended sessions) keeps retrying 429/529; v2.1.295+ `CLAUDE_CODE_RETRY_WATCHDOG_MAX_WAIT_MS`
  caps the wait; v2.1.305+ `CLAUDE_CODE_RETRY_WATCHDOG_USAGE_LIMIT_RECHECK_MS` (60000..10800000) sets the recheck cadence. It blocks (does not fail fast).
  Known bug #104690 (2.1.309): on a subscription with usage credits OFF, the watchdog treats the limit as a spend cap and exits at once with
  "You've hit your monthly spend limit ... your session limit resets 9:20pm". So with credits off the watchdog does NOT wait (confirm per version).
- Default `-p` (no watchdog) = fail fast; docs state exit 0 on success, non-zero on failure, and in-run failures are printed as the result on stdout.

Observed (bug reports; **treat as unstable**):
- 2.1.304 (#103807): exit 1, JSON `{"type":"result","subtype":"success","terminal_reason":"api_error","is_error":true,"api_error_status":429,"api_error":"usage_limit_reached"}`.
  Note contradictory `subtype:"success"`; key on `is_error`/`api_error`/`api_error_status`, not `subtype`.
- 2.1.49 (#79500): same shape but exit 0 and text "API Error: Rate limit reached" on stdout. Older versions exit 0.
- 2.1.303/304 (#103820): long stream-json run on weekly limit exited 1 with NO rate_limit rejection event, no result; message only in the transcript
  JSONL (`~/.claude/projects/<slug>/<session>.jsonl`, entry with `isApiErrorMessage: true`).
- #103253: exit 1 "No messages returned from query" when a `rate_limit_event` arrives after the result.
- Docs: ensure a missing `structured_output` is checked; SIGTERM exit code is 143.

## 5. Extra usage ("usage credits") and keeping it OFF

- Max/Ultra: claude.ai > Settings > Usage > "Usage credits": Enable/disable, monthly spend limit ("Adjust limit" or unlimited), prepaid funds, optional auto-reload.
  Off means only plan-included usage; at the limit, requests stop (support 12429409). `/usage-credits` in the CLI opens that page; `/usage` shows a credits row while on.
- Some models (Fable; Sonnet 4.6 1M context) can bill to usage credits instead of included limits ("Requires usage credits" in the picker) -> do not select them.
- Hard guarantees to give the bot no way to pay: (a) keep usage credits OFF (verify in settings; it is per account, not per machine);
  (b) never set `ANTHROPIC_API_KEY`/`ANTHROPIC_AUTH_TOKEN`/`apiKeyHelper`/Bedrock-Vertex vars on botPC (an API key silently wins in `-p`);
  (c) do not claim the monthly API credits / link a Console org (that is a separate $100 pool that applies only to API-key runs; harmless but avoid mixing);
  (d) no Console payment method / auto-reload; (e) have the bot assert at start-up that `/status` (or `system/init`) shows a subscription login. **Unconfirmed**: exact
  `claude auth status` JSON for asserting login method; check in Phase 0.
- `--max-budget-usd` and `total_cost_usd` are client-side estimates, not billing under a subscription.

## 6. Stretching quota

- `--model haiku|sonnet|opus`, or full IDs. `default` on Max resolves to Opus 5.5; Fable never default and may bill credits. Use `--model sonnet` (or `haiku`) explicitly.
- Effort: `--effort low|medium|high|xhigh|max` or `CLAUDE_CODE_EFFORT_LEVEL`; Opus 5.5/Sonnet 5.5/Haiku 5.5 default `medium`. Thinking cannot be disabled on 5.5 models;
  lower effort instead. `MAX_THINKING_TOKENS` only affects fixed-budget models.
- Reduce overhead: `--bare` is not an option (needs API key). Instead keep the working dir minimal, restrict tools (`--allowedTools`), trim CLAUDE.md, no MCP servers/skills,
  pipe the PR diff/comments in on stdin (10 MB cap) rather than letting Claude explore, `--max-turns`, `--permission-prompts none` (v2.1.259+), `/clear`-style fresh sessions per run
  (no `--continue`) so context stays small and cache re-read cost does not accrue.
- Prompt suggestions are skipped near the limit; idle compaction / session recap use tokens while idle (settings `idleCompaction`).

## Proposed limit-handling behavior for varjo

1. Preflight before each run: if `state.blocked_until` (persisted, e.g. SQLite) is in the future, skip and log; do not spawn `claude`.
2. Run `claude -p --output-format json --model sonnet --effort low ...` (stream-json if the `rate_limit_event` is wanted) with a hard wall-clock timeout, **no**
   `CLAUDE_CODE_RETRY_WATCHDOG`, single concurrent run, `ANTHROPIC_API_KEY` unset.
3. Detect "limit reached" with OR over: JSON `api_error == "usage_limit_reached"`; `api_error_status == 429`; `is_error:true` and result text matching
   `/hit your (session|weekly|Opus|Sonnet) limit/` or `Usage limit reached`; any `rate_limit_event.status == "rejected"`. Ignore `subtype`. Also treat a non-zero exit with
   empty stdout as "unknown failure", then check the transcript JSONL for `isApiErrorMessage` before classifying (#103820).
4. Parse reset: epoch `resetsAt` from `rate_limit_event` if present, else parse "resets 3:45pm"/"Mon 12:00am" (zone unspecified; assume the botPC local zone and
   verify). Set `blocked_until = reset + 2-5 min jitter`. If unparsable: exponential fallback 30 min -> 1 h -> 2 h, capped at the 5 h window (weekly: only a parsed time, else 6 h steps).
5. Weekly vs session: `rateLimitType`/message distinguishes them; on weekly, `blocked_until` may be days away -> notify once and stay idle (also pause polling GitHub if cheap).
6. Never retry inside the same poll cycle. Failed work item stays queued (idempotent, requirements s.10); retry only after `blocked_until`. At most 1 probe per window after the reset.
7. Notify once per blocked window (Discord/Notion/etc.), including reset time and queue depth; send a "resumed" message after the first success. Do not notify on every poll.
8. Pre-emptive guard: when a `rate_limit_event`/statusline shows `five_hour` utilization >= ~0.9, stop starting new jobs until `resetsAt`.
9. Other failures (401/expired login, 403 `oauth_org_not_allowed`, 5xx/529) are a different class: auth -> alert and stop; transient -> bounded backoff (max 3, >= 1 min apart).
10. Subscribe to warnings in logs (85% message) for observability; record every run's `total_cost_usd`/duration as a trend signal only.

## Open questions for hands-on validation (Phase 0)

- Actual exit code, stdout shape and exact text of a limit hit on 2.1.306 with `--output-format json` and `stream-json` (cannot be forced cheaply; consider a mock `ANTHROPIC_BASE_URL` returning 429 with `anthropic-ratelimit-unified-*` headers as in #104690).
- Whether `rate_limit_event` arrives in plain `json` mode, and whether a `statusLine` hook runs under `-p`.
- Time zone of "resets" text; whether a `setup-token` token and an interactive login on the same account refresh independently without clobbering.
- Whether the programmatic-usage split (paused) is revived; whether "ordinary, individual usage" is read to include a persistent scheduled poller.
- Dispatch and Desktop-over-SSH quota treatment (not covered here).
