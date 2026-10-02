# Graph-database and graph-memory ecosystem for a Claude Code codebase knowledge graph

Survey date: 2026-10-01. Emphasis on Neo4j, since the reader already runs Neo4j in production for a documentation knowledge graph. Every item below was checked against a fetched page unless marked UNVERIFIED. Version numbers come from PyPI JSON or GitHub release pages. Two fetched GitHub pages (the Neo4j Labs releases page and the CodeGraphContext releases page) rendered years one lower than PyPI, so PyPI is treated as authoritative for those.

## Part 1. Neo4j official assets

### 1.1 Official Neo4j MCP server (github.com/neo4j/mcp)

This is now the primary server. It is written in Go, with a PyPI mirror named neo4j-mcp-server. Releases: v1.6.0 on 10 Sep 2026, v1.5.3 on 11 Jun 2026, v1.5.2 on 23 Apr 2026, v1.5.1 on 15 Apr 2026, v1.5.0 on 24 Mar 2026. PyPI shows 1.6.0 uploaded 10 Sep 2026. The repository license file is GPL-3.0 and PyPI records GPL-3.0-only. Running it as a separate process is unaffected, but embedding it in a proprietary product is a real caveat.

Tools: get-schema, read-cypher, write-cypher, list-gds-procedures, plus further GDS tools when the GDS plugin is present. The server runs in an adaptive mode when GDS is absent. Read-only mode is enforced by running EXPLAIN on each statement and the write tool is disabled when the read-only flag is set. v1.6.0 deprecated legacy environment variables in favour of NEO4J_MCP_* names, so the docs page still shows NEO4J_READ_ONLY while the repository shows NEO4J_MCP_READ_ONLY. Other settings: schema sample size (default 100), stdio or HTTP transport, per-request bearer or basic auth over HTTP, TLS, CORS allow-list, a /healtz endpoint, and telemetry on by default. APOC is required. Install via pip, Homebrew (brew install neo4j-mcp) or a static binary. Repository showed 296 stars.

A community guide dated 3 May 2026 gives the Claude Code registration:

```
claude mcp add --transport stdio -e NEO4J_URI=bolt://127.0.0.1:7687 -e NEO4J_USERNAME=neo4j -e NEO4J_PASSWORD=... -e NEO4J_DATABASE=neo4j -e NEO4J_READ_ONLY=true -- neo4j-mcp neo4j-mcp
```

The same guide warns that registering the server directly loads its tool schemas into every conversation, and suggests a skill file layer above the raw MCP tools. No official Claude Code plugin from Neo4j was found. Several third-party Neo4j plugins exist on plugin hubs. UNVERIFIED.

### 1.2 Neo4j Labs MCP servers (github.com/neo4j-contrib/mcp-neo4j)

MIT licensed, Python, maintained in parallel with the Go server.

| Package | Version | PyPI upload date |
|---|---|---|
| mcp-neo4j-cypher | 0.6.0 | 10 Apr 2026 |
| mcp-neo4j-memory | 0.4.5 | 23 Feb 2026 |
| mcp-neo4j-aura-manager | 0.4.8 | 23 Feb 2026 |
| mcp-neo4j-data-modeling | 0.8.2 | 22 Dec 2025 |

mcp-neo4j-cypher launched December 2024 and exposes get_neo4j_schema, read_neo4j_cypher and write_neo4j_cypher. It has agent-hygiene knobs the Go server lacks: a response token limit that truncates with tiktoken (NEO4J_RESPONSE_TOKEN_LIMIT), a read timeout defaulting to 30 seconds, a schema sample size defaulting to 1000 nodes per label, and stdio, HTTP or SSE transport. Tomaž Bratanič's "Production-proofing your Neo4j Cypher MCP server" (8 Oct 2025) recommends read-only mode, timeouts, the token limit, stripping embedding vectors and lists over 128 items from results, DNS-rebinding and CORS protection, and a read-only database user.

mcp-neo4j-memory stores Memory nodes with name, type and observations plus typed relations, and offers read_graph, search_nodes, find_nodes, create_entities, delete_entities, create_relations, delete_relations, add_observations and delete_observations. It is a generic agent-memory store, not a code graph.

mcp-neo4j-data-modeling validates node, relationship and model structures, renders Mermaid, imports and exports Arrows, emits OWL Turtle, Pydantic and neo4j-graphrag schema, generates constraint and ingestion Cypher, and ships seven example models including one for software dependencies. It is useful for designing the code-graph schema before writing it.

mcp-neo4j-aura-manager handles Aura instance lifecycle.

### 1.3 Related first-party agent assets

- GDS Agent (github.com/neo4j-contrib/gds-agent). MIT, 97 stars, run with uvx gds-agent. Exposes centrality, community detection, path finding, similarity, node embedding and ML pipeline algorithms over stdio or HTTP. Needs the GDS plugin or an Aura Graph Analytics session. Ships a .mcpb bundle and a skill zip for Claude Desktop.
- Neo4j post "Agentic graph analytics with external MCP providers" (8 Dec 2025, Bratanič). Two design rules: keep raw graph data out of the LLM context by having one tool run the whole fetch-and-project pipeline internally, and group related algorithms into one tool that takes the algorithm name as a parameter rather than exposing dozens of tools.
- neo4j-contrib/neo4j-skills. MIT, 113 stars. Packages Cypher, data modeling, GDS, vector index, GraphRAG and agent-memory skills in the agentskills.io format and targets Claude Code, Codex, Gemini CLI and Antigravity via npx skills add.
- Google MCP Toolbox (googleapis/genai-toolbox) has a Neo4j source since December 2024.
- Aura Agent (hosted). Relevant mainly as a design reference. Exactly three tool types: Cypher Template, Similarity Search and Text2Cypher. Templates are recommended for common and repeated questions, with results of 10 to 50 rows and no embeddings. Agents can be exposed as MCP endpoints. All agents run in GCP europe-west1.

### 1.4 neo4j-graphrag Python package

Version 1.22.0 uploaded to PyPI on 1 Oct 2026, Apache-2.0, Python 3.10 to 3.14, Neo4j 5.18.1 or later. SimpleKGPipeline chains a data loader (PDF and Markdown), a text splitter, an optional chunk embedder, a schema builder, an LLM entity-relation extractor, a KG writer and an entity resolver (exact, spaCy semantic, RapidFuzz fuzzy). The schema guides but does not strictly enforce extraction. There is no source-code loader, so for a code graph you build the structural graph yourself and use only the retrieval half.

Retrievers: VectorRetriever, VectorCypherRetriever, HybridRetriever, HybridCypherRetriever, Text2CypherRetriever, ToolsRetriever, and bridging retrievers for Weaviate, Pinecone and Qdrant. Text2CypherRetriever accepts an optional neo4j_schema string and few-shot examples, needs no embedder, and raises Text2CypherRetrievalError because generated Cypher is not guaranteed valid. Anthropic is a supported LLM provider alongside OpenAI, Azure, Vertex, Mistral, Cohere, Ollama and LangChain. In-index filtering via the SEARCH clause needs Neo4j 2026.01 or later.

### 1.5 GDS algorithms for code graphs

Current GDS docs are version 2026.09. PageRank is production-quality (gds.pageRank.stream/stats/mutate/write), damping 0.85, with sourceNodes enabling personalized PageRank from a seed symbol. Leiden is production-quality, wants an UNDIRECTED projection, and returns hierarchical levels when includeIntermediateCommunities is set, which maps onto package, module and function communities. Louvain, Label Propagation and WCC are in the same category. Node Similarity supports Jaccard, Overlap and Cosine with a bipartite framing and O(n²) space, so run it on file-to-symbol or module-to-dependency projections rather than the whole graph. KNN, FastRP, Node2Vec and HashGNN cover embeddings.

Editions: the Community edition includes all algorithms but caps concurrency at 4 cores and the model catalog at 3. Enterprise needs a license key and adds cluster writes, Arrow import and export, backup and restore, unlimited models. Aura Graph Analytics provides enterprise features without a license file.

### 1.6 Vector indexes

Available since Neo4j 5.13 with CREATE VECTOR INDEX, dimensions 1 to 4096, cosine or Euclidean, relationship vector indexes, scalar or binary quantization. Native VECTOR property storage arrived in 2025.10, multi-label indexes in 2026.01, high-fidelity quantized search in 2026.07. The db.index.vector.queryNodes procedure is deprecated as of 2026.04 in favour of the SEARCH clause introduced in 2026.01.

### 1.7 Fine-grained security

Label and relationship-type privileges for TRAVERSE, READ and MATCH are long-standing. Property-based access control is Enterprise, AuraDB Business Critical and Virtual Dedicated Cloud only. Form: GRANT READ {*} ON GRAPH * FOR (n) WHERE n.classification IS NULL TO role. Each rule is restricted to a single property. Cypher 25 (2026.08) allows the property on either side of the comparison and value IN n.listProperty. The docs warn that a DENY rule fails open when its criteria cannot be evaluated. Attribute-based access control arrived in 2026.03 with CREATE AUTH RULE ... SET CONDITION over OIDC claims, and native user tags in 2026.06. ABAC roles are added to RBAC roles rather than replacing them. Multi-database is Enterprise-only. Community Edition allows exactly one standard database, which matters for co-locating a code graph with the documentation graph.

### 1.8 Neo4j and codebase knowledge graphs

No first-party Neo4j blog or developer guide devoted to building a codebase knowledge graph was found. Searches on that phrase returned third-party repositories and papers. The Neo4j-backed evidence that exists:

- CodexGraph (arxiv.org/abs/2408.03910, NAACL 2025 long paper, v3 27 Jul 2026). Selects Neo4j and Cypher. Nodes: MODULE, CLASS, FUNCTION, FIELD, METHOD, GLOBAL_VARIABLE. Edges: CONTAINS, HAS_METHOD, HAS_FIELD, INHERITS, USES. The agent writes Cypher itself.

| Benchmark, GPT-4o | CodexGraph | AutoCodeRover | BM25 |
|---|---|---|---|
| CrossCodeEval Lite Python, exact match | 27.90% | 21.20% | 21.20% |
| SWE-bench Lite, pass@1 | 22.96% | 22.96% | 3.11% |
| EvoCodeBench, recall@1 | 11.87% | 11.17% | n/a |

- LocAgent (arxiv.org/abs/2503.09089, Mar 2025). Directed heterogeneous code graph. 92.7% file-level localization, 12% pass@10 improvement on issue resolution, fine-tuned Qwen2.5-Coder-32B at roughly 86% lower cost than proprietary models.
- Codebase-Memory (arxiv.org/abs/2603.27277, 28 Mar 2026) is the paper behind DeusData/codebase-memory-mcp. Tree-sitter over 66 languages into a SQLite store, not a graph database. Across 31 repositories: 83% answer quality against 92% for a file-exploration agent, 10x fewer tokens, 2.1x fewer tool calls. The repository is MIT, 45.6k stars, 162 languages, 17 tools including a Cypher-like query tool, and installs with one curl command. It is the strongest competitor to any graph-DB approach and the right baseline to benchmark against.
- techsavvyash/codegraph. Code property graph in Neo4j from SCIP indexers (Go, TypeScript, Python, JVM). Labels Service, File, Module, Function, Class, Variable, Symbol, APIRoute. Relationships CONTAINS, CALLS, DEFINES, REFERENCES, INHERITS_FROM, IMPLEMENTS, DEPENDS_ON, CALLS_API. Nine MCP tools: codegraph_find, codegraph_expand, codegraph_path, codegraph_source, codegraph_schema, codegraph_cypher, codegraph_entry_points, codegraph_flows, codegraph_render. 2 stars, no visible license. Design reference only.
- Abhishek-Aditya-bs/CodeGraph. MIT, 22 stars, 14 languages. Class, Function, Interface, File, Package nodes plus CodeChunk nodes with 3072-dimensional OpenAI embeddings, bridged by REPRESENTS, CONTAINS_CHUNK and PART_OF_FILE. No MCP server.
- Sylphie-Labs/codebase-pkg. MIT, 1 star. TypeScript via ts-morph and Python via stdlib ast into Neo4j. Eight MCP tools for Claude Code: getModuleContext, getFunctionDetail, getDataFlow, getRecentChanges, getConstraints, getLogContext, searchContent, judgeConformity. Change nodes linked by CHANGED_IN. Very early.
- CodeGraphContext. MIT, 4.2k stars, 0.6.13 uploaded 6 Sep 2026. Tree-sitter over 23 or more languages. Backends: FalkorDB Lite by default on Unix, Kuzu, LadybugDB, Neo4j (neo4j>=5.15.0), Nornic. Tools for indexing, call chains, dead code and complexity. Configures Claude Code via cgc mcp setup. Of the open tools it is the most mature route to a Neo4j-resident, tree-sitter-derived code graph.
- JohT/code-graph-analysis-pipeline. GPL-3.0, 34 stars, v4.0 May 2026. Runs jQAssistant for Java, the TypeScript plugin experimentally, SCIP for Go, Rust, C++, Ruby, Python and C#, then GDS Leiden, PageRank, HashGNN, FastRP with UMAP, Jaccard and topological sort across twelve analysis domains. CLI pipeline with Cypher files, no MCP server, but ships AGENTS.md. Its Cypher directory is a ready-made library of code-graph queries.

## Part 2. jQAssistant

Core is GPL-3.0, developed by BUSCHMAIS (Maven coordinates com.buschmais.jqassistant). Org page lists 2.9.0 released 28 Jan 2026 and the manual documents 2.9.1. It ships an embedded Neo4j Community Edition (4.x on JDK 11, 5.x on JDK 17 or later) and can instead write to a remote Neo4j over bolt by setting the store URI. That remote mode is how it coexists with an existing server.

Built-in scanners: Java bytecode, packages, manifests, properties and service loaders; Maven 3 POMs; XML and XSD; JSON; YAML; JUnit reports; files, directories and URLs. Rules are Cypher: concepts enrich the graph with labels and relationships, constraints detect violations. The Maven plugin or CLI runs scan, analyze and server, configured through .jqassistant.yml.

Plugins: the jqassistant-plugin org adds TypeScript (GPL-3.0, 4 stars, ingests JSON produced by a separate analyzer), Spring (updated Sep 2026), C4, PlantUML, OpenAPI and AsyncAPI. Kontext E maintains a Git plugin at 2.4.0 that records commits, authors, branches, file changes, renames and submodules, plus PMD, SpotBugs, JaCoCo and Checkstyle plugins. No Python plugin was found: a guessed repository URL returned 404 and the org page lists none. Python support is UNVERIFIED and probably absent. Enterprise use and commercial support are not documented in the fetched pages. UNVERIFIED. Maturity signals: a decade of releases and a stable plugin API, but low GitHub star counts.

Exposing jQAssistant to an agent takes three routes:

1. Point any Neo4j MCP server at the jQAssistant store.
2. 2015xli/jqassistant-graph-rag. Apache-2.0, 13 stars. Links bytecode nodes to source files, writes multi-level summaries from method to project, and exposes get_graph_schema, execute_cypher_query and get_file_source_code_by_path over MCP with OpenAI, DeepSeek or Ollama. No vector index use.
3. JohT/code-graph-analysis-pipeline as described above, for the enrichment and GDS queries.

## Part 3. Other graph databases

### Memgraph
In-memory C++ server. Business Source License for Community, Memgraph Enterprise License for Enterprise. v3.13.1 on 14 Sep 2026, 4.6k stars, Cypher-compatible. Official MCP server mcp-memgraph 0.4.1, MIT, uploaded 14 Sep 2026, inside memgraph/ai-toolkit. Tools: run_query, search_schema, get_node_schema, get_relationship_schema, plus database switching when auth is on. Stdio or streamable HTTP. The toolkit also carries a LightRAG storage backend. No first-party code-graph example found. Cognee lists Memgraph as a community adapter outside core release testing.

### FalkorDB
SSPLv1, a Redis module built on GraphBLAS sparse matrices, 6.5k stars, openCypher with extensions. v6.0 moved the engine to Rust; v6.0.1 released 1 Oct 2026. FalkorDBLite 0.10.0 (2 May 2026, Python 3.12 or later) runs an embedded Redis sub-process over a Unix socket. Official MCP server is TypeScript, MIT: query_graph, query_graph_readonly, list_graphs, get_graph_schema, get_node_schema, get_relationship_schema, read-only mode, Docker images. FalkorDB/code-graph is MIT, 348 stars, analyzes Python, Java and C#, and ships seven stdio MCP tools (index_repo, search_code, find_symbol, get_neighbors, get_file_neighbors, impact_analysis, find_path) plus a Claude Code skill. A search snippet claimed twelve tools including a security group. The README lists seven, so the larger count is UNVERIFIED. GraphRAG-SDK 1.0 shipped April 2026 under Apache-2.0 with ontology discovery, LLM extraction, entity resolution, and combined vector, full-text and text-to-Cypher retrieval with MENTIONED_IN provenance edges. SSPL is the main enterprise caveat.

### Kuzu and LadybugDB
kuzudb/kuzu was archived on 10 Oct 2025 with 0.11.3 as the last release, MIT. A secondary source attributes the archival to an Apple acquisition. UNVERIFIED. LadybugDB is the MIT community fork, 1.8k stars, embedded and serverless, Cypher, native full-text and vector indexes, pip install ladybug. Graphiti marks Kuzu deprecated. CodeGraphContext and Cognee support both kuzu and ladybug backends. Embedded Kuzu-family stores suit per-repo local code graphs but cannot share a database with a Neo4j documentation graph.

### NebulaGraph
Apache-2.0, distributed server, 12.4k stars, nGQL with openCypher compatibility. nebulagraph-mcp-server 0.1.1 (17 Mar 2025, Apache-2.0, Python 3.12 or later, NebulaGraph 3.x) offers schema exploration, queries, path finding and neighbor discovery. No code-graph story found.

### TigerGraph
Proprietary. Community Edition is free including production use, limited to 300 GB of graph plus vector data, 16 CPUs and a single server, and cannot be upgraded in place. Official MCP server tigergraph-mcp 1.0.3 (16 Sep 2026, Apache-2.0, TigerGraph Inc) has 69 tools spanning GSQL execution, schema, loading jobs, vector operations and natural-language generation of both GSQL and openCypher. Requires TigerGraph 4.1 or later. 5 stars. No code-graph story found.

### Amazon Neptune
Managed service. AWS Labs MCP server 1.1.1 (8 Sep 2026, Apache-2.0) supports openCypher, Gremlin and SPARQL on Neptune Database and openCypher on Neptune Analytics. It runs any query sent to it, so read-only must be enforced through IAM. Graphiti and Cognee both support Neptune. No code-graph story found.

### ArangoDB
BUSL 1.1 since 3.12 with a four-year reversion to Apache 2.0. A secondary blog states the Community Edition is capped at 100 GiB and non-commercial use. UNVERIFIED against arangodb.com. No official MCP server could be verified. arango-solutions/arango-solutions-mcp is Apache-2.0 with 81 tools, version 2.0.0, ArangoDB 3.12 or later, no read-only mode. ravenwits/mcp-server-arangodb is a smaller community TypeScript server. No code-graph story found.

## Part 4. Graph-memory layers for agents

### Graphiti and Zep
Apache-2.0, graphiti-core 0.30.2 uploaded 8 Sep 2026, 31.4k stars. Backends: Neo4j 5.26 or later, FalkorDB 1.1.2 or later (now default), Neptune, deprecated Kuzu. On Neo4j it creates labels Entity, Episodic, Community and Saga; relationships RELATES_TO, MENTIONS, HAS_MEMBER, HAS_EPISODE, NEXT_EPISODE; full-text indexes episode_content, node_name_and_summary, community_name, edge_name_and_fact; range indexes on uuid, group_id, name and created_at. Namespacing is a group_id property on every node and edge within one Neo4j database, whereas FalkorDB uses a separate graph per group. MCP Server 1.0 arrived 10 Nov 2025 with 13 tools (add_memory, search_nodes, search_memory_facts, delete_entity_edge, delete_episode, get_entity_edge, get_episodes, summarize_saga, build_communities, add_triplet, get_episode_entities, clear_graph, get_status), streamable HTTP by default, stdio for desktop, YAML configuration. Claude Code connects to http://localhost:8000/mcp/. No code ingestion; every write goes through an LLM for extraction. It coexists with a doc graph through distinct labels and group_id, but issue #1022 (25 Oct 2025) reported invalid multi-label SET Cypher on the Neo4j provider, so verify the fix is in your version. Neo4j published Daniel Chalef's Graphiti post on 24 Mar 2025.

### Cognee
Apache-2.0, v1.6.1 on GitHub (24 Sep 2026), 1.6.2 on PyPI, 31.3k stars. GRAPH_DATABASE_PROVIDER accepts kuzu (default), kuzu-remote, ladybug, ladybug-remote, neo4j (self-hosted, Desktop, Aura), postgres (demo), neptune, neptune_analytics and community-maintained memgraph, with FalkorDB also referenced as a community adapter. The code graph pipeline uses an external extractor called enola, is deterministic, and needs no LLM or embedding provider. It extracts modules, symbols, routes, storage, services and dependency manifests, rolls symbol edges up to module architecture views, and supports query_facts, architecture (Mermaid or Graphviz), insights, explore, traverse, find_path, impact_analysis and delta. The MCP server exposes remember, recall, forget and cognify_status, with code search routed through recall, and registers in Claude Code with claude mcp add cognee-http -t http http://localhost:8000/mcp. Issue #1160 (July 2025) said the code pipeline handled only Python. Current language coverage is UNVERIFIED. Cognee writes its own node types into Neo4j, so coexistence is possible but the schema is Cognee's, not yours.

### Mem0
Graph memory was removed from the open-source SDK. PR #4805 deleted the Neo4j, Memgraph, Kuzu, Apache AGE and Neptune graph stores, and issue #6591 notes AGENTS.md still advertises them. The hosted platform now runs a built-in graph with no external database. Mem0 is no longer a Neo4j building block.

### LightRAG and nano-graphrag
LightRAG is MIT, 40k stars, lightrag-hku 1.5.7 uploaded 2 Jul 2025, supports Neo4j, Memgraph and PostgreSQL graph storage among others. Document-oriented, no code parser, no MCP server in the README. nano-graphrag is MIT, 0.0.8.2 from 19 Oct 2024, has a Neo4j storage class, and looks dormant.

### Microsoft GraphRAG
MIT, 36.2k stars, graphrag 3.2.0 uploaded 23 Sep 2026, yet the README states the project is largely in maintenance mode. Produces Leiden community summaries over Parquet outputs. neo4j-contrib/ms-graphrag-neo4j (MIT, 89 stars) ports the method onto Neo4j with GDS Leiden and labels itself experimental and unoptimized for large graphs. Neo4j's own LLM Knowledge Graph Builder added community summaries in its first 2025 release. None index code.

### HippoRAG
MIT, 2.0.0a4 on PyPI (24 Jun 2025), 2.0.0a5 in the README, built on networkx and igraph with no Neo4j dependency. Ships a hipporag-mcp server for retrieval, QA and indexing. HippoRAG 2 reports a 7% gain on associative memory tasks over the best embedding model. The personalized PageRank idea transfers directly to Neo4j via GDS PageRank with sourceNodes, which is the useful takeaway rather than the library itself.

## Part 5. Text-to-Cypher reliability for agents

Neo4j's Text2Cypher 2024 dataset (7 Nov 2024) has 44,387 instances split 39,554 train and 4,833 test from 16 sources. A 2025v1 dataset exists on Hugging Face under Apache-2.0 in the 10K to 100K bucket. Its card did not render, so size and changes are UNVERIFIED. The fine-tuned Gemma-2-9B-it post (15 Nov 2024) reported gains in Google BLEU and exact match, with fine-tuned GPT-4o and GPT-4o-mini exceeding 30% exact match, and Neo4j called it research rather than production. Ozsoy et al. published the method at GenAIK 2025. Follow-up Neo4j posts in 2025 analyzed model struggles (3 Mar 2025), hard-example selection (29 Apr 2025) and multilingual performance (8 Jul 2025).

The newer agent-focused papers matter more for design:

| Study, date | Headline finding |
|---|---|
| CypherTurn, EMNLP 2026 oral, 29 Sep 2026 | 721 sessions, 5,927 turns, 7 graphs, 15 models. Best single-turn execution accuracy 64.7%. Session-level correctness under 5%. Autonomous mode reorders the leaderboard top ("autonomy divergence"). |
| LAST-CQ, REALM at EMNLP 2026, 11 Sep 2026 | Detecting failure and retrying is what drives recovery. Recovers 91.7% of single-pass failures. Raw database error strings work almost as well as LLM-written feedback. Parallel sampling degrades quality 10 to 11%. |
| PIPE-Cypher, 7 Jun 2026 | Zero-shot transfer to enterprise schemas is weak. Schema-specific example banks help compatible model families. 3,000 examples, 11 models. |
| Cazzaro et al., 12 Jun 2026 | Grounded synthetic data lets small local models compete with large proprietary ones on text-to-Cypher. |

Neo4j's Text2Cypher guide (9 Feb 2026, Alex Gilmore) gives a complexity-by-frequency matrix. Low-complexity, low-frequency questions suit generated Cypher. Anything frequent or complex should be a pre-written tool. Large schemas should be served as retrieved subschemas rather than dumped whole. Validation should combine regex checks, the CyVer library, automatic relationship-direction fixes and a correction loop. The guide names MCP servers as the most flexible production route. Aura Agent operationalizes the same split with Cypher Templates, Similarity Search and Text2Cypher.

Recommended pattern for Claude Code:

1. A small set of curated, parameterized tools for the questions you can predict.
2. Raw read-only Cypher as the exploration fallback, behind a read-only database user and EXPLAIN validation.
3. Schema delivered compactly through a skill file with label conventions and example queries, rather than a large sampled schema on every call.
4. Database errors returned verbatim to the model, with a retry budget of one or two and no parallel sampling.
5. Result sanitization: strip embeddings and long lists, cap rows, enforce a token limit.

## (a) Neo4j-compatible building blocks for a code graph

| Block | Role | Version, date | License | Notes |
|---|---|---|---|---|
| neo4j/mcp | Agent access: schema, read/write Cypher, GDS procedures | 1.6.0, 10 Sep 2026 | GPL-3.0 | Read-only via EXPLAIN, HTTP auth, no token limit |
| mcp-neo4j-cypher | Agent access with token limit and timeout | 0.6.0, 10 Apr 2026 | MIT | Needs APOC, stdio/HTTP/SSE |
| mcp-neo4j-data-modeling | Design and validate the code-graph schema | 0.8.2, 22 Dec 2025 | MIT | Mermaid, Arrows, constraint Cypher |
| gds-agent | Algorithm tools over MCP | PyPI, 97 stars | MIT | Needs GDS or Aura Graph Analytics |
| neo4j-skills | Cypher, modeling and GDS skills for Claude Code | 113 stars | MIT | agentskills.io format |
| neo4j-graphrag | Vector, hybrid, VectorCypher and Text2Cypher retrievers | 1.22.0, 1 Oct 2026 | Apache-2.0 | KG builder is text-only |
| GDS Leiden, PageRank, Node Similarity, FastRP | Communities, hubs, similar modules, embeddings | 2026.09 | Community capped at 4 cores | Leiden needs undirected projection |
| Vector index plus SEARCH clause | Docstring and chunk embeddings | SEARCH since 2026.01 | Core | queryNodes deprecated 2026.04 |
| Property-based privileges, ABAC | Hide sensitive properties from the agent user | ABAC 2026.03 | Enterprise, Aura BC/VDC | DENY fails open |
| jQAssistant plus Git plugin | JVM bytecode, Maven, Git history into Neo4j over bolt | 2.9.x, Jan 2026; Git 2.4.0 | GPL-3.0 | No Python scanner |
| CodeGraphContext, Neo4j backend | Tree-sitter code graph for 23+ languages with MCP | 0.6.13, 6 Sep 2026 | MIT | Default backend is FalkorDB Lite; switch to Neo4j |
| CodexGraph schema | Reference schema for an agent-queried code graph | NAACL 2025 | Research | Six labels, five edge types |
| JohT pipeline Cypher library | Ready-made GDS and code-smell queries | v4.0, May 2026 | GPL-3.0 | CLI only |
| Graphiti on Neo4j | Temporal episodic memory beside the code graph | 0.30.2, 8 Sep 2026 | Apache-2.0 | Own labels, group_id namespacing, LLM on every write |
| Cognee, neo4j provider | Deterministic code-architecture graph | 1.6.x, Sep 2026 | Apache-2.0 | Language coverage unverified |
| DeusData/codebase-memory-mcp | Non-graph-DB baseline to benchmark against | 2026 | MIT | SQLite, 162 languages, 45.6k stars |

## (b) Recommended minimal stack

Keep one Neo4j server. On Enterprise, give the code graph its own database so privileges, backups and schema stay separate. On Community you have one database, so namespace with a Code prefix on labels and a repo property on every node.

1. Structural graph. Build it with a tree-sitter indexer writing straight to Neo4j. CodeGraphContext with the Neo4j backend is the quickest start. Reshape its output toward the CodexGraph label set (Module, Class, Function, Method, Field, GlobalVariable) with CONTAINS, HAS_METHOD, INHERITS, CALLS, IMPORTS and USES edges, plus File nodes. Add jQAssistant over bolt only if you have JVM code. Add the Kontext E Git plugin or a small script for commit history so the agent can ask what changed, modelled as Commit nodes with CHANGED_IN edges.
2. Bridge to the documentation graph. Create explicit edges from Function or Module nodes to existing documentation Chunk and Concept nodes, for example DOCUMENTED_BY and MENTIONS. Populate them with a nightly Cypher job that matches qualified names and with vector similarity between docstring embeddings and chunk embeddings.
3. Analytics. Run GDS Leiden on an undirected CALLS plus IMPORTS projection and PageRank on CALLS, writing community and pagerank properties so the agent can ask for the important symbols in a subsystem without reading files. Store docstring embeddings in a vector index.
4. Agent access. Expose the graph through the official neo4j-mcp in read-only mode under a dedicated read-only database user, or the Labs mcp-neo4j-cypher if you want the token limit. Then write a thin MCP server of your own with six to eight parameterized tools: find_symbol, callers, callees, impact_of_change, module_summary, path_between, docs_for_symbol, recent_changes. Keep raw read-cypher as the fallback.
5. Context. Put the schema, label conventions and example queries in a Claude Code skill rather than relying on the sampled schema tool.
6. Operations. Re-index on a git hook or CI job. Hide anything sensitive with property-based privileges on Enterprise. Benchmark against DeusData/codebase-memory-mcp, the SQLite-based incumbent whose published numbers favour it for raw exploration cost.

Items marked UNVERIFIED: existence of a jQAssistant Python plugin, jQAssistant commercial support and named enterprise users, the Apple acquisition of Kuzu, ArangoDB Community Edition limits, the twelve-tool count for FalkorDB code-graph, the contents of the text2cypher-2025v1 card, Cognee's current code-graph language coverage, and the quality of third-party Neo4j Claude Code plugins.

## (c) URLs used

- https://github.com/neo4j/mcp
- https://github.com/neo4j/mcp/releases
- https://raw.githubusercontent.com/neo4j/mcp/main/LICENSE.txt
- https://neo4j.com/docs/mcp/current/
- https://neo4j.com/docs/mcp/current/installation/
- https://neo4j.com/docs/mcp/current/configuration/
- https://pypi.org/pypi/neo4j-mcp-server/json
- https://github.com/neo4j-contrib/mcp-neo4j
- https://github.com/neo4j-contrib/mcp-neo4j/tree/main/servers/mcp-neo4j-cypher
- https://github.com/neo4j-contrib/mcp-neo4j/releases
- https://pypi.org/pypi/mcp-neo4j-cypher/json
- https://pypi.org/pypi/mcp-neo4j-memory/json
- https://pypi.org/project/mcp-neo4j-memory/
- https://pypi.org/pypi/mcp-neo4j-data-modeling/json
- https://pypi.org/project/mcp-neo4j-data-modeling/
- https://pypi.org/pypi/mcp-neo4j-aura-manager/json
- https://neo4j.com/developer/genai-ecosystem/model-context-protocol-mcp/
- https://neo4j.com/blog/developer/production-proofing-cypher-mcp-server/
- https://neo4j.com/blog/developer/knowledge-graphs-claude-neo4j-mcp/
- https://gaetanopiazzolla.github.io/graph%20database/ai/claude/2026/05/03/neo4j-mcp-claude.html
- https://github.com/neo4j-contrib/gds-agent
- https://neo4j.com/blog/developer/agentic-graph-analytics-external-mcp-providers-neo4j/
- https://github.com/neo4j-contrib/neo4j-skills
- https://neo4j.com/docs/aura/aura-agent/
- https://neo4j.com/docs/neo4j-graphrag-python/current/
- https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html
- https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_kg_builder.html
- https://pypi.org/project/neo4j-graphrag/
- https://neo4j.com/docs/graph-data-science/current/algorithms/
- https://neo4j.com/docs/graph-data-science/current/algorithms/page-rank/
- https://neo4j.com/docs/graph-data-science/current/algorithms/leiden/
- https://neo4j.com/docs/graph-data-science/current/algorithms/node-similarity/
- https://neo4j.com/docs/graph-data-science/current/installation/
- https://neo4j.com/docs/graph-data-science/current/introduction/
- https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/vector-indexes/
- https://neo4j.com/docs/operations-manual/current/authentication-authorization/property-based-access-control/
- https://neo4j.com/docs/operations-manual/current/authentication-authorization/attribute-based-access-control/
- https://neo4j.com/docs/operations-manual/current/database-administration/
- https://github.com/jqassistant
- https://github.com/jqassistant/jqassistant
- https://jqassistant.github.io/jqassistant/current/
- https://github.com/jqassistant-plugin
- https://github.com/jqassistant-plugin/jqassistant-typescript-plugin
- https://github.com/kontext-e/jqassistant-plugins
- https://github.com/kontext-e/jqassistant-git-plugin
- https://github.com/2015xli/jqassistant-graph-rag
- https://mcpmarket.com/server/jqassistant-graph-rag
- https://github.com/JohT/code-graph-analysis-pipeline
- https://raw.githubusercontent.com/JohT/code-graph-analysis-pipeline/main/AGENTS.md
- https://arxiv.org/abs/2408.03910
- https://arxiv.org/html/2408.03910
- https://arxiv.org/abs/2503.09089
- https://arxiv.org/abs/2603.27277
- https://github.com/DeusData/codebase-memory-mcp
- https://github.com/techsavvyash/codegraph
- https://github.com/Abhishek-Aditya-bs/CodeGraph
- https://github.com/Sylphie-Labs/codebase-pkg
- https://github.com/CodeGraphContext/CodeGraphContext
- https://pypi.org/project/codegraphcontext/
- https://pypi.org/pypi/codegraphcontext/json
- https://memgraph.com/docs/ai-ecosystem/mcp
- https://github.com/memgraph/ai-toolkit
- https://github.com/memgraph/memgraph
- https://memgraph.com/docs/release-notes
- https://pypi.org/pypi/mcp-memgraph/json
- https://github.com/FalkorDB/code-graph
- https://github.com/falkordb/falkordb-mcpserver
- https://github.com/FalkorDB/GraphRAG-SDK
- https://github.com/FalkorDB/FalkorDB
- https://github.com/FalkorDB/FalkorDB/releases/tag/v6.0.1
- https://github.com/kuzudb/kuzu
- https://github.com/LadybugDB/ladybug
- https://pypi.org/pypi/nebulagraph-mcp-server/json
- https://github.com/vesoft-inc/nebula
- https://pypi.org/pypi/tigergraph-mcp/json
- https://github.com/tigergraph/tigergraph-mcp
- https://www.tigergraph.com/community-edition/
- https://awslabs.github.io/mcp/servers/amazon-neptune-mcp-server
- https://pypi.org/pypi/awslabs.amazon-neptune-mcp-server/json
- https://github.com/arango-solutions/arango-solutions-mcp
- https://github.com/getzep/graphiti
- https://help.getzep.com/graphiti/getting-started/mcp-server
- https://help.getzep.com/graphiti/core-concepts/graph-namespacing
- https://raw.githubusercontent.com/getzep/graphiti/main/graphiti_core/graph_queries.py
- https://github.com/getzep/graphiti/issues/1022
- https://pypi.org/project/graphiti-core/
- https://blog.getzep.com/graphiti-hits-20k-stars-mcp-server-1-0/
- https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/
- https://github.com/topoteretes/cognee
- https://github.com/topoteretes/cognee/tree/main/cognee-mcp
- https://docs.cognee.ai/guides/code-graph
- https://docs.cognee.ai/setup-configuration/graph-stores
- https://pypi.org/pypi/cognee/json
- https://github.com/mem0ai/mem0/issues/6591
- https://docs.mem0.ai/platform/features/graph-memory
- https://github.com/HKUDS/LightRAG
- https://pypi.org/pypi/lightrag-hku/json
- https://pypi.org/pypi/nano-graphrag/json
- https://github.com/microsoft/graphrag
- https://pypi.org/pypi/graphrag/json
- https://github.com/neo4j-contrib/ms-graphrag-neo4j
- https://github.com/OSU-NLP-Group/HippoRAG
- https://pypi.org/pypi/hipporag/json
- https://neo4j.com/blog/developer/introducing-neo4j-text2cypher-dataset/
- https://neo4j.com/blog/developer/fine-tuned-text2cypher-2024-model/
- https://aclanthology.org/2025.genaik-1.11.pdf
- https://huggingface.co/datasets/neo4j/text2cypher-2025v1
- https://neo4j.com/blog/genai/text2cypher-guide/
- https://arxiv.org/abs/2609.36987
- https://arxiv.org/abs/2609.12746
- https://arxiv.org/abs/2606.08481
- https://arxiv.org/abs/2606.14325
