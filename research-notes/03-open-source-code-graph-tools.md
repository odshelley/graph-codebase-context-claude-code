# Open-source code-graph tools and MCP servers for Claude Code

Survey date: 2026-10-01. Audience: an enterprise team that already runs Neo4j.

Every URL below was fetched live on 2026-10-01. Star counts, fork counts, push dates and licence identifiers come from the GitHub REST API on that date, not from READMEs or third-party listings. Release tags come from the `releases/latest` endpoint. Anything I could not confirm from a fetched page is marked UNVERIFIED.

## A caution on star counts

Five of the most-starred repositories in this space were created in 2026 and already show between 30k and 123k stars: Graphify (123k, created April 2026), colbymchenry/codegraph (73k, January 2026), GitNexus (48k, August 2025), codebase-memory-mcp (46k, February 2026) and code-review-graph (32k, February 2026). Their Hacker News submissions peak at 12 points. I could not verify star provenance. Treat stars in that cohort as a weak signal and weigh commit cadence, licence, architecture and published evidence instead.

## Tool-by-tool findings

### code-graph-rag (vitali87)

1. URL: https://github.com/vitali87/code-graph-rag (live). Commercial site: https://code-graph-rag.com/
2. Description: Parses a multi-language codebase with tree-sitter and sharpens the result with compiler-grade frontends where available: libclang for C/C++, go/types for Go, Roslyn for C#, javac for Java, Jedi for Python. Builds a knowledge graph of modules, classes, functions and call edges, with Qdrant for vector search. Offers natural-language to Cypher querying, AST-based surgical patching with diff preview, dead-code detection by call-graph traversal, eBPF runtime-trace ingestion, ast-grep structural rewrites, real-time incremental indexing and graph export.
3. Backend: Memgraph (hard dependency) plus Qdrant.
4. Languages: Python, TypeScript, TSX, JavaScript, Rust, Go, Java, C, C++, C#, PHP, Lua, Dart fully supported. Scala in development. Ruby, Kotlin, Swift, Elixir, Haskell, Solidity, Bash, Nix structurally supported.
5. Claude Code integration: runs as an MCP server; the README names Claude Code as a client. Exposes query and edit capabilities over the graph (natural-language Cypher, patching, dead-code, tracing).
6. Licence: MIT.
7. Maturity: 5,191 stars, 692 forks, created 2025-06-16, pushed 2026-10-01, latest release v0.1.38 on 2026-09-29, 392 open issues. Individual maintainer with a commercial arm.
8. Enterprise readiness: the commercial site advertises managed cloud, on-premise and air-gapped deployment, custom parsers, SLAs and integration consulting. No telemetry statement found on the README or site. Incremental indexing is a headline feature. Multi-repo: described as a monorepo tool; cross-repo linking UNVERIFIED.
9. Evidence: Memgraph's blog post (2025-08-28) shows a live demo computing a maximum call chain of 11 on the tool's own repo. No benchmark. Marketing only.
10. Caveats: Memgraph is required; 392 open issues indicate fast churn. Whether the Bolt driver can be pointed at Neo4j is not claimed anywhere I fetched.

### GitNexus (abhigyanpatwari)

1. URL: https://github.com/abhigyanpatwari/GitNexus (live).
2. Description: Tree-sitter indexing into a knowledge graph of files, symbols, calls, imports, inheritance, interface implementations, clusters and execution flows. CLI uses native bindings; the web UI runs the same pipeline in WASM in-browser.
3. Backend: LadybugDB, native and persistent for the CLI, WASM in-memory per session for the web UI.
4. Languages: TypeScript, JavaScript, Python, Java, Kotlin, C#, Go, Rust, PHP, Ruby, Swift, C, C++, Objective-C, Dart, Zig.
5. Claude Code integration: `claude mcp add gitnexus -- npx -y gitnexus@latest mcp`. Setup also installs skills into `.claude/skills/` and writes CLAUDE.md context. Nineteen MCP tools: list_repos, query, context, impact, trace, detect_changes, check, rename, cypher, route_map, tool_map, shape_check, api_impact, explain, pdg_query, read_file, grep, group_list, group_sync. `GITNEXUS_MCP_READ_ONLY=1` restricts to read-only tools.
6. Licence: PolyForm Noncommercial 1.0.0 per the LICENSE file. The API reports NOASSERTION. The licence text contains no separate commercial-licence clause.
7. Maturity: 47,680 stars, 5,179 forks, created 2025-08-02, pushed 2026-10-01. Individual maintainer.
8. Enterprise readiness: `gitnexus analyze --watch` performs debounced incremental refresh. `serve` has no authentication by default. Web UI is capped near 5k files by browser memory. Embeddings opt-in.
9. Evidence: README argument only. HN posts at 1 to 3 points. MarkTechPost coverage (2026-04-24) is promotional.
10. Caveats: the noncommercial licence blocks enterprise use unless the maintainer offers terms. npm 11 install crashes are documented.

### code-review-graph (tirth8205)

1. URL: https://github.com/tirth8205/code-review-graph (live).
2. Description: Tree-sitter builds a structural map of functions, classes, imports and call/inheritance edges with targeted fallbacks for unsupported languages, kept updated by content-hash incremental re-parse, and served over MCP so the agent reads only what a change touches. FTS5 keyword search plus optional vector search.
3. Backend: one SQLite file under `.code-review-graph/`. No external database.
4. Languages: Python, JS/TS/TSX, Go, Rust, Java, C/C++, C#, VB.NET, Ruby, Kotlin, Swift, PHP, Scala, Solidity, Dart, R, Perl, Lua, Objective-C, shell, Elixir, Zig, PowerShell, Julia, ReScript, GDScript, Nix, Verilog, SQL, Terraform, Ansible YAML, Spring Boot config, Vue, Svelte, Astro, notebooks.
5. Claude Code integration: `code-review-graph install` auto-detects Claude Code and other clients, writes CLAUDE.md, and installs skills explore-codebase, review-changes, debug-issue and refactor-safely. Thirty MCP tools including build_or_update_graph_tool, get_minimal_context_tool, get_impact_radius_tool, get_review_context_tool, query_graph_tool, traverse_graph_tool, semantic_search_nodes_tool, detect_changes_tool, get_architecture_overview_tool, dead-code, flow and community-detection tools.
6. Licence: MIT.
7. Maturity: 31,891 stars, 2,916 forks, created 2026-02-26, pushed 2026-09-18. Individual maintainer.
8. Enterprise readiness: local-first, no telemetry, cloud embeddings opt-in. Incremental: a two-file edit on a 3,000-file repo takes about 2.5 s. Single-repo.
9. Evidence: self-published benchmark over six repositories and 13 commits with pinned SHAs, fixed Leiden seed and CPU-deterministic embeddings, reporting a median 63x per-question token reduction (range 35x to 358x) and 0.69 average impact F1. The README states the impact "recall 1.0" is circular because ground truth is derived from the same graph edges. Saurabh Sharma's comparison (2026-07-02) repeats these numbers without re-measuring. HN: 12 points.
10. Caveats: small single-file changes may cost more than a plain read; search ranking and flow detection are self-described as needing work.

### codebase-memory-mcp (DeusData)

1. URL: https://github.com/DeusData/codebase-memory-mcp (live). Paper: https://arxiv.org/abs/2603.27277
2. Description: Single static binary. Tree-sitter with 162 vendored grammars, plus hybrid LSP semantic type resolution for Python, TypeScript/JavaScript/JSX/TSX, PHP, C#, Go, C, C++, Java, Kotlin, Rust and Perl. Indexes functions, classes, call chains, HTTP routes and cross-service links; background watcher re-indexes on change.
3. Backend: SQLite in WAL mode under `~/.cache/codebase-memory-mcp/`.
4. Languages: 162 claimed.
5. Claude Code integration: MCP server installed via an install script. Seventeen tools: index_repository, list_projects, delete_project, index_status, check_index_coverage, search_graph, trace_path, detect_changes, query_graph, get_graph_schema, compare_graphs, get_code_snippet, get_file_outline, get_architecture, search_code, manage_adr, ingest_traces.
6. Licence: MIT.
7. Maturity: 45,623 stars, 3,742 forks, created 2026-02-24, pushed 2026-10-01, 615 open issues. Organisation account.
8. Enterprise readiness: states it runs 100% locally and collects no telemetry. CROSS_* edges link nodes across multiple repos indexed into the same store. Scale claim: Linux kernel, 28M LOC, indexed in 3 minutes. No auth layer (local binary).
9. Evidence: the arXiv preprint (Vogel, Meyer-Eschenbach, Kohler, Grünewald, Balzer, submitted 2026-03-28) evaluates 31 repositories and reports 83% answer quality against 92% for a file-exploration agent, with ten times fewer tokens and 2.1 times fewer tool calls. Russ McKendrick's field report (2026-05-10) confirms token savings on his own projects and notes the graph does not index text content, so grep is still needed. The paper is the strongest published number in this survey and shows a quality trade-off, not a free win. Whether the authors are the maintainers is not stated.
10. Caveats: large indexes may time out in MCP clients; unsigned Windows binary.

### codegraph (colbymchenry)

1. URL: https://github.com/colbymchenry/codegraph (live).
2. Description: Native Rust kernel with tree-sitter grammars compiled in. Pre-indexes the repo into a symbol graph and auto-syncs via OS file events (FSEvents, inotify, ReadDirectoryChangesW) with debouncing.
3. Backend: SQLite with FTS5 at `.codegraph/codegraph.db`.
4. Languages: 20 primary (TypeScript, JavaScript, Python, Go, Rust, Java, C#, PHP, Ruby, C, C++, Swift, Kotlin, Scala, Dart, Lua and others) plus extended Svelte, Vue, Astro, Solidity, Terraform, Nix, COBOL.
5. Claude Code integration: MCP server. One listed tool, codegraph_explore; codegraph_node, codegraph_search, codegraph_callers, codegraph_callees, codegraph_impact, codegraph_files and codegraph_status are unlisted by default. Auto-allows permissions when Claude Code is a target.
6. Licence: MIT.
7. Maturity: 72,767 stars, 4,673 forks, created 2026-01-18, pushed 2026-10-01, 526 open issues. Individual maintainer.
8. Enterprise readiness: fully local. No telemetry statement found. Single-repo. Files over 1 MB skipped.
9. Evidence: self-run benchmark on seven codebases, median of four runs per arm on Claude Opus 4.8 with the CLI blocked in both arms, claiming 88% fewer tool calls, 53% faster, 62% fewer tokens. The baeseokjae comparison (2026-07-06) measured an 8-second index on a 50k-line TypeScript project but otherwise repeats vendor claims. HN: 1 point.
10. Caveats: WSL2 and Windows shared checkouts need separate indices.

### CodeGraphContext

1. URL: https://github.com/CodeGraphContext/CodeGraphContext (live). Docs: https://codegraphcontext.github.io/
2. Description: Tree-sitter plus SCIP indexers (scip-clang for C/C++ with compile_commands.json, scip-dotnet via Roslyn for C#) index local code into a graph of symbols, calls, imports, inheritance and complexity.
3. Backend: pluggable. FalkorDB Lite default on Unix with Python 3.12+, KuzuDB cross-platform fallback, LadybugDB embedded, external Neo4j, and a "Nornic DB" enterprise option.
4. Languages: 23 including Python, JavaScript, TypeScript, Java, C, C++, C#, Go, Rust, Ruby, PHP, Swift, Kotlin, Dart, Perl, Lua, Scala, Haskell, Elixir, HTML, CSS, TSX, Solidity.
5. Claude Code integration: `codegraphcontext mcp setup` configures Claude among other clients. Twenty-one MCP tools including analyze_code_relationships (query types find_callers, find_callees, find_all_callers, find_importers, class_hierarchy, overrides, dead_code, call_chain, module_deps, variable_scope, find_complexity) and execute_cypher_query, which runs read-only Cypher against whichever backend is configured. `cgc watch` updates the graph on change.
6. Licence: MIT.
7. Maturity: 4,243 stars, 832 forks, created 2025-08-16, pushed 2026-09-29, latest release v0.5.7 on 2026-08-08. Single named maintainer.
8. Enterprise readiness: self-hosted against your own Neo4j; auth is Neo4j's. Incremental via watch. Multi-repo UNVERIFIED. No telemetry statement found.
9. Evidence: none published. Regolo and other blog coverage is promotional.
10. Caveats: SCIP for C/C++ needs a compilation database and falls back to tree-sitter without it. The v0.5.7 notes mention fixing "silent data loss" in Ruby, C/C++, C# and JavaScript parsers, so earlier indexes may be incomplete.

### Graphify (safishamsi / Graphify Labs)

1. URL: https://github.com/safishamsi/graphify (live).
2. Description: a `/graphify` skill first, server second. Code is parsed locally with tree-sitter (37 grammars), deterministically and with no LLM call. Docs, PDFs, SQL schemas, Terraform and media route through the IDE session's model for concept extraction. Produces a report of "god nodes" and surprising cross-module links.
3. Backend: `graph.json` plus `graph.html`. Exports to GraphML, Neo4j Cypher, FalkorDB and SVG. Graph capped at 512 MiB by default.
4. Languages: 37 tree-sitter grammars.
5. Claude Code integration: `graphify install` registers the skill, installs PreToolUse hooks and optional CLAUDE.md. Also runs as an MCP server via `python -m graphify.serve graphify-out/graph.json`. `graphify update .` is incremental; post-commit hooks rebuild on commit and checkout.
6. Licence: Apache-2.0 per the API; the README also mentions MIT.
7. Maturity: 123,061 stars, 11,862 forks, created 2026-04-03, pushed 2026-09-30, 1,516 open issues. Graphify Labs, a YC S26 company with an enterprise offering.
8. Enterprise readiness: states no telemetry, no usage tracking; query logging is opt-in. Code never leaves the machine. Neo4j is a batch export, not the live store.
9. Evidence: internal LOCOMO (recall@10 0.497, QA 45.3%) and LongMemEval-S (76%) numbers for memory tasks, not code tasks. The "71.5x fewer tokens" figure in blogs is third-party anecdote.
10. Caveats: anomalous star growth; extras required for PDF, office, video and Neo4j.

### CodeGraph (codegraph-ai)

1. URL: https://github.com/codegraph-ai/CodeGraph (live).
2. Description: tree-sitter semantic graph of functions, classes, imports and call chains with a persistent memory layer (BM25 plus semantic search over stored memories).
3. Backend: RocksDB at `~/.codegraph/graph.db`.
4. Languages: 38; COBOL, Fortran, Perl, Dart, Zig and R need an extra-languages build.
5. Claude Code integration: MCP server configured in `~/.claude.json`. 42 community tools including get_ai_context, get_callers, get_callees, analyze_impact, pr_context, memory_store, memory_search. 27 more tools in a Pro tier.
6. Licence: Apache-2.0, open-core.
7. Maturity: 104 stars, created 2026-04-04, pushed 2026-10-01. Solo developer.
8. Enterprise readiness: graph-only mode for CI; no telemetry statement found. Linux needs glibc 2.30+.
9. Evidence: self-reported 60 files/s indexing and sub-100 ms queries. No benchmark.
10. Caveats: early and open-core.

### code-graph-mcp (sdsrss)

1. URL: https://github.com/sdsrss/code-graph-mcp (live).
2. Description: tree-sitter AST graph with BLAKE3 Merkle-tree incremental re-index, hybrid BM25 plus vector search, recursive-CTE call-graph traversal with cycle detection, and HTTP route tracing for Express/Connect, Flask/FastAPI, Go net/http and Rust axum.
3. Backend: SQLite with FTS5 and sqlite-vec at `.code-graph/index.db`.
4. Languages: 19.
5. Claude Code integration: ships as a Claude Code plugin with skills explore and index, a code-explorer agent, auto-index hooks, StatusLine integration and self-update. Tools: project_map, semantic_code_search, get_call_graph, get_ast_node, module_overview, ast_search, find_references.
6. Licence: MIT.
7. Maturity: 83 stars, created 2026-03-10, pushed 2026-09-29. Individual.
8. Enterprise readiness: local only. Full index of its own 283-file repo in about 2 s.
9. Evidence: own latency numbers only.
10. Caveats: the README candidly lists unresolved cases: calls through module paths, receiver methods shared across types, cross-file constant and type uses.

### stakgraph (Stakwork)

1. URL: https://github.com/stakwork/stakgraph (live).
2. Description: tree-sitter queries extract nodes; cross-file calls resolved optionally via LSP. Runs as an HTTP server backed by Neo4j with embedding and visualisation. Links endpoints across repos, for example a backend `POST /api/users` to the frontend fetch that calls it. `POST /ingest_async` with webhooks.
3. Backend: Neo4j.
4. Languages: 16 including TypeScript, JavaScript, Python, Go, Rust, Ruby, Java, Kotlin, Swift, C#, PHP, C/C++, Angular, Svelte, Bash, TOML.
5. Claude Code integration: MCP server named for Cursor, Claude Code, Windsurf and OpenCode. Tools: stakgraph_search, stakgraph_map, stakgraph_code, stakgraph_shortest_path. No setup snippet in the README.
6. Licence: none found. The API returns null, and LICENSE, LICENSE.md and the README contain no licence. Treat as all-rights-reserved until clarified.
7. Maturity: 112 stars, created 2025-03-07, pushed 2026-10-01, 59 open issues. Company maintainer.
8. Enterprise readiness: multi-repo and async ingestion are first-class. Needs language servers for full LSP resolution.
9. Evidence: none.
10. Caveats: the missing licence is the blocker.

### Small Neo4j-native projects

- codeKG, https://github.com/eviking/codeKG. Apache-2.0, 1 star, created 2026-06-08, pushed 2026-09-30. Tree-sitter plus XML parsers for Java, Python, C++, JS/TS, and Salesforce Apex/LWC/Aura/Flows and SAP ABAP, into Neo4j, served over SSE at localhost:8002 and wired into Claude Code via `.mcp.json` and a generated CLAUDE.md. Tools: answer_question, get_class_context, get_module_context, get_change_impact, search_classes, check_violations, list_arch_policies, capture_insight, submit_session_telemetry. Multi-repo console with per-commit ingestion. Needs Docker and an Anthropic key for NL features. Note the telemetry tool.
- codebase-pkg, https://github.com/Sylphie-Labs/codebase-pkg. MIT, 1 star, pushed 2026-09-30. ts-morph for TypeScript/TSX and Python `ast` into Neo4j with pgvector Postgres for conformity checks. Tools: getModuleContext, getFunctionDetail, getDataFlow, getRecentChanges, getConstraints, searchContent, judgeConformity. Delta sync by SHA-256 per entity with change-history nodes. Self-described 0.x.
- graph-codebase-mcp, https://github.com/eric050828/graph-codebase-mcp. 9 stars, last push 2025-07-07, Python AST only, OpenAI embeddings required. Stale.
- neuro-code, https://github.com/mohitmishra786/neuro-code. MIT, 0 stars, Python only, tree-sitter to Neo4j. Hobby.
- CodeGraph (Abhishek-Aditya-bs), https://github.com/Abhishek-Aditya-bs/CodeGraph. MIT, 22 stars, pushed 2026-06-01. Neo4j entity graph plus 3072-dim vector index with Streamlit UI. No MCP server found.
- denzharkov/codegraph-mcp, 1 star, pure JS plus WASM, SQLite. Hobby.

### Potpie

1. URL: https://github.com/potpie-ai/potpie (live). Blog: https://potpie.ai/blog/building-a-distributed-knowledge-graph-pipeline
2. Description: v2.0 reframes Potpie as a "context graph" CLI daemon indexing code, structure, decisions, source history and workflows. The March 2026 engineering post documents tree-sitter extraction of FILE, CLASS, FUNCTION and INTERFACE nodes with CONTAINS, CALLS, INHERITS and IMPLEMENTS edges, written to Neo4j in batches of 1,000, with a 35-million-line repo processed in hours.
3. Backend: Neo4j per the March 2026 post. The v2.0.1 release notes do not name Neo4j and PR text references FalkorDB. Current default backend UNVERIFIED. Docker compose files returned 404.
4. Languages: not stated in the current README.
5. Claude Code integration: `potpie skills install --agent claude-code` installs instructions and skills. The v2.0 notes include "move daemon into root package and remove MCP", so there is no MCP server as of v2.0.1.
6. Licence: Apache-2.0.
7. Maturity: 5,737 stars, 685 forks, created 2024-08-12, pushed 2026-09-30, v2.0.1 released 2026-08-31. Company-backed.
8. Enterprise readiness: local daemon; a PR snippet indicates a PostHog telemetry sink with an anonymous install id and opt-out. Details UNVERIFIED.
9. Evidence: scale anecdote in the blog post only.
10. Caveats: mid-rewrite. Watch, do not adopt yet.

### FalkorDB code-graph

1. URL: https://github.com/FalkorDB/code-graph (live). Docs: https://docs.falkordb.com/genai-tools/code-graph
2. Description: custom source analysers under `api/analyzers/` index into FalkorDB with a GraphRAG chat UI.
3. Backend: FalkorDB or embedded FalkorDBLite.
4. Languages: Python, Java, C# only.
5. Claude Code integration: `cgraph-mcp` stdio server with index_repo, search_code, find_symbol, get_neighbors, get_file_neighbors, impact_analysis, find_path, plus a Claude Code skill under `skills/code-graph/`. The FalkorDB docs page does not mention MCP or Claude.
6. Licence: MIT.
7. Maturity: 348 stars, 54 forks, created 2024-01-03, pushed 2026-09-24. FalkorDB Ltd.
8. Enterprise readiness: Docker or FalkorDB Cloud.
9. Evidence: none.
10. Caveats: README admits legacy analyser tests fail on a clean checkout. Demo-grade.

### Blarify (Blar)

1. URL: https://github.com/blarApp/blarify (live).
2. Description: Python library that parses a repo into nodes and edges (imports, definitions, references) using SCIP for reference resolution, with LSP fallback.
3. Backend: Neo4j (container manager in-repo) with FalkorDB tagged.
4. Languages: Python, JavaScript, TypeScript, Ruby, Go, C#.
5. Claude Code integration: none. No MCP server found. A CLAUDE.md exists for contributors.
6. Licence: MIT.
7. Maturity: 232 stars, pushed 2026-08-17. Company maintainer.
8. Enterprise readiness: incremental update is listed as future work.
9. Evidence: none.
10. Caveats: a building block, not an integration.

### mcp-code-graph (JudiniLabs / CodeGPT)

URL: https://github.com/JudiniLabs/mcp-code-graph (live). MIT wrapper, 402 stars, last push 2025-11-25. Graphs are built and hosted by CodeGPT's cloud; private graphs need an API key, organisation id and graph id. Tools: list-graphs, get-code, find-direct-connections, nodes-semantic-search, docs-semantic-search, get-usage-dependency-links, folder-tree-structure. Data egress and ten months without a push rule it out.

### Augment Context Engine MCP

URL: https://www.augmentcode.com/product/context-engine-mcp (live). Proprietary hosted service. The indexer runs locally but an Augment account and credits are required and the engine has no source repository. Vendor benchmark: 300 Elasticsearch PRs, three prompts each, 900 attempts, scored on five dimensions, claiming +80% for Claude Code with Opus 4.6. Vendor-run, not independent. The community wrapper https://github.com/aj47/auggie-context-mcp (MIT, 38 stars) was last pushed 2026-01-15. Out of scope for an open-source shortlist.

### Sourcegraph MCP and SCIP

URLs: https://sourcegraph.com/mcp and https://github.com/sourcegraph/scip (both live). SCIP is Apache-2.0, 820 stars, pushed 2026-09-29, with indexers scip-java (Java, Scala, Kotlin), scip-typescript, scip-clang, scip-ruby, scip-python, scip-dotnet, scip-dart, scip-php, plus rust-analyzer. The Sourcegraph MCP server (search, navigation, history, Code Finder) is a feature of the proprietary platform and the page does not state self-hosting or licence terms. Use SCIP indexers as a precise-reference input to a graph you own; CodeGraphContext and Blarify already do this.

### Nuanced

URL: https://github.com/nuanced-dev/nuanced (live). Python-only enriched call graphs as JSON. MIT, 128 stars, archived 2026-03-05, last push 2025-06-26. Dead.

### Graphiti (Zep)

URL: https://github.com/getzep/graphiti (live). Temporal knowledge graph framework. Backends: Neo4j 5.26+, FalkorDB 1.1.2+, Amazon Neptune; Kuzu deprecated. The MCP server exposes add_memory, add_triplet, search_nodes, search_memory_facts, summarize_saga, build_communities, get_episode_entities, delete_entity_edge, delete_episode, clear_graph, get_status over HTTP at `/mcp/`. No code parser; entities come from LLM extraction. Anonymous telemetry on by default, disabled with `GRAPHITI_TELEMETRY_ENABLED=false`. Apache-2.0, 31,358 stars, pushed 2026-09-30, v0.30.2 on 2026-09-08. Company-backed. Good agent-memory layer on Neo4j, wrong tool for a symbol graph.

### Cognee

1. URL: https://github.com/topoteretes/cognee (live). Docs: https://docs.cognee.ai/
2. Description: AI memory platform. The code path parses ASTs to extract function signatures, call graphs, class hierarchies and import dependencies as typed graph entities (per the 2026-06-04 guide, updated 2026-09-03). Documents go through a six-stage cognify pipeline with LLM entity extraction.
3. Backend: Kuzu/Ladybug file-based by default (not for production); Neo4j self-hosted and Neo4j Aura recommended for production; Neptune and Neptune Analytics recommended on AWS; Postgres as demo; Memgraph community-maintained outside core release testing.
4. Languages: not stated on the pages fetched.
5. Claude Code integration: a dedicated plugin, `claude plugin install cognee-memory@cognee`, hooks SessionStart, UserPromptSubmit, PostToolUse, Stop, PreCompact and SessionEnd. MCP: `claude mcp add --transport http cognee http://localhost:8000/mcp` for Docker, or stdio via uv. Tools: remember, recall, forget, code_search ("source-code structure, symbols, call and dependency paths, change impact, architecture, and index deltas"), search_tools, call_tool, cognify_status. The older cognify and search tools have been removed.
6. Licence: Apache-2.0.
7. Maturity: 31,288 stars, 3,148 forks, created 2023-08-16, pushed 2026-10-01, v1.6.1 on 2026-09-24. Company-backed.
8. Enterprise readiness: self-hostable; Neo4j recommended for multi-agent concurrency. Telemetry on by default, disabled with `TELEMETRY_DISABLED=true`.
9. Evidence: BEAM memory benchmark (0.79 at 100K tokens) and one testimonial. Nothing code-specific.
10. Caveats: code graph is a side feature of a memory platform and its documentation is thin.

### LightRAG and nano-graphrag

URLs: https://github.com/HKUDS/LightRAG (MIT, 39,951 stars, pushed 2026-10-01, HKU academic maintainers) and https://github.com/gusye1234/nano-graphrag (MIT, 3,991 stars, last push 2026-01-27). Both support Neo4j (LightRAG also Memgraph, Postgres, NetworkX, MongoDB) but are document-oriented LLM entity extractors with no code parser and no MCP server. Not code-graph tools.

### Serena (oraios)

URL: https://github.com/oraios/serena (live). LSP-based symbol retrieval and editing with no persistent graph. 40+ languages. Tools: find_symbol, symbol_overview, find_referencing_symbols, find_declaration, find_implementations, diagnostics, rename, move, inline, safe_delete, replace_symbol_body, insert_after_symbol, search_for_pattern. Claude Code: `claude mcp add --scope user serena -- serena start-mcp-server --context claude-code --project-from-cwd`. The docs recommend a system-prompt override and four hooks (remind, activate, cleanup, auto-approve) because recent Claude Code builds bias toward built-in tools. Dual licence: SolidLSP MIT, Serena GPL-3.0-or-later. 29,935 stars, pushed 2026-09-30, v1.7.0 on 2026-08-09. Evidence is user quotes. Complementary to any graph tool for precise edits.

### Claude Context (Zilliz)

URL: https://github.com/zilliztech/claude-context (live). AST-chunked embeddings into Milvus or Zilliz Cloud with Merkle-tree incremental re-index. Tools: index_codebase, search_code, clear_index, get_indexing_status. MIT, 12,584 stars, last push 2026-07-14. Evaluation on 30 SWE-bench Verified instances with GPT-4o-mini, three runs each, reports a 39.4% token reduction and 36.3% fewer tool calls at equal F1 of 0.40. Requires a vector database and embedding provider, so code leaves the machine unless Milvus and Ollama are self-hosted. Not a graph.

### ast-grep MCP, repomix, code-index-mcp

- https://github.com/ast-grep/ast-grep-mcp: structural pattern search only (find_code, find_code_by_rule, dump_syntax_tree). MIT, 470 stars, pushed 2026-10-01, labelled experimental.
- https://github.com/yamadashy/repomix: packs files with optional tree-sitter signature compression; six MCP tools. MIT, 28,639 stars, pushed 2026-09-29. No graph.
- https://github.com/johnhuang316/code-index-mcp: tree-sitter for ten languages, build_deep_index adds call tracking. MIT, 1,000 stars, last push 2026-07-27. No graph database.

### Aider repo map (reference design)

URLs: https://aider.chat/docs/repomap.html and https://aider.chat/2023/10/22/repomap.html. Tree-sitter tags for definitions and references, a file-level dependency graph, PageRank to choose identifiers within a default 1k-token budget. No benchmark published for the map itself. Several 2026 tools above are this idea with persistence and MCP added.

### Joern and wrappers

- https://github.com/joernio/joern: code property graphs for C/C++, Java, binary, JavaScript, Python, Kotlin on the Flatgraph store, Scala query language. Apache-2.0, 3,540 stars, pushed 2026-10-01, v4.0.644 released 2026-10-01. No MCP in the main repo.
- https://github.com/Lekssays/codebadger: containerised MCP wrapper from QCRI, accepted at an ICSE 2026 workshop. GPL-3.0, 171 stars, pushed 2026-08-31, v0.7.0-beta on 2026-08-21. Twelve languages. Tools include generate_cpg, list_methods, list_calls, get_call_graph, get_cfg, run_cpgql_query (arbitrary CPGQL), find_taint_sources, find_taint_sinks, find_taint_flows, get_program_slice, get_variable_flow and memory-safety detectors. Production needs Postgres and Redis.
- https://github.com/BlockSecCA/joern-mcp: 5 stars.
Right tools for security review, heavy for daily navigation.

### CodeQL, Kythe, Glean, stack-graphs

- https://github.com/github/codeql: libraries MIT, but the CLI needs a commercial licence for closed-source code. No MCP server in the repo.
- https://github.com/kythe/kythe: Apache-2.0, 2,159 stars, pushed 2026-09-18, C++, Go and Java indexers, no MCP.
- https://github.com/facebookincubator/Glean: BSD (API: NOASSERTION), 1,417 stars, pushed 2026-10-01, Angle query language, native C++, Hack, Haskell, JS/Flow, Python plus SCIP/LSIF import, no MCP.
- https://github.com/github/stack-graphs: archived 2025-09-09.
Enterprise-grade indexers that need a custom bridge to Claude Code.

### Database-side MCP servers

- https://github.com/neo4j-contrib/mcp-neo4j: MIT, 986 stars, pushed 2026-09-09, Neo4j Labs. mcp-neo4j-cypher exposes schema read plus read and write Cypher; also mcp-neo4j-memory, mcp-neo4j-data-modeling, mcp-neo4j-cloud-aura-api. STDIO, SSE and HTTP.
- https://neo4j.com/labs/genai-ecosystem/agent-skills/coding-skills/: lists get-schema, read-cypher, write-cypher, list-gds-procedures and Claude Code skills. No codebase indexer.
- https://github.com/memgraph/ai-toolkit: MIT, 112 stars, pushed 2026-10-01, run_query and schema tools.
Any graph loaded into Neo4j becomes queryable from Claude Code through these with zero code.

## Neo4j fit summary

- Native Neo4j backend with an MCP server: CodeGraphContext, Cognee, stakgraph, codeKG, codebase-pkg, graph-codebase-mcp.
- Neo4j as an export target: Graphify (Cypher export).
- Neo4j-capable but not code-aware: Graphiti, LightRAG, nano-graphrag.
- Neo4j per March 2026 docs, current status UNVERIFIED: Potpie.
- Memgraph-native, Bolt and Cypher compatible: code-graph-rag. A driver swap to Neo4j is not claimed and I did not verify it.
- Industry signal: SiliconANGLE (2026-09-28) reports Blitzy running proprietary Neo4j code graphs for thousands of agents and an 84.95% SWE-Bench Pro score. That validates the architecture, not any open tool.

## Comparison table

| Tool | Parser | Backend | Languages | Claude Code integration | Licence | Maturity (stars, last push) | Enterprise fit |
|---|---|---|---|---|---|---|---|
| code-graph-rag | tree-sitter + compiler frontends | Memgraph + Qdrant | 13 full, 8 partial | MCP server | MIT | 5.2k, 2026-10-01 | Good; on-prem support sold; Memgraph dependency |
| GitNexus | tree-sitter | LadybugDB | 16 | MCP + skills + CLAUDE.md | PolyForm Noncommercial | 47.7k, 2026-10-01 | Blocked by licence |
| code-review-graph | tree-sitter | SQLite | ~40 | MCP + skills + install hooks | MIT | 31.9k, 2026-09-18 | Good local-first; no Neo4j |
| codebase-memory-mcp | tree-sitter + LSP | SQLite | 162 | MCP binary | MIT | 45.6k, 2026-10-01 | Good local-first; multi-repo; arXiv eval |
| codegraph (colbymchenry) | tree-sitter (Rust) | SQLite | 20 + 10 | MCP + watcher | MIT | 72.8k, 2026-10-01 | Good local-first |
| CodeGraphContext | tree-sitter + SCIP | FalkorDB Lite / Kuzu / Neo4j | 23 | MCP + Cypher passthrough + watch | MIT | 4.2k, 2026-09-29 | Best Neo4j option |
| Graphify | tree-sitter, no LLM for code | graph.json, Neo4j export | 37 | skill + hooks + MCP | Apache-2.0 | 123k, 2026-09-30 | Good; Neo4j export only |
| CodeGraph (codegraph-ai) | tree-sitter | RocksDB | 38 | MCP | Apache-2.0, open-core | 104, 2026-10-01 | Early |
| code-graph-mcp (sdsrss) | tree-sitter | SQLite + sqlite-vec | 19 | plugin + hooks | MIT | 83, 2026-09-29 | Early |
| stakgraph | tree-sitter + LSP | Neo4j | 16 | MCP + HTTP | none found | 112, 2026-10-01 | Blocked until licensed |
| codeKG | tree-sitter | Neo4j | 5 + Salesforce, ABAP | SSE MCP + CLAUDE.md | Apache-2.0 | 1, 2026-09-30 | Niche, unproven |
| codebase-pkg | ts-morph + Python ast | Neo4j + pgvector | 2 | MCP | MIT | 1, 2026-09-30 | 0.x |
| Potpie | tree-sitter | Neo4j per 2026-03 blog; UNVERIFIED now | not stated | skills; MCP removed in v2.0 | Apache-2.0 | 5.7k, 2026-09-30 | Wait for v2 to settle |
| FalkorDB code-graph | custom | FalkorDB | 3 | MCP + skill | MIT | 348, 2026-09-24 | Demo |
| Blarify | SCIP + LSP | Neo4j / FalkorDB | 6 | none | MIT | 232, 2026-08-17 | Library only |
| Cognee | AST for code, LLM for docs | Neo4j / Aura / Neptune / Kuzu | not stated | plugin with hooks + MCP code_search | Apache-2.0 | 31.3k, 2026-10-01 | Good; disable telemetry |
| Graphiti | LLM extraction | Neo4j / FalkorDB / Neptune | n/a | MCP | Apache-2.0 | 31.4k, 2026-09-30 | Memory only, not code |
| LightRAG | LLM extraction | Neo4j / Memgraph / PG / NetworkX | n/a | none | MIT | 40.0k, 2026-10-01 | Not a code tool |
| Serena | LSP | none | 40+ | MCP + hooks | MIT / GPL-3.0 | 29.9k, 2026-09-30 | Complement; GPL caveat |
| Claude Context | AST chunking | Milvus | 14 | MCP | MIT | 12.6k, 2026-07-14 | Vector; egress unless self-hosted |
| codebadger (Joern) | Joern CPG | Flatgraph | 12 | MCP, Docker | GPL-3.0 | 171, 2026-08-31 | Security use only |
| mcp-code-graph (CodeGPT) | cloud | CodeGPT cloud | not stated | MCP | MIT wrapper | 402, 2025-11-25 | Data egress; stale |
| Augment Context Engine | proprietary | proprietary | not stated | MCP | proprietary | n/a | Out of scope |
| Kythe / Glean / SCIP | language-specific | custom | varies | none | Apache / BSD | active | Needs a bridge |
| Nuanced | Python static | JSON | 1 | none | MIT | archived | Dead |

## Top five picks for an enterprise Neo4j shop

1. CodeGraphContext. The only actively maintained, permissively licensed tool that writes a symbol graph straight into an external Neo4j and exposes read-only execute_cypher_query to Claude Code alongside typed relationship queries (callers, callees, hierarchy, dead code, module deps). SCIP support gives precise references for C/C++ and C# where tree-sitter guesses. Risk: single maintainer at 0.5.x, a recent parser data-loss fix, and no published accuracy numbers. Pilot on one repo and measure before rollout.

2. Cognee. Company-backed, Apache-2.0, Neo4j recommended for production and for multi-agent concurrency, a deterministic AST code path exposed as code_search, and the only native Claude Code plugin here that hooks the full session lifecycle. Risk: the code graph is a side feature of a memory platform with thin docs. Set TELEMETRY_DISABLED=true.

3. code-graph-rag. The deepest parser stack (compiler frontends on top of tree-sitter), runtime-trace ingestion, and the only project selling on-prem and air-gapped support. Memgraph speaks Bolt and Cypher, so existing Cypher skills transfer, but running a second graph database is a real cost. Ask the maintainer whether Neo4j is supported before committing.

4. Graphify. Deterministic, no LLM touches code, no telemetry, incremental with post-commit hooks, and it exports Cypher you can load into the existing Neo4j and query through mcp-neo4j-cypher. Risk: Neo4j is a batch export rather than the live store, and the star count is anomalous for a six-month-old repo.

5. codebase-memory-mcp as the local-first fallback. It has the only arXiv-published evaluation (31 repos, 83% versus 92% answer quality at ten times fewer tokens), cross-repo linking, zero dependencies and a no-telemetry statement. It does not touch Neo4j, so use it where a per-developer index is acceptable.

Honourable mentions. stakgraph is Neo4j-native with cross-repo API linking and async ingestion and would be a top-three pick if it had a licence file; ask Stakwork. Serena pairs with any of the above for symbol-precise edits, mindful of GPL-3.0. codebadger is the choice for taint and slicing questions in security review. Potpie is worth re-checking once v2 stabilises and its backend is documented.

## All URLs used

GitHub pages and API:
- https://github.com/vitali87/code-graph-rag
- https://api.github.com/repos/vitali87/code-graph-rag
- https://api.github.com/repos/vitali87/code-graph-rag/releases/latest
- https://code-graph-rag.com/
- https://github.com/abhigyanpatwari/GitNexus
- https://api.github.com/repos/abhigyanpatwari/GitNexus
- https://raw.githubusercontent.com/abhigyanpatwari/GitNexus/main/LICENSE
- https://github.com/tirth8205/code-review-graph
- https://api.github.com/repos/tirth8205/code-review-graph
- https://github.com/DeusData/codebase-memory-mcp
- https://api.github.com/repos/DeusData/codebase-memory-mcp
- https://arxiv.org/abs/2603.27277
- https://github.com/colbymchenry/codegraph
- https://api.github.com/repos/colbymchenry/codegraph
- https://github.com/CodeGraphContext/CodeGraphContext
- https://api.github.com/repos/CodeGraphContext/CodeGraphContext
- https://api.github.com/repos/CodeGraphContext/CodeGraphContext/releases/latest
- https://github.com/safishamsi/graphify
- https://api.github.com/repos/safishamsi/graphify
- https://github.com/codegraph-ai/CodeGraph
- https://api.github.com/repos/codegraph-ai/CodeGraph
- https://github.com/sdsrss/code-graph-mcp
- https://api.github.com/repos/sdsrss/code-graph-mcp
- https://github.com/stakwork/stakgraph
- https://api.github.com/repos/stakwork/stakgraph
- https://raw.githubusercontent.com/stakwork/stakgraph/main/README.md
- https://github.com/eviking/codeKG
- https://api.github.com/repos/eviking/codeKG
- https://github.com/Sylphie-Labs/codebase-pkg
- https://api.github.com/repos/Sylphie-Labs/codebase-pkg
- https://github.com/eric050828/graph-codebase-mcp
- https://api.github.com/repos/eric050828/graph-codebase-mcp
- https://api.github.com/repos/mohitmishra786/neuro-code
- https://api.github.com/repos/Abhishek-Aditya-bs/CodeGraph
- https://api.github.com/repos/denzharkov/codegraph-mcp
- https://github.com/potpie-ai/potpie
- https://raw.githubusercontent.com/potpie-ai/potpie/main/README.md
- https://api.github.com/repos/potpie-ai/potpie
- https://api.github.com/repos/potpie-ai/potpie/releases/latest
- https://github.com/potpie-ai/potpie/releases/tag/v2.0.1
- https://potpie.ai/blog/building-a-distributed-knowledge-graph-pipeline
- https://docs.potpie.ai/
- https://github.com/FalkorDB/code-graph
- https://api.github.com/repos/FalkorDB/code-graph
- https://docs.falkordb.com/genai-tools/code-graph
- https://github.com/blarApp/blarify
- https://api.github.com/repos/blarApp/blarify
- https://github.com/JudiniLabs/mcp-code-graph
- https://api.github.com/repos/JudiniLabs/mcp-code-graph
- https://www.augmentcode.com/product/context-engine-mcp
- https://api.github.com/repos/aj47/auggie-context-mcp
- https://sourcegraph.com/mcp
- https://github.com/sourcegraph/scip
- https://api.github.com/repos/sourcegraph/scip
- https://github.com/nuanced-dev/nuanced
- https://api.github.com/repos/nuanced-dev/nuanced
- https://github.com/getzep/graphiti
- https://api.github.com/repos/getzep/graphiti
- https://api.github.com/repos/getzep/graphiti/releases/latest
- https://github.com/getzep/graphiti/tree/main/mcp_server
- https://github.com/topoteretes/cognee
- https://api.github.com/repos/topoteretes/cognee
- https://docs.cognee.ai/
- https://docs.cognee.ai/cognee-mcp/mcp-overview
- https://docs.cognee.ai/cognee-mcp/mcp-tools
- https://docs.cognee.ai/setup-configuration/graph-stores
- https://docs.cognee.ai/cognee-mcp/integrations/claude-code
- https://cognee.ai/blog/guides/ai-coding-agent-persistent-codebase-memory
- https://github.com/HKUDS/LightRAG
- https://api.github.com/repos/HKUDS/LightRAG
- https://github.com/gusye1234/nano-graphrag
- https://api.github.com/repos/gusye1234/nano-graphrag
- https://github.com/oraios/serena
- https://api.github.com/repos/oraios/serena
- https://api.github.com/repos/oraios/serena/releases/latest
- https://oraios.github.io/serena/02-usage/030_clients.html
- https://github.com/zilliztech/claude-context
- https://api.github.com/repos/zilliztech/claude-context
- https://github.com/zilliztech/claude-context/tree/master/evaluation
- https://github.com/ast-grep/ast-grep-mcp
- https://api.github.com/repos/ast-grep/ast-grep-mcp
- https://github.com/yamadashy/repomix
- https://api.github.com/repos/yamadashy/repomix
- https://github.com/johnhuang316/code-index-mcp
- https://api.github.com/repos/johnhuang316/code-index-mcp
- https://aider.chat/docs/repomap.html
- https://aider.chat/2023/10/22/repomap.html
- https://github.com/joernio/joern
- https://api.github.com/repos/joernio/joern
- https://api.github.com/repos/joernio/joern/releases/latest
- https://github.com/Lekssays/codebadger
- https://api.github.com/repos/Lekssays/codebadger
- https://api.github.com/repos/Lekssays/codebadger/releases/latest
- https://raw.githubusercontent.com/Lekssays/codebadger/main/docs/available-tools.md
- https://api.github.com/repos/BlockSecCA/joern-mcp
- https://github.com/github/codeql
- https://api.github.com/repos/github/codeql
- https://github.com/kythe/kythe
- https://api.github.com/repos/kythe/kythe
- https://github.com/facebookincubator/Glean
- https://api.github.com/repos/facebookincubator/Glean
- https://github.com/github/stack-graphs
- https://api.github.com/repos/github/stack-graphs
- https://github.com/neo4j-contrib/mcp-neo4j
- https://api.github.com/repos/neo4j-contrib/mcp-neo4j
- https://neo4j.com/labs/genai-ecosystem/agent-skills/coding-skills/
- https://api.github.com/repos/memgraph/ai-toolkit
- https://memgraph.com/blog/graphrag-for-devs-coding-assistant

Independent commentary and evidence:
- https://www.saurabhsharma.dev/blogs/code-graph-mcp-tools-comparison/
- https://www.russ.cloud/2026/05/10/codebase-memory-mcp-giving-claude-code-and-codex-a-map/
- https://baeseokjae.github.io/posts/ai-coding-agent-code-index-mcp-comparison-2026/
- https://siliconangle.com/2026/09/28/knowledge-graphs-blitzy-coding-agents-codebase-context-neo4jgraphsummit/
- HN Algolia searches at https://hn.algolia.com/api/v1/search for codebase-memory-mcp, code-review-graph, GitNexus, code-graph-rag and "codegraph claude code"

Search-result pages consulted for discovery (not cited for facts): WebSearch results pointing to marktechpost.com, hoangyell.com, mcpservers.org, glama.ai, skillsllm.com, lobehub.com, codegraphcontext.github.io, regolo.ai, awesomeclaudeplugins.com.

Pages that returned 404 and contributed nothing:
- https://www.augmentcode.com/changelog/context-engine-mcp-in-ga
- https://raw.githubusercontent.com/potpie-ai/potpie/main/docker-compose.yaml
- https://raw.githubusercontent.com/potpie-ai/potpie/main/docker-compose.yml
- https://raw.githubusercontent.com/stakwork/stakgraph/main/LICENSE
- https://raw.githubusercontent.com/stakwork/stakgraph/main/LICENSE.md
