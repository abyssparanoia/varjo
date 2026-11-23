# Phase 0 validation runbook

This runbook turns the Phase 0 validation issues (#7, #8, #10, #12, #14, #15 and the new #31 to #34) into hands-on experiments that one person can run on real hardware: botPC (the always-on MacBook), mainPC (the everyday MacBook) and a phone. Each experiment maps 1:1 to its issue. The settled direction (#30): T3 Code is the runtime, UI and remote layer, and varjo is reduced to the GitHub event adapter plus glue. The top priority is #32: threads that varjo starts from the outside through T3 Code's MCP server must be first-class in T3 Code's clients. #9, #11 and #13 are superseded and keep only a stub. Sources: `docs/requirements.md` (§2, §4, §6, §9), `docs/research/*.md`, and T3 Code's own docs (repo `pingdotgg/t3code`, branch `main`: `docs/user/providers-claude.md`, `faq.md`, `remote-access.md`, `background-service.md`, `mobile-notifications.md`, `permission-modes.md`, `outside-agents.md`, `source-control.md`, `managed-policy.md`, `docs/internals/overview.md`, `t3-connect.md`, `apps/server/src/scheduledTasks/*`, `packages/contracts/src/scheduledTask.ts`). Scheduled tasks and webhooks have no user-facing doc page yet; their semantics come from source and release notes v0.1.7 to v0.1.10. Anything marked "unconfirmed" is not verified by the research and is what these experiments exist to settle.

This runbook was revised for T3 Code on 2026-11-23 (#30). Round 1 (the `claude` CLI plus Remote Control experiments) keeps its results in the issues and in git history (commit 8bdc33d, issue #11). The `headless.sh` helper below is kept only for the CLI steps in #8 and for the fallback launcher.

## How to use this runbook

- Run one experiment per issue, in the order given below. Do not skip the Setup block: most failures come from an unset env var, an API key left in the shell, or a service environment that differs from the login shell.
- When an experiment ends, paste its filled **Results** template as a comment on that issue, then tick the matching checkbox in #18.
- Never run two CLI sessions against the same session ID at once (the docs say concurrent writers interleave into one transcript). For T3 Code, drive a thread from one client at a time while an experiment records its behaviour, unless the step says otherwise.
- Commands are plain POSIX sh. Placeholders: `<uuid>` session ID (create with `uuidgen | tr 'A-Z' 'a-z'`), `<repo-path>` absolute repo path (identical on both Macs), `<pr-number>`, `<key>` project key (`echo <repo-path> | sed 's/[^A-Za-z0-9]/-/g'`), `<env-url>` the T3 Code environment host as shown in the app, `<thread-id>` a T3 thread id, `<t3-label>` the launchd label of the `t3 up` service (discover it, see #31).
- Where an exact T3 Code command, UI path or MCP call shape is not known from its docs, the step says to discover and record it. Record the exact shape in the Results; do not guess.
- Work files live in `~/varjo-phase0/` on botPC, never inside a repo. Do not paste tokens, pairing links, webhook URLs (they carry a secret token) or transcript contents into issues.

Shared helper, create once on botPC as `~/varjo-phase0/headless.sh` (`chmod +x`). It is used only by the CLI steps in #8 and by the CLI fallback. It encodes the former headless defaults (`dontAsk`, read-only allowlist, 15 turns, Sonnet, low effort):

```sh
#!/bin/sh
# usage: headless.sh --session-id|--resume <uuid> "<prompt>" [extra claude flags]
# The prompt comes before the flags: variadic flags such as --allowedTools swallow a trailing positional prompt.
# stdin is /dev/null to skip the 3 s stdin wait.
mode=$1; id=$2; prompt=$3; shift 3
exec claude -p "$mode" "$id" "$prompt" --output-format json \
  --permission-mode dontAsk --max-turns 15 --model sonnet --effort low \
  --allowedTools "Read,Grep,Glob,Bash(gh pr view *),Bash(gh pr diff *),Bash(git log *),Bash(git diff *)" \
  "$@" < /dev/null
```

## Prerequisites

| Machine | Requirement |
|---|---|
| botPC | macOS version recorded (`sw_vers`); T3 Code, a stable release pinned and recorded (`t3 --version`), installed via the desktop app or `npx t3@latest`, with `t3 up` for the background service; Claude Code CLI still installed (`claude --version`, `claude doctor`) for `claude auth login`; `gh` and `git` installed; Tailscale installed (needed for the #31 Tailscale route and for SSH access); same macOS username and workspace layout as mainPC; plugged into AC; Remote Login (sshd) on only for the #31 SSH fallback and #14 |
| mainPC | T3 Code desktop app (current stable version recorded) or app.t3.codes in a browser; signed in to the T3 Connect account; Claude Code CLI optional (same login as botPC for #8 concurrency); same username and repo paths as botPC |
| Phone | T3 Code iOS/Android app, signed in to the same T3 Connect account; Tailscale app for the #31 Tailscale route |
| Accounts | A T3 Connect account (Tailscale is the alternative route, see #31); the Max account used for `claude auth login` |

Safety checklist (do before any experiment, re-check after #8):

- [ ] `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` unset on botPC in every shell and in `launchctl getenv`; no `apiKeyHelper` or Bedrock/Vertex vars (an API key silently wins in `-p` and in the Agent SDK too).
- [ ] Usage credits OFF at claude.ai > Settings > Usage (per account, so one check covers both Macs). No Console payment method or auto-reload.
- [ ] `claude auth status --text` on botPC shows a subscription login, not an API key.
- [ ] botPC has its own `claude auth login`. Never copy Claude credentials, `~/.claude.json` or the Keychain item between Macs.
- [ ] `t3 --version` recorded with the pinned stable release (T3 Code is pre-1.0, with 1 to 2 stable releases a week plus nightlies; do not upgrade mid-experiment).
- [ ] "Require approval for new devices" is ON in T3 Code on botPC.
- [ ] Held webhooks are OFF unless #33 route 3 is being tested (held webhook bodies are stored on the relay for up to 24 h).
- [ ] varjo-launched threads never run in Full access. Use a read-only or Supervised mode; any run that writes uses Auto-accept edits in a scratch repo only.
- [ ] The MCP OAuth credential (#32) is stored `0600` on botPC only and never synced to mainPC.
- [ ] GitHub: a fine-grained token limited to the target repos (Contents, Pull requests, Metadata; no admin, no workflow). Direct pushes to `main` blocked by branch protection. Use a scratch repo for anything that writes. The one exception is #33 route 3, which needs webhook admin on the scratch repo only.
- [ ] Never pick Fable or "requires usage credits" models; pin Sonnet for varjo-launched threads (the CLI helper pins `--model sonnet`).

## Experiment order and dependencies

1. **#7 and #8 first, in parallel.** Hardware stability and auth gate everything; both start a long soak (48 h and 7 days) that runs while later work proceeds. #8 includes installing T3 Code and `t3 up`, so #31 can start right after #8's first T3 steps.
2. **#31** (reachability and the service) needs T3 Code installed and running from #8. It settles T3 Connect vs Tailscale (O4) and the service label that later steps use. The reboot sub-checks interrupt the #7 soak, so do those after the 48 h are complete or restart the soak clock.
3. **#32** (top priority) needs #8 and #31: the MCP server is reached through the same route. It also yields the usage-limit observability that #8's last T3 item needs.
4. **#34 and #10** need #32's working launch path (and #8's login). #34 decides whether CLAUDE.md, rules, settings, hooks and memory load at all; #10 settles the service environment (PATH, `gh`) and the read-only summary. They can run on the same day.
5. **#33** (trigger routes) needs #10 (`gh` and PATH in the service environment) and #32 (the MCP launch call). Route 3 also needs #31's T3 Connect.
6. **#12 and #15** need #32's MCP-launched thread and #33's PR worktree decisions. #12 also needs #31 (mobile reachability, device approval). #15 reuses the scratch repo and PR from #33.
7. **#14** needs #34's memory-key result and the finished sessions from earlier steps. Same day as #15.
8. **#30 decision** (T3 Connect vs Tailscale, trigger route, Max terms), then the docs revision (see "After Phase 0").

Order in short: #7 #8 (parallel, soak) → #31 → #32 → #34 #10 → #33 → #12 #15 → #14 → #30 decision. Needs soak: #7 (48 h lid closed), #8 (7 days unattended Claude login through the service, plus a natural usage-limit capture if one happens), #31 and #32 (service restart and reboot checks). Close the soak-bound issues only after the soak ends.

## #7 botPC lid-closed stability

**Goal:** botPC stays awake and reachable for 48 h with the lid closed, and returns after power loss.

**Setup:** botPC on AC, no external display, lid closed after step 3. A second device (mainPC) to probe reachability.

**Steps:**
1. Record hardware: `system_profiler SPHardwareDataType | grep -E "Model|Chip|Memory"; sw_vers; pmset -g`.
2. Apply sleep prevention (record which were needed): `sudo pmset -a sleep 0 disksleep 0 displaysleep 0 autorestart 1`; for clamshell without a display also try `sudo pmset -a disablesleep 1`. Alternative: `caffeinate -dimsu` in a launchd job, or Amphetamine. Verify with `pmset -g assertions`.
3. Install a heartbeat LaunchAgent (`~/Library/LaunchAgents/com.varjo.phase0.heartbeat.plist`, `RunAtLoad`, `KeepAlive`) running `sh -c 'while :; do date -u +%FT%TZ >> ~/varjo-phase0/heartbeat.log; sleep 60; done'`. Load: `launchctl bootstrap gui/$(id -u) <plist>`. Enable automatic login so the GUI domain exists after a reboot (note FileVault implications).
4. Close the lid. From mainPC every few hours: `ping -c3 <botpc-host>` and `ssh <botpc-host> uptime` (or Tailscale name). When #31 exists, also confirm the T3 Code environment is still reachable from mainPC.
5. Load test for 30+ minutes with the lid closed: run a real build/test loop in a repo plus a headless Claude run; record `pmset -g therm` and `sudo powermetrics --samplers thermal -n 1` before, during, after.
6. After 24 h change network (move to another Wi-Fi or toggle Wi-Fi) and confirm SSH and the T3 Code connection recover without touching botPC.
7. Pull the power cable until the battery drops a few percent, plug back in; then `sudo shutdown -r now` and confirm login, heartbeat resume and `autorestart` behaviour. Optionally a real power cut.
8. After 48 h: count heartbeat lines (`wc -l ~/varjo-phase0/heartbeat.log`, expect about 2880) and list gaps over 2 minutes: `awk '{cmd="date -j -u -f %Y-%m-%dT%H:%M:%SZ "$1" +%s"; cmd | getline t; close(cmd); if (p && t-p>120) print $1, t-p; p=t}' ~/varjo-phase0/heartbeat.log`.

**Expected:** no gaps beyond the deliberate reboot; reachable after wake and network change; thermals steady without throttling.

**Pass:** 48 h uptime, lid closed, no unexpected sleep, remote access works throughout, config documented in `docs/setup/botpc.md`. **Partial:** works only with an extra tool (Amphetamine, `disablesleep`) or with intermittent wake gaps under 5 minutes. **Fail:** sleeps with lid closed despite all settings.

**If it fails:** use a cheap HDMI dummy plug or a real external display to force clamshell mode, or keep the lid ajar; re-evaluate hardware (requirements §2.1).

**Results:**
```markdown
Date: | Hardware model: | botPC macOS: | Claude Code: | T3 Code (`t3 --version`):
Outcome: Pass / Partial / Fail
pmset settings / tools used:
Uptime achieved (h), unexpected gaps:
Thermals (idle / load, throttling?):
Reboot + power-loss recovery (heartbeat resumed? autologin?):
Network change / 24 h reachability:
Errors (exact text):
Screenshots:
```

## #8 Claude on botPC with the Max subscription (CLI and T3 Code)

**Result (2026-11-11, CLI, Claude Code 2.1.306, round 1):** Pass for the same-day block; see the results comment on issue #8. Case table: (a) normal exit 0, `is_error:false`; (b) invalid `CLAUDE_CODE_OAUTH_TOKEN` exit 1, `is_error:true`, `subtype:"success"` (useless), text "Failed to authenticate. API Error: 401 Invalid bearer token"; (d) `--max-turns 1` exit 1, `subtype:"error_max_turns"`, `session_id` present; (c) usage limit not hit. launchd with no TTY worked with both the Keychain login and `setup-token`. launchd's default PATH (`/usr/bin:/bin:/usr/sbin:/sbin`) lacks `claude`, `gh`, `node`, `go`, `docker`; `gh` found its keyring token without `GH_TOKEN`. Guard rule: classify on exit code, `is_error` and result text, never on `subtype`. Still open from round 1: natural limit capture and mainPC plus botPC concurrency (both repeated below for T3 Code). `setup-token` and the CLI launchd job are no longer needed unless the CLI fallback launcher is built, so the 7-day CLI soak is dropped.

**Goal:** botPC authenticates on its own; T3 Code picks up that login and runs threads on the subscription with no API key, including from the `t3 up` service; the exact failure shapes are captured for limit handling (`docs/research/max-plan.md`); subscription use through the Agent SDK is checked against the Max terms (#30 decision 3).

**Setup:** `env | grep -E 'ANTHROPIC|CLAUDE_CODE_OAUTH'` empty. Create `~/varjo-phase0/headless.sh` (CLI part only). `mkdir -p ~/.config/varjo`. T3 Code installed and pinned (see Prerequisites).

**Part 1: CLI steps (recorded in round 1; rerun only if the CLI fallback is built)**
1. Login: `claude auth login` (on botPC itself, or over SSH from mainPC with the URL/code flow; record whether a browser on botPC was needed). Then `claude auth status --json` and save the JSON shape.
2. Interactive run: `claude` in a scratch dir, ask a one-line question, `/status`, `/usage`; exit.
3. (a) normal: `cd <repo-path>; ~/varjo-phase0/headless.sh --session-id $(uuidgen | tr 'A-Z' 'a-z') "Say ok" > a.json 2> a.err; echo "exit=$?"`. Record `is_error`, `subtype`, `api_error`, `session_id`.
4. (b) invalid token: `CLAUDE_CODE_OAUTH_TOKEN=bad ~/varjo-phase0/headless.sh --session-id $(uuidgen | tr 'A-Z' 'a-z') "Say ok" > b.json 2> b.err; echo "exit=$?"`. Record exit code and JSON.
5. (d) turn exhaustion: `... "Run git log -1 and summarize" --max-turns 1`. Record exit code, JSON, and whether `session_id` is still present.
6. (c) usage limit: do not burn quota. If a limit is hit naturally during any experiment, save the full stdout JSON, stderr and exit code verbatim.
7. `claude setup-token` and the launchd headless job (`com.varjo.phase0.headless`) are fallback-only; their results stay in the issue.

**Part 2: T3 Code steps (re-scoped under #30)**
1. Confirm the login is in place (`claude auth status --text` shows a subscription login) and `t3 --version` is recorded. In T3 Code on botPC, open the Claude provider settings (`docs/user/providers-claude.md`). Record whether it picks up the existing `claude auth login` credential with no prompt and no API key.
2. In a scratch repo registered as a T3 Code project, start a thread (Supervised, Sonnet) with "Say ok". Confirm it completes on the subscription (no API key, usage credits stay off), and record the thread status values you see. Compare the 5-hour usage window with the CLI's `/usage` before and after (the T3 FAQ says it is the same limit since v0.1.7).
3. Service: run `t3 up` (`docs/user/background-service.md`). Discover and record the launchd label and plist path (`launchctl list | grep -i t3`; `ls ~/Library/LaunchAgents`). Close every terminal and the desktop app, then start a thread from mainPC or the phone and confirm the Claude login works from the service with no interactive terminal. Record the service's environment (`launchctl print gui/$(id -u)/<t3-label>`, structure only) and the exact way to set env for it.
4. Concurrency: use the `claude` CLI on mainPC (same account) and run a thread on botPC at the same time, prompts in both; afterwards `claude auth status` on mainPC and a new thread on botPC; confirm neither asks to log in again.
5. Soak: keep the service running for 7 days. Once a day start one small thread (by hand from the phone, or a T3 scheduled task if the UI exposes one; record which) and note the day of any re-login prompt.
6. Usage-limit signals: do not burn quota. If a limit hits naturally, record what the thread shows (state, error text, push notification) and, once #32 exists, what `t3_thread_read` / `t3_thread_wait` return for that thread (exact fields), because the varjo guard has to classify from them. If none hits during the soak, record "not hit" and re-check #32 step 7 output for a failed or stopped thread.
7. Max terms (#30 decision 3): read the current Anthropic terms and Agent SDK documentation on subscription use, and the T3 FAQ passage on Max. Record the quoted sentences, URLs and date. This is the user's decision in #30; the step only collects the wording.

**Expected:** T3 Code reuses the CLI login; the service runs threads without a terminal; no re-login for 7 days; limit shapes appear as a failed or stalled thread with the limit text (unconfirmed).

**Pass:** a thread runs end to end on the subscription from the launchd-run service with no terminal, login survives 7 days, concurrency does not invalidate either login, limit observability recorded (or "not hit" with a plan). **Partial:** works only after an extra step on botPC (a one-time login inside T3 Code, an env var in the service), limit shape not captured, or the Max terms unclear. **Fail:** T3 Code needs an API key or a browser on every start, or refresh breaks either machine.

**If it fails:** if T3 Code cannot use the login, check the service environment first; if the Agent SDK path is not allowed under the Max terms, return to the CLI launcher in git history (commit 8bdc33d, issue #11), which needs `setup-token` for launchd. Do not fall back to an API key.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code: | T3 Code (`t3 --version`):
Outcome: Pass / Partial / Fail
T3 Code picked up `claude auth login` without prompt? Provider settings path:
`t3 up` label / plist path / how env is set for the service:
Thread from the service with no terminal (status values, duration):
Concurrent mainPC CLI + botPC thread ok?:
7-day soak log (day of any re-login):
| Case | where observed | state / error text | via t3_thread_read / t3_thread_wait? | notes |
| usage limit (natural, or "not hit") | | | | |
Max terms wording (quote, URL, date):
Errors (exact text):
Screenshots:
```

## #9 Claude Desktop on mainPC to botPC (SSH fallback)

**Superseded (closed under #30).** Result: not run under T3 Code. Remote access from mainPC is now T3 Code's job (T3 Connect, Tailscale or SSH), validated by #31; Claude Desktop's SSH connection is no longer in the plan. Background and the earlier research: `docs/research/remote-access.md`.

## #31 T3 Code service reachable from mainPC and the phone without inbound ports

**Goal:** T3 Code runs on botPC as a background service that survives logout, lid-closed operation and a reboot, and is reachable from mainPC and the phone on another network with no inbound port, first through T3 Connect, then through Tailscale HTTPS.

**Setup:** T3 Code installed and pinned on botPC (#8); botPC on AC with #7's settings; mainPC and phone with T3 Code and a T3 Connect account; Tailscale on botPC, mainPC and phone with an ACL allowing only mainPC and phone to reach botPC; `~/varjo-phase0/` exists; a scratch repo registered as a T3 Code project with one finished thread.

**Steps:**
1. Service: `t3 up`; discover the launchd label and plist (`launchctl list | grep -i t3`; `ls ~/Library/LaunchAgents`) and record `<t3-label>`. Confirm it keeps running after you close every terminal, quit the desktop app and log out of the SSH session. If the plist is a LaunchAgent, check what happens on GUI logout and record it (autologin from #7 keeps the GUI domain).
2. Lid closed: with #7's settings, close the lid and confirm from mainPC that the service answers (see step 3).
3. T3 Connect: sign in with the same T3 Connect account on botPC and mainPC (`docs/user/remote-access.md`, `docs/internals/t3-connect.md`). Add the botPC environment in the mainPC desktop app and at app.t3.codes. From a different network (phone hotspot, mainPC joined to it), open the existing thread and send a message. Record the exact UI steps.
4. No inbound port: on botPC `sudo lsof -iTCP -sTCP:LISTEN -nP` before enabling T3 Connect and after; record any new listener and whether it is loopback only. Confirm the router forwards nothing and that a scan from a non-ACL device finds nothing.
5. Approval: with "Require approval for new devices" on, try to add the environment from a third device (a private browser window at app.t3.codes, or a second phone). Confirm it is blocked until you approve it on botPC or an approved device. Revoke it afterwards and confirm access ends.
6. Server-side storage: read `docs/internals/t3-connect.md` and `docs/user/remote-access.md`, and check the T3 Code settings for relay-related options. Record what the relay stores (device auth tokens; held webhook bodies and their retention if enabled), what is end to end, and the telemetry setting per `docs/user/telemetry.md`. Quote the sentences.
7. 3-machine cap: record how many machines are signed in (botPC, mainPC; whether the phone counts) and what the docs and the app say about the cap, the fourth machine and pricing. Do not buy anything. Record whether it matters for mainPC plus botPC.
8. Tailscale HTTPS: with T3 Connect signed out or the relay disabled for this step, follow `docs/user/remote-access.md` for the Tailscale route including Tailscale HTTPS; record the exact commands and settings you used. From the phone hotspot (not on the home LAN) with the Tailscale app on, open the thread from the phone and from mainPC. Record whether push notifications still arrive (they need T3 Connect; see #12).
9. SSH fallback: from the mainPC desktop app add an SSH-based connection to botPC (`ssh botpc true` first; sshd key-only, `PasswordAuthentication no`). Open the existing thread and send a message. Record setup effort and limits.
10. Restart: `launchctl kickstart -k gui/$(id -u)/<t3-label>` and then `sudo shutdown -r now`; after login confirm the service is back and the thread opens from mainPC without touching botPC (deeper thread-survival checks are in #32 step 6).

**Expected:** T3 Connect works through an outbound tunnel with no new listener; Tailscale HTTPS works without any third-party relay; push only with T3 Connect.

**Pass:** from mainPC on another network, an existing thread on botPC opens and accepts a message, with no inbound port exposed; both routes documented in `docs/setup/remote-access.md` with their trade-offs (this is O4 in `docs/design/architecture.md` §9). **Partial:** only one route works, or the service needs a manual restart after reboot. **Fail:** neither route is reachable from another network without an inbound port.

**If it fails:** fall back to the SSH-based connection from the desktop app (step 9) and keep the phone path on Tailscale; if no route works, stop and escalate (requirements §2.3).

**Results:**
```markdown
Date: | botPC macOS: | T3 Code (`t3 --version`): | mainPC app version: | phone app version:
Outcome: Pass / Partial / Fail
Service label / plist / survives logout, lid closed, reboot?:
T3 Connect: other-network test (mainPC, phone), exact UI steps:
New listening ports on botPC (before / after):
"Require approval for new devices": blocked until approved? Revoke works?:
Server-side storage (device tokens, held webhooks, retention, telemetry) with quotes:
3-machine cap (what counts, 4th machine, matters here?):
Tailscale HTTPS: commands/settings, other-network test, push behaviour:
SSH fallback: works? effort:
Errors (exact text):
Screenshots:
```

## #10 Claude in repositories cloned on botPC (CLI and T3 Code thread)

**Result (part 1, 2026-11-11, CLI, Claude Code 2.1.306):** Partial; see the results comment on issue #10. Repos used: varjo (docs only), rapid-go (Go; `.claude/CLAUDE.md`, path-scoped `.claude/rules/*`, agents), varjo-phase0-scratch (sh). launchd PATH needed `/opt/homebrew/bin:/usr/local/bin:~/.anyenv/envs/nodenv/shims` prepended (all of `claude`, `gh`, `node`, `go`, `docker` were missing from the default PATH); `gh` found its keyring token under launchd. A launchd headless run exited 0 in varjo with Keychain and `setup-token`; a throwaway-worktree run with `acceptEdits` and a test allowlist edited, ran `sh test.sh` and showed the diff without a prompt. Open from part 1: the fine-grained `gh` token with the out-of-scope denial check (the current token is a classic OAuth token with full access), the mainPC `/memory` comparison, and the PR-summary task once the scratch repo has a PR. CLAUDE.md and rules loading inside T3 Code moves to #34.

**Goal:** a T3 Code thread started in 2+ real repos runs with the right environment (PATH, toolchains, `gh` with a least-privilege token) from the `t3 up` service, and produces a read-only PR summary using only the Phase 1 allowlist (architecture.md §4.3).

**Setup:** two real repos cloned at `<repo-path>` (same layout as mainPC), registered as T3 Code projects; `gh auth login --with-token` with the fine-grained token; the `t3 up` service running (#8, #31); the #32 launch path available for step 6.

**Part 1: CLI baseline (steps from round 1; rerun only the open parts)**
1. `gh auth status`; `gh repo view <other-repo-not-in-scope>` must fail (token scope); `gh pr list -L 3` must succeed.
2. PATH under launchd: a job whose script runs `command -v claude gh git node go docker; echo "$PATH"` and logs; fix the plist `EnvironmentVariables` PATH until all resolve.
3. Headless read-only run in each repo from launchd: `~/varjo-phase0/headless.sh --session-id <uuid> "Summarize the latest merged PR (gh pr view). Max 5 bullets."`

**Part 2: T3 Code thread (re-scoped under #30)**
1. Service environment: start a thread in repo 1 with a permission mode that lets you run one command at a time (Supervised, approve each). Ask it to run `command -v gh git node go docker; printenv PATH` (one command per Bash call). Compare with your login shell. Record what is missing and how env is set for the `t3 up` service (the `launchctl print` output from #8, structure only, and the T3 Code setting or plist change that fixed it). Restart the service and confirm the fix persists.
2. `gh` from a thread: ask the thread to run `gh auth status` and `gh pr list -L 3`; then `gh repo view <other-repo-not-in-scope>` must fail. Repeat `command -v gh; gh auth status` from the launchd job that `varjo poll` will use (reuse the part 1 step 2 job) and record both results.
3. Read-only summary: in each repo, start a thread in the read-only mode that #32 maps to (record the mode and its allowlist), model Sonnet, with the prompt "Summarize the latest merged PR using only gh pr view, gh pr diff and git log. Max 5 bullets." Confirm it completes without a prompt, uses only those commands, and that an attempt to edit or run other Bash commands is blocked. Record turns, duration and which commands ran.

**Expected:** the service PATH differs from the login shell and needs an explicit fix; the read-only mode allows only the listed commands.

**Pass:** a scripted run in each repo completes the PR-summary task, inside a T3 Code thread (read-only mode) and from launchd for the `gh` check. **Partial:** works in one repo or needs per-repo env. **Fail:** `gh` unusable non-interactively, or no read-only mode can run `gh pr view` without prompting.

**If it fails:** set PATH and `GH_TOKEN` explicitly in the service environment, or wrap in `zsh -lc`; document the permission mode used.

**Results:**
```markdown
Date: | botPC macOS: | T3 Code (`t3 --version`):
Outcome: Pass / Partial / Fail
Repos (names, languages):
Tools missing from the service PATH, fix applied (how env is set):
`gh` scope check (out-of-scope repo denied?) in thread / in launchd job:
Read-only mode used (name, allowlist), summary per repo (turns, duration):
Blocked-action check (edit / other Bash refused?):
Errors (exact text):
Screenshots:
```

## #11 CLI-created sessions viewable and resumable (top priority, superseded)

**Result (2026-11-02):** Pass, M2 + arm-ahead selected; see issue #11 comment. Remaining on botPC at the time: V3 second Mac, V7 reboot, server mode under launchd / 10-minute outage.

**Superseded (closed under #30).** The Remote Control experiment (M1 `--bg`, M2 headless + arm-ahead, server mode) is replaced by #32. The M2 + arm-ahead design on Claude Code 2.1.306 stays as the fallback launcher: the steps and results live in git history (commit 8bdc33d) and issue #11, and the launcher is built only if #32 or #34 fails. Research: `docs/research/remote-access.md`.

## #32 A thread launched from outside T3 Code via MCP is visible and resumable (top priority)

**Goal:** a script on botPC can create, continue and read a T3 Code thread through the environment's MCP server; the same thread is a first-class thread on mainPC and the phone; it survives a service restart. This decides whether varjo can be only a launcher.

**Setup:** #8 and #31 done; `<env-url>` known; the `t3 up` service running; a scratch repo registered as a T3 Code project with an open PR `<pr-number>`; a varjo-style PR worktree: `cd <repo-path>; git fetch origin pull/<pr-number>/head && git worktree add --detach <repo-path>-pr-<pr-number> FETCH_HEAD`; the phone and mainPC apps signed in. Read `docs/user/outside-agents.md` before starting. Credential files created with `umask 077`. Do not paste tokens.

**Steps:**
1. CLI MCP client: `claude mcp add --transport http t3 https://<env-url>/mcp`, then `claude mcp login t3`. Record whether the OAuth step needs a browser on botPC (also try it over SSH), which access levels the approval screen offers (Read only / Supervised / Auto-accept edits / Auto / Full access), and the lowest level that lets `t3_thread_launch` succeed. In that Claude session ask it to call `t3_thread_list` and record the result shape.
2. Plain script client: discover the OAuth metadata the MCP server advertises (record the URLs and grant types) and implement the OAuth device flow without a browser in a small script in `~/varjo-phase0/` (record the language). Store the credential `0600` under `~/.config/varjo/`, botPC only. Call the MCP `tools/list` and save the tool names and input schemas (`t3_thread_launch`, `t3_thread_wait`, `t3_thread_read`, `t3_thread_list`, `t3_thread_search`, the message tool and the stop tool) with secrets redacted.
3. Launch: call `t3_thread_launch` with the project, workspace mode `existing worktree` at the varjo-style PR worktree, the read-only mode (record which T3 mode this maps to; Supervised with a read-only allowlist is the target), model Sonnet (low effort if exposed), and a fixed prompt: a fixed instruction plus a delimited data block, asking for a 5-bullet summary and a fenced JSON block at the end. Capture exactly what comes back (thread id, URL, status) and which MCP argument names you needed. Record the thread's final assistant message via `t3_thread_wait` then `t3_thread_read`, and whether the fenced JSON block parses.
4. Visibility: confirm the thread appears in the sidebar of the mainPC desktop app and in the phone app with the full history. From the phone, send a turn; then call `t3_thread_read` and confirm the phone's turn is present. Open the stored thread URL on the phone and on mainPC; record the URL format and whether it opens the right thread.
5. Follow-up: with nobody having the thread open, send a follow-up message into the same thread from the script (the "additional review on the same PR" case); wait and read. Also test a follow-up while the thread is still running and record what the server does.
6. Escalation: sign in a second script client approved at the lowest level (step 1), and try to launch with a higher permission mode (for example Auto-accept edits). Record the error; a thread an outside agent starts must not exceed the approved level.
7. Restart: `launchctl kickstart -k gui/$(id -u)/<t3-label>`; then `sudo shutdown -r now`. After each, `t3_thread_list` must still show the thread, `t3_thread_read` returns the history, and a new message continues it (native resume). Record whether a thread that was mid-run is marked failed or resumes.
8. Latency: record seconds from the launch call to the first token visible in the app, and to the final message; record whether `t3_thread_wait` is usable (does it return at turn end, what timeout, does it block the script) for a synchronous "summary is ready" step. Also stop a running thread via the stop tool, then `t3_thread_read` it, and record the shape for #8's limit observability.

**Expected:** MCP-launched threads look like any other thread; the access level the sign-in was approved with caps the mode; threads survive restart.

**Pass:** a script on botPC can create, continue and read a thread; the same thread is usable from the phone and mainPC; it survives a service restart. The exact MCP tool calls and the OAuth setup are written into this section's Results and then into `docs/design/architecture.md` §4.3. **Partial:** works but only with a higher access level than expected, or `t3_thread_wait` is unusable (polling `t3_thread_read` is needed), or an in-flight thread is lost on restart. **Fail:** MCP-launched threads do not appear in clients, cannot be continued, or the OAuth needs a browser that cannot be provided.

**If it fails:** return to the CLI launcher recorded in git history (commit 8bdc33d, issue #11) and reopen the Remote Control items; this is a **user decision** (requirements §4.3), do not propose a custom UI before it.

**Results:**
```markdown
Date: | botPC macOS: | T3 Code (`t3 --version`): | Claude Code (CLI client):
Outcome: Pass / Partial / Fail
OAuth setup (CLI client / script device flow, browser needed?), lowest access level for `t3_thread_launch`:
Tool names + input schema (saved file path), identifiers returned (thread id, URL):
Launch request used (project, workspace mode, permission mode mapping, model, prompt template):
Summary JSON block parsed? Final message extracted via:
Visible on mainPC sidebar / phone, full history? Phone turn seen by t3_thread_read?:
Follow-up into idle thread / running thread:
Escalation attempt (error text):
Service kill / reboot: listed, readable, resumable? In-flight thread state:
Latency: launch to first token (s) | to final message (s) | t3_thread_wait usable (timeout, behaviour):
Stopped/failed thread shape from t3_thread_read:
Errors (exact text):
Screenshots:
```

## #33 GitHub review event delivery into T3 Code

**Goal:** compare three ways a review event can start or continue a thread and record a recommendation for #30 (expected: route 1 default, route 3 optional): (1) varjo poller to MCP, (2) T3 interval task only, (3) GitHub webhook to the T3 Connect relay to a T3 webhook task.

**Setup:** #10, #31 and #32 done; a scratch repo (webhook admin for route 3 only) registered as a T3 Code project with an open PR `<pr-number>` that you can review from a second account or by submitting a review as yourself; fine-grained `gh` token; the varjo-style PR worktree from #32; read `packages/contracts/src/scheduledTask.ts` and `apps/server/src/scheduledTasks/*` for the task and webhook fields (no user doc page yet). Held webhooks stay OFF until step 6.

**Steps:**
1. Route 1, environment: from a launchd job (the same plist pattern as #10, PATH set explicitly) run `command -v gh git; gh auth status; gh pr list -L 3` and log. This is the environment `varjo poll` runs in; record fixes. Also try the same as a T3 `interval` task prompt to see whether the task environment differs.
2. Route 1, launch: submit a review on the scratch PR; from a script call `gh api` for the reviews (or `gh pr view <pr-number> --json reviews`), then `t3_thread_launch` (#32 step 3 call) with `existing worktree` pointing at the varjo-style PR worktree and the review text in a delimited data block. Record the elapsed time from review submitted to thread started, and from the poll interval you used; the GitHub permissions needed (read PRs only).
3. Route 2: in the T3 UI create an `interval` task (minimum 60 s) in the scratch project whose prompt runs `gh` to find new reviews. Record the fields it exposes, whether the workspace is fixed per task (so it cannot target a worktree per PR), how new vs seen would be tracked (the agent's own files or prompt), and what happens when two PRs have reviews at once. Expected outcome: unsuitable for multi-PR fan-out; record why.
4. Route 3, endpoint: with T3 Connect on (#31) create a `webhook` task in the scratch project. Record the URL shape (`/v1/hooks/:endpointKey/:hookId/:token`; do not paste it), the signature options (HMAC-SHA256), the JSON-path filter syntax, the prompt template syntax (`{{path}}`, for example `{{body.pull_request.number}}`) and dedupe options.
5. Route 3, GitHub: add a repo webhook (content type `application/json`, a secret kept on botPC only, event "Pull request reviews") pointing at the endpoint. Check that the signature passes (GitHub's `X-Hub-Signature-256` with the `sha256=` prefix is accepted by T3's HMAC-SHA256 check; record any setting needed for the prefix). Break the secret on purpose and redeliver from GitHub's "Recent Deliveries"; confirm a rejection. Set a JSON-path filter on `action` and `review.state` (and the PR author, if the filter supports it) so only `submitted` reviews on your own PRs start a thread; submit a comment-only edit and a review on someone else's PR to confirm they are filtered. Put an injection string in a review body and confirm the template hardening keeps it as data.
6. Route 3, dedupe and hold: use "Redeliver" for the same delivery and confirm exactly one thread (or one skipped run). Then turn on held webhooks, stop the botPC service for a few minutes (`launchctl bootout gui/$(id -u)/<t3-label>`, then `launchctl bootstrap gui/$(id -u) <plist>` to bring it back), submit a review while it is down, and confirm the held delivery arrives after it returns (and that the 24 h hold holds for the lid-closed case). Turn held webhooks OFF again afterwards and delete the test repo webhook.
7. Latency and permissions summary: for each route, record seconds from review submitted to thread started and the GitHub permission needed (route 1: read PRs; route 2: read PRs; route 3: repo webhook admin plus the T3 Connect relay).

**Expected:** route 1 works with no GitHub webhook rights; route 2 cannot fan out across PRs; route 3 works with HMAC and filters and covers lid-closed gaps through the relay hold.

**Pass:** route 1 end to end (poller to `existing worktree` thread) and a recorded verdict on routes 2 and 3 (route 3 end to end on one test repo, or a recorded reason it is not usable). **Partial:** route 3 untestable (no webhook admin), route 1 only. **Fail:** route 1 cannot start threads in a varjo-created worktree (then the `EventSource` and worktree design in architecture.md §4.1 and O3 must be revisited).

**If it fails:** route 1 problems are the PATH and `gh` fixes from #10 or the `existing worktree` limits from #15; route 3 problems stay an optional-adapter question only.

**Results:**
```markdown
Date: | botPC macOS: | T3 Code (`t3 --version`):
Outcome: Pass / Partial / Fail -> recommended default route: 1 / 2 / 3 (record in #30)
Route 1: launchd env (gh/PATH ok?), existing worktree launch ok?, latency (s), permissions needed:
Route 2: task fields / workspace fixed? / dedup possible? / verdict:
Route 3: signature (X-Hub-Signature-256, sha256= prefix) ok? bad secret rejected? filter used / selects only submitted own-PR reviews? template + injection check? duplicate delivery deduped? held delivery after restart? latency (s), permissions:
Errors (exact text):
Screenshots:
```

## #34 Claude via T3 Code (Agent SDK) loads CLAUDE.md, rules, settings, hooks and Auto Memory

**Goal:** a table of what loads from where inside a T3 Code thread, for the repo root and a T3-created worktree, matching the CLI table in #15's results; plus how the Auto Memory key is chosen and whether hooks fire. This decides whether "the repository's `CLAUDE.md` and `.claude/rules/` are applied as-is" still holds, and whether the handoff hook (architecture.md §4.9) is possible.

**Setup:** a scratch repo registered as a T3 Code project, with on the PR branch: `CLAUDE.md` and `.claude/CLAUDE.md` each with `PHASE0-MARKER-<name>`, `.claude/rules/x.md` (unscoped) and `.claude/rules/y.md` (path-scoped with `paths:` frontmatter to a file `src/y.txt`), `.claude/settings.json` with `{"env":{"PHASE0_ENV":"repo"}}` and a permissions entry you can observe, one skill and one agent; a user-level marker in `~/.claude/CLAUDE.md` and `~/.claude/settings.json` (`{"env":{"PHASE0_USER":"1"}}`); hooks in `.claude/settings.json`: a `PostToolUse` hook with `matcher: "Bash"` and `if: "Bash(gh pr create *)"` and a `Stop` hook, each running `cat >> ~/varjo-phase0/hook-<event>.json`. `touch ~/varjo-phase0/marker` before step 3. Thread permission mode: Supervised (approve each action).

**Steps:**
1. Find how T3 Code configures the SDK: shallow-clone the pinned tag into `~/varjo-phase0/t3code-src` and run `grep -rn "settingSources\|CLAUDE_CONFIG_DIR\|claude-agent-sdk" apps/server packages | head -40`. Record the file and line of the `settingSources` (or equivalent) value, any `CLAUDE_CONFIG_DIR` handling per provider instance, and the SDK version from the lockfile.
2. Root thread: in a thread at the repo root ask: "Quote the PHASE0-MARKER lines from CLAUDE.md and .claude/CLAUDE.md; list the rules you have loaded; list skills and agents; run printenv PHASE0_ENV; then run printenv PHASE0_USER and printenv CLAUDE_CONFIG_DIR; one command per Bash call." Then ask it to read `src/y.txt` and re-list the rules (path-scoped rule). Record which items loaded and which did not.
3. Worktree thread: repeat step 2 in a thread with workspace mode `new worktree` on the PR branch (T3-created), and again with `existing worktree` at the varjo-style worktree (see #15). Record loaded-from for each.
4. User level: confirm `~/.claude/CLAUDE.md` marker and `PHASE0_USER` appear in the thread (loaded or not).
5. Hooks: with approvals on, have a thread run `gh pr create` in the scratch repo (scratch only) and then finish. Check `~/varjo-phase0/hook-PostToolUse.json` and `hook-Stop.json` exist. Record the stdin keys only (`head -c 300`, structure; do not read contents), whether the `if` matcher filtered other Bash calls, and whether a hook can read the PR URL.
6. Auto Memory: ask the thread to remember a fact ("remember: phase0-t3-fact"), then `find ~/.claude -newer ~/varjo-phase0/marker -type f | sed 's/[^/]*$//' | sort -u` (structure only). Record the `~/.claude/projects/<key>/memory/` directory used for the repo root and for the worktree, compare it with the `<key>` the `claude` CLI uses on the same path (#15: memory under the repo key, transcripts under the worktree key), and where session data lives. If a thread-private `CLAUDE_CONFIG_DIR` appears in step 1 or 2, record the directory and how memory behaves under it.
7. Gaps: list every item that did not load and decide per item: a varjo responsibility (for example inject rules into the prompt) or an upstream T3 Code issue to file. Cross-check #14 using the memory key recorded here.

**Expected:** unknown. The SDK loads project and user settings only if `settingSources` includes them; memory and hooks follow the same switch.

**Pass:** the table below is complete for root and worktree, with every gap assigned an owner (varjo or upstream). **Partial:** CLAUDE.md and rules load but hooks or memory behave differently from the CLI (record, plan around). **Fail:** project CLAUDE.md and rules are not loaded and cannot be enabled.

**If it fails:** inject the repo's `CLAUDE.md` and rules into the prompt from varjo, or return to the CLI launcher (commit 8bdc33d, issue #11); a failure here with no workaround triggers the fallback rule (see "After Phase 0").

**Results:**
```markdown
Date: | botPC macOS: | T3 Code (`t3 --version`): | Agent SDK version:
Outcome: Pass / Partial / Fail
settingSources value (file:line) / CLAUDE_CONFIG_DIR handling:
| Item | root | T3 new worktree | existing worktree | notes |
| CLAUDE.md / .claude/CLAUDE.md | | | | |
| rules (unscoped / path-scoped) | | | | |
| .claude/settings.json (env, permissions) | | | | |
| skills / agents | | | | |
| user ~/.claude/CLAUDE.md and settings.json | | | | |
| hooks (PostToolUse `if` / Stop; stdin keys) | | | | |
| Auto Memory key (root / worktree) vs CLI key | | | | |
Gaps -> owner (varjo / upstream):
Errors (exact text):
Screenshots:
```

## #12 Operating botPC from the phone (T3 Code mobile app)

**Goal:** the requirements §2.3 flow completes with the phone as the only client, using the T3 Code mobile app, and the MCP-launched thread from #32 is usable from it.

**Setup:** #31 (reachability, device approval) and #32 (an MCP-launched thread) done; scratch repo with branch protection on `main`; T3 Code iOS/Android app signed in; phone on mobile data (Wi-Fi off) for the Connect run, and again on the Tailscale route.

**Steps:**
1. Phone only, mobile data: open the thread launched through MCP in #32, read the summary, send an instruction, and answer a permission prompt. Confirm the next `t3_thread_read` from the script sees these turns.
2. Push notifications: trigger finish, fail (for example a denied or stopped run), approval request and question in threads, with the phone locked, once via T3 Connect (`docs/user/mobile-notifications.md`) and once via the Tailscale-only route. Record which arrive, the timing, and the Tailscale-only experience (expected: no push; the thread list still works).
3. §2.3 flow, phone as the only client: create a new thread in the scratch repo project with workspace mode `new worktree`, send "Add a line to README, run the tests, commit on a branch and open a PR", answer each permission prompt on the phone, then commit, push and open the PR from T3 Code's source-control UI (`docs/user/source-control.md`). Confirm the PR link opens on the phone and the result is visible as the thread's last turn.
4. Device approval: with "Require approval for new devices" on, sign in on a second, unapproved phone or browser and confirm it is blocked until approved on botPC.
5. On botPC `sudo lsof -iTCP -sTCP:LISTEN -nP`: no new listening ports during the flow.

**Expected:** instruction, worktree, changes, tests, PR, result visible; Full access is not selected by default; unapproved devices are blocked.

**Pass:** the full flow once from mobile data with the phone as the only client, and the MCP-launched thread from #32 is usable from the phone. **Partial:** works only on Wi-Fi or Tailscale, push missing on the Tailscale route (expected, record), or needs one botPC-side manual step. **Fail:** cannot drive or approve from the phone, or an unapproved device is not blocked.

**If it fails:** revisit the route (T3 Connect vs Tailscale) in #31; as last resort the SSH-based connection from the desktop app on mainPC (not a phone path).

**Results:**
```markdown
Date: | botPC macOS: | T3 Code: | Phone app version / OS:
Outcome: Pass / Partial / Fail
MCP-launched thread usable from phone (read / instruct / approve)?:
Push (finish / fail / approval / question): T3 Connect timing | Tailscale-only behaviour:
§2.3 steps completed (instruction / worktree / tests / commit+push+PR / result seen):
Mobile data confirmed? Route used:
Unapproved device blocked?:
New listening ports on botPC:
Errors (exact text):
Screenshots:
```

## #13 Dispatch (desk-check only, superseded)

**Superseded (closed under #30).** Recorded result: No. Dispatch runs only inside Claude Desktop on the paired machine and cannot target botPC; it is recorded in `docs/research/remote-access.md`. The smartphone entry point is now the T3 Code mobile app (#12, #31). Nothing left to check.

## #14 Auto Memory sharing between mainPC and botPC

**Goal:** a memory round trip mainPC to botPC to mainPC with zero data loss, plus session-copy resume. These are `claude` CLI results and stay the reference for the memory sync design; T3 threads reuse the same `~/.claude/projects/<key>/memory/` only if #34 step 6 says so.

**Setup:** same absolute `<repo-path>` on both Macs; compute `<key>`; both on the same Claude Code version; SSH from mainPC to botPC works. Everything here uses rsync over SSH, no `--delete`.

**Steps:**
1. mainPC: `ls ~/.claude/projects/<key>/memory/`. Scan for secrets: `grep -rEi 'token|secret|password|sk-ant|ghp_|AKIA' ~/.claude/projects/<key>/memory/`. The sync set is `projects/<key>/memory/` only (never `.claude.json`, `sessions/`, `settings.json`, credentials).
2. Dry run then copy: `rsync -avn --update -e ssh ~/.claude/projects/<key>/memory/ botpc:.claude/projects/<key>/memory/` then without `-n` (create the dir first if needed).
3. botPC: `cd <repo-path>; claude`, `/memory`, ask "what do your notes say about <topic in the copied memory>". Confirm the files are listed and used.
4. On botPC ask Claude to remember a new fact, exit. `rsync -av --update -e ssh botpc:.claude/projects/<key>/memory/ ~/.claude/projects/<key>/memory/`. Compare with `shasum` before/after: nothing lost.
5. Conflict: edit the same line of `MEMORY.md` on both machines, rsync in both directions, record which edit survives and whether anything signals the conflict (rsync `--update` is last-write-wins).
6. External edit mid-session: start `claude` on botPC (any terminal); from mainPC rsync a changed `MEMORY.md`; in the live session ask about the new line; then start a fresh session. Record whether the live session re-reads it.
7. Worktree: `claude -p ... --worktree phase0-mem "remember: phase0-worktree-fact"`; `ls -d ~/.claude/projects/*phase0-mem*` and check whether `memory/` landed under `<key>` or the worktree key.
8. Fallback tests (only if keys differ): set `autoMemoryDirectory` in `~/.claude/settings.json` to a synced dir and confirm a per-repo value works from user scope; try symlinking `~/.claude/projects/<key>` and check key resolution.
9. Session copy: choose one finished, secret-free `<uuid>` on mainPC. `rsync -av -e ssh ~/.claude/projects/<key>/<uuid>.jsonl ~/.claude/projects/<key>/<uuid>/ botpc:.claude/projects/<key>/` (note: copy the directory with its own rsync call). On botPC `cd <repo-path>; claude --resume <uuid>`: history, cwd, working. Also `claude --resume ~/.claude/projects/<key>/<uuid>.jsonl`.
10. Different key: on botPC copy the same files into `~/.claude/projects/<other-key>/` (clone at `<alt-repo-path>`), `cd <alt-repo-path>; claude --resume <uuid>`. Then leave both copies present and retry (docs: duplicates may yield "not found"). Never resume the same session on both machines at once.
11. T3 cross-check (after #34): take the memory directory a T3 thread read and wrote in #34 step 6 (root and worktree) and confirm it is the `<key>` directory that steps 2 and 4 sync. Write a fact through a T3 thread on botPC, run the step 4 rsync back to mainPC, and confirm it arrives with no loss. If #34 found a different directory or a thread-private `CLAUDE_CONFIG_DIR`, record the path and use it as the sync unit instead.

**Expected:** identical paths give identical keys and everything lines up; session copy resume is unconfirmed (research: inferred); T3 threads share the CLI's memory directory only if the SDK loads it from the same place.

**Pass:** memory round trip with zero loss and a documented checklist of excluded paths, with the T3 thread's memory directory identified. **Partial:** memory works but session copy needs the same key or a path argument, or T3 uses a different directory. **Fail:** botPC does not load copied memory or data is lost.

**If it fails:** `autoMemoryDirectory`; or `CLAUDE_CONFIG_DIR` plus `CLAUDE_CODE_PROJECT_DIR_NAME` per launch; or a private Git repo of the memory folder.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code (both machines): | T3 Code:
Outcome: Pass / Partial / Fail
Key used / paths identical?:
Memory loaded on botPC? Round trip loss (shasum diff)?:
Concurrent edit: winner / signal given:
External edit during live session re-read?:
Worktree memory location:
autoMemoryDirectory / symlink results (if tried):
Session copy: same key / different key / --resume <path> / duplicate copies:
T3 thread memory directory (root / worktree) = synced key? Round trip via T3 ok?:
Excluded paths checklist:
Errors (exact text):
Screenshots:
```

## #15 Git worktree lifecycle (T3-created and varjo-created worktrees)

**Result (CLI, 2026-11-01 and 2026-11-11, Claude Code 2.1.316 / 2.1.306):** Partial, see the two comments on issue #15; the CLI results stay the reference for the fallback launcher. Config loading passes (CLAUDE.md, rules, settings, skill, agent all load from the worktree); the main checkout's uncommitted `.claude/settings.local.json` IS inherited (not "not inherited" as expected) and overrides a worktree's own local file on the same key; transcripts are keyed by the worktree path, memory by the repo path; `--worktree` directories are locked for the session's lifetime (`git worktree unlock` before `git worktree remove`); `claude rm` does not apply to `-p` worktrees; a resumed session whose worktree is gone continues in the current directory. Open from the CLI round: the PR-bound `--worktree "#<pr>"` and `--from-pr` steps, dropped unless the CLI fallback is built. What loads inside a T3 thread moves to #34 (the loading table to match is in the 2026-11-01 comment on issue #15 and the 2026-11-11 results).

**Goal:** document the worktree lifecycle for worktrees that T3 Code creates (`new worktree` workspace mode) and that varjo creates and hands to T3 (`existing worktree`), and settle O3 (worktree ownership) in `docs/design/architecture.md` §9.

**Setup:** the scratch repo and PR `<pr-number>` from #33, registered as a T3 Code project; #32's script available; a second commit ready to push to the PR branch. `touch ~/varjo-phase0/marker` before step 1.

**Steps:**
1. T3-created: start a thread with workspace mode `new worktree`. Record where T3 puts the worktree (`git worktree list` in `<repo-path>`, `ls`), how it names the branch, which base it uses, and whether the worktree is locked (`git worktree list --porcelain`). Check that the PR head is not the base: can you choose the PR branch, or must varjo fetch `pull/<pr-number>/head` into it?
2. varjo-created: `cd <repo-path>; git fetch origin pull/<pr-number>/head && git worktree add --detach <repo-path>-pr-<pr-number> FETCH_HEAD`. Start a thread with workspace mode `existing worktree` at that path (UI first, then the MCP call from #32 step 3). Record whether T3 accepts a detached checkout at `pull/<pr-number>/head`, the exact error otherwise, and whether T3 adds anything to the worktree (untracked files, locks, a branch).
3. Reuse: send a second launch or a follow-up for the same PR (#32 step 5). Is the same worktree reused, or does T3 create another (check the `git worktree list` count)?
4. Update to a new head: push a second commit to the PR branch. While the thread is idle, update the varjo worktree: `git fetch origin pull/<pr-number>/head && git checkout --detach FETCH_HEAD` in it, then send a follow-up asking the thread to run `git log -1 --oneline`. Does the thread see the new tip? Repeat for the T3-created worktree and record what a fast-forward there needs.
5. Cleanup: find T3 Code's worktree cleanup setting (discover where it is; `docs/user/source-control.md`) and record its default. Delete or archive the T3-created thread and see whether T3 removes its worktree. Do the same for the varjo-created thread: the varjo worktree must NOT be deleted by T3. Then do varjo's own cleanup: `git worktree unlock` if locked, `git worktree remove <path>`, and check the main checkout is intact (`git status`) and the thread still opens (cwd missing: record what it does).
6. Transcript and layout record: `find ~/.claude -newer ~/varjo-phase0/marker -type f | sed 's/[^/]*$//' | sort -u` (structure only) to see whether T3 keys anything by worktree path; memory key belongs to #34.

**Expected:** T3 names branches and picks locations itself and cannot be pointed at a PR head directly (hence O3 option (b), varjo-managed `existing worktree`); fast-forward while idle works.

**Pass:** layout and lifecycle (create / update to head SHA / remove) for both strategies written to `docs/design/worktrees.md`, and O3 decided. **Partial:** `existing worktree` works only with a named branch instead of a detached head, or T3's cleanup needs a setting change to leave varjo's worktrees alone (document it). **Fail:** `existing worktree` rejects a varjo-managed PR worktree and T3's `new worktree` cannot be made to check out the PR head.

**If it fails:** use the `new worktree` mode (O3 option (a)) and have varjo run `git fetch origin pull/<pr-number>/head` plus checkout inside T3's worktree before each launch; or return to the CLI sweeper design from the round 1 results.

**Results:**
```markdown
Date: | botPC macOS: | T3 Code (`t3 --version`):
Outcome: Pass / Partial / Fail -> O3: (a) T3 new worktree / (b) varjo existing worktree
T3-created: location / branch naming / base / locked?:
varjo-created at pull/<n>/head: accepted by `existing worktree`? detached ok? error text:
Second launch reused the worktree?:
Fast-forward to new head while idle (varjo / T3 worktree):
T3 worktree cleanup setting (default) vs varjo-created worktree deleted? Thread after removal:
Errors (exact text):
Screenshots:
```

## Known unknowns this runbook resolves

| Unconfirmed item (source) | Experiment |
|---|---|
| T3 Code picks up `claude auth login` with no API key; Claude login from the `t3 up` service with no terminal; 7-day survival (`providers-claude.md`, `background-service.md`) | #8 |
| Service PATH and env: `claude`, `gh`, toolchains missing under launchd; how to set env for the `t3 up` service (`background-service.md`, #8 round 1) | #8, #10 |
| Usage-limit signals in T3 (thread state, error text, push) and whether `t3_thread_read` / `t3_thread_wait` expose them to the varjo guard (max-plan §4/§5, `faq.md`) | #8 (with #32 step 8) |
| Subscription use through the Agent SDK vs the Max plan terms (#30 decision 3, `faq.md`) | #8 |
| Concurrent mainPC CLI and botPC T3 Code logins under one account (max-plan) | #8 |
| Exit codes and JSON for usage limit, bad token, max-turns; `claude auth status` JSON; `setup-token` plus launchd (claude-code §2/§18.3/§18.8, max-plan) | #8 (CLI part, recorded in round 1) |
| `gh` scope and non-interactive use from a thread and from `varjo poll`; read-only mode runs the Phase 1 allowlist (requirements §6, architecture §4.3) | #10 |
| Hardware stability, sleep prevention, reboot recovery (requirements §2.1) | #7 |
| T3 Code service survives logout, lid closed, reboot; reachable from another network without an inbound port (`remote-access.md`) | #31 |
| T3 Connect storage (device tokens, held webhooks) and "Require approval for new devices" (`t3-connect.md`) | #31 (decision 1 in #30 / O4) |
| 3-machine cap and whether it matters (`t3-connect.md`, `faq.md`) | #31 |
| Tailscale-only route and push behaviour; SSH fallback from the desktop app (`remote-access.md`, `mobile-notifications.md`) | #31, #12 |
| Access level needed for `t3_thread_launch`; OAuth without a browser for the MCP server; escalation cap (`outside-agents.md`) | #32 steps 1, 2, 6 |
| A thread launched via MCP is visible on mainPC and the phone with full history and resumable (`outside-agents.md`) | #32 steps 3, 4, 5 |
| Thread survival across service restart and reboot (native resume) (`overview.md`) | #32 step 7 |
| `t3_thread_wait` usability, launch latency, MCP call shapes (`outside-agents.md`) | #32 step 8 |
| Interval-task suitability for multi-PR fan-out (source: `scheduledTask.ts`) | #33 route 2 |
| Webhook HMAC with GitHub's `X-Hub-Signature-256` (`sha256=` prefix), JSON-path filter, template injection hardening, dedupe (source, release notes v0.1.7 to v0.1.10) | #33 route 3 |
| Held-webhook delivery after the host returns (source, `t3-connect.md`) | #33 route 3 step 6 |
| Review-to-thread latency and the GitHub permissions each route needs (requirements §3.2) | #33 |
| `settingSources` / CLAUDE.md and rules loading in T3 threads, root vs worktree; `CLAUDE_CONFIG_DIR` (`providers-claude.md`, `apps/server`) | #34 |
| Hooks (`PostToolUse` with `if`, `Stop`) under the SDK and their stdin (architecture §4.9) | #34 step 5 |
| Memory key under T3 vs the CLI key; worktree vs repo keying of memory (auto-memory) | #34 step 6, #14 step 11, #14 step 7 |
| Phone: drive and approve an MCP-launched thread, push for finish/fail/approval/question, unapproved device blocked, §2.3 flow (`mobile-notifications.md`, `source-control.md`) | #12 |
| T3 `new worktree` location, branch naming; `existing worktree` with a `pull/<n>/head` checkout; reuse; fast-forward while idle (`source-control.md`, requirements §2.1) | #15 |
| T3 worktree cleanup setting vs varjo's cleanup on PR close (architecture §4.10, O3) | #15 step 5 |
| Cross-machine memory round trip; session copy resume; `autoMemoryDirectory`; external `MEMORY.md` edit (auto-memory, claude-code §1/§18.1) | #14 |
| T3 Code version pinning and update policy (pre-1.0, 1 to 2 releases a week); managed policy cap; telemetry opt-out | #16 policy |
| `rate_limit_event` in plain `json` mode and statusLine in `-p` (max-plan); Claude `auto` permission mode (claude-code §18.11) | not covered; only captured if a limit hits naturally in #8 |

## After Phase 0

- Collect the filled Results from #7, #8, #10, #12, #14, #15 and #31 to #34. They feed `docs/design/architecture.md` §4 to §6 and §9 to §10: the T3 launcher and MCP credential (#32, §4.3), the trigger route and `EventSource` (#33, §4.1, O5), worktree ownership (#15, §4.10, O3), the handoff hook and memory sync unit (#34, #14, §4.9), limit handling (#8, §4.6), the notification default and the optional adapter (#12, #31, O2, O4), and the Phase 0 mapping.
- Record #30 decisions 1 to 3: T3 Connect vs Tailscale (O4, from #31 and the storage notes), the trigger route (from #33), and subscription use through the Agent SDK under the Max terms (from #8).
- Update `docs/setup/botpc.md` (#7, #8, #31 service label and env), `docs/setup/remote-access.md` (#31, both routes with trade-offs) and `docs/design/worktrees.md` (#15) as their pass criteria require. The #9, #11 and #13 results stay as recorded above and in `docs/research/remote-access.md`.
- Fallback rule: if #32 or #34 fails, return to the CLI launcher recorded in git history (commit 8bdc33d, issue #11: M2 + arm-ahead, with `setup-token` for launchd) and reopen the Remote Control items (#9, #11, #12 as originally scoped). This is a **user decision** (requirements §4.3); do not propose a custom UI before it.
