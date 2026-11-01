# Claude Code capabilities relevant to varjo

## Sources and versions

- Fetched 2026-11-01. Local CLI: `claude --version` = 2.1.306 (`claude --help` read locally).
- Docs (live fetch):
  - Sessions: https://code.claude.com/docs/en/sessions.md
  - Headless: https://code.claude.com/docs/en/headless.md
  - CLI reference: https://code.claude.com/docs/en/cli-reference.md
  - Hooks: https://code.claude.com/docs/en/hooks.md
  - Skills: https://code.claude.com/docs/en/skills.md
  - Worktrees: https://code.claude.com/docs/en/worktrees.md
  - Memory: https://code.claude.com/docs/en/memory.md
  - Background sessions: https://code.claude.com/docs/en/agent-view.md
  - Authentication: https://code.claude.com/docs/en/authentication.md
  - Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview.md
  - Remote Control: https://code.claude.com/docs/en/remote-control.md
  - Cloud sessions: https://code.claude.com/docs/en/claude-code-on-the-web.md
  - Scheduled tasks (routines): https://code.claude.com/docs/en/routines.md
  - GitHub Actions: https://code.claude.com/docs/en/github-actions.md
- Not read: desktop.md, permission-modes.md, errors.md, code-review.md, Dispatch. Dependent items are marked "unconfirmed".
- Fetch truncation: hook per-event JSON for SessionEnd/Stop/Notification was not fully read.

## 1. Session lifecycle

- Storage: `~/.claude/projects/<project>/<session-id>.jsonl`. `<project>` = cwd with non-alphanumerics replaced by `-` (truncated and hashed past 200 chars). Config root: `CLAUDE_CONFIG_DIR`. Project dir name: `CLAUDE_CODE_PROJECT_DIR_NAME` (needs `CLAUDE_CONFIG_DIR`, v2.1.234+).
- Session IDs are UUIDs. `--session-id <uuid>` pins one at creation.
- Retention: `cleanupPeriodDays` (default 30 days per sessions page).
- Transcript format is internal and may change; use `--output-format json`, `/export`, or hook `transcript_path`.
- Resume:
  - `claude -c` / `--continue`: most recent conversation in cwd. Skips `-p`/SDK sessions and `/loop` sessions. `claude -p --continue` includes them.
  - `claude -r <id|name>` / `--resume <id|name>`: works from any directory. Also accepts an absolute `.jsonl` path. Bare `--resume` opens the picker.
  - ID lookup: current project and its worktrees, then other projects (only if exactly one matches).
  - `--fork-session`: new ID on resume (copy of history).
  - `-n/--name <name>` or `/rename`: resume with `--resume <name>`. Default display names (`my-app-3f`) are NOT resume handles.
- `claude -p --resume <id> "prompt" --output-format json`: documented follow-up pattern into an existing session.
- Resume restores history, model, agent, and terminal permission mode. It does NOT restore `--mcp-config`, `--settings`, `--plugin-dir`, `--fallback-model`, `--add-dir`; pass them again.
- Cross-machine by copying files: NOT documented. Path encoding means the file must go under the matching `projects/<encoded-cwd>/` on botPC, or be resumed by absolute path. Unconfirmed.

Varjo relevance: store the UUID from `--session-id` (or `session_id` in JSON output) per repo+PR. Use `--resume <id>`, not `--continue` (ambiguous across PRs).

Open: copy-between-machines resume; whether `--name` stays unique enough to be a fallback handle.

## 2. Headless mode

- `claude -p "<prompt>"`. Exit 0 on success, non-zero on failure. Failures inside a run are printed as the result on stdout. SIGTERM exits 143. Invalid flags fail before the run.
- `-p` skips the trust dialog; project hooks and `.mcp.json` servers still run unless `--bare`.
- `--bare`: skips hooks, plugins, MCP (except `--mcp-config`), auto memory, CLAUDE.md discovery. Auth is only `ANTHROPIC_API_KEY` or `apiKeyHelper`; OAuth and keychain are never read. Not usable with a Max login.
- Flags (exact):
  - `--output-format text|json|stream-json`. `json` fields include `result`, `session_id`, `total_cost_usd`, `num_turns`, `structured_output`.
  - `--input-format text|stream-json`
  - `--max-turns <n>`: exits with error when reached.
  - `--max-budget-usd <amount>`: client-side estimate.
  - `--allowedTools` / `--allowed-tools`, `--disallowedTools`, `--tools`
  - `--permission-mode acceptEdits|auto|bypassPermissions|manual|dontAsk|plan`; `--permission-prompts host|none`; `--permission-prompt-tool mcp__<server>__<tool>`
  - `--dangerously-skip-permissions` (= bypassPermissions). Docs say only for sandboxes without internet.
  - `--json-schema '<schema>'`: validated output in `structured_output`. Absent if run ends without valid output; check before use.
  - `--no-session-persistence`: not resumable.
  - `--exclude-dynamic-system-prompt-sections`: moves cwd, env, memory paths, git status into the first user message (cache reuse). Ignored with `--system-prompt`.
- Usage-limit and auth error exit codes: unconfirmed. `api_retry` error categories (stream-json): `authentication_failed`, `billing_error`, `rate_limit`, `overloaded`, `invalid_request`, `model_not_found`, `server_error`, `max_output_tokens`, `unknown`, and others.
- `ANTHROPIC_API_KEY`, when present, is always used in `-p` and overrides subscription login.

Varjo relevance: review worker = `claude -p --resume <id> --output-format json --json-schema ... --permission-mode dontAsk --permission-prompts none --allowedTools ...`. Avoid default Manual mode in `-p` (since v2.1.302 it denies unapproved calls).

Open: exit codes for usage limit and auth errors; whether `--max-turns` exit still prints `session_id`.

## 3. Hooks

- 33 events: `SessionStart`, `Setup`, `SessionEnd`, `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `StopFailure`, `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `Notification`, `MessageDisplay`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PreCompact`, `PostCompact`, `PreModelSwitch`, `PostModelSwitch`, `Elicitation`, `ElicitationResult`.
- Matchers:
  - `SessionStart`: `startup`, `resume`, `clear`, `compact`, `fork` (input `source`).
  - `SessionEnd`: `clear`, `resume`, `logout`, `prompt_input_exit`, `other`. Budget 1.5 s default, up to 60 s.
  - `Notification`: `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_*`, `agent_needs_input`, `agent_completed`, `quota_auto_resume_*`, `plugin_notification`.
  - `Stop`: no matcher. Input has `last_assistant_message`. Exit 2 or `decision: "block"` keeps Claude working.
- Common input: `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `permission_mode`, `prompt_id`, `scratchpad_dir`.
- Headless: hooks run in `-p` unless `--bare`.
- Git/PR: no commit, push, or PR event. Workaround: `PreToolUse`/`PostToolUse` on `Bash` with `if: "Bash(gh pr create *)"`.
- Defined in: settings files, plugins (`hooks/hooks.json`), skill frontmatter, subagent frontmatter.

Varjo relevance: `Stop.last_assistant_message` and `SessionEnd` drive state updates and mainPC notification. `PostToolUse` + `gh pr create` `if` records the PR URL for the PR↔session map.

Open: exact SessionStart/SessionEnd/Stop/Notification fields; whether `if` filters fire in `-p`.

## 4. Skills and slash commands

- Locations: `~/.claude/skills/<name>/SKILL.md`; `.claude/skills/<name>/SKILL.md` (repo); plugin `skills/<name>/SKILL.md` (invoked `/plugin:skill`).
- Frontmatter: `name`, `description`, `when_to_use`, `argument-hint`, `arguments`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `disallowed-tools`, `context: fork`, `agent`, `hooks`, `model`, `effort`, `background`, `paths`, `shell`.
- Args: `$ARGUMENTS`, `$ARGUMENTS[N]` / `$N`, named via `arguments`.
- `claude -p "/skill-name args"` works. Only the first skill on the line loads.
- Terminal-only built-ins (`/login`) are unavailable in `-p`.
- No skill or hook runs automatically at PR creation. Use a `PostToolUse` hook or a wrapper that calls `claude -p "/skill ..."` after `gh pr create`.

Varjo relevance: package the review-summary prompt, handoff template (requirement 5.4), and resume recipe as repo skills.

Open: `--json-schema` combined with a skill-driven `-p` run.

## 5. Git worktrees

- `claude -w <name>` / `--worktree <name>`: creates `<repo>/.claude/worktrees/<name>/` on branch `worktree-<name>`. Base is default branch (`worktree.baseRef` = `"fresh"`, or `"head"`). Name omitted: generated.
- `claude --worktree "#1234"` (or PR/MR URL): fetches `pull/1234/head` from origin, creates `.claude/worktrees/pr-1234`. Quote `#`. Fails non-interactively if git/ssh would prompt.
- `-w` with `-p` skips the trust check.
- `--tmux` (requires `--worktree`): tmux session for the worktree.
- Reusing a name reopens the worktree. It resets to the default branch only when clean, on its own branch, with no unique commits (or merged PR). Otherwise it keeps the old tip. PR-reference names always keep the old tip.
- Tools: `EnterWorktree`, `ExitWorktree`.
- Isolation checks in a worktree session block edits and git commands that target the main checkout.
- `.worktreeinclude` (gitignore syntax) copies gitignored files such as `.env` into new worktrees.
- Cleanup:
  - Interactive exit: clean unnamed worktrees removed; dirty or named ones prompt.
  - `-p` runs: no cleanup; lock stays until a later sweep. Use `git worktree remove` (`git worktree unlock` if locked).
  - `claude rm <id>`: removes a background session's worktree when safe. Refuses on unpushed commits; keeps dirty worktrees; leaves worktrees you created with `git worktree add`.
- Config resolution inside a worktree:
  - CLAUDE.md and `.claude/rules/` at the worktree root load from the worktree. Exception: subagents in the default `.claude/worktrees/` location do not load them.
  - `.claude/settings.json` comes from the worktree checkout. `${CLAUDE_PROJECT_DIR}` in hooks stays on the main checkout; the hook `cwd` field follows the session.
  - "Don't ask again" approvals are saved to the main checkout's `.claude/settings.local.json` and apply to all worktrees.
  - Project plugins from the main checkout load in worktrees.
  - If the worktree has no `.claude/skills`, `agents`, or `commands` directory, the main checkout's copies load.
  - Auto memory is per repo, shared across worktrees.
- Non-interactive resume into a worktree (v2.1.212+). A refusal produces stream-json `result` with subtype `error_during_execution` and `startup_failure_reason` `worktree_resume_refused` or `worktree_unverified`.

Varjo relevance: requirement 3.3 step 4 maps to `claude --worktree "#<n>"`. Varjo still needs a stale-worktree sweeper for `-p` runs.

Open: whether a second `--worktree "#<n>"` run reuses the same worktree and its state; whether `claude rm` can clean a worktree created by a `-p` run.

## 6. Auto Memory (overview)

- Per repository, shared across worktrees. First 200 lines or 25 KB loaded each session.
- Location: `<config>/projects/<project>/memory/` (confirmed by the sessions page example).
- Enable/disable setting name: unconfirmed (memory page not fully read). `--bare` skips it.
- Details belong to the separate memory research agent.

## 7. Agent SDK vs CLI

- Packages: Python `claude-agent-sdk`; TypeScript `@anthropic-ai/claude-agent-sdk`.
- Quickstart uses API key auth. The overview says Anthropic "does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK." That targets products offered to others. Whether a personal self-run tool is covered is unconfirmed.
- `--bare` docs say it needs `ANTHROPIC_API_KEY`, implying non-bare `claude -p` uses subscription login.
- Recommendation: for a Max-plan orchestrator, drive the CLI (`claude -p`, `--bg`, `-w`) from launchd-managed scripts. Use the SDK only if you need in-process callbacks, and then check the ToS and plan on an API key.
- Managed Agents is a separate Anthropic-hosted product, not relevant here.

## 8. Remote-related inventory

- `claude remote-control` (server mode): many sessions, `--spawn same-dir|worktree|session`, `--capacity` (default 32), `--name`, `--continue`, `--session-id`, `--permission-mode`, `--create-session-in-dir`. Doc: https://code.claude.com/docs/en/remote-control.md
- `claude --remote-control [name]` / `--rc`, or `/remote-control` / `/rc` inside a session.
- `remoteControlAtStartup` setting for auto-connect.
- Requirements: claude.ai login on Pro/Max/Ultra/Team/Enterprise. API keys unsupported. Not available with a non-Anthropic `ANTHROPIC_BASE_URL`. Disabled by `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` or `DISABLE_GROWTHBOOK`.
- The computer must stay awake and the `claude` process must keep running. Reconnects after sleep or network drop.
- `setup-token` tokens cannot establish Remote Control sessions.
- Dispatch: deep dive is for the separate agent. Background sessions in section 10 are the local foundation.
- `claude --teleport [session]` / `/teleport`: pull a cloud session to the terminal.
- `claude --cloud "<task>"`, `claude -p "<msg>" --cloud <id>`: cloud sessions. `--environment <ccpool_...>` for self-hosted. `--remote` is a deprecated alias.
- Routines: see section 15.
- `claude --desktop [--continue|--resume <id>]`: opens Desktop on a session.

## 9. GitHub integration

- `/install-github-app`: github.com only; needs `gh` authenticated. Installs the Claude GitHub App, sets a secret (`ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`), and opens a PR adding `.github/workflows/claude.yml` (optional `claude-code-review.yml`).
- `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) works on Pro/Max/Ultra/Team/Enterprise, so the Action can run on a subscription.
- `anthropics/claude-code-action@v1`: `@claude` mentions, or automation mode with a `prompt`. Runs on GitHub-hosted runners, not botPC.
- `/autofix-pr`: cloud session watching CI and comments. Cloud, not local.
- Code Review plan gating: unconfirmed (code-review.md not read).
- Varjo relevance: low. Varjo polls with `gh` on botPC. The Action would be a separate execution path with no shared local state.

## 10. Background sessions (requirement a)

- Start: `claude --bg "<prompt>"` / `--background` (prints ID; rejected with `-p`). In agent view, or `/background` / `/bg`.
- `claude --bg --resume <id>`: continues under the same ID, or starts a copy with a `note:` line if already running (v2.1.257+).
- `claude --bg --exec '<cmd>'`: shell job instead of a session.
- Subcommands:
  - `claude agents` (`--cwd`, `--json`, `--all`).
  - `claude attach <id|name>` (name works while running; v2.1.290+).
  - `claude logs <id|name>`.
  - `claude stop <id>` (alias `kill`). Conversation kept.
  - `claude respawn [id]` / `--all`: restart on the current version; resumes saved conversation.
  - `claude rm <id>`: removes session list entry and worktree when safe. Transcript stays resumable.
- Lifecycle:
  - A supervisor runs sessions. It starts on first background or agent view. Closing the terminal does not stop sessions.
  - Sleep is preserved.
  - Reboot or shutdown stops running sessions. They show `Needs input`, `failed` (recent progress), or `stopped`. Attach or reply restarts from the saved conversation.
  - On macOS/Linux, sessions stop if no supervisor reconnects within about a minute. Run `claude agents` to start one. launchd is not mentioned.
  - Idle sessions unattached about an hour are stopped; `Ctrl+T` pins.
- Resume from another terminal: `claude --resume <id> --bg` or `claude attach <id>`. A live session in another process is refused. Plain `--resume` on a live session is not stated.
- Permissions: start like a new session in that directory, or `claude agents --permission-mode`. Persist across supervisor restart.
- File isolation: edits happen in a worktree under `.claude/worktrees/`. Disable with `worktree.bgIsolation: "none"`.
- Expired login stops background sessions from progressing.

Varjo relevance: `--bg` plus `attach`/`logs` gives a native per-PR worker with supervision. Reboot recovery is not provided; varjo needs a launchd job that runs `claude agents` and `respawn`.

Open: whether `--bg` sessions reach mainPC via Remote Control; stability of `agents --json`.

## 11. `--from-pr` and PR linking (requirement c)

- `claude --from-pr [value]`: "Resume a session linked to a PR by PR number/URL, or open interactive picker." Accepts number, GitHub/GHE PR URL, GitLab MR URL, Bitbucket PR URL. "Sessions are linked automatically when Claude creates the pull request."
- Linking is automatic from Claude's own `gh` commands. Agent view shows a `#1234` label when a session edits, comments on, or closes a PR, or when `gh pr checkout` / a push resolves an open PR.
- The docs do not say where the link is stored. Whether `-p`/SDK sessions are included in `--from-pr` is unconfirmed (they are excluded from the picker and `--continue`).
- `-n/--name` does not create the link.
- PRs created outside Claude are not linked.
- `--from-pr` is an interactive picker and resume helper. It is not a stable API.

Varjo relevance: keep varjo's own map (repo, PR, session ID, worktree). Use `--from-pr` only as a human recovery path, after confirming it works for `-p` sessions.

Open: create a PR from a `-p` session, then test `claude --from-pr <n>`, and find where the link is stored.

## 12. `claude setup-token` and unattended auth (requirement d)

- `claude setup-token`: "Set up a long-lived authentication token (requires Claude subscription)." Browser flow prints a one-year OAuth token. Not stored by Claude Code. Set `CLAUDE_CODE_OAUTH_TOKEN`.
- Model requests only: no Remote Control, no claude.ai connectors. Ignored by `--bare`.
- Precedence (highest first):
  1. Cloud provider env vars
  2. `ANTHROPIC_AUTH_TOKEN`
  3. `ANTHROPIC_API_KEY` (always used in `-p` when present)
  4. `apiKeyHelper`
  5. `CLAUDE_CODE_OAUTH_TOKEN`
  6. Anthropic profile / federation
  7. `/login` subscription OAuth (default for Max)
- Storage: `/login` uses macOS Keychain (fallback `~/.claude/.credentials.json` 0600). The setup-token value is stored by you (plist, secrets file, or Keychain).
- Unattended Max runs: documented for CI and scripts where browser login is not available. Caveats:
  - An `ANTHROPIC_API_KEY` in the launchd environment overrides it.
  - `/login` can expire; background sessions and Remote Control stop when it does.
  - If set, `CLAUDE_CODE_OAUTH_TOKEN` also overrides `/login` in new sessions.
  - Do not add `--bare`.
- Unconfirmed: token behavior beyond one year; whether it shares weekly usage limits with interactive use (expected yes).

## 13. `claude purge` and retention (requirement e)

- `claude purge [path]`: deletes "all Claude Code state for a project (transcripts, tasks, file history, config entry)". Options: `--all`, `--dry-run`, `-i/--interactive`, `-y/--yes`.
- `cleanupPeriodDays` (default 30): transcripts older than this are swept. Raise it on botPC for long-running PRs.
- `CLAUDE_CODE_SKIP_PROMPT_HISTORY` and `--no-session-persistence` suppress writes.

## 14. Permission modes and headless prompts (requirement f)

- `--permission-mode` choices: `acceptEdits`, `auto`, `bypassPermissions`, `manual`, `dontAsk`, `plan` (`manual` = `default`).
  - `dontAsk`: denies anything that would prompt. Allowed and no-approval calls still run. `AskUserQuestion`, connector tools set to `ask`, and network-path reads are denied even if allowed.
  - `acceptEdits`: auto-approves edits and common fs commands (`mkdir`, `touch`, `mv`, `cp`).
  - `auto`: classifier decides. Plan eligibility is in permission-modes.md (unread).
  - `plan`: read-only.
- `-p` default: Manual since v2.1.302 when nothing sets a mode.
- `--permission-prompts host|none`: `host` (default) sends prompts to the SDK host or the `--permission-prompt-tool`. `none` denies them, and Claude is told not to retry. Requires v2.1.259+.
- `--permission-prompt-tool mcp__<server>__<tool>`: a varjo approval service could sit here.
- stream-json: denials appear as `permission_denied`; result lists `permission_denials`.
- `--json-schema`: see section 2.

Recommendation: `--permission-mode dontAsk` (or `auto`), `--permission-prompts none`, explicit allowlist (`Read`, `Grep`, `Glob`, `Bash(gh pr view *)`, `Bash(gh api *)`). Keep `Edit`, `Write`, and `git push` out until phase 2.

## 15. Cloud and teleport inventory (requirement g)

- `--cloud [description|session_id|url]`: new cloud session, or with `-p` queues a message into an existing one. Repo cloned from GitHub at current branch; local-only repos are bundled (`CCR_FORCE_BUNDLE=1`).
- `--environment <ccpool_...>`: self-hosted environment.
- `--teleport [session]` / `/teleport` / `/tp` / `/tasks` then `t`: pulls a cloud session local. Requires clean git, same repo, pushed branch, claude.ai login (not API key).
- Routines / scheduled tasks (https://code.claude.com/docs/en/routines.md): research preview. Run on Anthropic cloud (or self-hosted). Triggers: schedule (cron), API (per-task endpoint with bearer token), GitHub event (e.g. `pull_request.opened`). Each run is a new cloud session. No access to botPC files. Created via `/schedule` or claude.ai/code/routines.

## 16. Features that could replace custom varjo components

| Requirement area | Built-in feature | Still needed in varjo |
| --- | --- | --- |
| Session per PR, resume (4.1, 4.2) | `--session-id`, `--resume`, `-p --resume`, JSON `session_id` | State store: repo, PR, session ID, last event ID, status |
| Headless review (3.3 steps 7-8) | `-p --output-format json --json-schema` | Prompt/skill text and schema |
| Worktree per PR (3.3 step 4) | `--worktree "#<n>"`, `.worktreeinclude`, `claude rm`, `git worktree remove` | Sweeper for `-p` worktrees |
| Long-running per-PR worker | `--bg`, `attach`, `logs`, `stop`, `respawn`, `agents` | Reboot recovery via launchd |
| Remote view on mainPC/phone | Remote Control (`remote-control --spawn worktree`, `--rc`), claude.ai/code | Notifier (Stop/Notification hooks) |
| Review-event detection | None local. Routines GitHub trigger is cloud-only. | `gh api` poller with per-PR cursors |
| Unattended Max auth | `setup-token` + `CLAUDE_CODE_OAUTH_TOKEN` | Secure token storage |
| Logs and traceability | `logs`, transcripts, `--debug-file`, stream-json | Varjo event log |
| Context sharing (5.1, 5.4) | CLAUDE.md, rules, skills, auto memory (per repo, shared across worktrees) | Handoff template and generator skill; selective memory sync |
| PR-created handoff (5.4) | `PostToolUse` hook on `gh pr create` | Handoff content |
| Cleanup (requirement 10) | `cleanupPeriodDays`, `claude rm`, `claude purge` | Policy: which PRs are closed |
| Approval gate (phase 2) | `--permission-prompt-tool`, `PermissionRequest` hook, `--permission-prompts none` | Approval UI and relay |

## 17. Contradictions or refinements to docs/requirements.md

1. Req 4.3 / Phase 0 (CLI sessions visible in Desktop on mainPC): Remote Control is the documented phone/web path. Desktop cross-machine visibility is unconfirmed (desktop.md unread).
2. Req 7 (Agent SDK on Max): the SDK docs push API keys and bar claude.ai login for third-party products. Use the CLI for a personal Max setup.
3. Req 5.3 (Auto Memory): memory is per repo and shared across worktrees. This answers part of the open question. Cross-machine sync is still open.
4. Req 5.4 (handoff "at PR creation" by Skill or Hook): no PR-created event exists. Use a `PostToolUse` hook on `gh pr create`.
5. Req 10 (usage-limit graceful stop): no documented exit code for usage limits in `-p`. Unconfirmed.
6. Phase 0 (Max auth): supported for the CLI. `setup-token` for unattended use. `--bare` and `ANTHROPIC_API_KEY` break it.
7. Background sessions do not survive reboot. launchd is still required.

## 18. Open questions for hands-on validation

1. Copy a transcript JSONL to a different Mac under the matching `projects/<encoded-cwd>/`, then `claude --resume <id>`.
2. Do repeated `claude -p --resume <id> --output-format json` calls from different cwds keep full context?
3. Exit codes and JSON for: usage limit, expired login, bad `CLAUDE_CODE_OAUTH_TOKEN`, `--max-turns`, `--max-budget-usd`.
4. Does `--worktree "#<n>"` with `-p` reuse the same worktree on a second run? Can `claude rm` clean a `-p`-created worktree?
5. Does `--from-pr <n>` find a `-p` session that opened a PR via `gh pr create`? Where is the link stored?
6. Is a `--bg` session visible through Remote Control on mainPC or the phone?
7. After reboot, does a launchd job running `claude agents` / `respawn` restore sessions?
8. Does `setup-token` plus `CLAUDE_CODE_OAUTH_TOKEN` work in launchd with `ANTHROPIC_API_KEY` unset, without `--bare`?
9. Full stdin JSON for SessionEnd, Stop, Notification (read remaining hooks.md).
10. Does `PostToolUse` with `if: "Bash(gh pr create *)"` fire in `-p` runs and get `cwd` and `session_id`?
11. Does `auto` permission mode work on this Max account?
12. Is `claude remote-control --spawn worktree` viable given the Mac must stay awake and the process running (lid-closed behavior)?
13. Does the Agent SDK ToS restriction apply to a personal self-run orchestrator? Ask the user or check the terms.
