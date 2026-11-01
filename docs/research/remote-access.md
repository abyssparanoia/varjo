# Remote access research (issue #2)

Goal: view and continue Claude Code sessions that run non-interactively on botPC from Claude Desktop on mainPC and from a phone, with no public inbound port on botPC.

## Sources

Fetched 2026-11-01 (Claude Code CLI 2.1.306 locally). Everything below is from these pages unless marked "unconfirmed".

- https://code.claude.com/docs/en/desktop.md (SSH sessions, Open in, Dispatch section, CLI-to-Desktop handoff)
- https://code.claude.com/docs/en/remote-control.md
- https://code.claude.com/docs/en/claude-code-on-the-web.md (cloud sessions, `--cloud`, `--teleport`)
- https://code.claude.com/docs/en/mobile.md
- https://code.claude.com/docs/en/cli-reference.md, feature-availability.md, deep-links.md
- https://support.claude.com/en/articles/13947068 (Dispatch help article)

## A. Claude Desktop over SSH ("SSH sessions", Code tab)

- What: Desktop Code tab runs Claude Code on a remote Linux/macOS/Windows machine over SSH, with Desktop as the UI. Setup: environment selector > New environment > SSH tab > Add SSH connection (name, `user@host` or `~/.ssh/config` Host, port, optional key, worktree folder) > pick a remote folder.
- Remote side needs: sshd with SFTP subsystem; non-prompting login (Desktop logs in at least twice per connect); `MaxSessions` >= 2 (or Desktop >= v3.1623.0); ~1 GB free in `~/.claude/remote/` (not a noexec mount); Bash/Zsh/sh; outbound access Claude Code needs (else Desktop downloads locally and uploads). Desktop auto-installs Claude Code and a helper under `~/.claude/remote/`. The helper listens on no network port; it is reached through the SSH connection (unix socket).
- Login: the doc does not require `claude auth login` on the remote for a Claude-account Desktop. Desktop forwards/renews the sign-in. With a Claude account the sign-in is renewed only while Desktop is running and connected, so an unattended task "can stop when sign-in expires, possibly after a few minutes". The doc says not to count on overnight runs. (Whether the remote CLI's own login is used for botPC-side scripts is separate and unaffected.)
- `git` and `gh` must be installed and authenticated on the remote by the user (Desktop does not prompt for `gh` over SSH).
- Listing existing sessions: NOT documented. Docs describe SSH sessions as ones Desktop starts. `/resume` (list of CLI-started sessions) "is available in local sessions, not in SSH, WSL, or cloud sessions". Cross-session visibility also says Desktop sees only sessions it runs itself. So a session created by a CLI script on botPC appears in the SSH sidebar: unconfirmed, and the docs lean toward "no".
- Resume by ID: no documented SSH-side resume-by-ID. `claude --desktop --resume <id>` (CLI >= 2.1.285, macOS/x64 Windows, subscription auth) opens a session in Desktop, but only for a session in the local machine's history (i.e. run on the same Mac as Desktop), not on a remote host. Unconfirmed for SSH.
- Warning in docs: do not run `claude --resume` on the remote for a conversation whose process Desktop still runs; both write to the same conversation (`--fork-session` copies).
- Deep link: `claude://code/new?ssh_host=user%40host&ssh_port=..&ssh_folder=/path&q=prompt` (Desktop >= v2.110.0) opens the new-session page on a saved/new SSH connection after a confirm dialog. It starts a NEW session; it does not open an existing session by ID. No documented session-ID deep link.
- Network: any route SSH can use (ProxyJump, ProxyCommand, jump host, `ConnectTimeout`). Tailscale is not mentioned in the docs; it works in principle as ordinary SSH reachability (unconfirmed, not tested). sshd must be reachable from mainPC, so this needs a private path (Tailscale/LAN/VPN) to satisfy "no public inbound port".
- Disconnect behavior (Desktop >= v2.31226.0): the remote process keeps running while Desktop is disconnected, under a helper; helper stops after 30 days without a connection. Worktree-edit checks, connectors and workflows fail while disconnected (see doc "Limits while Desktop is disconnected").
- Plans: the SSH page states no plan restriction other than Claude account vs third-party provider setups. Plan requirement: unconfirmed (docs list none).
- Not supported over SSH: moving a session to cloud (Open in > Cloud); `Merge when ready` disabled.

## B. Remote Control (RC)

- What: connects claude.ai/code, the Claude iOS/Android app (Code tab) and Claude Desktop to a Claude Code session whose process runs on your machine. Execution and filesystem stay local; the web/mobile UI is a window into it.
- Invocation modes (CLI):
  - `claude remote-control` (server mode, no local interactive session; flags `--name`, `--spawn same-dir|worktree|session`, `--capacity N` default 32, `-c/--continue`, `--session-id <id>`, `--remote-control-session-name-prefix`; spacebar shows QR).
  - `claude --remote-control ["name"]` / `--rc` (interactive session also remote-controllable).
  - `/remote-control` or `/rc` inside an existing interactive session (CLI, Desktop local session, VS Code).
  - Auto-connect for every interactive session: `/config` > "Enable Remote Control for all sessions" or `remoteControlAtStartup: true` in `~/.claude/settings.json`.
- Existing session: yes, for a running interactive session via `/remote-control`. Also `claude --resume <id> --remote-control` style combination is plausible but unconfirmed. After a session was stopped, `claude --continue`/`--resume` reattaches RC if it was on. Server mode creates NEW sessions on demand from the client (the clients can start sessions in the server's directory/worktrees).
- Headless `-p`: not documented. RC is documented for interactive sessions and server mode only. A `claude -p` run being remote-controlled is unconfirmed (likely not supported). Server mode is the documented non-interactive-ish host (a long-lived process, ideally in tmux/launchd).
- Multiple sessions: server mode serves several concurrently (default capacity 32). Otherwise one remote session per interactive process; with auto-connect each process registers its own.
- Networking: outbound HTTPS only; the session registers with the Anthropic API and polls for work; "never opens inbound ports". Traffic is TLS via Anthropic. Short-lived scoped credentials. Transcript is stored on Anthropic servers while connected (retention per Data usage policy). Satisfies the "no inbound port" requirement.
- Connect: open session URL in browser, scan QR in Claude app, or pick it in the Code tab session list (computer icon with green dot when online). Desktop also lists RC-connected sessions (session list shows "devices signed in to your claude.ai account"). A session is identified by `claude.ai/code/<id>` URL (deep link-able in browser/app).
- Lifetime: no fixed link TTL documented. Stopped server sessions can be brought back for about 4 hours (`claude remote-control`, `--continue`, `--session-id`). Server mode exits after about 10 minutes of network outage; interactive mode retries indefinitely; a 30-minute presence-heartbeat failure disconnects an interactive session (rerun `/remote-control`). Permission prompts and AskUserQuestion stay open until answered.
- Requirements: Pro, Max, Ultra, Team, Enterprise (Team/Enterprise: Owner must enable the toggle). claude.ai subscription auth only: API keys, `ANTHROPIC_AUTH_TOKEN`, `apiKeyHelper`, Bedrock/Vertex/Foundry, and `ANTHROPIC_BASE_URL` other than api.anthropic.com disable it. Not available under ZDR/HIPAA. Local process must keep running; machine must be awake. First run of `claude remote-control` asks a one-time y/n confirmation (interactive; needs doing once by hand on botPC).
- Phone capabilities: send messages, attach images/files, answer permission prompts and questions, stop subagents/workflows, change model/effort, view diff. Local-only commands (`/plugin`, `/resume`) do not work remotely. Bypass permissions cannot be selected from the app. Push notifications when Claude finishes or needs a decision (`/config` toggles).
- Security: Trusted Devices option (enrollment, biometric after 18h) for RC. `disableRemoteControl` setting. `--sandbox` flag removed in v2.1.300; use settings-file sandboxing.

## C. Dispatch

- What: a persistent conversation inside Cowork in Claude Desktop; you message it from the Claude mobile app and it decides whether to do the work in Cowork or to spawn a Claude Code session. Code sessions it spawns appear in the Desktop Code tab sidebar; phone gets a push when done or needing approval.
- Executes on: the paired user's own computer, in the Desktop app ("Claude runs on: Your machine (Desktop)"), using local files, connectors and plugins. Not a cloud sandbox. Requires the computer awake, Desktop open, internet on both devices.
- Targeting botPC: the help article does not describe choosing a machine or repository, or pairing several computers. Claude picks session type from the task. Using botPC would mean running Desktop (signed in, awake, lid handling) on botPC and pairing the phone with it. Whether the folder/repo can be chosen (e.g. a cloned repo) is unconfirmed. Spawned sessions are local Desktop sessions, so they use botPC's local repos and credentials (`gh`, git) as any local session.
- PR creation/identity: not documented for Dispatch. As a local Desktop session it would use whatever `gh`/git identity is configured on that machine (inference, unconfirmed). By contrast, cloud sessions use the user's connected GitHub (Claude GitHub App or `/web-setup` gh token via a server-side proxy); Auto-fix replies are posted under the user's GitHub account but labeled as from Claude Code.
- Plans: Pro, Max, Ultra only; not Team/Enterprise. Help article says it is in limited beta and "isn't available to new users" (existing users keep it). One continuous thread; cannot start/manage several.
- Not related to SSH. Dispatch is orthogonal to cloud sessions.
- Cloud sessions / `--cloud` / `--teleport` (context): "Claude Code on the web" (claude.ai/code, mobile Code tab, Desktop "Cloud" env, `claude --cloud "task"`). Runs in Anthropic-managed VM (or self-hosted environment); clones from GitHub. `--teleport [id]` pulls a CLOUD session into a local terminal as a copy; one-way (cannot push a terminal session to cloud). `claude -p "msg" --cloud <session_id>` posts a follow-up to a cloud session from any machine. `--resume` does not list cloud sessions. Plans: Pro, Max, Ultra, Team, Enterprise (premium seats). Not for ZDR/HIPAA/CMEK orgs. `--cloud` and `--teleport` need claude.ai login. Not usable to expose a botPC-local session.

## Mobile app (Claude iOS/Android, Code tab)

- Is a client, not an execution host. From the phone: start/monitor cloud sessions and projects, drive a Remote Control session on your computer, message Dispatch.
- With Remote Control: list sessions (by name, online dot), view conversation, send messages, attach photos/files, approve/deny permissions and answer questions, switch Manual/Accept edits/Plan/Auto modes (not Bypass), push notifications.
- Cannot attach to Desktop SSH sessions or arbitrary CLI sessions unless RC is on for them (docs do not mention SSH sessions being visible on mobile; unconfirmed).
- `/mobile` (or `/ios`, `/android`) in the CLI shows an app-store QR code.

## Comparison

| Capability | SSH (Desktop) | Remote Control | Dispatch |
|---|---|---|---|
| Lists existing CLI-created sessions on botPC | Unconfirmed; docs imply no (only sessions Desktop starts) | Yes if the session has RC on (running process); stopped sessions only via `claude remote-control --continue/--session-id` within ~4 h or by `--resume` with RC | No (spawns its own sessions) |
| Resume by ID | No documented remote resume by ID (`claude --desktop --resume <id>` is local-only) | Yes: `claude remote-control --session-id <id>` (v2.1.200+); `claude --resume <id>` locally on botPC re-attaches RC if it was on | No |
| Deep link | `claude://code/new?ssh_host=...` starts a new session only | `claude.ai/code/<id>` URL and QR per session | None documented |
| Works from phone | Not documented | Yes (Claude app Code tab, browser) | Yes (its purpose) |
| Works from Desktop | Yes (its purpose) | Yes (Desktop lists RC sessions; unconfirmed UI path on a different Mac, docs say "Claude apps on devices signed in") | Yes, but Desktop is the executor (the paired machine) |
| Inbound port needed on botPC | sshd reachable by mainPC (needs private path such as Tailscale/LAN); not public-safe by default | None (outbound HTTPS only) | None documented (outbound to Anthropic) |
| Headless `-p` session supported | Unconfirmed; Desktop launches its own Claude Code processes | Unconfirmed, probably not; server mode is the non-interactive host | N/A (Desktop spawns sessions) |
| Plan requirement | None stated (Claude account or 3P) | Pro/Max/Ultra/Team/Enterprise; subscription auth only; Team/Ent Owner must enable | Pro/Max/Ultra only; limited beta, not for new users |

## Recommended order of Phase 0 experiments (validates #11: view a CLI-created session from another machine)

1. Remote Control with an interactive session on botPC (lowest effort, no inbound port). On botPC (inside tmux): `claude --remote-control "test"`, or start `claude` then `/remote-control`. Open it from mainPC browser/Desktop and phone. Then exit, run `claude --resume <id> --remote-control` (or `/remote-control`) and confirm it re-attaches to the same claude.ai session.
2. Remote Control server mode: `claude remote-control --spawn worktree` as a launchd/tmux service on botPC. Test creating sessions from the phone, `--session-id` resume, and the ~10-minute outage exit (needs a supervisor).
3. Probe the gap that matters for varjo: start a session non-interactively (`claude -p ... --session-id X`, then `claude --resume X` on botPC) and check whether the session (a) shows in claude.ai/code, (b) accepts `--remote-control` when combined with `-p` or `--resume`. If not, design varjo to run automation sessions as interactive/`--remote-control` processes or as server-mode sessions.
4. Desktop SSH over Tailscale (private path): add botPC as an SSH environment, check whether sessions created by CLI scripts under that folder appear in the sidebar, whether any resume works, and the sign-in expiry behavior when mainPC sleeps.
5. Dispatch last: needs Desktop running on botPC and pairing; high uncertainty (limited beta, no machine/repo targeting documented). Test only if 1-3 fail the phone requirement.
6. Cloud sessions/`--teleport` are not a substitute for botPC-local sessions; consider only as a fallback (sessions run in Anthropic VM, not on botPC).

## Unconfirmed list

- SSH sidebar showing CLI-created remote sessions; SSH sessions visible on mobile.
- RC with `claude -p`; Desktop-on-mainPC UI path for RC sessions started on another machine.
- Dispatch machine/repo targeting and GitHub identity for PRs; multi-computer pairing.
- Tailscale + Desktop SSH (no doc mention).
- Any fixed RC link TTL beyond the ~4 h resume window and outage timeouts.
