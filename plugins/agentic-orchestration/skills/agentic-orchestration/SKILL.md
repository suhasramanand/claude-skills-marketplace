---
name: agentic-orchestration
description: This skill should be used when the user asks about CLAUDE.md, "set up Claude Code", "sub-agents", "sub-agent", "hooks", "PostToolUse", "PreToolUse", "MCP server", ".mcp.json", "agentic workflow", "orchestrator-worker", "Writer/Reviewer", "parallel agents", "multi-agent", "git worktree agents", "agent delegation", "custom slash commands", "plugin", or discusses structuring a Claude Code project for agentic use. Applies production agentic-orchestration patterns from Anthropic's internal engineering practice.
version: 1.0.0
---

# Agentic Orchestration with Claude Code

You are an expert in production Claude Code agentic workflows. Apply the following patterns whenever you write, review, or advise on CLAUDE.md files, sub-agent definitions, hooks, MCP configs, or multi-agent architectures.

---

## 1. CLAUDE.md Architecture

Keep CLAUDE.md under ~200 lines. Every low-value instruction dilutes high-value ones — be ruthless.

Required sections in order:

```markdown
# Project: <name>

## Build & Test
<exact commands — copy-paste runnable>

## Architecture
<key systems, 3–5 bullet points>

## Conventions
<naming, branch policy, PR rules>

## MCP Servers
<which servers are wired, what they cover>

## Gotchas
<the hidden constraints, past incidents, non-obvious invariants>
```

The **Gotchas** section is the highest-signal content. Write it as "things that would surprise a new engineer": a DB column that looks nullable but isn't, a service that always needs a migration before deploy, a Kafka topic that requires SASL.

Never put obvious things in CLAUDE.md ("use meaningful variable names"). Every line should be something the model couldn't infer from reading the code.

---

## 2. Sub-agent Definition

Sub-agents live in `.claude/agents/<name>.md` and use YAML frontmatter:

```markdown
---
name: code-reviewer
description: >
  Invoked after completing any implementation task to do a read-only
  security, correctness, and style review. Use for: any PR touching auth,
  payments, or external APIs. Do NOT use for: doc-only changes.
tools: Read, Grep, Glob
model: opus
---

You are a senior security-focused code reviewer. Your only job is to read
the diff and report issues. You cannot edit files. Focus on:
- Auth bypass / privilege escalation
- SQL injection, XSS, command injection
- Secrets or PII in logs
- Missing input validation at system boundaries

Output: a numbered list of findings, severity (CRITICAL/HIGH/MEDIUM/LOW),
and a one-line remediation for each. If nothing found, say "LGTM."
```

Key rules:
- `description` drives auto-delegation — write it as "when to invoke / when not to invoke"
- `tools` is an allowlist; omit `Edit`/`Write`/`Bash` for read-only agents
- Sub-agents get isolated context windows — they don't see the parent's conversation
- Sub-agents cannot nest or share sibling state; pass artifacts via files or condensed summaries
- Use `model: opus` for judgment-heavy review agents; `sonnet` for mechanical tasks

Anthropic's delegation heuristic: **delegate when the task touches 10+ files or has 3+ independent pieces**; sequential dependent work stays in one session.

---

## 3. Hooks

Hooks live in `.claude/settings.json`. They execute deterministically — unlike prompts, they always run.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'cd \"$CLAUDE_PROJECT_DIR\" && [[ \"$CLAUDE_TOOL_INPUT_FILE_PATH\" == *.go ]] && gofmt -w \"$CLAUDE_TOOL_INPUT_FILE_PATH\" && go vet ./... 2>&1 | head -20 || true'"
          },
          {
            "type": "command",
            "command": "bash -c '[[ \"$CLAUDE_TOOL_INPUT_FILE_PATH\" == *.py ]] && ruff check --fix \"$CLAUDE_TOOL_INPUT_FILE_PATH\" 2>&1 | head -20 || true'"
          },
          {
            "type": "command",
            "command": "bash -c '[[ \"$CLAUDE_TOOL_INPUT_FILE_PATH\" == *.tf ]] && terraform fmt \"$CLAUDE_TOOL_INPUT_FILE_PATH\" 2>&1 || true'"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'path=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"; [[ \"$path\" == *\".env\"* || \"$path\" == *\"package-lock.json\"* || \"$path\" == *\".git/\"* ]] && echo \"Blocked: editing $path is not allowed\" && exit 2 || exit 0'"
          }
        ]
      }
    ]
  }
}
```

Exit code semantics: `0` = proceed, `1` = soft warning (shown to model), `2` = hard block (stops the tool call and returns feedback).

Add an HTTP audit hook for any write to prod-adjacent paths:

```json
{
  "type": "http",
  "url": "https://internal-audit.corp/claude-events",
  "headers": { "Authorization": "Bearer ${AUDIT_TOKEN}" }
}
```

---

## 4. MCP Server Configuration

Commit `.mcp.json` at project root — never store credentials in it, use env-var references:

```json
{
  "mcpServers": {
    "dbt": {
      "command": "uvx",
      "args": ["dbt-mcp"],
      "env": {
        "DBT_TOKEN": "${DBT_PERSONAL_ACCESS_TOKEN}",
        "DBT_HOST": "${DBT_CLOUD_HOST}"
      }
    },
    "kubernetes": {
      "command": "npx",
      "args": ["-y", "@manusa/kubernetes-mcp-server"],
      "env": {
        "KUBECONFIG": "${HOME}/.kube/config"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@github/mcp-server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

Scope rules:
- Use `-s project` for repo-specific servers (dbt, k8s cluster)
- Use `-s user` for cross-repo tools (GitHub, Slack notifications)
- Use `read-only` modes first; expand write access only after you trust the tool's blast radius
- MCP prompt-injection is a real attack surface — only install servers from trusted sources

---

## 5. Orchestrator-Worker Pattern

Use this when the task decomposes into **independent parallel threads**. Token usage scales ~15× vs single-agent chat — only worth it for large, parallelizable work.

```
Orchestrator (Opus, thinking enabled)
  ├── plans the full task
  ├── persists plan to memory.md
  ├── spawns Worker A  ──→ isolated context, condensed return
  ├── spawns Worker B  ──→ isolated context, condensed return
  └── reconciles condensed returns into a final artifact

Workers (Sonnet, tool-scoped)
  ├── receive only the slice they need
  ├── return a condensed summary + file paths touched
  └── never see each other's context
```

Implementation in Claude Code:

```bash
# Orchestrator prompt pattern
claude --model opus "
Read memory.md for the current plan.
Spawn a sub-agent to handle <slice A> with context from files X, Y, Z.
Spawn a sub-agent to handle <slice B> with context from files P, Q, R.
Wait for both, then reconcile into output.md.
Update memory.md with completed steps.
"
```

---

## 6. Writer/Reviewer Pattern

Run a fresh-context reviewer sub-agent after every non-trivial implementation. The reviewer catches what the implementer missed because it has no anchoring bias.

```markdown
# .claude/agents/reviewer.md
---
name: reviewer
description: >
  Read-only reviewer. Invoke after any implementation that touches business
  logic, auth, data models, or public APIs. Pass it the list of changed files.
tools: Read, Grep, Glob
model: opus
---

Review the provided files for: correctness, security, test coverage gaps,
and consistency with CLAUDE.md conventions. You have no knowledge of what
was intended — judge only what is written.
```

Always give the reviewer the diff context, not the full codebase — it needs signal, not noise.

---

## 7. Parallel Workflows with Git Worktrees

For large migrations or parallel independent features:

```bash
# Create isolated worktrees for parallel agent runs
git worktree add ../feature-a -b feature/migrate-auth-service
git worktree add ../feature-b -b feature/migrate-payment-service

# Run agents in parallel against each worktree
claude -p "Migrate the auth service to the new SDK" --project ../feature-a &
claude -p "Migrate the payment service to the new SDK" --project ../feature-b &
wait

# Review both, then merge
git -C ../feature-a diff main
git -C ../feature-b diff main
```

Use `claude -p` (print mode) for batch / headless runs. Pipe a list of files for large migrations:

```bash
find . -name "*.go" | xargs -I{} claude -p "Update {} to use the new logger interface per CLAUDE.md"
```

---

## 8. Common Anti-Patterns

| Anti-pattern | Correct approach |
|---|---|
| CLAUDE.md > 200 lines | Every line must earn its place — cut obvious guidance |
| Sub-agent with all tools | Allowlist only the tools the agent needs |
| Autonomous apply on prod | Human gate on any write to prod systems |
| One huge orchestration session | Decompose + isolated sub-agents + condensed returns |
| Secrets in `.mcp.json` | Env-var references only |
| No hooks | PostToolUse formatter + PreToolUse guard at minimum |
| Reviewer sees full codebase | Pass only the diff / changed file paths |
| Skipping effort level | Set `/effort max` for complex multi-file tasks |

---

## Quick Reference

```bash
# Add an MCP server (project scope)
claude mcp add dbt -- uvx dbt-mcp

# Run a sub-agent inline
claude "Spawn a code-reviewer sub-agent on the files changed in this PR"

# Headless batch loop
git diff --name-only main | xargs -I{} claude -p "Review {} for SQL injection per CLAUDE.md"

# Parallel worktrees
git worktree add ../scratch -b scratch/experiment
claude --project ../scratch "Try the refactor here without touching main"
git worktree remove ../scratch

# Inspect hook execution
claude --debug 2>&1 | grep hook
```
