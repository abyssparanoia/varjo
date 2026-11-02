# Phase 0 validation runbook

This runbook turns the nine Phase 0 validation issues (#7 to #15) into hands-on experiments that one person can run on real hardware: botPC (the always-on MacBook), mainPC (the everyday MacBook) and a phone. Each experiment maps 1:1 to its issue. The settled approach: Remote Control (RC) is the primary path for session visibility (#11); Desktop-over-SSH is an interactive fallback only; Dispatch is dropped, so #13 is a desk-check. Sources: `docs/requirements.md` (§2, §4, §6, §9) and `docs/research/*.md`. Anything marked "unconfirmed" is not verified by the research and is what these experiments exist to settle.

## How to use this runbook

- Run one experiment per issue, in the order given below. Do not skip the Setup block: most failures come from an unset env var or an API key left in the shell.
- When an experiment ends, paste its filled **Results** template as a comment on that issue, then tick the matching checkbox in #18.
- Never run two sessions against the same session ID at once (the docs say concurrent writers interleave into one transcript).
- Commands are plain POSIX sh. Placeholders: `<uuid>` session ID (create with `uuidgen | tr 'A-Z' 'a-z'`), `<repo-path>` absolute repo path (identical on both Macs), `<pr-number>`, `<key>` project key (`echo <repo-path> | sed 's/[^A-Za-z0-9]/-/g'`).
- Work files live in `~/varjo-phase0/` on botPC, never inside a repo. Do not paste tokens or transcript contents into issues.

Shared helper, create once on botPC as `~/varjo-phase0/headless.sh` (`chmod +x`). It encodes the settled headless defaults (`dontAsk`, read-only allowlist, 15 turns, Sonnet, low effort):

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
| botPC | macOS version recorded (`sw_vers`); Claude Code >= 2.1.306 via native install (`claude --version`, `claude doctor`); `gh`, `git`, `tmux` installed; Tailscale optional (needed only for #9 and for SSH access); same macOS username and workspace layout as mainPC; plugged into AC; Remote Login (sshd) on only if #9 is run |
| mainPC | Claude Desktop, current version recorded; signed in to claude.ai with the Max account; browser logged in to claude.ai; same username and repo paths as botPC |
| Phone | Claude app (iOS/Android) signed in to the same account, Code tab visible; Tailscale app only if #9 is run |

Safety checklist (do before any experiment, re-check after #8):

- [ ] `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` unset on botPC in every shell and in `launchctl getenv`; no `apiKeyHelper` or Bedrock/Vertex vars (an API key silently wins in `-p` and also disables RC).
- [ ] Usage credits OFF at claude.ai > Settings > Usage (per account, so one check covers both Macs). No Console payment method or auto-reload.
- [ ] `claude auth status --text` on botPC shows a subscription login, not an API key.
- [ ] botPC has its own `claude auth login` and its own `claude setup-token` token. Never copy credentials, `~/.claude.json` or the Keychain item from mainPC.
- [ ] GitHub: a fine-grained token limited to the target repos (Contents, Pull requests, Metadata; no admin, no workflow). Direct pushes to `main` blocked by branch protection. Use a scratch repo for anything that writes.
- [ ] Never pick Fable or "requires usage credits" models; the helper pins `--model sonnet`.

## Experiment order and dependencies

1. **#7 and #8 first, in parallel.** Hardware stability and auth gate everything; both start a long soak (48 h and 7 days) that runs while later work proceeds.
2. **#10** (repos, PATH under launchd, `gh` token) needs #8's login. Same day as #8.
3. **#11** (top priority) needs #8 and #10. Same day. Run M1 before M2. The sleep/reboot sub-checks interrupt the #7 soak, so do those after the 48 h are complete or restart the soak clock.
4. **#15** needs #10; reuses the #11 repo and worktree. Same day as #11 or the next.
5. **#9** SSH fallback: only needs a private path (LAN/Tailscale); run after #11 so the CLI-created sessions exist to look for. Skip if #11 passes on all Desktop sub-checks (record "not needed" and stop).
6. **#12** needs the winning launch mode from #11; same day.
7. **#14** independent of RC but needs two repos' worth of finished sessions from #10/#11/#15. Same day.
8. **#13** desk-check, no dependency; do any time, 15 minutes.

Same-day block: #8 (login, runs a to d), #10, #11, #15, #12, #14, #13. Needs soak: #7 (48 h lid closed), #8 (7 days unattended refresh, plus a natural usage-limit capture if one happens), #11 (reboot/sleep checks, plus re-check the RC session 24 h later). Close the soak-bound issues only after the soak ends.

## #7 botPC lid-closed stability

**Goal:** botPC stays awake and reachable for 48 h with the lid closed, and returns after power loss.

**Setup:** botPC on AC, no external display, lid closed after step 3. A second device (mainPC) to probe reachability.

**Steps:**
1. Record hardware: `system_profiler SPHardwareDataType | grep -E "Model|Chip|Memory"; sw_vers; pmset -g`.
2. Apply sleep prevention (record which were needed): `sudo pmset -a sleep 0 disksleep 0 displaysleep 0 autorestart 1`; for clamshell without a display also try `sudo pmset -a disablesleep 1`. Alternative: `caffeinate -dimsu` in a launchd job, or Amphetamine. Verify with `pmset -g assertions`.
3. Install a heartbeat LaunchAgent (`~/Library/LaunchAgents/com.varjo.phase0.heartbeat.plist`, `RunAtLoad`, `KeepAlive`) running `sh -c 'while :; do date -u +%FT%TZ >> ~/varjo-phase0/heartbeat.log; sleep 60; done'`. Load: `launchctl bootstrap gui/$(id -u) <plist>`. Enable automatic login so the GUI domain exists after a reboot (note FileVault implications).
4. Close the lid. From mainPC every few hours: `ping -c3 <botpc-host>` and `ssh <botpc-host> uptime` (or Tailscale name). When #11 exists, also confirm its RC session is still online (green dot).
5. Load test for 30+ minutes with the lid closed: run a real build/test loop in a repo plus a headless Claude run; record `pmset -g therm` and `sudo powermetrics --samplers thermal -n 1` before, during, after.
6. After 24 h change network (move to another Wi-Fi or toggle Wi-Fi) and confirm SSH/RC recover without touching botPC.
7. Pull the power cable until the battery drops a few percent, plug back in; then `sudo shutdown -r now` and confirm login, heartbeat resume and `autorestart` behaviour. Optionally a real power cut.
8. After 48 h: count heartbeat lines (`wc -l ~/varjo-phase0/heartbeat.log`, expect about 2880) and list gaps over 2 minutes: `awk '{cmd="date -j -u -f %Y-%m-%dT%H:%M:%SZ "$1" +%s"; cmd | getline t; close(cmd); if (p && t-p>120) print $1, t-p; p=t}' ~/varjo-phase0/heartbeat.log`.

**Expected:** no gaps beyond the deliberate reboot; reachable after wake and network change; thermals steady without throttling.

**Pass:** 48 h uptime, lid closed, no unexpected sleep, remote access works throughout, config documented in `docs/setup/botpc.md`. **Partial:** works only with an extra tool (Amphetamine, `disablesleep`) or with intermittent wake gaps under 5 minutes. **Fail:** sleeps with lid closed despite all settings.

**If it fails:** use a cheap HDMI dummy plug or a real external display to force clamshell mode, or keep the lid ajar; re-evaluate hardware (requirements §2.1).

**Results:**
```markdown
Date: | Hardware model: | botPC macOS: | Claude Code:
Outcome: Pass / Partial / Fail
pmset settings / tools used:
Uptime achieved (h), unexpected gaps:
Thermals (idle / load, throttling?):
Reboot + power-loss recovery (heartbeat resumed? autologin?):
Network change / 24 h reachability:
Errors (exact text):
Screenshots:
```

## #8 Claude Code auth with the Max subscription

**Goal:** botPC authenticates on its own, runs interactive and headless jobs with no API key, and the exact failure shapes are captured for limit handling (`docs/research/max-plan.md`).

**Setup:** `env | grep -E 'ANTHROPIC|CLAUDE_CODE_OAUTH'` empty. Create `~/varjo-phase0/headless.sh`. `mkdir -p ~/.config/varjo`.

**Steps:**
1. Login: `claude auth login` over SSH from mainPC (try the URL/code flow: open the printed URL in mainPC's browser and paste the code; record whether a browser on botPC was needed). Then `claude auth status --json` and save the JSON shape (unconfirmed: used to assert subscription login).
2. Interactive run: `claude` in a scratch dir, ask a one-line question, `/status`, `/usage`; exit.
3. Headless (a) normal: `cd <repo-path>; ~/varjo-phase0/headless.sh --session-id $(uuidgen | tr 'A-Z' 'a-z') "Say ok" > a.json 2> a.err; echo "exit=$?"`. Record `is_error`, `subtype`, `api_error`, `session_id`.
4. Token: `claude setup-token` (browser flow, prints a one-year token); store it with `umask 077; cat > ~/.config/varjo/oauth-token`. It is used only for launchd jobs.
5. (b) Invalid token: `CLAUDE_CODE_OAUTH_TOKEN=bad ~/varjo-phase0/headless.sh --session-id $(uuidgen | tr 'A-Z' 'a-z') "Say ok" > b.json 2> b.err; echo "exit=$?"`. Record exit code and JSON (`is_error`, `api_error`, `subtype`).
6. (d) Turn exhaustion: prompt needing a tool call with a 1-turn cap: `... "Run git log -1 and summarize" --max-turns 1` (later flag overrides; if not, call `claude -p` directly with the same flags). Record exit code, JSON, and whether `session_id` is still present.
7. (c) Usage limit: do not burn quota. If a limit is hit naturally during any experiment, save the full stdout JSON, stderr and exit code verbatim.
8. launchd headless: `~/varjo-phase0/launchd-run.sh` does `export CLAUDE_CODE_OAUTH_TOKEN=$(cat ~/.config/varjo/oauth-token)` then runs `headless.sh` in `<repo-path>` and appends exit code and JSON to `~/varjo-phase0/launchd.log`. Create `com.varjo.phase0.headless.plist` (`ProgramArguments` = `/bin/sh <script>`, `StartCalendarInterval` once per day, `EnvironmentVariables` PATH including the `claude` directory). `launchctl bootstrap gui/$(id -u) <plist>; launchctl kickstart -k gui/$(id -u)/com.varjo.phase0.headless`. Run it with no terminal attached (log out of SSH first). Also run once without the token file to see whether the Keychain login works under launchd.
9. Concurrency: use Claude Code interactively on mainPC and botPC at the same time, send prompts in both; afterwards `claude auth status` on both and confirm neither asks to log in.
10. Soak: keep the daily launchd job for 7 days; each day append `claude auth status --json | head -c 300` and the run's `is_error`. Note the day of any re-login prompt.

**Expected:** (a) exit 0, `is_error:false`; (b) non-zero exit with an auth error; (d) error result (subtype likely `error_max_turns`, unconfirmed); no re-login for 7 days.

**Pass:** headless succeeds from launchd with no terminal, login survives 7 days, shapes recorded. **Partial:** works only with the `setup-token` token and not with the Keychain login (acceptable, record which), or limit shape (c) not captured. **Fail:** headless needs a browser or API key, or refresh breaks either machine.

**If it fails:** use `setup-token` for all headless jobs (already the plan); if concurrent logins invalidate each other, give botPC a separate login and avoid simultaneous interactive use (refresh-token race, issue #88583 in `max-plan.md`). Do not fall back to an API key.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code:
Outcome: Pass / Partial / Fail
Login method (browser on botPC? over SSH?):
`claude auth status --json` keys:
| Case | exit code | is_error | api_error | subtype | notes |
| (a) normal | | | | | |
| (b) invalid CLAUDE_CODE_OAUTH_TOKEN | | | | | |
| (c) usage limit (natural, or "not hit") | | | | | |
| (d) --max-turns 1 | | | | | |
launchd headless (token / Keychain): | Concurrent mainPC+botPC ok?:
7-day refresh log (day of any re-login):
Errors (exact text):
Screenshots:
```

## #9 Claude Desktop on mainPC to botPC (SSH fallback)

**Goal:** confirm Desktop's official SSH connection works over a private path with no inbound public port; fallback for interactive access only.

**Setup:** botPC Remote Login on; key-only auth (`PasswordAuthentication no`); Tailscale on botPC, mainPC, phone with ACL allowing only mainPC and phone to reach botPC; `MaxSessions` >= 2; ~1 GB free in `~/.claude/remote/`. Run after #11. If #11 passes all Desktop sub-checks, record "not needed" and stop.

**Steps:**
1. mainPC `~/.ssh/config` entry `Host botpc` with `HostName <tailscale-name>`, key, `ConnectTimeout 15`; test `ssh botpc true` twice (Desktop logs in at least twice).
2. Desktop > Code tab > environment selector > New environment > SSH > Add SSH connection (name, `botpc`, folder `<repo-path>`). Start a session; record install prompt, UX, speed.
3. Sidebar check: do the CLI-created sessions from #11 (M1/M2, same folder) appear? Can any be resumed? (docs lean "no"; unconfirmed.)
4. Do not run `claude --resume` on botPC for a session Desktop is running (both write one transcript).
5. Sleep mainPC for 10 minutes mid-task; record sign-in expiry behaviour.
6. Network: test from the home LAN, then from phone hotspot (different network) over Tailscale. On botPC `sudo lsof -iTCP -sTCP:LISTEN -nP` and confirm sshd is not exposed beyond Tailscale/LAN and the router forwards nothing. From a non-ACL device confirm botPC is unreachable.

**Expected:** new Desktop-started sessions work; CLI-created sessions probably not listed.

**Pass:** from mainPC on a different network, a Claude Code session on botPC opens in Desktop with no inbound port, setup in `docs/setup/remote-access.md`. **Partial:** works for Desktop-started sessions only. **Fail:** cannot connect or requires a public port.

**If it fails:** rely on RC (#11); use plain `ssh botpc` plus `tmux` and `claude --resume <uuid>` for interactive access.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code: | Desktop version:
Outcome: Pass / Partial / Fail / Not needed
SSH setup + Tailscale ACL notes:
CLI-created sessions listed in SSH sidebar? Resume works?:
Different-network test (hotspot): | Listening ports on botPC:
mainPC-sleep sign-in expiry:
Errors (exact text):
Screenshots:
```

## #10 Claude Code in repositories cloned on botPC

**Goal:** headless runs in 2+ real repos work from launchd with project config, toolchains and a least-privilege `gh` token.

**Setup:** clone two real repos at the same absolute path layout as mainPC (`<repo-path>`). `gh auth login --with-token` with the fine-grained token.

**Steps:**
1. In each repo, interactive `claude`: `/memory` and ask which `CLAUDE.md` and `.claude/rules/*` files loaded; compare against mainPC.
2. `gh auth status`; `gh repo view <other-repo-not-in-scope>` must fail (token scope); `gh pr list -L 3` must succeed.
3. PATH check under launchd: a job whose script runs `command -v claude gh git tmux node go docker; echo "$PATH"` and logs. Fix the plist `EnvironmentVariables` PATH until all resolve. Check `gh` finds its token under launchd (Keychain vs `GH_TOKEN` in the env file).
4. Headless read-only run in each repo from launchd (same plist pattern as #8): `~/varjo-phase0/headless.sh --session-id <uuid> "Summarize the latest merged PR (gh pr view). Max 5 bullets."`
5. Tests and diff: in a throwaway worktree (`git worktree add ../<name>-phase0 -b phase0-scratch`) run `claude -p --permission-mode acceptEdits --max-turns 15 --model sonnet --effort low --allowedTools "Read,Grep,Glob,Edit,Bash(<test-cmd>),Bash(git diff *)" "Fix a typo in README, run the tests, show the diff"`. Verify no interactive prompt appears and exit code is 0.

**Expected:** project rules load; toolchains resolve once PATH is set; no prompts.

**Pass:** a scripted headless run in each repo completes the PR-summary task from launchd. **Partial:** works in one repo or needs per-repo env. **Fail:** project config not loaded or `gh` unusable non-interactively.

**If it fails:** pass `GH_TOKEN` via the env file, set PATH explicitly in the plist, or wrap in `zsh -lc`; document the permission mode used.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code:
Outcome: Pass / Partial / Fail
Repos (names, languages): | CLAUDE.md/rules loaded identically to mainPC?:
Tools missing from launchd PATH, fix applied:
`gh` scope check (out-of-scope repo denied?):
Headless task per repo (exit code, duration, session_id):
Errors (exact text):
Screenshots:
```

## #11 CLI-created sessions viewable and resumable from mainPC and phone (top priority)

**Result (2026-11-02):** Pass, M2 + arm-ahead selected; see issue #11 comment. Remaining on botPC: V3 second Mac, V7 reboot, server mode under launchd / 10-minute outage.

**Goal:** decide which launch mode (M1 or M2) makes a session created on botPC visible and controllable from phone, browser and Claude Desktop.

**Setup:** #8 and #10 done. In the shell used for M1/M2, `unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN` (the `setup-token` token cannot establish RC and overrides the login). Run `claude remote-control` once by hand to answer the first-run y/n prompt, then Ctrl+C. Pick an open PR `<pr-number>` in a scratch repo at `<repo-path>`. Prompt used: `"Read the PR diff and summarize it in 5 bullets, then wait."`

**Sub-checks (apply to M1 and M2; record Y/N plus notes):**
V1 visible in the Claude app Code tab (phone) | V2 visible at claude.ai/code in a browser on mainPC | V3 visible in Claude Desktop on mainPC | V4 follow-up message works and context is intact | V5 permission prompt can be approved from phone (ask: "Create PROBE.txt containing the date") | V6 survives mainPC sleep (sleep 10 min, wake, then V4 from phone only) | V7 after `claude stop <id>` and after botPC reboot.

**M1: `--bg` interactive session with RC on**
1. `cd <repo-path>; U1=$(uuidgen | tr 'A-Z' 'a-z'); claude --bg --remote-control --session-id $U1 --worktree "#<pr-number>" "<prompt>"`. Record the printed short ID and whether a `claude.ai/code/<id>` URL is printed (expected). If a flag combination is rejected, record the exact error and retry dropping one flag at a time (note which). Note (2.1.306): `--bg` ignores `--session-id` ("--bg manages the session id; ignoring --session-id"); read the real UUID and state from `claude agents --json` (`id`, `sessionId`, `cwd`, `state`, `status`, `name`), and the claude.ai URL only from `claude logs <id>` (never on the launch stdout).
2. `claude agents`, `claude logs <short-id>`. Run V1 to V6. Record the URL; tap it on the phone and open it on mainPC (can it go in a notification?).
3. V7: `claude stop <short-id>`; check the session's state in the app/browser; `claude attach <short-id>` and `claude --resume $U1 --remote-control`; does it re-attach to the same claude.ai session or create a new one? Then reboot botPC (log in if needed): `claude agents`, `claude respawn <short-id>`; does RC reconnect?
4. With the session running, `claude --resume $U1` on botPC: record the refusal/copy behaviour (do not type into both).

**M2: headless first, RC on demand**
1. `U2=$(uuidgen | tr 'A-Z' 'a-z'); ~/varjo-phase0/headless.sh --session-id $U2 "<prompt>" --worktree "#<pr-number>" | tee m2.json`. Confirm `session_id` equals `$U2` and a summary comes back.
2. Check V1 to V3 now (expected: not listed; headless RC is unconfirmed and probably unsupported).
3. Arm ahead from the worktree: `cd <repo-path>` (the worktree), then `claude --bg --resume $U2 --remote-control --permission-mode dontAsk -n <name>` (no tmux; `--bg` holds the process). Read the bg id from `claude agents --json`, then poll `claude logs <bg-id>` for the `https://claude.ai/code/session_01...` URL. Record the URL and whether the earlier headless history is present. Run V1 to V6. Pitfall: if a `--bg` entry already exists for `$U2` (even stopped), this starts a copy with a new session id; use `claude respawn <bg-id>` instead. Pass `--permission-mode` explicitly: a `--bg` session otherwise runs in the machine default mode (auto here) without prompts.
4. V7: `claude stop <bg-id>`, check the UI shows offline, run `~/varjo-phase0/headless.sh --resume $U2 "Reply with the PR title"`, then `claude respawn <bg-id>`; check same session id and same URL. Then reboot botPC and record whether `claude agents` / `claude respawn <bg-id>` (and `claude respawn --all`) restore the session. Pitfall: put the prompt before variadic flags such as `--allowedTools` (they swallow a trailing positional prompt) or pass it on stdin with `< /dev/null`.
5. Hand back: a phone-typed turn is persisted in the same local `.jsonl`; confirm `~/varjo-phase0/headless.sh --resume $U2 "What did I last ask from the phone?"` (after `claude stop <bg-id>`) sees it and the transcript stays continuous.

**Server mode (after M1/M2)**
1. Detached: `nohup claude remote-control --name varjo-phase0 --spawn worktree --capacity 4 < /dev/null > ~/varjo-phase0/rc.log 2>&1 &`. On 2.1.306 the help lists `--spawn same-dir|worktree|session`, `--capacity N`, `--session-id`, `--permission-mode`, `-c/--continue`, `--[no-]create-session-in-dir` and `--name`; it prints `https://claude.ai/code?environment=env_...` and pre-creates a session in cwd that is visible on the phone. Pass `--permission-mode` explicitly. `--session-id` takes the `session_01...` id of a session created through server mode (a uuid gives "Could not reach the server to look up session"; a `--bg` RC session id gives "has no environment_id"), and cannot be used with spawn flags. Start a session from the phone; reattach later with `claude remote-control --session-id <session_01...>`.
2. Run it under launchd (`KeepAlive`, `RunAtLoad`, log to `~/varjo-phase0/rc.log`); confirm the first-run y/n prompt does not hang the job once it was answered by hand. Disconnect Wi-Fi 12 minutes: server mode should exit after about 10 minutes (documented); confirm launchd restarts it.

**Expected (as observed on 2.1.306):** `--bg` prints no URL on stdout (it appears only in `claude logs <id>`). M2 headless is not listed until the `--bg --resume --remote-control` step, after which V1 to V4 work on phone and browser. Whether Desktop on a second Mac lists RC sessions from botPC (V3) is still unconfirmed.

**Pass:** one mode passes V1 to V6 on phone and in mainPC Desktop (browser acceptable for Desktop only if Desktop fails and is noted). **Partial:** visible and drivable only from phone/browser, or resumable only by ID. **Fail:** nothing is visible without SSH. A fail needs a **user decision** (requirements §4.3); do not propose a custom UI before it.

**If it fails:** fall back to the `claude --resume <uuid>` over SSH/tmux (#9 path) with a notification carrying the session ID and the resume command, or to server mode as the host for automation sessions.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code: | Desktop: | Claude app:
Outcome: Pass / Partial / Fail  -> preferred mode for Phase 1: M1 / M2 / server
| Check | M1 | M2 |
| URL printed / format | | |
| V1 phone | | |
| V2 browser mainPC | | |
| V3 Desktop mainPC | | |
| V4 follow-up | | |
| V5 approve prompt | | |
| V6 survives mainPC sleep | | |
| V7 stop / reboot | | |
`--resume` on botPC re-attaches the same claude.ai session?:
Server mode (--spawn present? launchd first-run prompt? 10-min outage?):
Errors (exact text):
Screenshots:
```

## #12 Operating botPC from the phone

**Goal:** the requirements §2.3 flow completes with the phone as the only client.

**Setup:** the winning mode from #11; scratch repo with branch protection on `main`; phone on mobile data (Wi-Fi off).

**Steps:**
1. Start (or reuse) an RC session on botPC: M1 as in #11, or `claude remote-control --spawn worktree` and start the session from the phone.
2. From the phone only: reach the existing session created earlier; send "Add a line to README, run the tests, commit on a branch and open a PR".
3. Answer each permission prompt on the phone; note push notification timing (`/config` toggles).
4. Confirm the PR link is visible and openable on the phone and the diff view works.
5. On botPC `sudo lsof -iTCP -sTCP:LISTEN -nP`: no new listening ports.

**Expected:** instruction, worktree, changes, tests, PR, result visible; Bypass mode is not selectable from the app.

**Pass:** the full flow once from mobile data with the phone as the only client. **Partial:** works only on Wi-Fi or needs one botPC-side manual step. **Fail:** cannot drive or approve from the phone.

**If it fails:** revisit server mode vs `--bg`; as last resort cloud sessions (`--cloud`) run in Anthropic's VM, not botPC (not a substitute).

**Results:**
```markdown
Date: | botPC macOS: | Claude Code: | Claude app:
Outcome: Pass / Partial / Fail
Mode used / session URL format:
Steps completed (instruction / worktree / tests / PR / result seen):
Mobile data confirmed? Push notifications?:
Errors (exact text):
Screenshots:
```

## #13 Dispatch (desk-check only)

**Goal:** confirm from the docs that Dispatch cannot target a specific machine; record and close.

**Steps (about 15 minutes, no hands-on setup):**
1. Read the Dispatch help article (https://support.claude.com/en/articles/13947068) and the Dispatch section of the Desktop docs (`desktop.md`). Look for machine/repo selection, multi-computer pairing and GitHub identity for PRs.
2. If the beta is visible, spend at most 5 minutes in Claude Desktop settings on mainPC looking at Dispatch settings; screenshot; do not pair or configure anything.
3. Record the yes/no for "Dispatch as the smartphone entry point for botPC".

**Expected:** no way to choose a target machine or repo (research: unconfirmed beyond the help article; limited beta, Pro/Max/Ultra only, not offered to new users).

**Pass / Fail:** Close as "No" if no machine/repo targeting is documented or visible (expected). Re-open only if the docs describe targeting, and then test after #12 fails.

**If it fails:** RC (#11, #12) is the smartphone path.

**Results:**
```markdown
Date: | Docs read (URLs, date): | Desktop version:
Outcome: No (cannot target botPC) / Yes
Evidence (quote the relevant sentences):
Beta visible in Desktop settings? Screenshots:
```

## #14 Auto Memory sharing between mainPC and botPC

**Goal:** a memory round trip mainPC to botPC to mainPC with zero data loss, plus session-copy resume.

**Setup:** same absolute `<repo-path>` on both Macs; compute `<key>`; both on the same Claude Code version; SSH from mainPC to botPC works. Everything here uses rsync over SSH, no `--delete`.

**Steps:**
1. mainPC: `ls ~/.claude/projects/<key>/memory/`. Scan for secrets: `grep -rEi 'token|secret|password|sk-ant|ghp_|AKIA' ~/.claude/projects/<key>/memory/`. The sync set is `projects/<key>/memory/` only (never `.claude.json`, `sessions/`, `settings.json`, credentials).
2. Dry run then copy: `rsync -avn --update -e ssh ~/.claude/projects/<key>/memory/ botpc:.claude/projects/<key>/memory/` then without `-n` (create the dir first if needed).
3. botPC: `cd <repo-path>; claude`, `/memory`, ask "what do your notes say about <topic in the copied memory>". Confirm the files are listed and used.
4. On botPC ask Claude to remember a new fact, exit. `rsync -av --update -e ssh botpc:.claude/projects/<key>/memory/ ~/.claude/projects/<key>/memory/`. Compare with `shasum` before/after: nothing lost.
5. Conflict: edit the same line of `MEMORY.md` on both machines, rsync in both directions, record which edit survives and whether anything signals the conflict (rsync `--update` is last-write-wins).
6. External edit mid-session: start `claude` in tmux on botPC; from mainPC rsync a changed `MEMORY.md`; in the live session ask about the new line; then start a fresh session. Record whether the live session re-reads it.
7. Worktree: `claude -p ... --worktree phase0-mem "remember: phase0-worktree-fact"`; `ls -d ~/.claude/projects/*phase0-mem*` and check whether `memory/` landed under `<key>` or the worktree key.
8. Fallback tests (only if keys differ): set `autoMemoryDirectory` in `~/.claude/settings.json` to a synced dir and confirm a per-repo value works from user scope; try symlinking `~/.claude/projects/<key>` and check key resolution.
9. Session copy: choose one finished, secret-free `<uuid>` on mainPC. `rsync -av -e ssh ~/.claude/projects/<key>/<uuid>.jsonl ~/.claude/projects/<key>/<uuid>/ botpc:.claude/projects/<key>/` (note: copy the directory with its own rsync call). On botPC `cd <repo-path>; claude --resume <uuid>`: history, cwd, working. Also `claude --resume ~/.claude/projects/<key>/<uuid>.jsonl`.
10. Different key: on botPC copy the same files into `~/.claude/projects/<other-key>/` (clone at `<alt-repo-path>`), `cd <alt-repo-path>; claude --resume <uuid>`. Then leave both copies present and retry (docs: duplicates may yield "not found"). Never resume the same session on both machines at once.

**Expected:** identical paths give identical keys and everything lines up; session copy resume is unconfirmed (research: inferred).

**Pass:** memory round trip with zero loss and a documented checklist of excluded paths. **Partial:** memory works but session copy needs the same key or a path argument. **Fail:** botPC does not load copied memory or data is lost.

**If it fails:** `autoMemoryDirectory`; or `CLAUDE_CONFIG_DIR` plus `CLAUDE_CODE_PROJECT_DIR_NAME` per launch; or a private Git repo of the memory folder.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code (both machines):
Outcome: Pass / Partial / Fail
Key used / paths identical?:
Memory loaded on botPC? Round trip loss (shasum diff)?:
Concurrent edit: winner / signal given:
External edit during live session re-read?:
Worktree memory location:
autoMemoryDirectory / symlink results (if tried):
Session copy: same key / different key / --resume <path> / duplicate copies:
Excluded paths checklist:
Errors (exact text):
Screenshots:
```

## #15 CLAUDE.md and rules inside Git worktrees

**Goal:** confirm project config loads from the worktree, find out how state is keyed, and document the worktree lifecycle.

**Setup:** a scratch repo at `<repo-path>` with, on the PR branch only: `CLAUDE.md` and `.claude/rules/x.md` each containing `PHASE0-MARKER-<name>`, a `.claude/settings.json` with `{"env":{"PHASE0_ENV":"worktree"}}`, one skill and one agent. In the main checkout, uncommitted `.claude/settings.local.json` with `{"env":{"PHASE0_LOCAL":"1"}}`. `touch ~/varjo-phase0/marker` before step 1.

**Steps:**
1. Create via Claude: `cd <repo-path>; ~/varjo-phase0/headless.sh --session-id <uuid> "Quote the PHASE0-MARKER lines from CLAUDE.md and rules; list skills and agents; run printenv PHASE0_ENV and printenv PHASE0_LOCAL" --worktree "#<pr-number>" --allowedTools "Read,Grep,Glob,Bash(printenv *)"` (the later `--allowedTools` replaces the default list; if not, add the entries to the wrapper). Layout under test: `<repo-path>/.claude/worktrees/pr-<pr-number>/`. Also compare a sibling layout: `git worktree add ../<repo>-pr-<pr-number> <pr-branch>` and start `claude -p` there with the same prompt.
2. Expected: markers, skill, agent and `PHASE0_ENV` come from the worktree; `PHASE0_LOCAL` is empty (not inherited; note what `.worktreeinclude` must replicate).
3. State keying: `ls -d ~/.claude/projects/*<repo-name>*`; is the transcript under a worktree key; memory under the repo key; does `claude agents --cwd <repo-path>` list it?
4. Second run: rerun step 1 with `--worktree "#<pr-number>"` and the `--resume <uuid>` form. Is the same directory reused and its state kept?
5. Cleanup: `claude rm <uuid>` (does it remove a `-p` worktree? else `git worktree remove <path>`, `git worktree unlock` if locked); then check the main checkout is intact (`git status`), and `claude -p --resume <uuid> "Reply ok"` still works (cwd missing: expected to resume in the current dir).
6. PR link: in the worktree run `claude -p --permission-mode acceptEdits --session-id <uuid2> --allowedTools "Read,Edit,Bash(git add *),Bash(git commit *),Bash(git push *),Bash(gh pr create *)" "Fix a typo, commit, push, and create a PR with gh pr create"` (scratch repo only). Then `claude --from-pr <number>`: is `<uuid2>` found? Where is the link: `grep -c '"pr-link"' ~/.claude/projects/<key>/<uuid2>.jsonl`; `find ~/.claude -newer ~/varjo-phase0/marker -type f | sed 's/[^/]*$//' | sort -u` (structure only; do not read contents).

**Pass:** layout and lifecycle (create / update to head SHA / remove) written to `docs/design/worktrees.md`. **Partial:** config loads but state keying or `claude rm` behaves differently from docs (document it). **Fail:** rules or settings not loaded from the worktree.

**If it fails:** copy needed files via `.worktreeinclude`, run `git worktree remove` in a varjo sweeper, and keep varjo's own PR-to-session map instead of `--from-pr`.

**Results:**
```markdown
Date: | botPC macOS: | Claude Code:
Outcome: Pass / Partial / Fail
Layout tested (.claude/worktrees vs sibling):
Loaded from worktree (CLAUDE.md / rules / settings / skill / agent):
PHASE0_LOCAL inherited? (expected no):
Keying: transcript key / memory key / agents list:
Second `--worktree "#n"` run reused same dir?:
`claude rm` on -p worktree / git worktree remove outcome:
`--from-pr <n>` finds -p session? link location (structure):
Errors (exact text):
Screenshots:
```

## Known unknowns this runbook resolves

| Unconfirmed item (source) | Experiment |
|---|---|
| CLI-created (botPC) sessions visible in Desktop/phone; RC UI path on another Mac (remote-access, claude-code §17.1) | #11 V1 to V3 |
| RC with headless `-p`; `claude --resume <id> --remote-control` (remote-access) | #11 M2 |
| `--bg` sessions visible through RC (claude-code §10/§18.6) | #11 M1 |
| `--spawn`/`--capacity` on 2.1.306, server mode under launchd, first-run y/n, 10-minute outage (remote-access, claude-code §18.12) | #11 server mode |
| `claude --resume` on botPC re-attaches the same claude.ai session; stop/reboot behaviour (remote-access) | #11 V7 |
| SSH sidebar shows CLI-created sessions; Tailscale + Desktop SSH; sign-in expiry (remote-access) | #9 |
| Dispatch machine/repo targeting, GitHub identity, multi-computer pairing (remote-access) | #13 |
| Exit codes and JSON for usage limit, bad token, max-turns; `claude auth status` JSON (claude-code §2/§18.3, max-plan §4/§5) | #8 |
| `setup-token` + launchd works with API key unset, Keychain under launchd; independent refresh (claude-code §18.8, max-plan) | #8 |
| Whether `rate_limit_event` appears in plain `json` mode; statusLine in `-p` (max-plan) | not covered; only captured if a limit hits naturally in #8 (c) |
| Second `--worktree "#n"` reuse; `claude rm` on `-p` worktree (claude-code §5/§18.4) | #15 |
| `--from-pr` finds `-p` sessions; link storage (claude-code §11/§18.5) | #15 |
| Worktree vs repo keying of memory/sessions (auto-memory) | #14 step 7, #15 |
| Cross-machine session copy resume; same vs different key; duplicate copies (auto-memory, claude-code §1/§18.1) | #14 |
| `autoMemoryDirectory` per-repo value; symlinked keys (auto-memory) | #14 step 8 |
| External `MEMORY.md` edit during live session (auto-memory) | #14 step 6 |
| Hook/`PostToolUse` on `gh pr create` in `-p` (claude-code §18.10), `auto` permission mode (§18.11), reboot restore via launchd (§18.7), Agent SDK ToS (§18.13) | Not covered by this runbook |

## After Phase 0

- Collect the filled Results from #7 to #15. They feed `docs/design/architecture.md` (#5): auth model (`setup-token` for launchd, own login for RC), repo path convention, limit handling (#8 table), session/PR mapping (#15), memory sync unit (#14), notification contents (#11 URL format).
- Phase 1 launch mode is settled by #11: M2 + arm-ahead (headless `-p --session-id`, then `claude --bg --resume --remote-control`). Server mode is the Phase 4 candidate. Re-check the remaining botPC items (V3 second Mac, V7 reboot, server mode under launchd / 10-minute outage). If any invalidates M2, escalate to the user (requirements §4.3).
- Update `docs/setup/botpc.md` (#7), `docs/setup/remote-access.md` (#9) and `docs/design/worktrees.md` (#15) as their pass criteria require, and record the #13 "No" in `docs/research/remote-access.md`.
