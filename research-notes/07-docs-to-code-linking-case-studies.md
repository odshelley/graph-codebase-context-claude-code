# Linking Documentation Graphs to Code Graphs: Methods, Tools, and Enterprise Case Studies

Research date: 2026-10-01. Scenario: an enterprise team with an existing Neo4j documentation graph (design docs, runbooks, ADRs, wiki) deciding whether and how to link a code graph for Claude Code.

Legend: (F) = page fetched and read directly. (S) = search snippet only, contents not confirmed. UNVERIFIED = a claim I could not confirm against a primary source. All dates are publication dates from the fetched page unless stated.

---

## Part A. Linking documentation graphs to code graphs

### A1. Traceability link recovery research

This is the academic home of the docs-to-code linking problem. The Karlsruhe (KIT) ARDoCo group dominates the recent record.

**LiSSA: Toward Generic Traceability Link Recovery through RAG** (ICSE 2025, F). Fuchß, Hey, Keim, Liu, Ewald, Thirolf, Koziolek. A generic retrieval-then-LLM-classification pipeline for three artifact pairs: requirements-to-code, documentation-to-code, and architecture documentation to architecture models. Retrieval narrows the quadratic candidate space to a few plausible pairs; the LLM decides which are genuine links and explains why. The authors state RAG "can significantly outperform the state-of-the-art on the code-related tasks" but that "further research is required to improve the performance of RAG-based approaches to be applicable in practice." Replication package on GitHub. Relevance: this is the exact pattern for LLM-inferred doc-to-code edges in Neo4j, with the explanation stored as evidence on the edge.

**Who's Who? LLM-assisted Software Traceability with Architecture Entity Recognition** (arXiv 2511.02434, Nov 2025, revised Mar 2026, F). Same group. ExArch uses an LLM to extract component names from architecture documentation and source code and builds a simple architecture model (F1 0.86). ArTEMiS recognises architectural entities in documentation and matches them to model entities (F1 0.81), matching heuristic baselines. Combined, they beat ArDoCode, the best baseline that needs no hand-made architecture model. Relevance: architecture entity recognition is how a design-doc mention becomes a typed edge to a Component or Module node.

**The ARDoCo Tool Landscape** (ASE 2026 tools track, arXiv 2606.28064, F). REST API exposing four pipelines (SAD-to-SAM, SAM-to-Code, SAD-to-Code, SAD-to-SAM-to-Code), a browser TraceView, and a VS Code extension TraceViz that overlays trace links on documentation. Publicly deployed, CC BY 4.0. Relevance: a ready-made link-recovery service whose output can be loaded into Neo4j.

**NL-PL Traceability Link Recovery Needs More than Textual Similarity** (arXiv 2509.05585, Sep 2025, F). Zou, Wang, Liang, Bi, Jin. Twelve open-source projects. A Heterogeneous Graph Transformer with edge types and Gemini 2.5 Pro with structural prompt context both beat baselines; the LLM variant improves F1 by 8.84 percent over HGNNLink. Relevance: direct evidence that code-structure edges improve requirement-to-code linking, which argues for linking docs into the code graph rather than into a flat index.

**R2Code** (IEEE COMPSAC 2026, arXiv 2604.22432, F). Self-reflective LLM framework with a bidirectional alignment network, explanation-guided consistency checking for confidence calibration, and context-adaptive retrieval. Average F1 up 7.4 percent over baselines; token use down up to 41.7 percent; five datasets, two languages. Relevance: confidence calibration is what decides which inferred links are written to the graph versus queued for review.

**ReqToCode** (arXiv 2603.13999, Mar 2026, preprint, F). Preventive traceability: generated "Traceable" code elements per requirement, referenced from code and tests, validated at build time with a graduated deprecation-to-build-failure lifecycle. No empirical evaluation. Relevance: the deterministic alternative for new code, alongside recovery for legacy code.

**Issue-to-commit linking.** LinkAnchor (ACM FSE 2026, arXiv 2508.12232, F) is an LLM agent with lazy tool access over commits, issue discussions, and files. It notes that only about 42.2 percent of GitHub issues are linked to commits and that an issue is often resolved by a chain of commits rather than one. DeepLink (JSS 2019) and FRLink (IST 2016) are the classical learned baselines (S). Relevance: Jira-to-commit edges are cheap when issue keys appear in commit messages; this literature covers the rest.

**Unsupervised Construction of Knowledge Graphs From Text and Code** (KDD workshop 2019, arXiv 1908.09354, F). Cao and Fairbanks exploit naming conventions: function names reflect the concepts they implement, so word embeddings plus clustering align text concepts to code entities with no labels. Relevance: the cheapest lexical linking tier.

### A2. Code graphs built for LLM agents

These define the code-side schema vocabulary the documentation graph must join against.

| System | Node types | Edge types | Store and interface | Headline result |
|---|---|---|---|---|
| CodexGraph (NAACL 2025; NUS, Xi'an Jiaotong, Alibaba Group; F, affiliations read from PDF) | MODULE, CLASS, FUNCTION, METHOD, FIELD, GLOBAL_VARIABLE | CONTAINS, HAS_METHOD, HAS_FIELD, INHERITS, USES | Neo4j; primary agent writes NL query, translation agent emits Cypher | 22.96 percent SWE-bench Lite pass@1 with GPT-4o; 27.90 percent EM on CrossCodeEval Lite. Authors admit no call edges, Python only, high token use, indexing cost |
| CGM, Ant Group CodeFuse (NeurIPS 2025, arXiv 2505.16901, F) | REPO, PACKAGE, FILE, TEXTFILE, CLASS, FUNCTION, ATTRIBUTE | contains, calls, extends, imports, implements | Graph injected into LLM attention via adapter; R4 chain Rewriter, Retriever, Reranker, Reader; graph built offline by AST plus symbol resolution with CHA | 43.00 percent SWE-bench Lite, 50.40 percent Verified, first among open-weight at publication; 14.29 percent SWE-bench-java |
| LocAgent (arXiv 2503.09089, F) | directory, file, class, function | contains, imports, invokes, inherits | Lightweight graph plus agent tools | up to 92.7 percent file-level localisation; fine-tuned Qwen at about 86 percent lower cost |
| RANGER (arXiv 2509.25257, F) | hierarchy to variable level; nodes carry descriptions and embeddings | cross-file dependencies | Cypher for entity queries, MCTS for NL queries | beats embedding baselines on CodeSearchNet, RepoQA, RepoBench |
| Repository Intelligence Graph (arXiv 2601.10112, Jan 2026, F) | buildable components, aggregators, runners, tests, external packages, package managers | dependency and coverage edges from CMake File API and CTest | Fully deterministic; JSON view handed to agents | Claude Code, Cursor, Codex on eight repos: accuracy up 12.2 percent, completion time down 53.9 percent; multilingual repos up 17.7 percent |
| Codebase-Memory (arXiv 2603.27277, Mar 2026, F) | Tree-sitter symbols, 66 languages, call graph, community detection | calls, imports | MCP | tokens down about 10x, tool calls down 2.1x over 31 repos; answer quality 83 vs 92 percent for file exploration |
| CodeCompass (arXiv 2602.20048, Feb 2026, F) | dependency graph | | MCP | 99.4 percent completion on hidden-dependency tasks vs 76.2 baseline and 78.2 BM25; but 58 percent of trials with graph access made zero tool calls |

Two production-grade precedents exist for documentation nodes inside a code graph: CGM's TEXTFILE node and Kythe's doc node (Part B).

Also relevant: Microsoft CodePlan (arXiv 2309.12499, F) uses a dependency graph of containment, imports, inheritance, overrides, calls, object construction, and field use to plan multi-file edits; Knowledge Graph Based Repository-Level Code Generation (arXiv 2505.14394, F) and Bridging Code Property Graphs and LLMs (ICSE 2026 workshop, arXiv 2603.24837, F; the codebadger MCP server over Joern CPGs found a new libtiff overflow and patched CVE-2025-6021 first try).

### A3. Commercial unified docs-plus-code products

**Unblocked** (vendor site, F). Ingests GitHub, GitLab, Bitbucket, Azure DevOps; Slack, Teams, Zendesk; Jira, Linear, Asana; Confluence, Notion, Coda, Drive, SharePoint; PostHog; Datadog, Sentry into "a single living knowledge graph." Traverses Jira ticket to linked PR to Slack thread to code in one pass. Pipeline stages: unified retrieval, conflict resolution (recency, authority, expert graph), permission enforcement, personalised relevance, token optimisation, citations, freshness with main branch as truth. Exposed to Claude Code, Cursor, Copilot, Codex via one MCP server with two tools, context_research and context_get_urls. Vendor A/B claim: 48 percent fewer tokens, 60 percent lower cost, 75 percent fewer correction cycles. Customer quotes: Drata (1 to 2 hours per engineer per day), Fingerprint, Fetch (350 plus engineers). Vendor figures, unaudited.

**Glean** (docs and product pages, F). Enterprise Graph of content, people, and activity across 275 plus connectors, built per single-tenant customer by machine learning, with entity linking for projects, people, customers, products. GitHub connector indexes repos, source files, commits, READMEs, issues and comments, PRs, reviews, diffs, commit statuses; one access group per private repo mirroring GitHub permissions; per-user OAuth required. Code Search is an agent tool; supports Claude Code and any MCP host via the developer platform. The connector docs do not describe explicit code-to-Jira or code-to-Confluence typed edges; linking happens in the entity layer.

**Swimm** (F). Smart Tokens and Smart Paths couple doc text to identifiers and file paths and auto-sync through refactors. Now positioned as an "Agentic Development Context Layer" for Copilot, Cursor, Claude Code, and MCP servers, with COBOL, JCL, PL/I support and a 100M plus LOC claim; a top-5 US bank quote says its application map was "100 percent accurate" (vendor-sourced). Relevance: a mature model of deterministic doc-to-symbol anchors that survive renames.

**Mintlify** (F). Mintlify Index (Aug 6, 2026) is a docs retrieval layer for coding agents over 9,000 sites with context, search, and contents endpoints and an MCP server. Blinded comparison vs Context7 over 150 tasks on 50 products: preferred in 96, 48 percent faster, 10 percent higher factual accuracy, 17 percent fewer critical errors. Mintlify also reads imports, type definitions, and cross-file references when drafting docs. Relevance: a measured demonstration that curated doc retrieval improves agent plans.

**DeepWiki, Cognition** (F). Launched May 2025; over 50,000 public repos indexed; private repos via Devin. CodeWiki (ACL 2026, arXiv 2510.24428, F) scored 68.79 vs DeepWiki's 64.06 on CodeWikiBench using hierarchical decomposition and recursive agent delegation. deepwiki-open (F, MIT, about 18k stars) is the self-hostable clone with codemaps; OpenDeepWiki and RepoWiki are alternatives (S). Relevance: these generate docs from code rather than link existing docs. Generated pages can be loaded as Document nodes with deterministic DESCRIBES edges because the generator knows which symbols it summarised.

**Sourcegraph** (F). MCP server exposes semantic and keyword search, go-to-definition, find references, commit history, diffs, Deep Search, and "ownership" signals to Claude Code, Codex, Amp, and Cursor; Code Finder returns files, line ranges, and rationale in one call. Sourcegraph Own (Mar 2023 blog) models accountability vs expertise with explicit, inferred (commits, review activity, service catalogs), and imported ownership. Stripe's Minions use Sourcegraph search. Named customers: Reddit, MathWorks, Leidos, Indeed, Canva, 1Password, Dropbox, HubSpot, Stripe, Blackstone.

**Moderne** (F). Lossless Semantic Trees, fully type-attributed and pre-computed, serialised to feed Moddy (Mar 2025), a multi-repo agent that invokes OpenRewrite recipes; the post says "knowledge graphs built from LST data enhance accuracy for architectural and business-purpose searches." Claims billions of lines and thousands of repos; no customer metrics in the announcement. Harness (Apr 2026, F) proposes the same three-layer model: LST plus embeddings, code knowledge graph, MCP.

### A4. Catalog, operational, ownership, history, and decision sources

**Backstage** (F). Kinds: Component, API, System, Domain, Resource, Group, User, Location. Well-known relations are bidirectional pairs: ownedBy/ownerOf, providesApi/apiProvidedBy, consumesApi/apiConsumedBy, dependsOn/dependencyOf, partOf/hasPart, parentOf/childOf, memberOf/hasMember. TechDocs attaches Markdown docs living with the code to an entity through an annotation so documentation is discoverable from the entity page. Relevance: the best off-the-shelf bridge between Service nodes and both ownership and docs; load through the catalog API.

**OpenTelemetry servicegraph connector** (F). Derives service-to-service edges from paired client and server spans, producer and consumer spans, and database client spans; emits request, failure, and latency metrics per edge; creates virtual nodes for uninstrumented peers. Netflix Service Topology (InfoQ, Jun 2026, F) fuses eBPF flow logs, IPC metrics, and traces into a live graph over thousands of services with a gRPC multi-hop API. Relevance: runtime CALLS edges between Service nodes, which static analysis cannot see across repo boundaries.

**Ownership.** CODEOWNERS gives deterministic path-glob to Group edges. checkOwners (F) infers confidence-scored ownership (0 to 1) from recency with a 90-day half-life, frequency, blame coverage, and review activity, plus bus factor and expertise decay. Relevance: store an explicit OWNS edge and an inferred EXPERT_IN edge with a confidence property.

**Git history and co-change.** code-maat (F) computes logical coupling as shared revisions over average revisions, with a temporal window to merge same-day commits; it evolved into CodeScene. codebase-memory-mcp (F) materialises this as a FILE_CHANGES_WITH edge. Lore (arXiv 2603.15566, Mar 2026, F) proposes git trailers carrying constraints and rejected alternatives so decision context survives in commits. Relevance: co-change edges reveal hidden coupling that neither docs nor imports show.

**ADRs.** adr-graph (F) parses frontmatter edges (supersedes, superseded_by, related), wikilinks, and Markdown links into a typed graph, and matches file paths to governing ADRs via code_paths globs, returning matched, no_explicit_match, or no_code_paths_declared so the agent never assumes governance. repowise (F) mines decisions from eight sources (ADR files, CHANGELOGs, PR bodies, WHY and DECISION comments, significant commits, READMEs, LLM-mined rationale, agent transcripts), backs each with evidence spans, links them to governed files in the dependency graph, and flags staleness when a governed file's last commit is newer than the decision, with "no LLM in the governance path." Relevance: two concrete recipes for Decision nodes with GOVERNS edges and staleness.

### A5. Claude Code-native code-graph tools

All fetched. **Graphify** (123.1k stars at fetch, Apache-2.0 and MIT) ingests code via Tree-sitter across 37 plus languages plus Markdown, RST, HTML, SQL schemas, configs, PDFs; code edges are deterministic, doc concepts are LLM-extracted, every edge carries EXTRACTED or INFERRED, NOTE and WHY comments become nodes, and it exports to Neo4j, FalkorDB, GraphML, Obsidian. **GitNexus** (PolyForm Noncommercial) exposes 19 MCP tools including impact, trace, detect_changes, and raw cypher over Function, Class, File, Interface nodes with CALLS, IMPORTS, EXTENDS, IMPLEMENTS, MEMBER_OF and Leiden communities. **codebase-memory-mcp** (MIT, 162 languages) has the richest schema seen: Project, Package, Folder, File, Module, Class, Function, Method, Interface, Enum, Type, Route, Resource; edges CONTAINS_*, DEFINES, IMPORTS, CALLS, HTTP_CALLS, ASYNC_CALLS, IMPLEMENTS, HANDLES, CONFIGURES, TESTS, USES_TYPE, FILE_CHANGES_WITH, plus a manage_adr tool; claims the Linux kernel (28M LOC) indexed in three minutes and a 99.2 percent token reduction on five structural queries. **code-review-graph plus Graphify** with Claude Code hooks (DEV post, F) reported about 90k tokens saved (5.5x) across 37 graph calls on a 1,336-node monorepo, single-user measurement. **CodeGraph** (F) reports 47 percent fewer tokens, 58 percent fewer tool calls, 22 percent less wall-clock across seven repos. Neo4j ships a Claude Code plugin for its MCP server and Agent Skills for Cypher 25 (S). Relevance: Graphify's confidence tags and codebase-memory-mcp's edge vocabulary are directly reusable.

### A6. Evidence that docs-plus-code context helps agents

- RIG: three commercial agents including Claude Code, accuracy plus 12.2 percent, time minus 53.9 percent, from a deterministic build graph alone.
- Codebase-Memory and CodeCompass: large token and completion gains, with the caveat that agents often ignore graph tools unless prompted.
- Mintlify Index vs Context7: curated doc retrieval won two thirds of blinded plan comparisons.
- RelationalAI at QCon AI Boston (Sep 12, 2026, InfoQ, F): "context bundles" of specs, standards, and prior work kept accuracy equal while cutting tokens and iterations; the "code as truth" pattern pairs intent (specs) with reality (code) in one graph with provenance from idea to production.
- Uber on its Neo4j config graph (F): the knowledge graph "had the biggest impact on accuracy of LLM queries on the knowledge graph, way more than what we could do with prompt engineering."
- Uber uReview (F) names the gap explicitly: the reviewer "only has access to the code, and not to other artifacts like past PRs, feature flag configurations, database schemas, technical documentation."

---

## Part B. Enterprise case studies

**Google Kythe** (kythe.io schema, SWE book ch. 17, 2025 migrations paper; all F). Compiler-instrumented extraction per build rule, merged into one global graph optimised for go-to-definition, find-usages, and file decorations. Node kinds: anchor, file, function, record, variable, package, constant, macro, doc, interface, plus type nodes tapp, tvar, tbuiltin, talias. Edge kinds: ref, defines/binding, childof, documents, param, typed, extends, overrides, instantiates, specializes. Every node has a five-field VName (signature, corpus, root, path, language). Documentation is a first-class doc node linked by a documents edge; bracketed references in doc text resolve through param edges. The index costs about one full build, so it runs once a day. Code Search serves about 200 queries per second over about 1.5 TB with sub-50 ms median latency and over a million daily queries, indexes full Piper history at 2.5x cost, and treats a Code Search link as a file's canonical location. Stated lesson: developers trust the index only when it "indexes all code, gives all results, and gives the desired results first." The Jan 2025 migrations paper used Kythe breadth-first "references-to-references" to three levels to find edit sites: int32-to-int64 at 80 percent AI-generated code and about 50 percent time saved; JUnit3-to-4 across 5,359 files and 149,000 plus lines at about 87 percent; lesson "LLM plus AST, better together."

**Meta Glean** (Dec 2024 post, Hackage benchmark, F). Open source; per-language schemas; Angle declarative logic query language; RocksDB storage; incremental indexing by stacking immutable databases so only changed code is reprocessed. Simple queries return in about a millisecond. Used at Meta for navigation, search, documentation generation, review tooling, dead-code detection, build dependency analysis, test coverage, and RAG for AI coding assistants. Languages: C++, Python, Hack, JavaScript, Rust, Erlang, Thrift, Haskell, plus LSIF/SCIP. Simon Marlow's May 2025 Hackage benchmark: 2,922 packages; Glean 470 s and 0.8 GB vs hiedb 1,021 s and 5.2 GB; find-references 0.39 s vs 2.3 s; caveat that incremental updates and type indexing remain gaps. "Billions of facts" appears only on the project site (S, UNVERIFIED).

**Uber** (Genie, EAg-RAG, Neo4j story, uReview, GraphSummit video page: F; Port write-ups: F but secondary). Genie (Oct 2024) is vector RAG over Engwiki, internal Stack Overflow, and requirement docs: 154 Slack channels, 70,000 plus questions, 48.9 percent helpful, about 13,000 hours saved, no graph. EAg-RAG (May 2025) added query optimiser and source identifier agents, BM25 plus vectors, and per-chunk metadata: 27 percent more acceptable answers, 60 percent fewer wrong ones. Config Knowledge Graph (Neo4j AuraDB, Nov 2025) models onboarding configs across 7 domains with REQUIRES, EVALUATES, CAN_LAUNCH_IN edges, 27 cross-domain safeguards, exposed via MCP, built by two engineers in under twelve weeks. uReview (Aug 2025) reviews over 90 percent of about 65,000 weekly diffs across six monorepos with 75 percent useful comments and 65 percent addressed. Port reports an "AI context graph" over services, ownership, design docs, Jira, incidents, with 1,000 plus MCP servers behind one gateway and over 70 percent of PRs agent-produced, but its two write-ups give conflicting figures (40M entries and 150 kinds vs 24M nodes and 80M edges across 30 plus systems), and the cited Uber post of Aug 27, 2026 returned 404 on the URLs I tried: UNVERIFIED. The 21,000 developer hours claim for Validator and AutoCover comes from secondary coverage of a LangChain talk: UNVERIFIED. Michelangelo is Uber's ML platform, not code tooling.

**Ant Group CGM** (F). See A2. Trained on 500k Python and 360k Java subgraphs from 20k repos, 200k issue-patch pairs, on 64 A100s with QLoRA; graph construction offline; CodeFuse reports 1.5M model downloads; 44.00 percent SWE-bench Lite for CGM-72B V1.2. Lesson: graph structure injected at the model level beat agentic Cypher querying on SWE-bench for open-weight models.

**Alibaba CodexGraph** (F). See A2. Demo lives in ModelScope-Agent (apps/codexgraph_agent). Author-stated lessons: missing call edges hurt, token cost is high, and the translation-agent handoff is a bottleneck.

**Microsoft** (F). CodePlan (2023) passes 5 of 6 repos on C# package migration and Python temporal edits where baselines pass none. Code Researcher (arXiv 2506.11060, Microsoft Research, May 2025 revised May 2026) uses four tools: search_definition via ctags, search_code via git grep, search_commits via git log, and close; three phases Analysis, Synthesis, Validation; 48 percent crash resolution on 200 Linux kernel crashes vs 31.5 SWE-agent and 31 Agentless, 54 percent at ten trajectories, 7 of 10 on FFmpeg; it probes about 10 files per trajectory vs 1.33 for typical agents. Commit history was decisive. I found no Microsoft-authored "GraphRAG for code" artifact; Microsoft GraphRAG targets text corpora.

**Stripe Minions** (stripe.dev, Feb 9, 2026, F). Over 1,000 agent PRs merged weekly; hundreds of millions of lines, mostly Ruby with Sorbet; 3 million plus tests; 10-second devboxes. Context comes from Toolshed, a central MCP server with over 400 tools covering docs, tickets, build status, and Sourcegraph code search. Links in the task are deterministically pre-hydrated before the agent loop. Lessons: reuse human dev infrastructure, cap CI retries at two, scope rules by directory, shift lint left.

**Shopify River** (May 28, 2026, F). One in eight merged PRs co-authored; 3,536 PRs in 30 days over 59,918 sessions; median session 19 minutes and 50 tool calls. Context lives in the World monorepo (consolidated spring 2024, Nix-built) as skills, intent docs, runbooks, AGENTS.md files, and zone knowledge, not a graph. Lessons: public multiplayer channels, decouple reasoning from sandbox execution, new agents as configuration profiles.

**LinkedIn** (QCon AI Boston, Sep 19, 2026, InfoQ, F). Organisational context layer over MCP: 600 plus playbooks as procedural memory, code search over 1,000 plus repos, docs, feature flags, task management; local MCP server on every laptop updated hourly; 8,000 daily users; 20 percent productivity gain with no reliability loss. Hierarchical playbooks that reference smaller ones solve the over-30-tools degradation problem. No code graph as such.

**Netflix** (F). E2EGraph (QCon London, Mar 17, 2026) models users, clients, services, and infrastructure with request edges enriched by latency, errors, versions, and geography, and feeds AutoSRE, a mixture-of-experts root-cause agent. Service Topology (Jun 2026) fuses eBPF, IPC metrics, and traces for thousands of services on a graph layer over Netflix's KV store. Neither is a code graph, but both are the runtime half of a unified schema.

**Sourcegraph customers** (F). MathWorks unified Perforce, GitHub, and GitLab search for 1,000 plus engineers by 2024 and all developers by Mar 2025 on a 40-year-old C, C++, Java, JavaScript codebase; a memory-leak diagnosis fell from weeks to under one week. Stated lesson: agentic tools need deep codebase context.

**Smaller enterprise talks** (abstracts F, videos not watched). Lockheed Martin at NODES 2025 (Thomas Orth, Nov 6, 2025) automated software architecture documentation from a code knowledge graph plus LLMs for codebases of thousands to millions of lines; technical detail is only in the video. Version 1's Decipher (NODES 2025) maps application code into Neo4j to surface business rules. Marsh McLennan's lead staff engineer (NODES 2025) showed a Neo4j code graph with File and Symbol nodes and DEFINED_IN, IMPORTS, CALLS, CONTAINS edges for Python, JavaScript, TypeScript, queried by agents for dead code, circular dependencies, and shotgun surgery. Morgan Stanley (eFinancialCareers citing WSJ, Jun 2025, F) trained models on its full codebase to reverse-engineer COBOL business logic into specs; a search snippet misattributed this to Goldman. Goldman's Devin rollout appeared only in a secondary newsletter: UNVERIFIED. Amazon: nothing public on code graphs found; Kiro is spec-driven. Airbnb's knowledge graph work is about trip data, not code.

---

## (a) Proposed unified schema sketch

Determinism labels: D = deterministic from a parser or system of record; L = LLM or ML inferred, must carry confidence, evidence, and review properties.

**Node labels**

- Code (D): Repo, Package, File, Module, Class, Interface, Function, Method, Field, Route, Test, ExternalPackage. Sources: Kythe VName, CodexGraph, CGM, codebase-memory-mcp. Properties: path, language, signature, line range, stable id built like a VName (corpus, root, path, signature, language).
- Architecture and operations (D): Service, API, System, Domain, Resource, Deployment. Sources: Backstage kinds, OTel, Netflix.
- People (D): User, Group. Sources: Backstage, CODEOWNERS, identity provider.
- History (D): Commit, PullRequest, Issue, Incident. Sources: git, Jira, LinkAnchor, Unblocked.
- Knowledge (existing graph): Document, Section, Decision (ADR), Runbook, Requirement, Concept. Sources: your graph, adr-graph, repowise, ArTEMiS.

**Relationship types**

- D code structure: CONTAINS, DEFINES, IMPORTS, CALLS, EXTENDS, IMPLEMENTS, USES_TYPE, TESTS, HTTP_CALLS. From Tree-sitter plus LSP or SCIP, Kythe-style.
- D history: AUTHORED, MODIFIES (Commit to File), FIXES (Commit to Issue when the key is present), MERGED_AS (PR to Commit), CHANGES_WITH (File to File, coupling strength and support count from code-maat).
- D ownership and catalog: OWNED_BY (path glob from CODEOWNERS or Backstage), PROVIDES_API, CONSUMES_API, DEPENDS_ON, PART_OF, MEMBER_OF, IMPLEMENTED_IN (Service to Repo or Package from the catalog annotation).
- D runtime: CALLS_AT_RUNTIME (Service to Service; request, error, and latency properties from the OTel servicegraph connector; virtual flag for uninstrumented peers).
- D docs-to-code anchors: REFERENCES_PATH, REFERENCES_SYMBOL (Swimm-style token anchors), LINKS_TO (URL to File, PR, or Issue), DOCUMENTED_BY (Backstage TechDocs annotation), GOVERNS (ADR code_paths glob with match status matched, no_explicit_match, no_code_paths_declared).
- L docs-to-code inferred: DESCRIBES (Document or Section to Module, Class, or Service; ArTEMiS-style entity recognition), TRACES_TO (Requirement to Function or Class; LiSSA-style retrieval then classification), RESOLVES (Issue to Commit chain without a key; LinkAnchor-style), ABOUT (Document to Concept), SUMMARISED_AS (community or module to generated description; RANGER and CGM style), EXPERT_IN (User to Package; checkOwners-style with decay).
- Properties on every L edge: confidence, method, evidence_span, model, created_at, reviewed (boolean), stale (set when the governed file's last commit is newer than the doc, per repowise).
- Provenance on every edge: source in {ast, lsp, git, codeowners, backstage, otel, jira, llm}, extracted_at, following Graphify's EXTRACTED vs INFERRED split.

**Linking recipe in tiers**: exact path, URL, symbol, and issue-key matches first; naming-convention and identifier-token matches second; retrieval plus LLM classification with calibrated confidence third; co-change and ownership as corroborating evidence rather than primary links. Expose the graph to Claude Code via the Neo4j MCP server plus a small set of task-shaped tools (impact, trace, who_owns, what_governs, docs_for), because CodeCompass shows agents under-use raw graph tools without explicit guidance, and pre-hydrate context from links in the task before the agent loop, as Stripe does.

---

## (b) Lessons learned from the case studies

1. Deterministic structure first, LLM inference second. Google's "LLM plus AST, better together," Graphify's confidence tags, repowise's no-LLM scoring path, and RIG's fully deterministic graph all point the same way.
2. Completeness builds trust. Google found developers abandon a code index that is not complete; partial docs-to-code coverage must be visible as such, which adr-graph's three-way match status models well.
3. Keep indexing incremental and cheap. Kythe is daily because it costs a build; Glean stacks immutable databases; code-review-graph updates in under half a second on commit. Plan refresh cadence per layer.
4. Commit history is a first-class signal. Code Researcher's gains came from search_commits; LinkAnchor shows issues map to commit chains, not single commits.
5. Docs linkage is a known gap even at Uber. uReview names the missing artifacts explicitly; Genie never linked docs to code and plateaued below 50 percent helpfulness until agents and metadata were added.
6. Agents need task-shaped tools and prompting, not just Cypher access. CodeCompass's 58 percent zero-tool-call trials and CodexGraph's translation-agent overhead both show this; LinkedIn's hierarchical playbooks and Stripe's pre-hydration are the production answers.
7. Bundle context rather than flood it. RelationalAI's context bundles and Unblocked's token optimisation stage both report equal accuracy at lower token cost.
8. Permissions must mirror the source systems. Glean builds one access group per private repo; Uber's Genie excluded sources it could not expose in Slack.
9. Store evidence spans and staleness on inferred links so humans can audit and the graph can age (repowise, LiSSA explanations, R2Code calibration).
10. Expect token and time savings from a graph layer, with a modest accuracy trade-off on some tasks (Codebase-Memory's 83 vs 92 percent) and larger gains on multilingual and hidden-dependency tasks (RIG, CodeCompass).
11. Runtime topology and ownership are the parts a static code graph cannot supply, and both have mature deterministic sources (OTel servicegraph, Backstage, CODEOWNERS).
12. Measure beyond SWE-bench. The 2026 research sweep by Anthony West and the MathWorks story both stress blast radius, affected tests, and reviewer burden as the enterprise metrics that matter.

---

## (c) All URLs used

Part A research:
- https://fuchss.org/conferences/icse25/ (F)
- https://github.com/ArDoCo/Replication-Package-ICSE25_LiSSA-Toward-Generic-Traceability-Link-Recovery-through-RAG (S)
- https://2025.refsq.org/details/refsq-2025-research-papers/8/Requirements-Traceability-Link-Recovery-via-Retrieval-Augmented-Generation (S)
- https://arxiv.org/abs/2511.02434 (F)
- https://arxiv.org/abs/2606.28064 (F)
- https://arxiv.org/abs/2509.05585 (F)
- https://arxiv.org/abs/2604.22432 (F)
- https://arxiv.org/abs/2603.13999 (F)
- https://arxiv.org/abs/2508.12232 (F)
- https://dl.acm.org/doi/10.1016/j.jss.2019.110406 (S)
- https://www.sciencedirect.com/science/article/abs/pii/S0950584916303792 (S)
- https://arxiv.org/abs/1908.09354 (F)
- https://arxiv.org/abs/2408.03910 (F)
- https://arxiv.org/html/2408.03910v3 (F)
- https://arxiv.org/pdf/2408.03910 (F)
- https://aclanthology.org/2025.naacl-long.7/ (F)
- https://github.com/modelscope/modelscope-agent (F)
- https://arxiv.org/abs/2505.16901 (F)
- https://arxiv.org/html/2505.16901 (F)
- https://github.com/codefuse-ai/CodeFuse-CGM (F)
- https://arxiv.org/abs/2503.09089 (F)
- https://arxiv.org/abs/2509.25257 (F)
- https://arxiv.org/abs/2601.10112 (F)
- https://arxiv.org/abs/2603.27277 (F)
- https://arxiv.org/abs/2602.20048 (F)
- https://arxiv.org/abs/2505.14394 (F)
- https://arxiv.org/abs/2603.24837 (F)
- https://arxiv.org/abs/2603.15566 (F)
- https://arxiv.org/abs/2510.24428 (F)
- https://arxiv.org/abs/2002.09440 (S)

Part A products and tools:
- https://getunblocked.com/ai-info/ (F)
- https://docs.glean.com/connectors/native/github/about (F)
- https://www.glean.com/product/knowledge-graph (F)
- https://www.glean.com/mcp/github (F)
- https://swimm.io/ (F)
- https://www.mintlify.com/blog/mintlify-index (F)
- https://cognition.com/blog/deepwiki (F)
- https://github.com/AsyncFuncAI/deepwiki-open (F)
- https://sourcegraph.com/mcp (F)
- https://sourcegraph.com/blog/our-vision-for-code-ownership (F)
- https://sourcegraph.com/blog/introducing-deep-search (F)
- https://moderne.ai/blog/introducing-moderne-multi-repo-ai-agent-for-transforming-code-at-scale (F)
- https://www.harness.io/blog/your-repo-is-a-knowledge-graph-you-just-dont-query-it-yet (F)
- https://backstage.io/docs/features/software-catalog/well-known-relations (F)
- https://backstage.io/docs/features/techdocs/ (F)
- https://github.com/backstage/backstage/blob/master/plugins/catalog-graph/README.md (F)
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/connector/servicegraphconnector/README.md (F)
- https://dev.to/smusali/meet-checkowners-confidence-scored-codeowners-from-your-git-history-4280 (F)
- https://github.com/adamtornhill/code-maat (F)
- https://github.com/democratize-technology/adr-graph (F)
- https://repowise.dev/guides/decisions-adr (F)
- https://github.com/Graphify-Labs/graphify (F)
- https://github.com/abhigyanpatwari/GitNexus (F)
- https://github.com/DeusData/codebase-memory-mcp (F)
- https://dev.to/mir_mursalin_ankur/graphify-code-review-graph-build-a-self-updating-knowledge-graph-for-claude-code-and-other-ai-j1m (F)
- https://toknow.ai/posts/codegraph-knowledge-graph-ai-coding-agents-fewer-tokens/ (F)
- https://memgraph.com/blog/graphrag-for-devs-coding-assistant (F)
- https://neo4j.com/blog/developer/codebase-knowledge-graph/ (F)
- https://github.com/cxt9/neo4j-mcp (S)
- https://neo4j.com/labs/genai-ecosystem/agent-skills/ (S)
- https://anthonywest.co.uk/research/code-intelligence-indexing-2026-openai (F)

Part B:
- https://kythe.io/docs/schema/ (F)
- https://abseil.io/resources/swe-book/html/ch17.html (F)
- https://arxiv.org/abs/2501.06972 (F)
- https://arxiv.org/html/2501.06972 (F)
- https://engineering.fb.com/2024/12/19/developer-tools/glean-open-source-code-indexing/ (F)
- https://simonmar.github.io/posts/2025-05-22-Glean-Haskell.html (F)
- https://glean.software/ (S)
- https://www.uber.com/blog/genie-ubers-gen-ai-on-call-copilot/ (F)
- https://www.uber.com/us/en/blog/enhanced-agentic-rag/ (F)
- https://neo4j.com/customer-stories/uber/ (F)
- https://neo4j.com/videos/graphsummit-nyc-25-ubers-config-knowledge-graph-genai-with-graphrag-and-mcp/ (F)
- https://www.uber.com/blog/ureview/ (F)
- https://newsletter.port.io/p/how-uber-built-a-software-factory (F, secondary)
- https://software-factories.port.io/uber (F, secondary)
- https://blog.tmcnet.com/blog/rich-tehrani/ai/how-uber-built-ai-agents-that-saved-21000-developer-hours.html (S, secondary)
- https://arxiv.org/abs/2309.12499 (F)
- https://arxiv.org/abs/2506.11060 (F)
- https://arxiv.org/html/2506.11060 (F)
- https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents (F)
- https://www.engineering.fyi/article/minions-stripe-s-one-shot-end-to-end-coding-agents (F)
- https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs (F)
- https://shopify.engineering/under-the-river (F)
- https://www.infoq.com/presentations/linkedin-context-engineering/ (F)
- https://qconlondon.com/presentation/mar2026/ontology-driven-observability-building-e2e-knowledge-graph-netflix-scale (F)
- https://www.infoq.com/news/2026/06/netflix-microservices-realtime/ (F)
- https://sourcegraph.com/case-studies/mathworks-accelerates-code-exploration-with-deep-search (F)
- https://neo4j.com/nodes-2025/agenda/leveraging-knowledge-graphs-and-llms-to-document-large-scale-codebases/ (F)
- https://neo4j.com/videos/leveraging-knowledge-graphs-and-llms-to-document-large-scale-codebases/ (F)
- https://neo4j.com/nodes-2025/agenda/beyond-vibe-coding-ai-agents-for-technical-debt-detection-with-neo4j/ (F)
- https://neo4j.com/nodes-2025/agenda/from-spaghetti-to-structure-decoding-complex-codebases-with-graphs-and-genai/ (F)
- https://www.infoq.com/presentations/knowledge-graphs-agentic-systems-patterns/ (F)
- https://www.efinancialcareers.com/news/morgan-stanley-llm-cobol (F)
- https://lucidate.substack.com/p/goldman-sachs-scales-ai-coding-to (S, UNVERIFIED)
- https://medium.com/airbnb-engineering/scaling-knowledge-access-and-retrieval-at-airbnb-665b6ba21e95 (S, not code-related)

## Gaps

The web search budget for this session was exhausted near the end, so I could not run a final search for software-architecture-recovery papers or locate Uber's primary August 2026 software-factory post (two URL guesses returned 404). Lockheed Martin's technical details exist only in an unwatched video. Uber context-graph figures and the Goldman Devin rollout remain UNVERIFIED.
