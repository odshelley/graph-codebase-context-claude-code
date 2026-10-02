# Claude Code Native Context Management Features (October 2026)
## Comprehensive Analysis for Enterprise Teams

---

## 1. Memory and Instruction Files
**https://code.claude.com/docs/en/memory**

Claude Code offers a three-tier memory system:

- **CLAUDE.md files** (hierarchical): Enterprise-managed at system level, user-level (`~/.claude/CLAUDE.md`), project-level (`./.claude/CLAUDE.md` or `./CLAUDE.md`), and local (`./CLAUDE.local.md` for personal preferences)
- **Auto memory**: Machine-local persistent notes stored in `~/.claude/projects/<project>/memory/` with `MEMORY.md` index (200 lines or 25KB loaded at session start) and topic files loaded on-demand
- **AGENTS.md compatibility**: Claude reads existing `AGENTS.md` files directly (v2.1.277+) when no `CLAUDE.md` is present, with configurable precedence via "Project instructions" setting (`claude-md-or-agents-md`, `claude-md-and-agents-md`, `claude-md`, or `managed-only`)
- **Path-scoped rules**: `.claude/rules/*.md` files with YAML `paths:` frontmatter load only when matching files are accessed (glob patterns like `src/**/*.ts`)
- **Imports**: `@path` syntax supports recursive imports (4-hop max) across CLAUDE.md and rules

**Size limits**: CLAUDE.md up to 4MB, auto memory `MEMORY.md` first 200 lines/25KB loaded at startup, topic files loaded on-demand. Over 200 lines costs adherence.

---

## 2. Context Window Handling
**https://code.claude.com/docs/en/context-window**

Claude Code simulates context composition with an interactive visualization. Key mechanisms:

- **Auto-compaction**: Triggered when context exceeds model limits; summarizes conversation and re-injects startup materials
- **What survives compaction**: System prompt, auto memory (MEMORY.md first 200/25KB), CLAUDE.md files (project-root only; nested rules reload on-demand), git status, invoked skills (first 5K tokens each, shared 25K budget), hooks' `additionalContext` fields
- **MCP tool schemas**: Deferred loading by default via tool search; set `ENABLE_TOOL_SEARCH=auto` to load upfront if <10% context window or `false` to load all
- **Context window sizes**: Claude Code supports 200K context on standard models; 1M context available on Opus 5.5+ (inferred from Sept 2026 blog post: "longer coding sessions use more context")
- **Token display**: `/status` shows session token usage; `claude --debug` logs context breakdown

---

## 3. Subagents
**https://code.claude.com/docs/en/sub-agents**

Subagents reduce main-context pollution through isolation:

- **Context isolation**: Fresh context window per non-fork subagent; loads system prompt, task prompt, CLAUDE.md, git status, preloaded skills, but NOT conversation history
- **Fork subagents**: Inherit full parent context (conversation, system prompt, model, permissions, prompt cache) and persist outputs separately
- **Model/tool restrictions**: Define in agent `.md` frontmatter (`tools: Read, Grep, Bash`); disallow with `disallowedTools: Write, Edit`
- **Built-in agents**: Explore (read-only search), Plan (research agent), General-purpose (full tools)
- **Parallelism**: Multiple subagents spawn and return summaries independently; careful with high-volume parallel work to avoid context pollution in main session
- **Auto-delegation**: Claude recognizes task descriptions and delegates matching agent's description

---

## 4. Skills
**https://code.claude.com/docs/en/skills**

Progressive disclosure minimizes context cost:

- **Loading strategy**: Skill descriptions (1% of context window budget by default) always load; full `SKILL.md` content loads only on invocation or when Claude auto-detects relevance
- **Content lifecycle**: Once loaded, stays in context across turns; re-attachment after compaction (first 5K tokens each, shared 25K budget)
- **Scope**: Personal (`~/.claude/skills/`), project (`.claude/skills/`), nested (subdirectory-specific), enterprise, plugins, claude.ai account synced
- **Dynamic content**: Inline commands (`` !`git status` ``) run when skill loads
- **Skill frontmatter**: `description` (always visible), `disable-model-invocation` flag, tool access control

---

## 5. MCP (Model Context Protocol)
**https://code.claude.com/docs/en/mcp**

Open standard for tool/data integrations:

- **Configuration**: Three transports—HTTP (recommended, `claude mcp add --transport http`), stdio (local, `--transport stdio`), WebSocket (`ws` type in JSON), deprecated SSE
- **Scopes**: Local (default, `~/.claude.json`), project (`.mcp.json`, shared), user (all projects)
- **Tool restrictions**: Organization can set `ask` (approval per call), `blocked` (filtered before Claude sees); check with `/mcp`
- **Output limits**: Default 25,000 tokens, warning at 10,000; increase via `MAX_MCP_OUTPUT_TOKENS` env var; tool-level limits via `_meta.anthropic/maxResultSizeChars`
- **Authentication**: OAuth 2.0, pre-configured credentials, dynamic headers via `headersHelper` script
- **Tool search** (deferred loading): By default only tool names listed in context; Claude loads specific schemas on-demand. `ENABLE_TOOL_SEARCH=auto` loads upfront if <10% window; `false` loads all schemas

---

## 6. Hooks
**https://code.claude.com/docs/en/hooks-guide** (main guide) and **https://code.claude.com/docs/en/hooks** (full reference)

Deterministic automation at lifecycle events:

- **Hook events** (30+): SessionStart, Setup, UserPromptSubmit, UserPromptExpansion, PreToolUse, PermissionRequest, PermissionDenied, PostToolUse, PostToolUseFailure, PostToolBatch, Notification, MessageDisplay, SubagentStart, SubagentStop, ConfigChange, CwdChanged, FileChanged, WorktreeCreate, WorktreeRemove, PreCompact, PostCompact, PreModelSwitch, PostModelSwitch, InstructionsLoaded, TaskCreated, TaskCompleted, SessionEnd, Elicitation, ElicitationResult, Stop, StopFailure, TeammateIdle
- **Hook types**: `command` (shell), `http` (POST to webhook), `mcp_tool` (call MCP server tool), `prompt` (single LLM eval), `agent` (subagent verification, experimental)
- **Context injection**: `UserPromptSubmit` hooks via `additionalContext` field; `SessionStart` hooks with `compact` matcher re-inject after compaction. Quote from docs: "Re-inject context after compaction...This example reminds Claude of project conventions and recent work."
- **Matchers** (narrow by tool/event): `Edit|Write`, `Bash(git *)`, `permission_prompt`, `compact`, regex patterns; MCP tools: `mcp__server__tool` or `mcp__.*__write.*`
- **Configuration**: `~/.claude/settings.json` (global), `.claude/settings.json` (project), `.claude/settings.local.json` (personal project), managed settings (org-wide), plugin `hooks/hooks.json`, skill frontmatter
- **Hook output**: Exit codes (0=no decision, 2=block) + stderr feedback; JSON output for structured decisions (`permissionDecision: allow|deny|ask`, `additionalContext`)

---

## 7. Built-in Search Behavior & Agentic-Search Philosophy

**https://code.claude.com/docs/en/** (Overview) and **https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code** (Sept 2026 workflow design post)

Anthropic's stated design position: **agentic search over indexing**.

**Key design choices:**
- **No shipped code index**: Claude Code provides Glob, Grep, Read as first-class tools; Claude decides what to search based on task context
- **Tool search (deferred MCP schemas)**: Reduces upfront token cost; Claude loads specific tool schemas only when needed via the tool-search mechanism
- **Offline-first**: Agentic search works without backend infrastructure or pre-indexed data
- **Context-aware reasoning**: Agent can adapt search strategy mid-session based on findings

**Anthropic's reasoning (inferred from documentation and blog):**
From the Claude Code overview: "Claude Code handles the tedious tasks that eat up your day: writing tests for untested code, fixing lint errors across a project, resolving merge conflicts, updating dependencies, and writing release notes...Claude Code traces the issue through your codebase, identifies the root cause, and implements a fix."

The Sept 2026 blog post "A harness for every task: dynamic workflows in Claude Code" states: "Spawn [multiple Claude Code agents] that work on different parts of a task simultaneously. A lead agent coordinates the work, assigns subtasks, and merges results." This reflects Anthropic's bet on **agentic parallelism** over indexed shortcuts—multiple agents searching independently is more flexible than querying a shared index.

**What this means for graph-layer gaps**: A semantic graph MCP would complement agentic search by offering direct type lookups, call-chain visualization, and symbol tables—trading agent flexibility for deterministic, context-efficient navigation on massive codebases.

---

## 8. Enterprise Controls
**https://code.claude.com/docs/en/managed-settings** (main) and **https://code.claude.com/docs/en/admin-setup** (decision table)

Organization-wide policy deployment:

- **Managed settings delivery**: Four mechanisms—server-managed (claude.ai admin console or gateway, highest priority), MDM/OS policy (macOS plist, Windows HKLM), file-based (`managed-settings.json` + `managed-settings.d/`), HKCU registry (fallback, user-writable)
- **Scope precedence** (highest to lowest): Managed > CLI (`--settings`) > Project local (`.claude/settings.local.json`) > Shared project (`.claude/settings.json`) > User (`~/.claude/settings.json`)
- **Permission enforcement**: Deny specific tools (`permissions.deny`), enforce bypass-mode disable, block file reads (e.g., `.env`, `package-lock.json`). Quote: "No user, project, local, or `--settings` value overrides them, apart from a few security-sensitive exceptions."
- **MCP allowlist lock**: `allowManagedMcpServersOnly` + `allowedMcpServers` list; blocks parent-supplied or user-added servers
- **Feature gates**: `disableBypassPermissionsMode`, `allowManagedPermissionRulesOnly`, `syncClaudeAiSkills`, `syncClaudeAiPlugins`, `enableArtifact`
- **Sandboxing**: `sandbox.enabled`, `sandbox.network.allowedDomains`, `sandbox.filesystem.allowRead`, ripgrep binary paths
- **Audit**: `/status` shows active managed source; `claude doctor` lists dropped settings; ConfigChange hooks log policy edits
- **Data retention**: `cleanupPeriodDays` for session transcripts (auto memory excluded); server-managed settings cached locally and revalidated on each fetch

---

## 9. Agent SDK Context Management
**https://code.claude.com/docs/en/agent-sdk/overview**

Programmatic agent harness in Python/TypeScript:

- **Automatic context management**: Inherits Claude Code's compaction logic, memory loading, skill/rule system
- **Context API**: Sessions support resume/fork, context summarization via agent loop
- **Tool access**: Same Read, Edit, Write, Bash, Glob, Grep, WebFetch tools; MCP servers, hooks, subagents all available
- **Memory**: Skills, CLAUDE.md, auto memory loaded like CLI; subagents have own memory when `memory: true` in agent frontmatter
- **Tool search** (MCP deferred loading): Same behavior as CLI; reduce token cost of large tool sets
- **Permissions**: Same modes as CLI (default, acceptEdits, bypassPermissions, dontAsk); hooks enforced in agent loops
- **GitHub repositories**: Python SDK (anthropics/claude-agent-sdk-python), TypeScript SDK (anthropics/claude-agent-sdk-typescript)

---

## 10. Recently Announced Features (2026)
**https://claude.com/blog** (multiple posts)

- **Opus 5.5 extended context** (Sept 24, 2026): "Coding sessions are longer and use more context. Claude Opus 5.5 is built with that in mind." Implies 1M context window support for sustained development
- **Artifact support in Claude Code** (June 18, 2026): Integrated artifacts directly in terminal/IDE
- **Managed Agents enhancements**: May 19 (self-hosted sandboxes, multiagent orchestration), Sept 30 ("How Anthropic's sales team rebuilt inbound with Claude Managed Agents")
- **Workflows** (Sept 15, 2026): Claude for Small Business workflows and integrations
- **Agent coaching** (Sept 29, 2026): "Agents you can coach: how Asana builds human-agent teams with Claude"

---

## Anthropic Engineering Posts (Verification Status)

**CONFIRMED URLs:**
1. https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code (Sept 2026) — discusses dynamic workflows, subagent orchestration, parallelism, and how the Claude Code team uses workflows internally
2. https://code.claude.com/docs/en/ (Overview) — design philosophy: tools over indexing, agents over backends
3. https://claude.com/blog (Sept 2026 blog posts on Opus 5.5 context, artifacts, agents, workflows)

**UNVERIFIED** (referenced in task but not located):
- Dedicated post "Effective context engineering for AI agents" — searched Anthropic blog/newsroom; only general posts found, no standalone engineering post on this topic
- "Code execution with MCP" — no standalone post located; MCP configuration covered in https://code.claude.com/docs/en/mcp
- Latent Space podcast episode on Claude Code architecture — not fetched

**Key insight from available posts**: Anthropic emphasizes **agentic parallelism and dynamic workflows** over static indexing. The Sept 2026 blog states agents should "work on different parts of a task simultaneously" with a "lead agent coordinat[ing] the work"—reflecting a deliberate choice of agent flexibility over pre-indexed data structures.

---

## Native Features vs. External Graph Layer Gap Analysis

| Dimension | Claude Code Native | Gap a Graph MCP Would Fill |
|-----------|--------------------|-----------------------------|
| **Code understanding** | Agentic grep/read + file traversal | Semantic indexing: fast cross-codebase queries without agent search overhead; direct symbol table/type info |
| **Cross-file context** | Agent infers from reads; limited by context window | Cached dependency graph, call chains, import resolution; instant cross-file dependency visualization |
| **Context efficiency** | Tool search (deferred MCP schemas); compaction re-injects startup only | Selective context injection: pull only the code Claude needs for a task; prune irrelevant files |
| **Long-horizon tasks** | Subagents + forks; skills for workflows | Persistent, multi-session task graph; track subtask dependencies without re-reading |
| **Memory persistence** | Auto memory (per-session learnings), CLAUDE.md (static rules) | Structural code facts (refactoring proposals, architecture debt) that change session-to-session |
| **Codebase-scale search** | Linear grep + glob; no semantic ranking | Vector embeddings or semantic search (symbols, types, patterns); ranking by relevance |
| **IDE integration** | MCP servers for external tools; no built-in AST/LSP | First-class type information, IDE-ready diagnostics; LSP bridge for real-time feedback |
| **Query optimization** | Claude decides search scope per prompt | Query planner: recommend search strategy (search types vs. implementations, trace call chains) |
| **Enterprise audit** | `/status`, `claude doctor`, ConfigChange hooks | Codebase graph snapshots for compliance; change-impact analysis |

---

## Complete Documentation URL List

1. **Memory & Instructions**: https://code.claude.com/docs/en/memory
2. **Context Window**: https://code.claude.com/docs/en/context-window
3. **Subagents**: https://code.claude.com/docs/en/sub-agents
4. **Skills**: https://code.claude.com/docs/en/skills
5. **MCP Overview**: https://code.claude.com/docs/en/mcp
6. **Hooks Guide**: https://code.claude.com/docs/en/hooks-guide
7. **Hooks Reference**: https://code.claude.com/docs/en/hooks
8. **Settings & Precedence**: https://code.claude.com/docs/en/settings
9. **Managed Settings**: https://code.claude.com/docs/en/managed-settings
10. **Admin Setup**: https://code.claude.com/docs/en/admin-setup
11. **Agent SDK Overview**: https://code.claude.com/docs/en/agent-sdk/overview
12. **Claude Code Overview**: https://code.claude.com/docs/en/
13. **CLI Reference**: https://code.claude.com/docs/en/cli-reference
14. **Claude Blog (Sept 24, 2026)**: https://claude.com/blog – "Coding sessions are longer and use more context. Claude Opus 5.5 is built with that in mind"
15. **Claude Blog (Sept 15, 2026)**: https://claude.com/blog – Claude for Small Business workflows
16. **Workflow Design Post (Sept 2026)**: https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code

---

## Key Design Insights & Summary

1. **Deferred tool loading (MCP tool search)** is Anthropic's core context-efficiency play for large tool sets; no upfront schema bloat.

2. **Compaction re-injects only startup materials**, requiring hooks or CLAUDE.md to preserve in-session learnings; session-scoped improvements (skills, hooks) don't auto-persist.

3. **No semantic code index**; Anthropic chose agentic search for offline capability and context-aware reasoning over indexed speed.

4. **Enterprises get three policy tiers**: server-managed (highest), endpoint (MDM/file), HKCU fallback; managed settings never override themselves, but lower-level security rules can tighten the lock.

5. **Subagents + forks** reduce main-context noise; design assumes you'll route high-volume research (file reads, tests) to subagents and keep main conversation clean.

6. **A graph-based MCP layer** would complement this by trading agentic flexibility for deterministic, context-efficient code navigation—most valuable for (a) massive codebases where agent search becomes expensive, (b) repeated queries (memo graph analysis), and (c) IDE-grade type/symbol queries.

---

## Conclusion

Claude Code's native context layer prioritizes **agentic flexibility over pre-indexed structure**. As of October 2026, the system offers rich memory hierarchies (CLAUDE.md, auto memory, path-scoped rules), progressive skill loading, deferred MCP tool schemas, and deterministic compaction with re-injection hooks. Enterprise controls (managed settings, permission enforcement, audit logging) provide tight policy control without sacrificing developer workflow.

A graph-based MCP designed to surface code relationships (dependencies, call chains, types, symbols) would address the primary gap: **context-efficient navigation of massive codebases without agentic search overhead**. The gap widens as codebase scale increases; for small teams, native tools suffice.

