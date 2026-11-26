# Worktree layout and lifecycle

Status: Draft from Phase 0 experiment #15 (parts 1 and 2, Claude Code 2.1.306, 2026-11-01 to 2026-11-27). Decides the facts behind open decision O3 in [architecture.md](architecture.md); the choice of strategy is still the user's.

## 1. What #15 established

| Question | Result |
|---|---|
| Does project config load from a worktree? | Yes. `CLAUDE.md`, `.claude/rules/`, `.claude/settings.json` (including `env`), skills and agents are read from the worktree's checkout, in both the `.claude/worktrees/` layout and a sibling-directory layout. |
| Is the main checkout's `.claude/settings.local.json` inherited? | Yes. Its `env` values are visible inside the worktree. No `.worktreeinclude` entry is needed for it, and botPC-local settings therefore apply to every worktree of that repo. |
| Where does state live? | Transcripts: under the worktree's own project key (`~/.claude/projects/<repo-key>--claude-worktrees-pr-<n>/`). Auto Memory: under the repo key only; no `memory/` is created under worktree keys. `claude agents --json` never lists `-p` sessions. |
| What does `--worktree "#<n>"` create? | `<repo>/.claude/worktrees/pr-<n>` on branch `worktree-pr-<n>` at the PR head, locked with reason `claude session pr-<n> (pid ...)`. |
| Is it reused? | Yes, by `--resume <uuid>` and by new session ids alike. |
| Is the tip refreshed when the PR head moves? | No. After the PR branch advanced, later `--worktree "#<n>"` runs kept the old commit. Only `git worktree remove` followed by a fresh run recreated it at the new head (the `worktree-pr-<n>` branch name is reused). |
| Does `claude rm <uuid>` clean up a `-p` worktree? | No: "No job matching". `git worktree remove` is refused while the lock is present, even after the process exited; `git worktree unlock` first. The branch `worktree-pr-<n>` is left behind. |
| Can the session continue after its worktree is gone? | Yes. `claude -p --resume <uuid>` continues in the current directory and prints a notice that the worktree binding was cleared. |
| Does `--from-pr <n>` find the headless session that created the PR? | Not with `-p`: it started a new session, from the main checkout and from the creating worktree alike. The link exists as a `"type":"pr-link"` line (`sessionId`, `prNumber`, `prUrl`, `prRepository`) in the creating session's transcript; no separate index was observed. |
| Pitfall | Compound Bash commands (`a; b`) do not match a `Bash(a *)` allowlist pattern. Prompts should ask for one command per call. Two `--allowedTools` flags are merged. |

## 2. Lifecycle varjo must implement

Both strategies from O3 need the same three operations; only who creates the directory differs.

### Create
- `ClaudeManaged`: pass `--worktree "#<n>"` on the first run; read back `<repo>/.claude/worktrees/pr-<n>`; verify `git -C <path> rev-parse HEAD` equals the PR `head_sha` from the poller.
- `VarjoManaged`: `git fetch origin pull/<n>/head`, `git worktree add --detach <root>/<owner>/<repo>/pr-<n> FETCH_HEAD`, run `claude` with cwd = that path (no `--worktree`).

### Update to the PR head
- Trigger: the poller reports a new `head_sha` for a PR with an existing worktree.
- Never reset a dirty worktree (`git status --porcelain` non-empty): mark the session `failed` for human attention.
- Clean worktree: `git fetch origin pull/<n>/head` then `git -C <path> checkout --detach FETCH_HEAD` (VarjoManaged) or `git -C <path> merge --ff-only FETCH_HEAD` on `worktree-pr-<n>` (ClaudeManaged; if not fast-forwardable, remove and recreate as below).
- Only while no Claude process holds the session lease.

### Remove (PR merged or closed, after the grace period)
1. Stop any running process for the session.
2. `git worktree unlock <path>` (ignore "not locked"), `git worktree remove <path>` (never `--force` on a dirty tree; report instead).
3. `git branch -D worktree-pr-<n>` for ClaudeManaged worktrees.
4. `git worktree prune`.
5. Keep the transcript; raise `cleanupPeriodDays` so it outlives the PR. `claude purge <path>` only for the worktree key, never the repo key.

## 3. Consequences for the design

- `--from-pr` is not a lookup varjo can use from headless sessions. varjo's own `sessions` table (PR to session id) stays the source of truth, as planned in architecture.md §4.3 and §5.
- Memory is per repo, shared by all worktrees, which is what §4.8 (memory sync per repo key) assumes.
- Inheritance of `.claude/settings.local.json` means botPC-wide local settings (for example permission rules) apply inside every worktree. Keep botPC's local settings minimal and reviewed (#16).
- Recommendation for O3: `VarjoManaged`, because the tip must track the PR head exactly and the lock/branch cleanup is varjo's job either way. `ClaudeManaged` remains viable if the fast-forward step and the unlock/branch deletion are implemented.
