# varjo — Personal AI Development Agent

Requirements and implementation plan.

## 1. Project overview

### Purpose

Turn a spare MacBook Pro into an always-on AI agent execution environment (hereafter `botPC`) that automates software development work, with Claude Code Max as the core engine.

From the everyday MacBook Pro (hereafter `mainPC`) and from a smartphone, the user can check what the agent is doing, chat with it, give additional instructions, and approve actions.

The goal is to reproduce the experience of continuously running agents such as OpenAI's dots or Grok Bot, as far as possible using existing Claude Code features and a local environment.

### Guiding principles

- Use Claude Code as the AI execution engine.
- Make the most of the existing Claude Code Max subscription and avoid additional pay-as-you-go API costs wherever possible.
- All AI-driven code execution happens on botPC.
- Do not run a publicly exposed server on botPC.
- Make the system usable over the internet from mainPC and from a smartphone.
- Prefer existing GUIs such as Claude Desktop.
- Do not re-implement anything that an existing tool already provides.
- Keep the architecture extensible to services other than GitHub.

## 2. System components

### 2.1 botPC

A spare MacBook Pro used as a dedicated, always-on machine.

Requirements:

- Runs continuously with the lid closed
- Claude Code CLI / Desktop available
- Git, GitHub CLI, language toolchains, Docker, etc. available
- Manages multiple Git repositories
- Isolates work per PR using Git worktrees
- Runs event-monitoring continuously or on a schedule
- Starts Claude Code work when an event fires
- Persists work history, event state, and session information
- Recovers from reboots and transient network failures

Verify macOS clamshell behavior, sleep prevention, power, and thermal handling on real hardware.

### 2.2 mainPC

The everyday MacBook Pro.

Requirements:

- Uses Claude Desktop as the primary GUI
- Can operate Claude Code running on botPC
- Receives event notifications such as PR reviews
- Can read review summaries in the GUI
- Can jump from a notification to the related Claude Code session
- Can inspect the work and give additional instructions or approvals

A custom GUI is not required. If Claude Desktop provides a good enough experience, no custom GUI will be built.

### 2.3 Smartphone

Even when mainPC is unavailable, the user must be able to hand development tasks to botPC.

Expected flow:

1. Send an instruction to Claude from the smartphone
2. Work starts in the target repository on botPC
3. A new session is created if needed
4. The working environment is isolated with a Git worktree
5. Code is changed, tests run, commits made
6. A PR is created on GitHub
7. The result is checked on the smartphone

Claude Remote Control and Dispatch are the first options to investigate.

## 3. GitHub PR review automation

### 3.1 Scope

By default, GitHub PRs authored by the user.

Multiple repositories are supported.

### 3.2 Event detection

Detect the following:

- New PR reviews
- New inline review comments
- Replies to reviews
- Changes Requested
- Re-reviews
- Later: CI failures, Issue updates, etc.

Use the GitHub CLI or the GitHub API.

Start with polling. Do not run a public server for webhooks.

To prevent duplicate processing, persist GitHub event IDs, review IDs, comment IDs, and similar identifiers.

### 3.3 Review handling

When a new review is detected:

1. Fetch the target PR and related comments
2. Identify the target repository
3. Get the PR's head commit
4. Create or update a dedicated worktree as needed
5. Load the project's Claude configuration
6. Load the PR-specific handoff context
7. Analyze the review with Claude Code
8. Generate a summary and proposed response
9. Notify mainPC
10. Leave the related session ready for follow-up instructions

### 3.4 Review summary

At minimum, include:

- PR title, number, URL
- Reviewer
- Review timestamp
- Severity
- Summary of the findings
- Files that need changes
- Proposed approach
- Open questions and items that need human judgment

In the initial phase, do not auto-reply to reviews and do not auto-push.

## 4. Claude Code session management

### 4.1 Principles

Give each PR or event its own independent Claude Code working context.

For additional reviews on the same PR, reuse the existing working context where possible.

Track the mapping between PRs and sessions so that sessions do not grow without bound.

### 4.2 Required data

- Repository identifier
- PR number
- Target branch and commit SHA
- Worktree path
- Claude Code session ID
- Last processed event ID
- Last run timestamp
- Processing state
- Error information

### 4.3 GUI integration

Prefer official features such as Claude Desktop's SSH connection and Claude Remote Control.

Verify the following on real hardware:

- Whether CLI sessions auto-created on botPC appear in Claude Desktop on mainPC
- Whether a session can be resumed from the GUI
- Whether deep links to a session are available
- Whether the same session can be accessed from a smartphone

If this is not achievable, consider connecting via Remote Control or resuming by session ID.

A custom chat UI is the last resort.

## 5. Project configuration and context sharing

### 5.1 Existing project rules

The user commits the following to each Git repository:

- `CLAUDE.md`
- `.claude/rules/`
- Other project-specific configuration

botPC clones the same repositories, so these are used as-is.

### 5.2 Global configuration

On mainPC, some shared agent configuration lives under `~/.claude/`.

Share only the configuration that is needed with botPC.

Unconditional syncing of the whole directory, which contains credentials, session history, and machine-specific settings, is prohibited.

### 5.3 Claude Auto Memory

Project-specific knowledge that Claude Code learned on mainPC should also be usable on botPC.

Investigate:

- Auto Memory under `~/.claude/projects/`
- How Claude Code identifies a project
- Memory sharing across worktrees
- Whether syncing between different Macs is feasible
- Conflicts on concurrent edits
- Format changes across Claude Code updates

For the initial implementation, consider selectively sharing the required memory files.

Candidates: Syncthing, Git, rsync over SSH.

### 5.4 Per-PR context handoff

Separately from Auto Memory, generate handoff information when a PR is created.

Example:

```markdown
# PR Context

## Goal
What this implementation is for

## Design Decisions
The design that was adopted and why

## Alternatives
Options considered but not adopted

## Constraints
Parts that must not be changed

## Known Issues
Known problems and follow-up work

## Verification
Tests that were run and their results

## Original Session
Identifier of the original Claude Code session
```

Consider generating this automatically with a Claude Code Skill or Hook at PR creation time.

Associate handoff information with the PR number, repository, and commit SHA.

Compare the GitHub PR body, a dedicated Git-tracked file, and private storage, and choose a storage location considering confidentiality and ease of management.

## 6. Remote access and security

### Hard requirements

- No publicly exposed inbound ports on botPC
- No admin UI exposed on a public IP
- No copying of credentials to a custom cloud server
- Least-privilege access to GitHub and AWS
- Direct pushes to `main` are prohibited
- Changes to production environments require human approval by default
- Clearly define how code and secrets sent to external services are handled

### Connectivity candidates

In order of preference:

1. Claude Remote Control
2. Claude Desktop's official SSH connection
3. Private connectivity via Tailscale

If Tailscale is used, configure access control explicitly.

Note that Claude Code inference runs in Anthropic's cloud, so code and context are sent to the model even when executing locally.

## 7. Future extensions

### Jira

- Detect Issue and comment updates
- Summarize content
- Convert into implementation tasks
- Create related Claude Code sessions
- Link to GitHub PRs

### Browser automation

Target systems that have no API.

Investigate Playwright and Claude in Chrome.

Use a logged-in browser on botPC and drive it from Claude Code when needed.

### Other

- Investigating CI failures
- Analyzing AWS logs
- Drafting Terraform changes
- Implementing from GitHub Issues
- Periodic repository maintenance

Event ingestion must be extensible via an adapter pattern.

## 8. Recommended technology stack

| Purpose | First choice |
|---|---|
| AI execution | Claude Code CLI |
| GUI | Claude Desktop |
| Smartphone control | Remote Control / Dispatch |
| GitHub integration | gh CLI / GitHub API |
| Work isolation | Git worktree |
| Scheduling | macOS launchd |
| Event state | SQLite |
| Connectivity | Official Claude features / Tailscale |
| Memory sharing | Git / rsync / Syncthing |
| Browser automation | Playwright / Claude in Chrome |

TypeScript is the first-choice implementation language, but if existing tools or shell scripts are sufficient, do not build a new application.

## 9. Implementation phases

### Phase 0 — Technical validation

Verify the following first:

- [ ] botPC runs stably with the lid closed
- [ ] Claude Code works with Max subscription authentication
- [ ] Claude Desktop on mainPC can connect to botPC
- [ ] Claude Code can run in repositories on botPC
- [ ] Auto-created sessions can be viewed and resumed from the GUI
- [ ] Claude Code on botPC can be operated from a smartphone
- [ ] Dispatch can use botPC's local environment
- [ ] An Auto Memory sharing approach is viable
- [ ] `CLAUDE.md` and rules apply correctly inside Git worktrees

### Phase 1 — GitHub review MVP

- [ ] Monitor the user's own PRs
- [ ] Detect new reviews
- [ ] Prevent duplicate event processing
- [ ] Create a worktree for the target PR
- [ ] Summarize reviews with Claude Code
- [ ] Notify mainPC
- [ ] Interact from Claude Desktop

### Phase 2 — Review fixes

- [ ] Keep a working context per PR
- [ ] Change code from fix instructions
- [ ] Run tests
- [ ] Review the diff
- [ ] Push after human approval

### Phase 3 — Context sharing

- [ ] Safe sharing of Auto Memory
- [ ] Generate handoff on PR creation
- [ ] Hand off from mainPC to botPC
- [ ] Hand results back from botPC to mainPC

### Phase 4 — Mobile and external integrations

- [ ] Request new development tasks from a smartphone
- [ ] Implement, test, and create PRs on botPC
- [ ] Jira event integration
- [ ] Browser automation integration

## 10. Non-functional requirements

- Event processing must be idempotent
- Prevent concurrent writes to the same PR
- Failed processing can be retried
- Monitoring recovers after botPC reboots
- Stop gracefully when the Claude Code usage limit is exceeded
- Unused Claude sessions and worktrees can be cleaned up
- Execution logs and errors are traceable
- Arbitrary command execution from the outside is not allowed
- External input such as PR comments is treated as untrusted data
- Automation runs with the minimum required privileges

## 11. Instructions to Claude Code

Based on this document, start with technical research and design.

**Do not implement every feature at once.**

Do the following first:

1. Survey the latest official Claude Code documentation.
2. Check the specifications of Claude Desktop's SSH connection, Remote Control, and Dispatch.
3. Confirm which automation approaches and usage limits apply under the Claude Code Max subscription.
4. Verify the feasibility of session creation, resumption, and GUI visibility.
5. Investigate the Auto Memory storage format and how to share it between machines.
6. Identify which parts can be covered by existing tools.
7. Decide the minimal set of components that must be built.
8. Write the Phase 0 validation plan and concrete steps.
9. Where things are unclear or multiple options exist, confirm with the user before implementing.

In particular, treat **whether a Claude Code session auto-created from the CLI can be viewed and continued from Claude Desktop on another machine** as the top-priority validation item.

If this is not feasible, propose an alternative architecture built on existing official features.

The end goal is not to build a custom AI agent platform.

**It is to reuse Claude Code as much as possible and build only the minimum necessary event monitoring, context sharing, and orchestration.**
