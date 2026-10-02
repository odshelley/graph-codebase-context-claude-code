# Commercial and Enterprise Graph-Augmented Code-Context Platforms for Claude Code

Survey date: 2026-10-01. Audience: an enterprise team evaluating options that already runs Neo4j and wants code to stay on-prem.

Method: every factual claim below comes from a page fetched during this session unless marked UNVERIFIED. Source dates are given where the page shows one. Vendor claims are labelled as vendor claims. Open-source graph MCP servers are out of scope here and are covered separately.

## 1. Headline findings

1. **Anthropic ships no code index or code-graph product for Claude Code and recommends none.** Its documented position is agentic search over the filesystem (Glob, Grep, Read), plus optional Language Server Protocol (LSP) plugins for precise symbol navigation, with MCP as the extension point for anything else.
2. **Only three commercial products expose a true structural, symbol-level representation to Claude Code:** Sourcegraph (SCIP precise index), Moderne (Lossless Semantic Trees), and the JetBrains IDE's built-in MCP server (IDE index). Everything else marketed as a "graph" or "context engine" is embeddings-based, undisclosed, or a graph that powers the vendor's own review product without being queryable by your agent.
3. **No commercial product in this set runs on, or exports to, a customer-operated Neo4j.** The only open index format present is SCIP, which the team could load into Neo4j themselves.
4. **Market churn is high.** Tabnine was acquired by Tricentis (2026-07-30). Windsurf became Devin Desktop under Cognition. Amp spun out of Sourcegraph (2025-12-02). Cody Free and Pro were discontinued (2025-07-23). Cursor dropped embedding-based indexing. Blar.io and Nuanced pivoted away from code graphs. CodeSee and Stepsize were acquired.

## 2. Anthropic's official position

### What the docs say

Claude Code's documentation describes the agentic loop as "gather context, take action, verify results" using built-in file, search, execution, web, and code-intelligence tools. The search category is defined as "Find files by pattern, search content with regex, explore codebases" (https://code.claude.com/docs/en/how-claude-code-works, fetched 2026-10-01).

Code intelligence means LSP, not a graph: "A code intelligence plugin gives Claude the live diagnostics and go-to-definition that your editor has, so Claude catches type errors and missing imports that its own edits introduce before you run your build, and finds definitions and references by symbol instead of by text search." Anthropic maintains official LSP plugins for C/C++, C#, Go, Java, Kotlin, Lua, PHP, Python, Ruby, Rust, Swift, and TypeScript/JavaScript, with Shopify maintaining Liquid (https://code.claude.com/docs/en/plugins/code-intelligence). The LSP tool is read-only and does not run in cloud sessions.

The best-practices page says: "If you work with a typed language, install a code intelligence plugin to give Claude precise symbol navigation and automatic error detection after edits," and "Let Claude fetch what it needs. Tell Claude to pull context itself using Bash commands, MCP tools, or by reading files." It never mentions indexing, RAG, embeddings, or a remote index (https://code.claude.com/docs/en/best-practices).

The MCP page documents enterprise controls: `managedMcpServers` for org-provided servers, `allowedMcpServers` / `deniedMcpServers` allowlists and denylists, and `disabledMcpjsonServers` to block project-level servers (https://code.claude.com/docs/en/mcp). The security page states: "Anthropic reviews connectors against its listing criteria before adding them to the Anthropic Directory, but does not security-audit or manage any MCP server," and "We encourage either writing your own MCP servers or using MCP servers from providers that you trust" (https://code.claude.com/docs/en/security).

### The agentic-search rationale

The widely quoted rationale is from Boris Cherny, head of Claude Code, on X: "Early versions of Claude Code used RAG + a local vector db, but we found pretty quickly that agentic search generally works better. It is also simpler and doesn't have the same issues around security, privacy, staleness, and reliability." The X post (https://x.com/bcherny/status/2017824286489383315) returned HTTP 402 to a direct fetch, so the quote is verified only via secondary sources that link it, including vadim.blog (2026-03-03). Gergely Orosz's Pragmatic Engineer write-up of his podcast with Cherny (2026-03-04) summarises: "Claude Code's 'agentic search' is really just glob and grep, and it outperformed RAG," noting the team tried "local vector databases, recursive model-based indexing, and other fancy approaches" and found stale indexes and permission complexity. That sentence is Orosz's paraphrase, not a direct Cherny quote.

### Independent evidence on agentic search vs RAG

Evidence is thin and not code-specific. Robert Eubanks' local-hardware study (2026-04-06, 42 questions over four documentation corpora, Mac Mini M4 Pro) scored RAG at 3.2/5, cloud-assisted agentic search at 4.7/5, and fully local agentic search at 3.2/5, concluding "the architecture is not the problem. The local model's capability is the bottleneck." The Milvus blog arguing the opposite (2025-08-25) is Zilliz vendor content claiming a "40% token reduction" with no published methodology.

A notable corroborating data point: Cursor has also moved to a local grep index. Its current docs state "Cursor does not upload file paths or code to build a search index, and it does not store embeddings of your codebase for search" (https://cursor.com/docs/agent/tools/search), whereas its 2026-01-27 engineering blog still described Merkle-tree embedding indexing in Turbopuffer. The change surfaced on Hacker News in late August 2026 with no official Cursor statement found.

## 3. Product entries

### 3.1 Sourcegraph

- **Canonical URL:** https://sourcegraph.com/docs/api/mcp (live).
- **What it indexes / representation:** Every repository, branch, and revision into a trigram search index; the vendor positions this as "deterministic search" returning "complete result sets with accurate totals" rather than ranked approximations. Precise navigation uses SCIP (SCIP Code Intelligence Protocol), a language-agnostic symbol index with GA indexers for Go, TypeScript/JavaScript, C/C++/CUDA, Java/Kotlin/Scala, Rust, Python, Ruby, and C#/Visual Basic. Indexes are produced manually, in CI, or by auto-indexing executors; search-based navigation is the fallback (https://sourcegraph.com/docs/code-navigation/precise-code-navigation). SCIP moved to open governance on 2026-03-25 with a steering committee of Catherine Gasnier (Meta), Jamy Timmermans (Uber), and Michal Kielbowicz (Sourcegraph); the project now lives at scip-code.org with an RFC-style SEP process (https://sourcegraph.com/blog/the-future-of-scip). Deep Search is an agentic layer that iterates over the index "until it is confident in its answer" and runs inside your instance "with only LLM calls made externally" (https://sourcegraph.com/docs/deep-search).
- **Claude Code integration:** Official, GA since 2026-02-25 with OAuth Dynamic Client Registration on by default (https://sourcegraph.com/changelog/mcp-ga). Setup:

  ```
  claude mcp add --transport http sourcegraph https://sourcegraph.example.com/.api/mcp
  ```
  then `/mcp` for browser OAuth. A token-header variant exists for project `.mcp.json` (https://sourcegraph.com/docs/api/mcp/client-integrations). Tools: read_file, list_files, list_repos, list_refs, keyword_search, nls_search, evaluator, go_to_definition, find_references, commit_search, diff_search, compare_revisions, get_contributor_repos, code_finder, deepsearch, deepsearch_read. Admins can disable deepsearch via `SRC_MCP_DISABLE_DEEPSEARCH_TOOL=true`. A community plugin (https://github.com/sourcegraph-community/sourcegraph-claudecode-plugin) wraps the server with a `searching-sourcegraph` skill; it needs Claude Code 1.0.33+ and a token with the `mcp` scope.
- **Deployment / compliance:** Single-tenant Sourcegraph Cloud or self-hosted on Kubernetes with Helm ("the most robust, scalable, and vetted self-hosted solution") or Docker Compose (https://sourcegraph.com/docs/self-hosted/deploy). MCP requires Cloud or self-hosted 7.0 on the Enterprise plan. An offline executor deployment guide exists. ARM64 is not supported in production. Deep Search is "not supported for BYOK customers" and "uses a combination of frontier and specialized models selected by Sourcegraph," so a fully air-gapped install keeps search and precise navigation but loses Deep Search. SOC 2 / ISO claims were on no fetched page: UNVERIFIED.
- **Pricing:** Enterprise from a $16,000 minimum annual contract, scaling with team size, with credits for AI features that roll over at renewal (https://sourcegraph.com/pricing). Cody Enterprise at $59/user/month appears only in third-party pages: UNVERIFIED.
- **Scale claims:** Case studies cite Reddit at 2,000+ repos, F5 at 350+, CERN at a 15-million-line Java codebase, MathWorks across Perforce, GitHub, and GitLab (https://sourcegraph.com/case-studies). The monorepo doc describes horizontal index sharding but states no limits (https://sourcegraph.com/docs/admin/monorepo). The "300,000+ repositories" figure appears only in third-party summaries: UNVERIFIED.
- **Independent evidence:** Meta and Uber taking SCIP governance seats is a real adoption signal. The one hands-on third-party comparison found (https://baeseokjae.github.io/posts/ai-coding-agent-code-index-mcp-comparison-2026/, 2026-07-06) reports 20 to 40 minute initial indexing on large codebases and recommends Sourcegraph for multi-repo enterprises; measurements are the author's own.
- **Caveats / lock-in:** Cody Free and Pro ended 2025-07-23 and Cody is Enterprise-only (https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans). Amp became a separate company on 2025-12-02 with Dan Adler as Sourcegraph CEO (https://sourcegraph.com/blog/why-sourcegraph-and-amp-are-becoming-independent-companies). The context-compare page is marketing. Lock-in is moderate: SCIP is open and portable, while Deep Search and the MCP tool surface are proprietary.

### 3.2 Augment Code

- **Canonical URLs:** https://www.augmentcode.com/product/context-engine-mcp and https://docs.augmentcode.com/context-services/mcp/overview.md (live).
- **Representation:** Embeddings. The connector docs say "Context Engine converts query to vector" and "stores embeddings and handles queries," with file-hash-based incremental updates and only changed files sent for embedding (https://docs.augmentcode.com/context-services/context-connectors/how-it-works.md). No graph is documented anywhere. Index state can persist locally or in S3.
- **Claude Code integration:** Official quickstart (https://docs.augmentcode.com/context-services/mcp/quickstart-claude-code.md). Local mode runs the Auggie CLI as a stdio MCP server:

  ```
  npm install -g @augmentcode/auggie@latest
  auggie login
  claude mcp add-json auggie --scope user '{"type":"stdio","command":"auggie","args":["--mcp","--mcp-auto-workspace"]}'
  ```
  Remote mode connects to Augment's hosted HTTP server configured at app.augmentcode.com/mcp/configuration and indexes the default branches of selected GitHub repositories. One tool is documented: `codebase-retrieval`. Twelve clients have quickstarts (Claude Code, Codex, Cursor, Zed, Copilot, OpenCode, Kilo Code, Kiro, Antigravity, Gemini CLI, Roo Code, Droid).
- **Deployment / compliance:** SaaS. The security page lists SOC 2 Type II, ISO/IEC 42001, customer-managed encryption keys, GDPR, CCPA, HIPAA with BAA, and "we never train on our customer's proprietary data"; it does not mention VPC, on-prem, or air-gapped (https://www.augmentcode.com/security). On-prem and air-gapped claims appear only on Augment's own comparison pages about its Cosmos platform; the independent SiliconANGLE coverage of Cosmos (2026-06-05) says nothing about deployment. Treat as UNVERIFIED.
- **Pricing:** Standard $20/month flat and Business $100/month flat, each capped at 50 seats; Enterprise custom with CMEK and multi-region compute (https://www.augmentcode.com/pricing). Context Engine MCP queries are metered at "$0.03–$0.06 per query" average, billed at LLM provider rates plus a 40% service fee.
- **Scale claims:** No documented limits. "400,000+ files" and "500,000 files" figures appear only in Augment marketing pages.
- **Independent evidence:** None. The vendor benchmark (blog 2026-02-06, updated 2026-06-18) used 300 Elasticsearch PRs with three prompts each and claims +80% for Claude Code with Opus and +71% for Cursor with Opus on correctness, completeness, best practices, code reuse, and documentation. No replication found.
- **Caveats / lock-in:** The remote index lives in Augment's cloud, remote mode is GitHub-only per docs, retrieval is proprietary, and the product is being repositioned around the Cosmos agent platform.

### 3.3 Greptile

- **Canonical URL:** https://www.greptile.com/docs/mcp-v2/setup (live).
- **Representation:** The homepage says "Builds a graph of your repo - files, functions, and dependencies" and the docs say "a complete graph of your codebase - every function, class, and dependency." The self-host stack requires PostgreSQL with pgvector plus Redis, so the implementation is at least partly vector-based; internals are undisclosed.
- **Claude Code integration:** Official:

  ```
  claude mcp add --transport http greptile https://api.greptile.com/mcp
  claude mcp login greptile
  ```
  plus an official plugin (https://github.com/greptileai/claude-plugin) adding `/greptile:review` and `/greptile:login`; it needs Claude Code 2.1+ and Node 22+. **Key finding:** the full tool list (https://www.greptile.com/docs/mcp-v2/tools) covers account, pull requests, code reviews, comment search, custom context patterns, synthesized knowledge-base documents, and analytics. No tool searches source code or answers natural-language questions about the codebase. The graph powers Greptile's review product and is not exposed as a context API.
- **Deployment / compliance:** Cloud, or self-hosted on Enterprise via Docker Compose (single VM) or Kubernetes with Helm; air-gapped supported; BYO LLM via OpenAI-compatible endpoints or AWS Bedrock; supports GitHub Enterprise Server, GitLab Self-Managed, Bitbucket Data Center, Perforce (https://www.greptile.com/docs/deployment-options, https://www.greptile.com/docs/security/selfhost). Sizing runs from 4 cores / 16 GB for 5 to 10 developers to 10 to 20+ nodes for 500+. SOC 2 is claimed on the homepage as "independent audits, reports available on request"; the "Type II" qualifier appears only in third-party pages: UNVERIFIED.
- **Pricing:** Free (50 credits, one developer), Pro at $30/seat/month with 50 credits and $1 per additional credit, Enterprise custom with SSO/SAML and self-hosting (https://www.greptile.com/pricing).
- **Scale claims:** "Over 22,000+ teams," "any programming language." Vendor.
- **Independent evidence:** Customer logos only (Nvidia, Brex, Retool, PostHog, Zapier among them).
- **Caveats:** Buys review quality, not agent context.

### 3.4 Qodo

- **Canonical URL:** https://docs.qodo.ai/developer-tools/context-engine-mcp (live).
- **Representation:** "A deep research agent for large codebases" using "Retrieval-Augmented Generation (RAG) with agentic reasoning." Marketing says it maintains "a continuously updated map of how your code is structured and connected." No graph database or embedding details are disclosed.
- **Claude Code integration:** Tools are `deep_research`, `context_ask`, and `get_context`, each accepting `repositories` and `session_id` scoping (https://docs.qodo.ai/developer-tools/context-engine-mcp-tools). Docs name Claude Desktop, Cursor, Windsurf, and GitHub Copilot. Claude Code is not named, but it is a standard MCP server usable from any client, and the homepage lists "Codex, Claude, and Kiro" as agents Qodo works with.
- **Deployment / compliance:** Cloud, hybrid, or "as an MCP server for air-gapped environments"; Enterprise offers single-tenant SaaS or on-prem. Homepage claims SOC 2 Type II, zero data retention, and BYOK for OpenAI, Anthropic, Azure OpenAI, or self-hosted models (https://www.qodo.ai/).
- **Pricing:** 14-day trial; Pro Team at $30/month base plus $0.012 per credit pooled, up to 30 users; Enterprise custom above 30 (https://www.qodo.ai/pricing).
- **Scale claims:** "Thousands of repositories"; a "500-repo enterprise" example. Vendor.
- **Independent evidence:** None. The vendor's DeepCodeBench (blog 2025-09-10; 1,144 questions from PRs in eight open-source repos) scores Qodo 80%, Codex 74%, Claude Code 64%, Gemini CLI 45%. Vendor-run.
- **Caveats:** Context Engine is an Enterprise-tier feature; air-gapped specifics are undocumented.

### 3.5 Unblocked

- **Canonical URL:** https://getunblocked.com/unblocked-mcp/ (live).
- **Representation:** The ai-info page (updated 2026-09-17) says it "traverses a single knowledge graph spanning all connected data sources" through an "eight-stage intelligence pipeline that retrieves, filters, and ranks organizational knowledge." Sources: GitHub, GitLab, Bitbucket, Azure DevOps (including self-managed), Slack, Teams, Zendesk, Jira, Linear, Asana, Confluence, Notion, Google Drive, SharePoint, Datadog, Sentry, Snowflake, PostHog, and CI systems (https://getunblocked.com/ai-info/).
- **Claude Code integration:** Named explicitly ("Claude Code, Cursor, Windsurf, Copilot, or any MCP-compatible agent"). Two tools: `context_research` and `context_get_urls`.
- **Deployment / compliance:** SaaS on AWS; SOC 2 Type II, CASA Tier II, GDPR; encryption at rest with AWS-managed keys (https://getunblocked.com/security/). Enterprise tier lists "On-premises deployment options" (https://getunblocked.com/pricing/).
- **Pricing:** Platform at $29/user/month annual or $35 monthly, which includes MCP access; Enterprise by contact; 21-day trial.
- **Independent evidence:** Vendor metrics only ("9/10 mergeable code, 60% token savings, 83% faster"), no named customers. $20M Series A per the vendor.
- **Caveats:** The knowledge graph is internal and not exposed as a graph; it is organisational context rather than symbol structure.

### 3.6 Tabnine (now Tricentis)

- **Canonical URL:** All tabnine.com marketing URLs 301 to tricentis.com. Surviving docs at https://docs.tabnine.com/.
- **Status:** Tricentis acquired Tabnine on 2026-07-30 to fold the Enterprise Context Engine into its Agentic Quality Engineering Platform (https://www.devopsdigest.com/tricentis-acquires-tabnine; https://sapinsider.org/blogs/tricentis-tabnine-acquisition-agentic-sap-testing/). The $500M price appears in one secondary source: UNVERIFIED.
- **Representation:** Described as "a hybrid graph-plus-vector knowledge model of enterprise systems" and "a structured, continuously updated knowledge graph." The surviving docs say it generates "higher-level summaries, services, dependencies, and structured architectural insights" from connected remote repositories (https://docs.tabnine.com/main/administering-tabnine/managing-your-team/context-engine.md).
- **Claude Code integration:** Not documented. The docs say Context Engine outputs are consumed inside Tabnine Agent; a remote `tabnine-context` MCP server exists but is documented only for Tabnine's OpenCode plugin and exposes `tabnine_context_*` tools (https://docs.tabnine.com/main/getting-started/tabnine-plugin-for-opencode/mcp-servers.md). Pre-acquisition claims that it "works alongside Claude Code" survive only in third-party summaries: UNVERIFIED.
- **Deployment:** Historical on-prem, VPC, and air-gapped claims are UNVERIFIED post-acquisition; current docs list allowlist IPs, implying SaaS.
- **Evidence:** Tricentis cites customer-reported "up to 2x AI accuracy," "up to 80% reduction in token consumption," and "up to 50% faster resolution." Vendor.
- **Assessment:** Do not plan on this product until Tricentis publishes a roadmap for the coding products.

### 3.7 Moderne

- **Canonical URL:** https://docs.moderne.io/user-documentation/agent-tools/mcp/overview/ (live).
- **Representation:** The Lossless Semantic Tree (LST), "a compiler-accurate code model that adds rich type and semantic attribution while preserving formatting, comments, and style," resolving "every symbol to its exact type" against the real classpath and build; LSTv3 supports incremental compilation (https://moderne.ai/platform/lst; https://moderne.ai/blog/local-moderne-mcp-deterministic-tools-coding-agents, 2026-04-17). Languages: Java, JavaScript, TypeScript, C#, Python, Go, Kotlin, Groovy, Scala, COBOL, XML, YAML, with Java most mature (https://www.moderne.ai/solutions/language-support).
- **Claude Code integration:** Named alongside Cursor, Windsurf, Copilot, Amp, Codex, Cline. The local MCP runs via `mod mcp` in the Moderne CLI as a subprocess with no network transmission of code, and is marked experimental; a remote MCP runs on the Moderne Platform against ingested repos. Tools: `trigrep_search`, `trigrep_structural_search` (Comby syntax), `grep`, `find_types`, `find_methods`, `find_annotations`, `find_implementations`, `symbols_overview`, `change_type`, `change_method_name`, `pattern_replace`, `edit_code`, `analyze_code`, `search_recipes`, `learn_recipe`, `run_recipe`, `build_status`, `lst_status`, plus "Prethink" precomputed dependency graphs and service boundaries (https://www.moderne.ai/product/moddy).
- **Deployment / compliance:** SaaS Platform, or Moderne DX for "estate-wide code change entirely airgapped" where "your code never leaves your network" (https://moderne.ai/moderne-dx). The DX page lists Trigrep and Prethink but does not explicitly confirm MCP inside DX. Compliance certifications were on no fetched page: UNVERIFIED.
- **Pricing:** Sales-led; a free community tenant exists.
- **Scale claims:** "Tens of millions of repositories" and "tens of billions of lines parsed"; "7,200" recipes. Vendor.
- **Independent evidence:** Weak. The FreshBrew arXiv benchmark (https://arxiv.org/abs/2510.04852, 2025-10) on 228 Java repos found the best agent migrates 52.3% of projects to JDK 17, but the abstract does not name Moderne. The Moderne docs themselves warn that "agents inconsistently invoke local MCP tools."
- **Caveats:** Java-centric; recipes and LST serialisation are proprietary to the platform even though OpenRewrite is open source.

### 3.8 Glean

- **Canonical URL:** https://developers.glean.com/guides/mcp/claude-code (live).
- **Representation:** An enterprise knowledge graph over connectors; code arrives via GitHub, GitLab, or Bitbucket connectors and the code-indexing method is undocumented. No symbol graph.
- **Claude Code integration:** Official plugin:

  ```
  /plugin marketplace add gleanwork/claude-plugins
  /plugin install glean@glean-plugins
  ```
  OAuth with Dynamic Client Registration; Claude Code is listed as a user-configurable host among 21 (https://developers.glean.com/guides/mcp/supported-hosts.md). The tool list including `code_search` comes from third-party pages, not Glean docs: UNVERIFIED.
- **Deployment / compliance:** Glean-hosted on GCP, or customer-hosted as a managed service in your GCP or AWS account where "Glean still has minimal access to operate it like a SaaS service"; no air-gapped option (https://docs.glean.com/get-started/prepare/about-deployment). Certifications: ISO 42001, ISO 27001, SOC 2 Type II, HIPAA, GDPR, TX-RAMP Level 2 (https://www.glean.com/security).
- **Pricing:** Not public.
- **Assessment:** Suits docs-plus-code organisational context, not code structure.

### 3.9 Zilliz Claude Context

- **Canonical URL:** https://github.com/zilliztech/claude-context (MIT, 12.6k stars).
- **Representation:** Hybrid BM25 plus dense vectors, AST chunking with character fallback, Merkle-tree incremental indexing. Backends: self-hosted Milvus or Zilliz Cloud; embeddings via OpenAI, Voyage, Ollama, or Gemini.
- **Claude Code integration:** Tools `index_codebase`, `search_code`, `clear_index`, `get_indexing_status`:

  ```
  claude mcp add claude-context -e OPENAI_API_KEY=... -e MILVUS_ADDRESS=... -e MILVUS_TOKEN=... -- npx @zilliz/claude-context-mcp@latest
  ```
- **Deployment:** Fully self-hostable with Milvus and Ollama.
- **Evidence:** Vendor "~40% token reduction" claim. Not a graph. Included because Zilliz is a commercial vendor; the server itself is open source.

### 3.10 GitHub Copilot

- **Canonical URLs:** https://github.com/github/github-mcp-server and https://docs.github.com/en/copilot/concepts/context/spaces (live).
- **Representation:** A semantic code search index backs Copilot Chat and the Copilot cloud agent; it is not exposed to third-party agents and is unavailable on GHE.com and GitHub Enterprise Server for non-GitHub repos (https://docs.github.com/en/copilot/concepts/context/repository-indexing). Indexing dropped from about five minutes to under 60 seconds in March 2025.
- **Claude Code integration:** The remote MCP at `https://api.githubcopilot.com/mcp/` works with Claude Code. Copilot Spaces (curated context containers of repos, files, PRs, issues, notes) are readable through the remote-only `copilot_spaces` toolset (`list_copilot_spaces`, `get_copilot_space`). GHE Cloud uses `copilot-api.SUBDOMAIN.ghe.com/mcp`; GHES is local-deployment only.
- **Caveats:** Sourcegraph's claims that Copilot indexes only the default branch and files to about 350 KiB are competitor claims.

### 3.11 Cursor and Devin Desktop (formerly Windsurf), comparison points only

Neither exports its context to Claude Code. Cursor now uses a local Instant Grep index (changelog 2.1, 2025-11-21) and an Explore subagent; it holds SOC 2 Type II, ISO 27001:2022, ISO 42001:2023, and AIUC-1 (https://cursor.com/security, updated 2026-08-25). Cognition acquired Windsurf on 2025-07-14 (https://cognition.com/blog/windsurf) and renamed it Devin Desktop, stating "only the name and branding are changing" (https://devin.ai/desktop). Its docs describe RAG over a local index, remote repository indexing for Teams and Enterprise, and proprietary "M-Query" retrieval (https://docs.devin.ai/desktop/context-awareness/overview). The 2024-02-20 remote-indexing post describes embedding retrieval plus Lucene-style keyword search with reranking. "Riptide" appears only in third-party pages: UNVERIFIED. Devin Desktop Teams is "$80/mo + $40/mo per full seat"; Enterprise is custom.

### 3.12 JetBrains

IDEs from 2025.2 bundle an MCP Server plugin that "external clients such as Claude Code, Codex, VS Code, and others" can use over stdio, Streamable HTTP, or SSE. Tools include call hierarchy ("uses IDE call hierarchy data, so it provides more precise call relationships with less noise"), symbol information, find usages, symbol search, rename, build, lint/diagnostics, run configurations, and debugging on Ultimate editions (https://www.jetbrains.com/help/ai-assistant/mcp.html). Precise, local, and included with the IDE, but single-project and IDE-bound. A third-party "Code Intelligence MCP" marketplace plugin exists but its page would not render: UNVERIFIED.

### 3.13 Adjacent products that are not code graphs

- **CodeScene:** MCP runs fully locally, exposes Code Health, hotspot, ownership, delta-review, and refactoring-business-case tools; installs as a Claude Code plugin via `/plugin marketplace add codescene-oss/codescene-mcp-server` then `/plugin install codescene@codescene`; priced at €18 (Standard) or €27 (Pro) per active author per month with on-prem on both tiers (https://github.com/codescene-oss/codescene-mcp-server; https://www.codescene.com/pricing).
- **Semgrep:** The MCP server moved into the main `semgrep` binary on 2025-10-28; tools include `security_check`, `semgrep_scan`, `semgrep_scan_with_custom_rule`, `get_abstract_syntax_tree`, `semgrep_findings`; install with `claude mcp add semgrep uvx semgrep-mcp`. The Guardian plugin bundles the MCP server, hooks on every file write, and skills (https://github.com/semgrep/mcp; https://github.com/semgrep/mcp-marketplace).
- **CodeQL:** GitHub's `advanced-security/codeql-development-mcp-server` is for authoring CodeQL queries, not serving codebase context.
- **Weaviate / Pinecone:** Weaviate ships a built-in MCP at `/v1/mcp` with hybrid search and AST-chunked code (blog 2026-05-21). Pinecone's Claude Code plugin (2026-02-11) is for building apps on Pinecone. Both are infrastructure, not products.

### 3.14 Pivoted or acquired

- **Blar.io** is now a product-onboarding agent with no code-graph offering (https://blar.io/).
- **Nuanced** archived its Python call-graph library on 2026-03-05 and its homepage now sells a spec-driven coding workspace (https://github.com/nuanced-dev/nuanced; https://nuanced.dev/).
- **CodeSee** was acquired by GitKraken on 2024-05-14 and **Stepsize** by ClickUp, both per search results only, not fetched.
- **Swimm** claims dependency maps "queryable by MCP," on-prem and air-gapped deployment, SOC 2 and ISO 27001, and COBOL/JCL/PL/I support on its homepage, but its docs site documents no MCP server and Claude Code support is UNVERIFIED (https://swimm.io/; https://docs.swimm.io/).

## 4. Comparison table

| Product | Representation | Claude Code integration | On-prem / air-gap | Public price | Independent evidence |
|---|---|---|---|---|---|
| Sourcegraph | Search index + SCIP symbol graph, precise | Official HTTP MCP (GA 2026-02-25), 16 tools, community plugin | Yes, K8s/Compose; Deep Search needs Sourcegraph LLMs, not BYOK | $16K/yr minimum | SCIP governance by Meta/Uber; one hands-on third-party review |
| Augment Code | Embeddings, proprietary | Official; local stdio (Auggie) or hosted HTTP; 1 tool | SaaS; VPC/air-gap only on marketing pages, UNVERIFIED | $20/$100 flat tiers; $0.03-0.06 per query | Vendor benchmark only |
| Greptile | "Graph" + pgvector, undisclosed | Official HTTP MCP + plugin; review tools only, no code search | Yes, Compose/Helm, air-gapped, BYO LLM | $30/seat/mo | Customer logos only |
| Qodo | RAG + agentic reasoning, undisclosed | Any MCP client; 3 tools; Claude Code not named | Single-tenant SaaS, on-prem, air-gapped MCP | $30/mo base + $0.012/credit | Vendor benchmark only |
| Unblocked | Internal knowledge graph | Named; 2 tools | Enterprise on-prem option | $29/user/mo | Vendor metrics only |
| Tabnine / Tricentis | Hybrid graph + vector | None documented for Claude Code | UNVERIFIED post-acquisition | None | Vendor claims |
| Moderne | LST, compiler-accurate | Named; local CLI MCP (experimental) + Platform MCP | Yes, Moderne DX air-gapped | Sales-led | Weak; FreshBrew is adjacent |
| Glean | Enterprise knowledge graph, no symbol graph | Official plugin, OAuth DCR | Customer-hosted managed in GCP/AWS; no air-gap | None | None |
| Zilliz Claude Context | Hybrid BM25 + vectors | Official MCP, 4 tools | Yes, self-hosted Milvus | Free / Zilliz Cloud | Vendor claim |
| GitHub Copilot | Semantic index (not exposed) + Spaces | Remote MCP; Spaces readable | GHES local-only; no semantic index on GHES | Copilot licence | None |
| JetBrains IDE MCP | IDE index, precise | Built-in since 2025.2 | Local | Included in IDE | Not needed |
| CodeScene | Code Health metrics | Plugin | Yes | €18-27/author/mo | None |
| Cursor / Devin Desktop | Local grep / RAG | None | SaaS | n/a | n/a |

## 5. Assessment for an on-prem enterprise that already runs Neo4j

The realistic shortlist is short.

**Sourcegraph self-hosted** is the only mature commercial option that delivers a precise cross-repo symbol graph to Claude Code through an official, GA MCP server, and SCIP is now community-governed so the index data is portable. The trade-offs are that Deep Search is unavailable under BYOK or air-gapped setups and that the minimum contract is $16K per year. SOC 2 status should be confirmed with the vendor since it was not on any fetched page.

**Moderne** is the strongest choice if the estate is Java-heavy and the goal is deterministic refactoring and type-accurate search. It runs fully air-gapped via Moderne DX, but its local MCP is experimental, DX's MCP availability is not explicitly confirmed, and compliance claims were not verifiable.

**Greptile** self-hosts well and is air-gap capable with BYO LLM, but its MCP does not expose the graph, so it buys review quality rather than agent context.

**Qodo and Unblocked** both offer on-prem Enterprise tiers but with undisclosed internals and vendor-only evidence. **Augment** should be treated as SaaS until its deployment claims appear in docs. **Glean** has no air-gapped mode and Glean retains operational access even in customer-hosted deployments. **Tabnine** should be paused pending Tricentis' roadmap.

**On Neo4j specifically:** no vendor in this set runs on or exports to a customer Neo4j. The practical graph-in-Neo4j path is to generate SCIP indexes with the open indexers in CI, load them into Neo4j with a custom or open-source loader, and expose that over an in-house MCP server. A likely landing configuration is Sourcegraph for search and precise navigation, Anthropic's official LSP plugins for in-session diagnostics, and a Neo4j-backed MCP for the cross-service, ownership, and runtime relationships that SCIP alone does not carry.

## 6. Every URL used

### Anthropic / Claude Code
- https://code.claude.com/docs/en/how-claude-code-works
- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/plugins/code-intelligence
- https://code.claude.com/docs/en/mcp
- https://code.claude.com/docs/en/security
- https://code.claude.com/docs/en/overview
- https://x.com/bcherny/status/2017824286489383315 (HTTP 402, not fetched)
- https://vadim.blog/claude-code-no-indexing/ (2026-03-03)
- https://newsletter.pragmaticengineer.com/p/building-claude-code-with-boris-cherny (2026-03-04)
- https://news.ycombinator.com/item?id=46631689
- https://robertheubanks.substack.com/p/anthropic-replaced-their-rag-pipeline (2026-04-06)
- https://milvus.io/blog/why-im-against-claude-codes-grep-only-retrieval-it-just-burns-too-many-tokens.md (2025-08-25, vendor)

### Sourcegraph
- https://sourcegraph.com/docs/api/mcp
- https://sourcegraph.com/docs/api/mcp/client-integrations
- https://sourcegraph.com/changelog/mcp-ga (2026-02-25)
- https://github.com/sourcegraph-community/sourcegraph-claudecode-plugin
- https://sourcegraph.com/resources/context-compare (vendor marketing)
- https://sourcegraph.com/pricing
- https://sourcegraph.com/docs/self-hosted/deploy
- https://sourcegraph.com/docs/code-navigation/precise-code-navigation
- https://sourcegraph.com/docs/deep-search
- https://sourcegraph.com/docs/admin/monorepo
- https://sourcegraph.com/case-studies
- https://sourcegraph.com/blog/the-future-of-scip (2026-03-25)
- https://sourcegraph.com/blog/why-sourcegraph-and-amp-are-becoming-independent-companies (2025-12-02)
- https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans (2025-06-25)

### Augment Code
- https://www.augmentcode.com/product/context-engine-mcp
- https://www.augmentcode.com/blog/context-engine-mcp-now-live (2026-02-06, updated 2026-06-18)
- https://docs.augmentcode.com/context-services/overview
- https://docs.augmentcode.com/context-services/mcp/overview.md
- https://docs.augmentcode.com/context-services/mcp/quickstart-claude-code.md
- https://docs.augmentcode.com/context-services/context-connectors/how-it-works.md
- https://www.augmentcode.com/security
- https://www.augmentcode.com/pricing
- https://siliconangle.com/2026/06/05/augment-code-launches-cosmos-bring-agentic-ai-software-development-teams/

### Greptile
- https://www.greptile.com/
- https://www.greptile.com/docs/introduction
- https://www.greptile.com/docs/mcp/overview
- https://www.greptile.com/docs/mcp-v2/setup
- https://www.greptile.com/docs/mcp-v2/tools
- https://www.greptile.com/docs/integrations/claude-code
- https://www.greptile.com/pricing
- https://www.greptile.com/docs/deployment-options
- https://www.greptile.com/docs/security/selfhost

### Qodo
- https://docs.qodo.ai/developer-tools/context-engine-mcp
- https://docs.qodo.ai/developer-tools/context-engine-mcp-tools
- https://www.qodo.ai/features/qodo-context-engine/
- https://www.qodo.ai/blog/introducing-qodo-aware-deep-codebase-intelligence-for-enterprise-development/ (2025-09-10)
- https://www.qodo.ai/pricing
- https://www.qodo.ai/

### Unblocked
- https://getunblocked.com/unblocked-mcp/
- https://getunblocked.com/ai-info/ (2026-09-17)
- https://getunblocked.com/pricing/
- https://getunblocked.com/security/

### Tabnine / Tricentis
- https://www.devopsdigest.com/tricentis-acquires-tabnine (2026-07-30)
- https://sapinsider.org/blogs/tricentis-tabnine-acquisition-agentic-sap-testing/
- https://docs.tabnine.com/
- https://docs.tabnine.com/main/administering-tabnine/managing-your-team/context-engine.md
- https://docs.tabnine.com/main/getting-started/tabnine-plugin-for-opencode/mcp-servers.md

### Moderne
- https://docs.moderne.io/user-documentation/agent-tools/mcp/overview/
- https://moderne.ai/blog/local-moderne-mcp-deterministic-tools-coding-agents (2026-04-17)
- https://moderne.ai/platform/lst
- https://www.moderne.ai/solutions/language-support
- https://www.moderne.ai/product/moddy
- https://moderne.ai/moderne-dx
- https://arxiv.org/abs/2510.04852

### Glean
- https://developers.glean.com/guides/mcp/claude-code
- https://developers.glean.com/guides/mcp.md
- https://developers.glean.com/guides/mcp/supported-hosts.md
- https://docs.glean.com/user-guide/mcp/engineering
- https://docs.glean.com/get-started/prepare/about-deployment
- https://www.glean.com/security

### Zilliz
- https://github.com/zilliztech/claude-context

### GitHub
- https://github.com/github/github-mcp-server
- https://docs.github.com/en/copilot/concepts/context/spaces
- https://docs.github.com/en/copilot/concepts/context/repository-indexing
- https://github.blog/changelog/2025-03-12-instant-semantic-code-search-indexing-now-generally-available-for-github-copilot/
- https://github.com/advanced-security/codeql-development-mcp-server

### Cursor
- https://cursor.com/docs/agent/tools/search
- https://cursor.com/changelog/2-1 (2025-11-21)
- https://cursor.com/blog/secure-codebase-indexing (2026-01-27)
- https://cursor.com/security (2026-08-25)
- https://news.ycombinator.com/item?id=49487825

### Windsurf / Devin Desktop
- https://cognition.com/blog/windsurf (2025-07-14)
- https://devin.ai/desktop
- https://docs.devin.ai/desktop/context-awareness/overview
- https://devin.ai/blog/remote-indexing-multirepo-announcement (2024-02-20)
- https://devin.ai/blog/windsurf-rebrand-announcement (2025-04-04)

### JetBrains
- https://www.jetbrains.com/help/ai-assistant/mcp.html

### CodeScene, Semgrep, vector databases
- https://github.com/codescene-oss/codescene-mcp-server
- https://www.codescene.com/pricing
- https://github.com/semgrep/mcp
- https://github.com/semgrep/mcp-marketplace
- https://weaviate.io/blog/coding-assistant-weaviate-mcp (2026-05-21)
- https://www.pinecone.io/blog/pinecone-plugin-for-claude-code/ (2026-02-11)

### Pivoted or acquired
- https://blar.io/
- https://nuanced.dev/
- https://github.com/nuanced-dev/nuanced
- https://archive.nuanced.dev/blog/nuanced-lsp-in-mcp
- https://swimm.io/
- https://docs.swimm.io/

### Third-party comparisons
- https://baeseokjae.github.io/posts/ai-coding-agent-code-index-mcp-comparison-2026/ (2026-07-06)
- https://corestory.ai/post/mcp-servers-codebase-context-ai-coding-agents (vendor-authored)
