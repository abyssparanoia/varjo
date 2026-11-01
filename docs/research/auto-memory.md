# Claude Code auto memory and session portability (issue #4)

## Sources

- Memory docs: https://code.claude.com/docs/en/memory (section "Auto memory"), fetched 2026-11-01.
- Sessions docs: https://code.claude.com/docs/en/sessions (sections "Resume a session", "Where transcripts are stored", "Name the project directory yourself"), fetched 2026-11-01.
- Local inspection of `~/.claude/` on macOS, Claude Code CLI 2.1.306 (structure only; no memory or transcript contents were read).
- Labels used below: **documented** (from the docs), **observed** (seen on this machine), **unconfirmed** (inferred or untested).

## Auto memory: documented behavior

- On by default in local sessions. Toggle with `/memory`, which writes `autoMemoryEnabled` to `~/.claude/settings.json`. Per project: `autoMemoryEnabled: false` in project settings. Env override: `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`.
- Location: `~/.claude/projects/<project>/memory/`. The docs say `<project>` is "derived from the git repository", so all worktrees and subdirectories of one repo share one memory directory. Outside a git repo the project root is used.
- Relocation knobs (documented): `autoMemoryDirectory` in any settings scope (user/project/local/policy/`--settings`); `CLAUDE_CODE_PROJECT_DIR_NAME` (v2.1.234+, only honoured together with `CLAUDE_CONFIG_DIR`, 1-64 chars of letters/digits/`-`/`_`, must be set in the shell env, not settings) pins the `<project>` name regardless of cwd.
- Format: `MEMORY.md` index (one line per memory) plus one markdown topic file per memory. Frontmatter carries a `type` of `user`, `feedback`, `project` or `reference`. Claude Code stamps a `modified` ISO 8601 field when Claude writes a file with frontmatter.
- Loading: only the first 200 lines or 25 KB of `MEMORY.md` load at session start; topic files are read on demand. Not loaded into subagents (except forks).
- Relation to CLAUDE.md: complementary. CLAUDE.md is human-written instructions (loaded in full, up to 4 MiB); auto memory is Claude-written learnings. Claude skips things CLAUDE.md already says or that are derivable from code.
- Sync statement (documented): "Auto memory is machine-local ... Files are not shared across machines or cloud environments." Files are plain markdown that "you can edit or delete at any time"; no statement about external edits while a session runs. Memory files are excluded from the `cleanupPeriodDays` (default 30 days) transcript sweep.

## On-disk layout (observed on macOS, CLI 2.1.306)

```
~/.claude/
  settings.json  CLAUDE.md  rules/  skills/  agents/  plugins/  hooks/
  history.jsonl                  # global prompt history (display, project, sessionId, timestamp, pastedContents)
  projects/
    <project-key>/               # one per cwd/repo, 43 dirs here, 733 MB total
      <session-uuid>.jsonl       # one transcript per session
      <session-uuid>/            # per-session side data
        subagents/*  tool-results/*
      memory/                    # auto memory (22 of 43 projects have one)
        MEMORY.md                # index (25 lines in the largest example)
        <topic>.md               # frontmatter keys: name, description, metadata
  sessions/                      # NOT transcripts: <pid>.json + <pid>.<hash>.key (live-process/peer state)
  file-history/ checkpoints/ jobs/ session-env/ shell-snapshots/ paste-cache/ state/ ...
~/.claude.json                   # per-user global state; has a `projects` map (33 entries) keyed by path
```

- varjo's own dir (`-Users-abyssparanoia-workspace-go-github-com-abyssparanoia-varjo`, 7.4 MB) contains one `.jsonl`, one session subdir, and an empty `memory/` (no `MEMORY.md` yet).
- Largest memory dir seen: 24 files, 108 KB, `MEMORY.md` plus topic files. Memory sizes are tiny compared with transcripts; `jobs/` alone is 4 GB (not session data needed for resume, unconfirmed).
- No `todos/` or `statsig/` directory exists in 2.1.306 (older versions used them).
- `~/.claude.json` top-level keys include `projects`, `userID`, `machineID`, `oauthAccount`, `mcpServers`, `numStartups` and many caches. It holds auth identity and machine identity: never sync it. Its per-project entries hold trust/allowed-tools flags and last-session stats (`lastSessionId`, etc.).

## Project key derivation and the cross-machine problem

- Rule (documented for transcripts, observed for memory): the key is the working-directory absolute path with every non-alphanumeric character replaced by `-` (so `/`, `.`, `_` all become `-`). Example: `/Users/abyssparanoia/workspace/go/github.com/abyssparanoia/varjo` becomes `-Users-abyssparanoia-workspace-go-github-com-abyssparanoia-varjo`. Paths over 200 chars are truncated to 200 and a hash is appended. The encoding is lossy (`a.b` and `a-b` collide).
- The key depends on the absolute path, not on the git remote. Consequences:
  - Same absolute path on both Macs (same username, same clone location): identical key, so a copied `projects/<key>/` lines up with no changes. This is the easiest case.
  - Different path on botPC (different username or checkout dir): different key. Memory written on one machine is invisible to the other unless the directory is renamed/copied under the other key, symlinked, or `autoMemoryDirectory` is set.
  - Symlink options (unconfirmed, untested): make `~/.claude/projects/<botPC-key>` a symlink to the synced dir, or create the repo at the same absolute path via a symlink. Claude Code may resolve symlinks to real paths before deriving the key, which would defeat the second trick.
  - Better candidate: `autoMemoryDirectory` in user `~/.claude/settings.json`, pointing at a per-repo synced directory. Untested whether it is absolute-only or supports per-project values from user scope (docs say it is read from any scope, and project-scope values are trust-gated).
  - Alternative: `CLAUDE_CONFIG_DIR` + `CLAUDE_CODE_PROJECT_DIR_NAME` pins the key per launch; usable by a daemon that launches sessions, but it merges everything into one project dir unless set per repo.
- Worktrees: docs say worktrees share the main repo's memory dir. Observed: 8 worktree-named project dirs exist (e.g. `...--claude-worktrees-<name>`), none has a `memory/` subdir, which is consistent with memory being redirected to the main repo's key. Transcripts, however, get a per-worktree key (observed from dir naming), so worktree sessions are keyed by their own path.

## Session transcripts and resumability

- Documented: `~/.claude/projects/<project>/<session-id>.jsonl`; the entry format is internal and changes between versions. 30-day retention by default (`cleanupPeriodDays`). `claude --resume <absolute-transcript-path>` resumes from an arbitrary `.jsonl` path.
- Documented: `claude --resume <session-id>` searches the current project and its worktrees, then every other project on the machine; it resolves only when exactly one other project holds the ID, so a hand-copied duplicate yields "not found". Resuming a session whose original directory is missing resumes in the current directory. Sessions started via `claude -p`/SDK are hidden from the picker but resumable by ID.
- Observed: transcript filename is a UUID. Every record carries `sessionId`, and 448 records carry `cwd`, `gitBranch`, `version` (absolute paths and CLI version are embedded). There is no separate session index; record types include `ai-title`, `last-prompt`, `pr-link`, `permission-mode`, `file-history-snapshot`. The picker reads the transcript files directly (unconfirmed whether it also reads `~/.claude.json` `lastSessionId`).
- To resume on another machine (inferred, **unconfirmed**): copy `<session-uuid>.jsonl` into the matching `projects/<key-on-target>/`, ideally with the sibling `<session-uuid>/` dir (subagents, tool-results; needed if the session used them). Then `claude --resume <uuid>` from the repo checkout, or `--resume <path>`. Embedded `cwd` values pointing at the source machine's path may be ignored or cause tool-path confusion; untested. The same CLI version on both hosts is prudent.
- Not carried by transcripts: `--mcp-config`, `--settings`, `--plugin-dir`, `--add-dir` (documented); permission grants; file edits (the working tree must be synced through git); `file-history/` checkpoints (only needed for `/rewind`, unconfirmed).
- Do not resume the same session simultaneously on two machines: documented that concurrent writers interleave into one transcript. Concurrent sync of a growing `.jsonl` would also corrupt it.
- `history.jsonl` is a global prompt-history file (not required for resume; unconfirmed) and mixes all projects, so it is unsuitable for syncing.

## Sync candidates

| Option | Pros | Cons | Conflict behavior | Exclude |
|---|---|---|---|---|
| Git (private repo holding memory dirs) | History, diff, review, merges are textual; fits small markdown memory | Needs commit/pull discipline or hooks; transcripts are large and churn constantly; secrets in transcripts risk leaking | Textual merge conflicts, `MEMORY.md` index conflicts likely when both machines add lines | transcripts, `~/.claude.json`, caches |
| rsync over SSH | Simple, one-shot, scriptable, good for per-session copy on demand (e.g. for varjo "resume elsewhere") | One-directional unless run both ways; `--delete` can destroy data; needs reachability (see requirements §6) | None: last write wins by mtime/size, so use `--update`, no `--delete` | everything except `projects/<key>/memory/` or a named session's `.jsonl` + dir |
| Syncthing | Continuous, bidirectional, no server | Syncs half-written `.jsonl`; conflict files `*.sync-conflict-*` appear inside `memory/` and Claude may read them as notes; mixes machine-specific state | Keeps both versions as conflict copies | `.stignore`: `/*` then re-include only `projects/*/memory/**`; never `.claude.json`, `sessions/`, `jobs/`, `debug/`, `telemetry/`, `shell-snapshots/` |

## Recommendation for Phase 0 experiment #14

1. Memory: use identical absolute repo paths on both Macs if possible; otherwise set `autoMemoryDirectory`. Sync only `projects/<key>/memory/` (plain markdown, kilobytes). Start with rsync over SSH without `--delete` (or a private Git repo of the memory folder), one direction at a time, to observe how Claude reacts to externally edited files and whether it re-reads `MEMORY.md` mid-session.
2. Sessions: test a manual copy of one finished `<uuid>.jsonl` (plus `<uuid>/`) from machine A to B, under both an identical-path key and a different-path key, and record whether `claude --resume <uuid>` and `--resume <path>` work, what happens to `cwd`, and whether CLI version skew matters.
3. Record in the experiment: does the symlink approach (key dir to synced dir) survive path resolution; does `autoMemoryDirectory` in user settings accept a per-repo value.

## Constraints for the Phase 3 design

- Treat the project key as machine-local; the sync layer must map `repo identity (e.g. git remote) -> local key` per machine rather than copy key names blindly.
- Sync units: memory directory (small, merge-friendly, bidirectional OK) vs session bundle (`<uuid>.jsonl` + `<uuid>/`, large, single-writer, copy only when the session is idle/ended).
- Never sync `~/.claude.json`, `settings.json` machine specifics, credentials or `sessions/` (contain identity and live-process state). Scan transcripts and memory for secrets before leaving the machine.
- Transcript format is internal and versioned: pin or check the CLI version on both machines; do not parse transcripts, use `--resume`, `/export` or hooks (`transcript_path`, `SessionEnd`) to discover sessions.
- Handle retention (30-day sweep deletes old transcripts, not memory) and external-edit semantics of `MEMORY.md` (200 line / 25 KB load limit).
- Open: behavior of `autoMemoryDirectory`, symlinked projects, cross-path resume, and external edits during a live session remain unconfirmed until experiment #14.
