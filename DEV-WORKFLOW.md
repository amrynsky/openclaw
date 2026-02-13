# OpenClaw Development Workflow & Agent Integration

## Overview

OpenClaw uses **AI agents as first-class participants** in the development workflow. Agents don't just assist developers — they execute structured, multi-step workflows with safety guardrails, deterministic artifact handoff, and maintainer checkpoints. The core philosophy:

> "Skills execute workflow. Maintainers provide judgment."

This document covers: the PR lifecycle pipeline, GitHub Copilot model integration, the coding agent skill, CI/CD automation, documentation agents, upstream sync, and the tool/skill architecture that makes it all work.

---

## The PR Lifecycle Pipeline (Three-Skill System)

The centerpiece of the dev workflow is a **three-skill pipeline** that processes pull requests from review through merge. Each skill is a prompt-based `SKILL.md` file that the agent follows step-by-step, invoking wrapper scripts for deterministic execution.

```
 ┌─────────────────────── PR Lifecycle Pipeline ──────────────────────────┐
 │                                                                        │
 │   PR #123 arrives                                                      │
 │         │                                                              │
 │         ▼                                                              │
 │   ┌─────────────────────────────────────────────┐                      │
 │   │  STEP 1: /review-pr                         │                      │
 │   │  (read-only analysis)                       │                      │
 │   │                                             │                      │
 │   │  1. scripts/pr-review <PR>                  │                      │
 │   │     → Creates .worktrees/pr-<PR>            │                      │
 │   │     → Fetches PR metadata via gh CLI        │                      │
 │   │  2. Analyze diff, tests, docs, changelog    │                      │
 │   │  3. Claim PR: gh pr edit --add-assignee     │                      │
 │   │  4. Write structured outputs:               │                      │
 │   │     .local/review.md   (human-readable)     │                      │
 │   │     .local/review.json (machine-readable)   │                      │
 │   │  5. scripts/pr review-validate-artifacts    │                      │
 │   │                                             │                      │
 │   │  Safety: Never modifies code, never pushes  │                      │
 │   └──────────────┬──────────────────────────────┘                      │
 │                  │                                                      │
 │       ┌──────────▼──────────┐                                          │
 │       │ MAINTAINER PAUSE    │  Evaluate: Is this the right solution?   │
 │       │                     │  Can we fix everything in prepare-pr?    │
 │       │ Questions to ask:   │  Any unresolved questions?               │
 │       │ 1. Problem real?    │                                          │
 │       │ 2. Optimal impl?   │                                          │
 │       │ 3. Fixable in prep?│                                          │
 │       └──────────┬──────────┘                                          │
 │                  │ (maintainer approves)                                │
 │                  ▼                                                      │
 │   ┌─────────────────────────────────────────────┐                      │
 │   │  STEP 2: /prepare-pr                        │                      │
 │   │  (make PR merge-ready)                      │                      │
 │   │                                             │                      │
 │   │  1. scripts/pr-prepare init <PR>            │                      │
 │   │  2. Read .local/review.json findings        │                      │
 │   │  3. Resolve all BLOCKER + IMPORTANT items   │                      │
 │   │  4. Rebase onto main                        │                      │
 │   │  5. Update changelog/docs if needed         │                      │
 │   │  6. Commit via scripts/committer:           │                      │
 │   │     "fix: <summary> (openclaw#<PR>)         │                      │
 │   │      thanks @<pr-author>"                   │                      │
 │   │  7. Run gates:                              │                      │
 │   │     pnpm build → pnpm check → pnpm test    │                      │
 │   │  8. Push safely:                            │                      │
 │   │     --force-with-lease against known SHA    │                      │
 │   │     auto-retry on lease conflict            │                      │
 │   │  9. Write .local/prep.env with HEAD SHA     │                      │
 │   │                                             │                      │
 │   │  Safety: Never pushes to main,              │                      │
 │   │  never runs gh pr merge                     │                      │
 │   └──────────────┬──────────────────────────────┘                      │
 │                  │                                                      │
 │       ┌──────────▼──────────┐                                          │
 │       │ MAINTAINER PAUSE    │  Review the actual code changes.         │
 │       │                     │  Check types, security, test coverage.   │
 │       └──────────┬──────────┘                                          │
 │                  │ (maintainer approves)                                │
 │                  ▼                                                      │
 │   ┌─────────────────────────────────────────────┐                      │
 │   │  STEP 3: /merge-pr                          │                      │
 │   │  (deterministic squash merge)               │                      │
 │   │                                             │                      │
 │   │  1. scripts/pr-merge verify <PR>            │                      │
 │   │     → Check all artifacts present           │                      │
 │   │     → Validate CI checks green              │                      │
 │   │     → Verify branch not behind main         │                      │
 │   │  2. scripts/pr-merge run <PR>               │                      │
 │   │     → Squash merge pinned to PREP_HEAD_SHA  │                      │
 │   │     → Add co-author trailers:               │                      │
 │   │       Co-authored-by: PR Author <email>     │                      │
 │   │       Co-authored-by: Reviewer <email>      │                      │
 │   │     → Post PR comment with SHA hashes       │                      │
 │   │  3. Cleanup worktree after MERGED state     │                      │
 │   │                                             │                      │
 │   │  Safety: --match-head-commit required,      │                      │
 │   │  never uses --auto, never runs git push     │                      │
 │   └─────────────────────────────────────────────┘                      │
 └────────────────────────────────────────────────────────────────────────┘
```

### Structured Review Handoff

The key innovation is the **machine-readable handoff** between skills. `/review-pr` produces `.local/review.json` with a strict schema:

```json
{
  "recommendation": "READY FOR /prepare-pr",
  "findings": [
    {
      "id": "F1",
      "severity": "BLOCKER",
      "title": "Missing input validation on user-supplied regex",
      "area": "src/channels/plugins/discord.ts",
      "fix": "Add regex safety check before new RegExp()"
    },
    {
      "id": "F2",
      "severity": "IMPORTANT",
      "title": "Missing changelog entry",
      "area": "CHANGELOG.md",
      "fix": "Add a Fixes entry for PR #123"
    }
  ],
  "tests": {
    "ran": ["pnpm test -- --grep discord"],
    "gaps": ["No test for malformed regex input"],
    "result": "pass"
  },
  "docs": "up_to_date",
  "changelog": "required"
}
```

Severity levels control what `/prepare-pr` must fix:
- **BLOCKER**: Must be resolved before merge — security issues, logic errors, broken tests
- **IMPORTANT**: Should be resolved — missing tests, docs gaps, changelog
- **NIT**: Optional improvements — style, naming, minor refactors

Recommendations:
- `READY FOR /prepare-pr` — proceed to preparation
- `NEEDS WORK` — send back to PR author
- `NEEDS DISCUSSION` — requires maintainer input
- `NOT USEFUL` — close PR

### Required Artifacts

Each skill produces artifacts consumed by the next:

| Artifact | Producer | Consumer | Purpose |
|----------|----------|----------|---------|
| `.local/pr-meta.json` | review-pr | prepare-pr, merge-pr | PR number, author, head SHA, base branch |
| `.local/pr-meta.env` | review-pr | prepare-pr, merge-pr | Shell-sourceable PR metadata |
| `.local/review.md` | review-pr | maintainer | Human-readable review (sections A-J) |
| `.local/review.json` | review-pr | prepare-pr | Machine-readable findings + recommendation |
| `.local/prep-context.env` | prepare-pr | merge-pr | Preparation context |
| `.local/prep.md` | prepare-pr | maintainer | Resolution report + gate results |
| `.local/prep.env` | prepare-pr | merge-pr | `PREP_HEAD_SHA` for merge pinning |

### Wrapper Scripts

The skills invoke wrapper scripts that encapsulate git and GitHub operations:

```bash
# Review phase
scripts/pr-review <PR>                    # Full review setup (worktree + metadata)
scripts/pr review-checkout-main <PR>      # Switch to main baseline for comparison
scripts/pr review-checkout-pr <PR>        # Switch to PR head for analysis
scripts/pr review-guard <PR>              # Branch guard before writing outputs
scripts/pr review-validate-artifacts <PR> # Semantic validation of review outputs

# Prepare phase
scripts/pr-prepare init <PR>              # Load review artifacts + setup
scripts/pr-prepare validate-commit <PR>   # Verify commit subject format
scripts/pr-prepare gates <PR>             # Run build/check/test gates
scripts/pr-prepare push <PR>              # Safe push with lease + retry
scripts/pr-prepare run <PR>               # One-shot: all of the above

# Merge phase
scripts/pr-merge verify <PR>              # Pre-merge validation only
scripts/pr-merge run <PR>                 # Deterministic squash merge
```

All wrappers are **cwd-agnostic** — they work from both the repo root and the PR worktree.

### Push Safety Model

The `/prepare-pr` push sequence has multiple safety layers:

```
1. Pre-push: Read remote HEAD SHA, verify it matches expected SHA
2. Push with --force-with-lease against known SHA
3. If lease conflict:
   a. Fetch new remote HEAD
   b. Rebase onto new head
   c. Re-run gates (build/check/test)
   d. Retry push (one attempt only)
4. Post-push: Retry PR API to verify head propagated
```

This prevents race conditions when multiple maintainers or CI jobs are active.

### Triage Order

PRs are processed **oldest to newest**. Rationale: older PRs are more likely to have merge conflicts and stale dependencies. Resolving them first prevents snowballing rebase pain across the queue.

---

## GitHub Copilot Integration

**Location**: `src/providers/github-copilot-auth.ts`, `src/providers/github-copilot-token.ts`, `src/providers/github-copilot-models.ts`

OpenClaw integrates with **GitHub Copilot** as an LLM model provider, allowing users with active Copilot subscriptions to use Copilot-backed models for agent conversations.

```
 ┌────────────── GitHub Copilot Auth Flow ──────────────┐
 │                                                       │
 │   User runs auth setup                                │
 │         │                                             │
 │         ▼                                             │
 │   GitHub Device Flow OAuth                            │
 │   CLIENT_ID = "Iv1.b507a08c87ecfe98"                  │
 │   1. POST github.com/login/device/code                │
 │      → Get user_code + verification_uri               │
 │   2. User visits github.com/login/device              │
 │      → Enters code, approves                          │
 │   3. Poll github.com/login/oauth/access_token         │
 │      → Receive access_token                           │
 │         │                                             │
 │         ▼                                             │
 │   Token Exchange                                      │
 │   POST api.github.com/copilot_internal/v2/token       │
 │      → Copilot API token                              │
 │      → API base URL (from token's proxy-ep)           │
 │      → Cached with 5-min safety margin                │
 │         │                                             │
 │         ▼                                             │
 │   Auth Profile Storage                                │
 │   $STATE_DIR/credentials/github-copilot.token.json    │
 │   Used as OpenAI-compatible provider                  │
 │   (api: "openai-responses")                           │
 └───────────────────────────────────────────────────────┘
```

**Supported Copilot models:**

| Model | Context Window | Max Output Tokens |
|-------|---------------|-------------------|
| `gpt-4o` | 128K | 8192 |
| `gpt-4.1` | 128K | 8192 |
| `gpt-4.1-mini` | 128K | 8192 |
| `gpt-4.1-nano` | 128K | 8192 |
| `o1` | 128K | 8192 |
| `o1-mini` | 128K | 8192 |
| `o3-mini` | 128K | 8192 |

**Integration pattern**: Copilot models are registered as `openai-responses` API backends with the `github-copilot` provider ID. The pi-ai runtime attaches Copilot-specific headers automatically. This means Copilot models seamlessly participate in agent model rotation and failover — if Copilot rate-limits, the agent falls back to configured API keys.

---

## Coding Agent Skill

**Location**: `skills/coding-agent/SKILL.md`

The coding agent skill enables the OpenClaw agent to **orchestrate other AI coding agents** (Codex, Claude Code, OpenCode, Pi Coding Agent) as background processes. This creates a "manager-worker" pattern where the OpenClaw agent delegates focused tasks to specialized coding agents.

```
 ┌──────────────── Coding Agent Orchestration ──────────────────┐
 │                                                               │
 │   OpenClaw Agent (orchestrator)                               │
 │         │                                                     │
 │         │ bash pty:true background:true                       │
 │         │ command:"codex exec 'Fix the auth bug'"             │
 │         │                                                     │
 │         ▼                                                     │
 │   ┌──────────────────────────────────────┐                    │
 │   │  Background PTY Session              │                    │
 │   │  sessionId: abc123                   │                    │
 │   │                                      │                    │
 │   │  ┌──────────┐  ┌──────────────────┐  │                    │
 │   │  │ Codex    │  │ Working Dir      │  │                    │
 │   │  │ CLI      │  │ ~/project        │  │                    │
 │   │  │ (--full- │  │                  │  │                    │
 │   │  │  auto)   │  │ Reads/writes     │  │                    │
 │   │  └──────────┘  │ files, runs      │  │                    │
 │   │                │ tests, commits   │  │                    │
 │   │                └──────────────────┘  │                    │
 │   └──────────────────────────────────────┘                    │
 │         │                                                     │
 │         │ process action:log sessionId:abc123                 │
 │         │ (monitor progress)                                  │
 │         │                                                     │
 │         │ process action:submit sessionId:abc123 data:"y"     │
 │         │ (approve actions)                                   │
 │         │                                                     │
 │         │ process action:kill sessionId:abc123                │
 │         │ (terminate when done)                               │
 └───────────────────────────────────────────────────────────────┘
```

**Supported coding agents:**

| Agent | Command | Key Flags | Best For |
|-------|---------|-----------|----------|
| **Codex CLI** | `codex exec` | `--full-auto`, `--yolo` | Autonomous code generation (GPT-5.2-codex) |
| **Claude Code** | `claude` | — | Anthropic Claude-powered coding |
| **OpenCode** | `opencode` | — | Alternative open-source coding agent |
| **Pi Coding Agent** | `pi` | `--prompt-caching` | Prompt caching for efficiency |

**Critical**: All coding agents require `pty:true` (pseudo-terminal allocation) because they are interactive terminal applications. Without PTY, output breaks or the agent hangs.

**Bash tool parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `command` | string | Shell command to run |
| `pty` | boolean | **Required for coding agents** — allocates pseudo-terminal |
| `workdir` | string | Working directory (agent sees only this folder) |
| `background` | boolean | Run in background, returns sessionId |
| `timeout` | number | Kill process after N seconds |
| `elevated` | boolean | Run on host instead of sandbox |

**Process monitoring actions:**

| Action | Purpose |
|--------|---------|
| `process action:list` | List all running/recent sessions |
| `process action:poll sessionId:XXX` | Check if session is still running |
| `process action:log sessionId:XXX` | Get session output (offset/limit) |
| `process action:write sessionId:XXX data:"y"` | Send raw data to stdin |
| `process action:submit sessionId:XXX data:"yes"` | Send data + Enter |
| `process action:send-keys sessionId:XXX` | Send key tokens or hex bytes |
| `process action:kill sessionId:XXX` | Terminate session |

**Parallel PR review pattern** (agent army):

```bash
# Deploy multiple Codex instances - one per PR
bash pty:true workdir:~/project background:true \
  command:"codex exec 'Review PR #86. git diff origin/main...origin/pr/86'"
bash pty:true workdir:~/project background:true \
  command:"codex exec 'Review PR #87. git diff origin/main...origin/pr/87'"

# Monitor all running agents
process action:list

# Post results to GitHub
gh pr comment 86 --body "<review content>"
```

---

## GitHub Skill

**Location**: `skills/github/SKILL.md`

The GitHub skill teaches the agent to use the `gh` CLI for all GitHub interactions:

- **PRs**: `gh pr checks`, `gh pr view`, `gh pr diff`, `gh pr merge`
- **CI**: `gh run list`, `gh run view`, `gh run view --log-failed`
- **Issues**: `gh issue list`, `gh issue view`, `gh issue create`
- **API**: `gh api repos/{owner}/{repo}/...` for advanced queries

**Dependency**: Requires `gh` CLI installed (auto-installable via brew or apt).

---

## CI/CD Pipeline

**Location**: `.github/workflows/`

### Main CI (`ci.yml`)

The CI pipeline uses **smart scope detection** to skip expensive jobs when only docs or unrelated areas change:

```
 ┌──────────────────── CI Pipeline ────────────────────────────┐
 │                                                              │
 │   Push to main or PR                                         │
 │         │                                                    │
 │         ▼                                                    │
 │   ┌──────────────────────┐                                   │
 │   │  docs-scope          │  Detect docs-only changes         │
 │   │  (always runs)       │  .github/actions/detect-docs      │
 │   └──────────┬───────────┘                                   │
 │              │                                               │
 │    ┌─────── docs_only? ────────┐                             │
 │    │ yes                   no  │                              │
 │    ▼                       ▼   │                              │
 │  Skip heavy         ┌─────────────────────┐                  │
 │  jobs               │  changed-scope      │                  │
 │                      │  Detect touched     │                  │
 │                      │  areas:             │                  │
 │                      │  • run_node?        │                  │
 │                      │  • run_macos?       │                  │
 │                      │  • run_android?     │                  │
 │                      └──────────┬──────────┘                  │
 │                                 │                             │
 │   ┌────────────────────────────┼──────────────────────────┐  │
 │   │                            │                          │  │
 │   ▼                            ▼                          ▼  │
 │ ┌──────────┐  ┌──────────────────────┐  ┌──────────────┐    │
 │ │ lint +   │  │ test + build          │  │ Platform     │    │
 │ │ format   │  │ (Node, Bun, Windows) │  │ (macOS,      │    │
 │ │ (always) │  │ (if run_node)        │  │  Android)    │    │
 │ │ pnpm     │  │ vitest, tsdown       │  │ (if touched) │    │
 │ │ check    │  │                      │  │              │    │
 │ └──────────┘  └──────────────────────┘  └──────────────┘    │
 │                                                              │
 │ Also: detect-secrets, protocol conformance, canvas bundle    │
 └──────────────────────────────────────────────────────────────┘
```

**Concurrency**: Per-PR cancellation — new pushes to a PR cancel the previous CI run.

**Fail-safe**: If scope detection fails, all jobs run (conservative default).

### Other Workflows

| Workflow | Purpose |
|----------|---------|
| `auto-response.yml` | Auto-respond to labeled issues/PRs (support, skill requests, TestFlight, third-party extensions) |
| `labeler.yml` | Automatic PR/issue labeling based on changed files |
| `stale.yml` | Mark old issues as stale |
| `formal-conformance.yml` | Protocol conformance checks |
| `docker-release.yml` | Container release pipeline |
| `install-smoke.yml` | Tests installer on fresh systems |
| `workflow-sanity.yml` | Validates workflow correctness |

### Custom GitHub Actions

| Action | Purpose |
|--------|---------|
| `.github/actions/setup-node-env/` | Node.js + pnpm + Bun setup |
| `.github/actions/setup-pnpm-store-cache/` | Cache optimization for CI |
| `.github/actions/detect-docs-changes/` | Smart CI gating for docs-only PRs |

### Dependabot

Configured via `.github/dependabot.yml` for automatic dependency updates.

---

## Documentation Agent (Mintlify Skill)

**Location**: `.agents/skills/mintlify/SKILL.md`

The Mintlify skill enables the agent to build and maintain documentation sites. It teaches the agent to:

- Understand the `docs/docs.json` configuration (navigation, theme, API specs)
- Write content in MDX with YAML frontmatter
- Use Mintlify built-in components (Cards, Tabs, Accordions, CodeGroups)
- Follow naming conventions and navigation structure
- Avoid custom components in favor of built-in ones

The docs site lives in `docs/` and is built with Mintlify's documentation platform.

---

## Upstream Sync Workflow

**Location**: `.agent/workflows/update_clawdbot.md`

For forks that diverge from upstream, a documented workflow guides the agent through synchronization:

```
1. Assess divergence
   git fetch upstream
   git rev-list --left-right --count main...upstream/main

2. Decision: Rebase (linear history) vs Merge (preserves history)
   - Few local commits → Rebase
   - Many local commits → Merge

3. Handle conflicts (common patterns)
   - package.json: Take upstream deps, keep local scripts
   - pnpm-lock.yaml: Accept upstream, regenerate
   - *.patch files: Usually take upstream
   - Source files: Merge carefully, prefer upstream structure

4. Post-sync verification
   pnpm install → pnpm build → scripts/restart-mac.sh

5. Swift 6.2 compatibility check
   grep -r "FileManager\.default\|Thread\.isMainThread" src/ apps/ --include="*.swift"
```

---

## Agent Tool Architecture

The agent's dev capabilities are built on a layered tool system:

```
 ┌───────────────── Agent Tool Layers ─────────────────────┐
 │                                                          │
 │  Layer 1: Core Tools (always available)                  │
 │  ├── bash/exec   — Shell command execution               │
 │  ├── process     — Background process management (PTY)   │
 │  ├── read        — File reading                          │
 │  ├── write       — File writing                          │
 │  ├── edit        — Diff-based code editing               │
 │  └── apply-patch — Patch application                     │
 │                                                          │
 │  Layer 2: OpenClaw Tools (gateway-provided)              │
 │  ├── config      — Get/apply/patch configuration         │
 │  ├── sessions    — Session management + spawning         │
 │  ├── subagent    — Cross-agent delegation                │
 │  └── restart     — Gateway restart                       │
 │                                                          │
 │  Layer 3: Skills (prompt-injected capabilities)          │
 │  ├── github      — gh CLI for PRs, issues, CI            │
 │  ├── coding-agent— Orchestrate Codex/Claude/OpenCode     │
 │  ├── review-pr   — Structured PR analysis                │
 │  ├── prepare-pr  — PR preparation + push                 │
 │  ├── merge-pr    — Deterministic squash merge             │
 │  ├── mintlify    — Documentation generation              │
 │  └── 50+ more    — Domain-specific skills                │
 │                                                          │
 │  Layer 4: Channel Tools (plugin-provided)                │
 │  ├── discord     — Message/moderation tools              │
 │  ├── telegram    — Message management                    │
 │  └── ...         — Channel-specific actions              │
 │                                                          │
 │  Policy Layer: Controls what's available                 │
 │  ├── Tool allowlists/denylists per agent                 │
 │  ├── Sandbox execution policy                            │
 │  ├── Group-level restrictions                            │
 │  └── Subagent delegation restrictions                    │
 └──────────────────────────────────────────────────────────┘
```

### Skills Loading Pipeline

Skills are discovered and loaded at agent startup:

```
1. Scan skill sources:
   - Bundled: skills/ (80+ built-in skills)
   - Workspace: .agents/skills/ (project-specific)
   - Plugins: extensions/*/skills/ (extension-provided)

2. Filter by eligibility:
   - Check required binaries (e.g., gh, codex, claude)
   - Check platform requirements (e.g., Node.js)
   - Apply agent-level skill allowlists

3. Build skill snapshot:
   buildWorkspaceSkillSnapshot() → cached in memory

4. Inject into system prompt:
   buildWorkspaceSkillsPrompt() → appended to agent context

5. Apply environment:
   applySkillEnvOverrides() → set PATH, env vars
```

**Gateway skill methods:**

| Method | Purpose |
|--------|---------|
| `skills.status` | Health report (installed, missing, eligibility) |
| `skills.bins` | List available skill binaries across workspaces |
| `skills.install` | Install skill dependency (npm/apt/brew) |
| `skills.update` | Fetch/sync skill updates |

---

## Multi-Agent Safety Conventions

When multiple agents work concurrently (e.g., parallel PR reviews), strict safety rules apply:

| Rule | Rationale |
|------|-----------|
| Do NOT create/drop git stash | Other agents may be working with stashed changes |
| Do NOT create/remove git worktrees | Unless explicitly requested — other agents may use them |
| Do NOT switch branches | Unless requested — could disrupt concurrent work |
| Commit only YOUR changes | Do not stage others' work-in-progress |
| Unrecognized files? Continue | Focus on your edits, don't investigate |
| Use `scripts/committer` wrapper | Manages staging safety, not bare `git add` |

---

## Quality Bar (Enforced by Agent Skills)

The PR workflow enforces a strict quality bar at every step:

### Review Quality
- Validate with reproducible problem + tested fix
- Do not trust PR code by default
- Analyze whether the PR is the most optimal implementation
- Check for reuse of canonical sources of truth

### Code Quality
- Strict types — no `any` in implementation code
- Validate external-input boundaries (CLI, env, network, tool output)
- Fix root causes, not symptoms
- Harden against security + abuse paths

### Gate Requirements
- `pnpm build` — TypeScript compilation
- `pnpm check` — Oxlint + Oxfmt (lint + format)
- `pnpm test` — Vitest full suite (skip only for docs-only PRs)

### Commit Format
```
fix: <summary> (openclaw#<PR>) thanks @<pr-author>

Co-authored-by: PR Author <author@example.com>
Co-authored-by: Reviewer <reviewer@example.com>
```

### Changelog Rules
- Keep latest released version at top (no "Unreleased" section)
- Add PR number and contributor thanks
- Pure test additions don't need changelog unless user-facing
- After publishing: bump version, start new top section

---

## Typical Workflow Examples

### Example 1: Complete PR Processing

```
Maintainer: "/review-pr 123"

Agent:
  1. scripts/pr-review 123
     → Creates .worktrees/pr-123 from origin/main
     → Fetches: gh pr view 123 --json title,body,author,...
     → Reads diff: gh pr diff 123
     → Claims: gh pr edit 123 --add-assignee <me>
  2. Analyzes code changes against main baseline
  3. Runs relevant tests in worktree
  4. Writes:
     .local/review.md  → Sections A-J with analysis
     .local/review.json → 3 findings (1 BLOCKER, 1 IMPORTANT, 1 NIT)
     recommendation: "READY FOR /prepare-pr"

Maintainer reviews findings, agrees with approach.

Maintainer: "/prepare-pr 123"

Agent:
  1. scripts/pr-prepare init 123
  2. Reads review.json, identifies BLOCKER + IMPORTANT
  3. Fixes regex injection vulnerability (BLOCKER)
  4. Adds missing changelog entry (IMPORTANT)
  5. Commits: "fix: add regex validation for user patterns (openclaw#123) thanks @contributor"
  6. Runs: pnpm build ✓ → pnpm check ✓ → pnpm test ✓
  7. Pushes with --force-with-lease to PR branch

Maintainer reviews code changes, verifies fix quality.

Maintainer: "/merge-pr 123"

Agent:
  1. scripts/pr-merge verify 123
     → All artifacts present ✓
     → CI checks green ✓
     → Not behind main ✓
  2. scripts/pr-merge run 123
     → Squash merge pinned to PREP_HEAD_SHA
     → Co-author trailers added
     → PR comment posted with SHA
     → Worktree cleaned up
```

### Example 2: Parallel PR Review (Agent Army)

```
Maintainer: "Review PRs 86, 87, and 88"

Agent (orchestrator):
  # Create isolated worktrees + launch Codex per PR
  bash pty:true background:true workdir:.worktrees/pr-86 \
    command:"codex exec 'Review this PR diff against main'"
  bash pty:true background:true workdir:.worktrees/pr-87 \
    command:"codex exec 'Review this PR diff against main'"
  bash pty:true background:true workdir:.worktrees/pr-88 \
    command:"codex exec 'Review this PR diff against main'"

  # Monitor all three
  process action:list
  # Wait for completion, collect results
  process action:log sessionId:XXX

  # Post consolidated findings
  gh pr comment 86 --body "<review-86>"
  gh pr comment 87 --body "<review-87>"
  gh pr comment 88 --body "<review-88>"
```

---

## Key Design Insights

### 1. Agent-Assisted, Maintainer-Controlled
The system explicitly separates execution (agent) from judgment (maintainer). Mandatory pauses between skills ensure humans evaluate technical direction, not just command success. Agents propose; maintainers decide.

### 2. Deterministic Artifact Pipeline
Every step produces machine-readable artifacts (JSON, ENV) with strict schemas. This eliminates ambiguity in handoffs between skills and enables verification at each stage.

### 3. Script-Wrapped Safety
Instead of having agents run raw git commands, all critical operations go through wrapper scripts (`scripts/pr-*`, `scripts/committer`) that enforce safety invariants — lease-based pushing, SHA pinning, artifact validation.

### 4. Skills as Prompts, Not Code
The PR workflow, coding agent orchestration, and GitHub interaction are all implemented as `SKILL.md` prompt files, not code plugins. This makes them trivially auditable, versionable, and modifiable by maintainers who understand the workflow.

### 5. Copilot as a Model Provider
GitHub Copilot isn't used as a code completion tool — it's integrated as a full model provider for agent conversations. This means Copilot-powered models participate in the same failover, rotation, and auth profile system as Anthropic, OpenAI, and Google models.

### 6. Worktree Isolation
All PR work happens in isolated git worktrees (`.worktrees/pr-<PR>`), keeping the main working directory clean. This is essential for parallel processing and prevents interference between concurrent agent workflows.
