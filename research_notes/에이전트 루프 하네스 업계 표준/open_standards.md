# Cross-vendor Open Standards for Coding-Agent Harnesses (as of 2026-11-18)

Scope: AGENTS.md, Agent Skills (SKILL.md), MCP, hooks, A2A / ACP / AG-UI / OTel GenAI, and a repo layout that works across Claude Code, Codex, Gemini CLI, Cursor, Copilot, and similar tools. All facts below were fetched on 2026-11-18 unless marked otherwise. Items marked "(secondary)" come from non-primary sources.

## 1. AGENTS.md: spec, governance, adoption, nesting, Claude Code relation, native files

### Takeaway
AGENTS.md is a free-form Markdown convention with no schema or version number. It is governed by the Agentic AI Foundation (AAIF) under the Linux Foundation. Essentially every major tool now reads it natively. Since v2.1.277 that includes Claude Code, but by default only when no CLAUDE.md exists. "Nearest file wins" is the shared nesting rule. Make AGENTS.md the single source of truth and use tool-specific files only as thin shims.

### Cited Findings
- **Spec:** plain Markdown, "README for agents". No required fields or headings. Typical sections cover the overview, build and test commands, code style, testing, security, and PR guidelines. The site claims more than 60k OSS projects use it. It shows no version number or release date. — [agents.md](https://agents.md)
- **Origin and governance:** it came out of a collaboration between OpenAI Codex, Amp, Jules (Google), Cursor, and Factory. It is now "stewarded by the Agentic AI Foundation under the Linux Foundation", and the copyright line reads "AGENTS.md a Series of LF Projects, LLC". — [agents.md](https://agents.md)
- **AAIF founding:** announced 2025-12-09. Founding projects were MCP (Anthropic), goose (Block), and AGENTS.md (OpenAI). Platinum members are AWS, Anthropic, Block, Bloomberg, Cloudflare, Google, Microsoft, and OpenAI. Tiers are Platinum, Gold, and Silver, and the release does not describe the detailed governance model. — [Linux Foundation press release](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- **Later AAIF additions:** A2A joined AAIF as a hosted project on 2026-08-17 ([AAIF blog](https://aaif.io/blog/a2a-joins-aaif); [Axios](https://axios.com/2026/08/17/a2a-agentic-ai-foundation-open-ai-standards)). Some outlets give 2026-08-20. agentgateway was reportedly added as the fourth project on 2026-06-04 (secondary, from a search summary, unverified). Member counts were 146 in Feb 2026 and 247 in Aug 2026 (secondary, unverified).
- **Adoption list on agents.md:** Codex, Jules, Factory, Aider, goose, opencode, Zed, Warp, VS Code, Devin, UiPath, Junie (JetBrains), Amp, Cursor, RooCode, Gemini CLI, Kilo Code, Phoenix, Semgrep, GitHub Copilot coding agent, Ona, Windsurf, and Augment Code. Claude Code is not on the list and the site has no FAQ entry for it. — [agents.md](https://agents.md)
- **Nesting rule in the spec:** the nearest AGENTS.md in the directory tree takes precedence, and an explicit user prompt overrides everything. OpenAI's main repo had 88 AGENTS.md files "at time of writing". — [agents.md](https://agents.md)
- **Migration recipes from agents.md:**
  - Rename an existing file and symlink the old name to it: `mv AGENT.md AGENTS.md && ln -s AGENTS.md AGENT.md`.
  - Aider: `read: AGENTS.md` in `.aider.conf.yml`.
  - Gemini CLI: set the context file name to AGENTS.md in `.gemini/settings.json`.
  - Source: [agents.md](https://agents.md)
- **Codex (OpenAI) discovery rules:**
  - Global: `~/.codex` (or `$CODEX_HOME`). Codex uses `AGENTS.override.md` if present, otherwise `AGENTS.md`, and takes only the first non-empty file.
  - Project: walks from the git root down to the cwd. In each directory it checks `AGENTS.override.md`, then `AGENTS.md`, then names in `project_doc_fallback_filenames`. Each directory contributes at most one file.
  - Order: files are concatenated root first, so files closer to the cwd come later and win.
  - Size cap: concatenation stops at `project_doc_max_bytes`, default 32 KiB.
  - Timing: the chain is built once per run or TUI session.
  - Source: [OpenAI Codex docs, AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md.md) (developers.openai.com/codex now 308-redirects to learn.chatgpt.com)
- **Claude Code native support (major change):**
  - Since v2.1.277, Claude Code reads AGENTS.md natively through the built-in plugin `cc-plugin-agents-md@builtin` (named `agents-md@builtin` before v2.1.285).
  - Default mode `claude-md-or-agents-md`: AGENTS.md loads only if no `CLAUDE.md`, `.claude/CLAUDE.md`, or `CLAUDE.local.md` exists in the cwd or above it. `~/.claude/CLAUDE.md`, managed CLAUDE.md, and `.claude/rules/` don't count toward that check.
  - Other modes: `claude-md-and-agents-md` (each directory's CLAUDE.md first, then its AGENTS.md, deduplicated), `claude-md`, and `managed-only`. Set the mode via `/config` → Project instructions, or via `pluginConfigs` in user or managed settings. Project-level settings are ignored for this.
  - What gets read: every `AGENTS.md` and `.claude/AGENTS.md` from the cwd upward at startup, plus subdirectory AGENTS.md on demand when a file there is read.
  - Not read: `AGENTS.local.md`, `AGENTS.override.md`, or anything under `.agents/`.
  - Differences from CLAUDE.md: `InstructionsLoaded` hooks don't fire for an AGENTS.md read this way, and `--add-dir` directories' AGENTS.md don't load.
  - Legacy workarounds still work. A `CLAUDE.md` containing `@AGENTS.md` is never double-read and is recommended for sessions that can't read AGENTS.md (versions before 2.1.277, the first session after an upgrade, or the plugin disabled). A `CLAUDE.md` symlinked to AGENTS.md is read once.
  - `@path` imports resolve relative to the importing file, with a maximum depth of 4 hops. External imports need one-time approval.
  - Source: [Claude Code docs, memory](https://code.claude.com/docs/en/memory)
- **GitHub Copilot:**
  - Supported files:
    - `.github/copilot-instructions.md` (repo-wide)
    - `.github/instructions/NAME.instructions.md` (path-specific via `applyTo` glob, with an `excludeAgent: code-review|cloud-agent` option)
    - `AGENTS.md` anywhere in the repo (nearest wins)
    - a single `CLAUDE.md` or `GEMINI.md` at the repo root
  - Precedence: personal, then repository, then organization, but all sets are sent and combined.
  - The docs do not say how a repo with both AGENTS.md and CLAUDE.md is resolved.
  - Code review reads instructions and skills from the PR head branch.
  - Source: [GitHub Docs, repository instructions](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)
- **Cursor:**
  - Project rules live in `.cursor/rules/` as `.mdc` files with `description`, `globs`, and `alwaysApply` frontmatter. Plain `.md` files there are ignored.
  - Rule kinds: User Rules, Team Rules (dashboard, enforceable), and AGENTS.md in the root, with nested AGENTS.md supported (more specific wins).
  - Precedence is Team, then Project, then User, and all rules are merged.
  - The page does not mention CLAUDE.md.
  - Source: [Cursor docs, rules](https://cursor.com/docs/context/rules)
- **Gemini CLI:** the native file is `GEMINI.md`. AGENTS.md requires a context-filename setting in `.gemini/settings.json` ([agents.md](https://agents.md)). Gemini CLI was replaced by Antigravity CLI for unpaid-tier and Google One users on 2026-06-18 ([Gemini CLI docs, hooks](https://geminicli.com/docs/hooks/)).

### Inferences
- In practice the recipe is: write `AGENTS.md` as the canonical file. Add a 1-line `CLAUDE.md` containing `@AGENTS.md` only if you need Claude-specific additions or support for older Claude Code versions. A CLAUDE.md that exists for any other reason suppresses AGENTS.md in the default mode, so it must import it. Point Gemini at AGENTS.md through settings. Copilot, Cursor, and Codex need nothing extra.
- Avoid relying on `AGENTS.override.md`, which is Codex-only and explicitly ignored by Claude Code.
- Keep the total chain under 32 KiB because of Codex's default cap, and use nested per-package AGENTS.md (for example `backend/AGENTS.md` for Go and `frontend/AGENTS.md` for Next.js).

### Gaps
- AGENTS.md has no formal versioned spec. I found no published change process.
- Exact AAIF governance documents (TSC, charter) were not retrieved.
- The current Gemini CLI setting key name (`context.fileName`) was not confirmed on a primary page this session.
- How Copilot handles AGENTS.md and CLAUDE.md together is unspecified.

## 2. Agent Skills (SKILL.md): format, progressive disclosure, support and locations

### Takeaway
Agent Skills is a small open spec at agentskills.io: a directory with a `SKILL.md` made of YAML frontmatter and a Markdown body, plus optional `scripts/`, `references/`, and `assets/`. Loading is three-tier progressive. `.agents/skills/` is the de facto cross-client directory. Codex, Gemini CLI, and Copilot read it, but Claude Code's docs list only `.claude/skills/`.

### Cited Findings
- **Required frontmatter:**
  - `name`: 1–64 characters, lowercase a–z, 0–9 and `-`, no leading, trailing, or double hyphen, and must match the directory name.
  - `description`: 1–1024 characters, covering what the skill does and when to use it.
- **Optional frontmatter:** `license`, `compatibility` (max 500 characters), `metadata` (string→string map), and `allowed-tools` (space-separated, **Experimental**).
- **Body:** free-form. Keep SKILL.md under 500 lines. File references should be relative and one level deep.
- **Validation:** the `skills-ref validate` CLI.
- Source for the format bullets above: [agentskills.io specification](https://agentskills.io/specification)
- **Progressive disclosure:** tier 1 is the catalog (name and description, about 50–100 tokens per skill, loaded at session start). Tier 2 is the full SKILL.md body (under 5,000 tokens recommended), loaded on activation. Tier 3 is scripts, references, and assets, loaded on demand. — [agentskills.io client implementation guide](https://agentskills.io/client-implementation/adding-skills-support)
- **Discovery convention:**
  - The spec does not mandate locations. The guide recommends scanning `<project>/.<client>/skills/`, `<project>/.agents/skills/`, `~/.<client>/skills/`, and `~/.agents/skills/`.
  - "`.agents/skills/` paths have emerged as a widely-adopted convention for cross-client skill sharing". Some clients also scan `.claude/skills/` for compatibility.
  - Project skills override user skills.
  - Project skills should be gated on folder trust.
  - Activation is either model-driven (file read or a dedicated `activate_skill` tool) or explicit (`/skill-name` or `$skill-name`).
  - Skill content should be protected from compaction.
  - Source: [agentskills.io client implementation guide](https://agentskills.io/client-implementation/adding-skills-support)
- **Claude Code:**
  - Locations:
    - Project: `.claude/skills/<name>/SKILL.md`, in the cwd and every parent up to the repo root
    - Nested: `<subdir>/.claude/skills/`, loaded lazily when a file there is touched
    - Personal: `~/.claude/skills/`
    - Also enterprise (managed) skills, plugin skills (`<plugin>/skills/`, invoked as `/plugin:skill`), and claude.ai-synced skills
  - Legacy `.claude/commands/*.md` still works.
  - Precedence is enterprise, then personal, then project.
  - It states compliance with the Agent Skills open standard and adds extensions: `when_to_use`, `argument-hint`, `arguments`, `disable-model-invocation`, `user-invocable`, `disallowed-tools`, `model`, `effort`, `context: fork`, `agent`, `background`, `hooks`, `paths`, `shell`.
  - Unknown fields are silently ignored in Claude Code but cause hard errors in claude.ai uploads and the Skills API.
  - The docs page does **not** mention `.agents/skills`.
  - Source: [Claude Code docs, skills](https://code.claude.com/docs/en/skills)
- **Codex:** repository skills live in `.agents/skills` in every directory from the cwd up to the repo root. User skills are in `$HOME/.agents/skills`, admin skills in `/etc/codex/skills`, and system skills are bundled. Symlinked skill folders are followed. Same-name skills are not merged. — [OpenAI Codex docs, skills](https://developers.openai.com/codex/skills) (via search snippet; the page was not fully fetched)
- **Gemini CLI:** discovery goes built-in, then extension, then user (`~/.gemini/skills/` or the `~/.agents/skills/` alias), then workspace (`.gemini/skills/` or the `.agents/skills/` alias), with higher tiers winning. Within a tier, `.agents/skills/` beats `.gemini/skills/`. The feature is not marked experimental. — [Gemini CLI docs, skills](https://geminicli.com/docs/cli/skills/)
- **GitHub Copilot:** project skills are read from `.github/skills`, `.claude/skills`, or `.agents/skills`. Personal skills are in `~/.copilot/skills` or `~/.agents/skills`. Supported surfaces are the Copilot app, Copilot CLI, cloud agent, code review, and agent mode in VS Code, Visual Studio, and JetBrains. — [GitHub Docs, agent skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
- **MCP:** "Skills over MCP" is listed as a notable MCP extension or working group. — [MCP specification](https://modelcontextprotocol.io/specification/latest)

### Inferences
- The most portable layout is canonical skills in `.agents/skills/<name>/`, which Codex, Gemini, and Copilot read natively, plus a symlink `.claude/skills → ../.agents/skills`, or per-skill symlinks, for Claude Code. Copilot also reads `.claude/skills`, so the reverse direction (canonical in `.claude/skills`, symlink at `.agents/skills`) also works.
- Keep frontmatter to spec fields if the skills must also upload to the claude.ai or Skills API. Claude-only fields are harmless in other CLIs, which ignore unknown fields, but can break strict validators.

### Gaps
- Agent Skills spec governance and versioning: the spec page shows no version number, date, or steward. I did not verify whether it moved under AAIF.
- Cursor's skills support and directory were not verified (the rules page doesn't mention skills).
- Claude Code reading `.agents/skills` natively was not found in the docs. The skills page was read only to 100k of 120k characters, so it may be mentioned in the unread tail.

## 3. MCP: version, transports, auth, registry, governance, per-tool project config

### Takeaway
The current MCP revision is **2026-07-28**, a major breaking release: a stateless core with no `initialize` handshake and no sessions, plus `server/discover`, MRTR, and Tasks moved to an extension. Transports are stdio and Streamable HTTP (HTTP+SSE is Deprecated). Auth is OAuth with Client ID Metadata Documents preferred over DCR. The official registry is still preview. MCP is governed under AAIF/LF with a SEP process and a formal deprecation policy.

### Cited Findings
- **Version and basis:** the current spec is based on `schema/2026-07-28/schema.ts`. JSON-RPC 2.0 is "Stateless, self-contained requests" with "Per-request capability negotiation".
- **Features:**
  - Server features: Resources, Prompts, Tools.
  - Client feature: Elicitation.
  - Optional extensions, negotiated via capabilities: Tasks, Skills over MCP, and MCP Apps (inline interactive UI).
- Source for the bullets above: [MCP specification (latest)](https://modelcontextprotocol.io/specification/latest)
- **2026-07-28 changelog (vs 2025-11-25):**
  - Sessions and the `Mcp-Session-Id` header are removed. Cross-call state now uses server-minted handles (SEP-2567).
  - The `initialize` handshake is removed. Version and capabilities travel in `_meta` on each request (SEP-2575).
  - New mandatory `server/discover`.
  - `subscriptions/listen` replaces the GET endpoint and `resources/subscribe`.
  - `ping` and `logging/setLevel` are removed.
  - Tasks became the extension `io.modelcontextprotocol/tasks` (SEP-2663).
  - MRTR (`resultType: "input_required"`) replaces server-initiated requests such as sampling, elicitation, and roots (SEP-2322).
  - SSE resumability (`Last-Event-ID`) is removed.
  - OpenTelemetry trace context (`traceparent`, `tracestate`, `baggage`) is documented in `_meta` (SEP-414).
  - `Mcp-Method` and `Mcp-Name` headers are required.
  - `ttlMs` and `cacheScope` are required on list results.
  - JSON Schema 2020-12 is allowed in tool schemas.
  - Auth: `iss` validation (RFC 9207) and issuer-bound credentials.
  - **Deprecated:** Roots, Sampling, Logging, the HTTP+SSE transport, and OAuth Dynamic Client Registration (in favor of Client ID Metadata Documents).
  - A feature lifecycle policy now sets a minimum 12-month deprecation window, and SEPs are filed as PR-based markdown in `seps/`.
  - Source: [MCP changelog 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- **Release timeline:** the release candidate date conflicts across secondary sources (May 21 vs June 29, 2026). Final release was 2026-07-28. — (secondary) [The New Stack](https://thenewstack.io/mcp-release-candidate-rewrite/); [MCP blog release tag](https://blog.modelcontextprotocol.io/tags/release/)
- **Governance:** contributed by Anthropic to AAIF/LF on 2025-12-09. — [LF press release](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- **Registry:** `registry.modelcontextprotocol.io` is still described as **preview** ("breaking changes or data resets may occur before general availability"). The API froze at v0.1 on 2025-10-24. Entries use `server.json` metadata, and the registry hosts metadata only. Publishing uses the `mcp-publisher` CLI. — [MCP registry docs](https://modelcontextprotocol.io/registry/about.md) (via search snippet); (secondary) [Atlan](https://atlan.com/know/ai-agent/what-is-an-mcp-registry/)
- **Per-tool project config (partially verified):**
  - Codex: project config lives in a trusted-project `.codex/` directory (`config.toml`, `hooks.json`). — [Codex hooks docs](https://learn.chatgpt.com/docs/hooks)
  - Gemini CLI: project settings are in `.gemini/settings.json`, merged over user `~/.gemini/settings.json` and system `/etc/gemini-cli/settings.json`. — [Gemini CLI hooks docs](https://geminicli.com/docs/hooks/)

### Inferences
- loop-harness's Go backend should target 2026-07-28 semantics (stateless, `server/discover`) while tolerating 2025-11-25 clients, because many CLIs lag. `server/discover` serves as the backward-compatibility probe on stdio.
- For one-config-many-tools, generate per-tool MCP files from a single source manifest, since no cross-tool MCP config file standard exists.

### Gaps
- Not verified on primary pages this session:
  - Claude Code's project file `.mcp.json` (widely known)
  - Codex's `[mcp_servers]` table in `config.toml`
  - Gemini's `mcpServers` key
  - Cursor's `.cursor/mcp.json`
  - VS Code/Copilot's `.vscode/mcp.json`
- I found no cross-vendor shared MCP config file standard.
- Which clients already implement 2026-07-28 is unknown.

## 4. Hooks and lifecycle events: is there a cross-tool standard?

### Takeaway
**There is no formal cross-tool hooks standard.** A de facto convention, modeled on Claude Code, has converged. Command handlers get JSON on stdin and return JSON on stdout. Exit code 2 means block, with stderr as the reason, and events are named PreToolUse/PostToolUse-style. Event names, casing, and payload fields differ per tool. Cursor explicitly loads Claude Code hooks, and Codex, Gemini, and Cursor set Claude env-var aliases.

### Cited Findings

#### Claude Code
- **Events (33):** SessionStart, Setup, UserPromptSubmit, UserPromptExpansion, PreToolUse, PermissionRequest, PermissionDenied, PostToolUse, PostToolUseFailure, PostToolBatch, Notification, MessageDisplay, SubagentStart/Stop, TaskCreated/Completed, Stop, StopFailure, TeammateIdle, InstructionsLoaded, ConfigChange, CwdChanged, DirectoryAdded, FileChanged, WorktreeCreate/Remove, Pre/PostCompact, Pre/PostModelSwitch, Elicitation, ElicitationResult, SessionEnd.
- **Config:** `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed settings, plugin `hooks/hooks.json`, and skill or subagent frontmatter. Entries merge across levels.
- **Handlers:** `command`, `http`, `mcp_tool`, `prompt`, and `agent` (experimental).
- **Common stdin fields:** `session_id`, `prompt_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`, and others.
- **Exit codes:** 2 blocks on blocking events (`PermissionRequest` ignores exit 2). Other codes are non-blocking unless `"onFailure": "block"` is set (v2.1.295+).
- Source: [Claude Code hooks docs](https://code.claude.com/docs/en/hooks)

#### Codex
- **Events:** PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, UserPromptSubmit, SubagentStop, Stop, PreModelRequest, PostModelRequest, Interrupt, SessionStart, SubagentStart, SessionEnd. These are Claude-style PascalCase names.
- **Config:** `/etc/codex/hooks.json`, `~/.codex/hooks.json`, trusted-project `.codex/hooks.json`, a `[hooks]` table in `config.toml`, plugin `hooks/hooks.json`, or managed `requirements.toml`. Non-managed hooks must be trusted via `/hooks`.
- **Payload:** stdin JSON with `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `model`. Exit 2 with stderr blocks.
- **Handlers:** `command` and `mcp_tool`. `prompt` and `agent` are parsed but skipped.
- **Enablement:** on by default (`[features] hooks=false` disables it).
- **Claude compatibility:** sets `CLAUDE_PLUGIN_ROOT` and `CLAUDE_PLUGIN_DATA` "for compatibility with existing plugin hooks".
- Source: [Codex hooks docs](https://learn.chatgpt.com/docs/hooks)

#### Gemini CLI
- **Events (11):** SessionStart, SessionEnd, BeforeAgent, AfterAgent, BeforeModel, AfterModel, BeforeToolSelection, BeforeTool, AfterTool, PreCompress, Notification. These names differ from Claude's.
- **Config:** in `settings.json` (project, then user, then system, then extensions).
- **Handlers:** `command` only, default timeout 60s.
- **Exit codes:** 0 means stdout JSON is parsed, 2 blocks.
- **Claude compatibility:** sets `CLAUDE_PROJECT_DIR` as a compatibility alias.
- Source: [Gemini CLI hooks docs](https://geminicli.com/docs/hooks/)

#### Cursor
- **Agent events:** sessionStart/End, preToolUse, postToolUse, postToolUseFailure, subagentStart/Stop, beforeShellExecution, afterShellExecution, beforeMCPExecution, afterMCPExecution, beforeReadFile, afterFileEdit, beforeSubmitPrompt, preCompact, stop, afterAgentResponse, afterAgentThought. These are camelCase.
- **Other events:** Tab hooks and `workspaceOpen`.
- **Config:** enterprise MDM, then team dashboard, then `.cursor/hooks.json`, then `~/.cursor/hooks.json`.
- **Payload:** stdin JSON (`conversation_id`, `generation_id`, `hook_event_name`, `workspace_roots`, and others). Stdout carries `permission: allow|deny|ask`. Exit 2 blocks. Errors fail open unless `failClosed` is set.
- **Claude compatibility:** "supports loading hooks from third-party tools like Claude Code", sets `CLAUDE_PROJECT_DIR`, and Claude Code hooks get `loop_limit` null by default.
- Source: [Cursor hooks docs](https://cursor.com/docs/agent/hooks)

#### GitHub Copilot
- The skills page does not document hooks or a hooks location. — [GitHub Docs, agent skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)

### Inferences
- For loop-harness, write hook *scripts* once, in Go or shell, that read stdin JSON, normalize `hook_event_name` across the PascalCase and camelCase variants, and use the exit-2 and stderr contract. Then register them from thin per-tool config files (`.claude/settings.json`, `.codex/hooks.json`, `.gemini/settings.json`, `.cursor/hooks.json`).
- The common intersection of events is SessionStart/End, user-prompt-submit, pre/post tool, pre-compact, and stop.
- The hook scripts are the natural telemetry tap for a dashboard: forward the normalized events to the Go backend.

### Gaps
- I found no AAIF or other standards-body effort on hooks.
- I did not verify Copilot (CLI or VS Code) hooks support or file locations.
- Cursor's "Third Party Hooks" reference (exactly which `.claude` files it reads) was not retrieved.

## 5. A2A, ACP (Zed), AG-UI, OpenTelemetry GenAI: relevance for a harness with a dashboard

### Takeaway
Each protocol covers a different layer:
- **ACP:** editor ↔ coding agent (JSON-RPC over stdio, like LSP). It is the most directly useful for driving many coding-agent CLIs from one harness.
- **AG-UI:** agent backend ↔ frontend event stream. Relevant to the Next.js dashboard.
- **A2A:** agent ↔ agent. Relevant only for remote or multi-agent delegation.
- **OTel GenAI semconv:** tracing vocabulary. Still in Development status, but already referenced by MCP.

### Cited Findings
- **A2A:**
  - Created by Google in April 2025 and hosted by the LF since June 2025.
  - v1.0 is the "first stable specification", announced in an LF press release dated 2026-04-09. It added multi-protocol support, multi-tenancy, modernized security, and Signed Agent Cards. It has more than 150 supporting organizations and is integrated in Azure AI Foundry, Copilot Studio, and Bedrock AgentCore. — [LF press release](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year)
  - Moved into AAIF on 2026-08-17. — [AAIF blog](https://aaif.io/blog/a2a-joins-aaif)
  - Secondary sources date v1.0.0 to 2026-03-12, a v1.0.1 patch to May 2026, and say v1.1 is at RC (unverified; the dev.to sources are low quality).
- **ACP (Agent Client Protocol):**
  - Standardizes editor/IDE ↔ coding-agent communication and is modeled on LSP.
  - Local agents run as editor subprocesses over JSON-RPC on stdio. Remote agents (HTTP/WebSocket) are "a work in progress".
  - Reuses MCP JSON types and adds coding-specific types such as diffs.
  - Source: [agentclientprotocol.com](https://agentclientprotocol.com/overview/introduction)
  - Created by Zed in August 2025, joined by JetBrains. The ACP Registry launched 2026-01-28 inside Zed and JetBrains IDEs (2025.3.2+), and agents are listed via GitHub PR. — [Zed blog](https://zed.dev/blog/acp-registry); [JetBrains blog](https://blog.jetbrains.com/ai/2026/01/acp-agent-registry/)
  - Claude Code and Gemini CLI are reportedly installable from the registry (secondary). Codex adapter status was not found.
- **AG-UI:**
  - An open, event-based protocol connecting agent backends to user-facing apps (streaming state, UI intents, user interactions) over HTTP and WebSockets.
  - Originated with CopilotKit with LangChain and CrewAI. The GitHub org is `ag-ui-protocol`.
  - Layering per AG-UI: AG-UI is agent↔user, MCP is agent↔tools, A2A is agent↔agent.
  - Integrations include Microsoft Agent Framework, Google ADK, Google Antigravity, AWS Strands and Bedrock AgentCore, Mastra, Pydantic AI, LlamaIndex, and community Claude Agent SDK, OpenAI Agents SDK, and Cloudflare Agents.
  - No version number is stated.
  - Source: [AG-UI docs](https://docs.ag-ui.com/introduction)
- **OTel GenAI semconv:**
  - Moved out of the core semconv repo into `open-telemetry/semantic-conventions-genai` (README: spans, metrics, events for GenAI, including MCP). The repo has no tagged releases yet and the Schema URL is "TODO". — [opentelemetry.io GenAI page](https://opentelemetry.io/docs/specs/semconv/gen-ai/); [GitHub repo](https://github.com/open-telemetry/semantic-conventions-genai)
  - Agent spans have **Development** status.
  - Operations: `create_agent` (CLIENT), `invoke_agent`, `invoke_workflow`, `invoke_{entrypoint}` (INTERNAL), `plan`, and `execute_tool` (separate doc).
  - Key attributes: `gen_ai.operation.name` (required), `gen_ai.agent.name`, `gen_ai.agent.id`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.conversation.id`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, and `error.type`.
  - Message and content capture is opt-in, with `_ref` variants for external storage.
  - Source: [agent spans doc](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md)
  - MCP 2026-07-28 documents propagation of `traceparent`, `tracestate`, and `baggage` via `_meta` and recommends OpenTelemetry instead of the deprecated MCP Logging. — [MCP changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)

### Inferences
- **Dashboard telemetry:** model the Go backend's run, iteration, and tool-call data on OTel GenAI agent spans (`invoke_agent` per loop iteration, `execute_tool` per tool call). The vocabulary is Development status, so pin the semconv version and expect renames.
- **Frontend streaming:** AG-UI is a ready-made event schema for streaming run state from the Go backend to Next.js, with CopilotKit as the client. A plain SSE design with OTel-shaped events is a lower-dependency alternative.
- **Controlling CLIs:** ACP is the closest thing to a standard for programmatically driving heterogeneous coding-agent CLIs over stdio. Consider it as one adapter type alongside per-CLI headless modes.
- **A2A:** low priority unless loop-harness delegates to remote agents.

### Gaps
- Not determined:
  - the exact current ACP protocol version
  - the ACP governance home (whether it is still Zed/JetBrains-owned)
  - an official Codex ACP adapter
  - AG-UI's event count and version
  - the OTel `OTEL_SEMCONV_STABILITY_OPT_IN` values for GenAI
- Not verified: which coding CLIs emit OTel GenAI spans natively.

## 6. Recommended repo layout for multi-agent-tool compatibility

### Takeaway
No vendor publishes a single canonical "multi-tool" layout. The layout below combines documented conventions (AGENTS.md as canonical, `.agents/skills/` as the cross-client skills directory) with each tool's native config directories, linked by symlinks or imports, and avoids duplicated content.

### Cited Findings
- agents.md recommends renaming to AGENTS.md and symlinking legacy names. — [agents.md](https://agents.md)
- agentskills.io calls `.agents/skills/` the widely adopted cross-client convention. — [agentskills.io guide](https://agentskills.io/client-implementation/adding-skills-support)
- Claude Code documents `@AGENTS.md` import and CLAUDE.md→AGENTS.md symlink as supported. Since v2.1.277 neither is needed when no CLAUDE.md exists. — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Codex symlinked skill folders are followed (via search snippet). — [Codex skills docs](https://developers.openai.com/codex/skills)
- `.claude/rules/` supports symlinks. Symlink targets outside the project need external-import approval. — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Copilot reads `.claude/skills` and `.agents/skills`, plus root `CLAUDE.md` and `GEMINI.md`. — [GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)

### Inferences (proposed layout for loop-harness)
```
AGENTS.md                      # canonical, tool-agnostic (<32 KiB total chain for Codex)
CLAUDE.md                      # optional: "@AGENTS.md" + Claude-only notes (needed if any CLAUDE.md exists)
backend/AGENTS.md              # Go-specific (nearest-wins in all tools)
frontend/AGENTS.md             # Next.js-specific
.agents/skills/<name>/SKILL.md # canonical skills (Codex, Gemini, Copilot native)
.claude/skills -> ../.agents/skills   # symlink for Claude Code
.claude/settings.json          # Claude hooks + permissions -> scripts/hooks/*
.codex/config.toml, .codex/hooks.json # Codex (trusted project)
.gemini/settings.json          # Gemini: context filename -> AGENTS.md, hooks, MCP
.cursor/hooks.json, .cursor/rules/*.mdc (only if needed; Cursor reads AGENTS.md)
.github/copilot-instructions.md (optional; Copilot reads AGENTS.md)
.mcp.json (+ generated per-tool MCP configs from one manifest)
scripts/hooks/                 # one implementation, normalizes event names across tools
```
- Generate the per-tool config files (MCP and hooks) from one manifest using a small Go generator in the harness, because no shared config standard exists for MCP or hooks.
- Windows symlink caveat: git symlinks need `core.symlinks`. Prefer the `@AGENTS.md` import for CLAUDE.md where symlinks are risky.

### Gaps
- I found no official vendor-published multi-tool repo template. Widely adopted community templates (for example, ones syncing rules across tools) were not researched within the call budget.
- Cursor's native skills directory and whether it reads `.agents/skills` were not verified.
