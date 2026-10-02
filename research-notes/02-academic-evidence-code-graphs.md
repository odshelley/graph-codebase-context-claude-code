# Graph-Based Repository Representations for LLM Coding Agents: A Verified Literature Review

Prepared 2026-10-01. Every claim below comes from a page fetched during this session. Numbers are quoted as they appear on the fetched page, with the arXiv version noted where it matters. Items that could not be confirmed from a fetched page are marked UNVERIFIED.

## Executive summary

Static-analysis code graphs reliably improve code localization, and localization gains carry through to issue resolution, but the measured end-to-end gains over a strong grep-style agent are small (roughly 2 to 5 points on SWE-bench style benchmarks) and often not statistically separable at the sample sizes used. Graphs help most on multi-hop dependency questions, function-level and line-level localization, and multi-file edits. Embedding indexes beat plain agentic search on read-only codebase question answering in two independent 2025-2026 evaluations (Cursor's internal benchmark and an academic SWE-QA study), while Anthropic's own internal result for Claude Code went the other way. No fetched source shows a graph beating a frontier single-loop grep agent on SWE-bench Verified by more than a few points with the model held fixed. The most consistent finding across the literature is that the graph should rank and filter context rather than be dumped into the prompt: every system that fed large subgraphs to the model lost accuracy.

---

## Part 1. Systems and their evidence

### 1.1 CodexGraph

**Citation.** Xiangyan Liu, Bo Lan, Zhiyuan Hu, Yang Liu, Zhicheng Zhang, Fei Wang, Michael Qizhe Shieh, Wenmeng Zhou. "CodexGraph: Bridging Large Language Models and Code Repositories via Code Graph Databases." Proceedings of NAACL 2025, Volume 1: Long Papers, pp. 142-160. DOI 10.18653/v1/2025.naacl-long.7. arXiv:2408.03910 (v1 Aug 2024, v3 Jul 2026).
URLs: https://aclanthology.org/2025.naacl-long.7/ ; https://arxiv.org/abs/2408.03910 ; https://arxiv.org/html/2408.03910

**Method.** Node types are MODULE, CLASS, METHOD, FUNCTION, FIELD, GLOBAL_VARIABLE. Edge types are CONTAINS, INHERITS, HAS_METHOD, HAS_FIELD, USES. The graph is built by static analysis in two phases: a single-pass "shallow indexing" then an edge-completion pass using depth-first search to resolve cross-file relations. It is stored in a graph database and queried in Cypher. The agent follows a "write then translate" protocol: the primary LLM agent writes natural-language queries and a separate translation agent converts them to syntactically valid Cypher, iterating until the retrieved context suffices.

**Evaluation.** CrossCodeEval Lite (Python), SWE-bench Lite, EvoCodeBench, all with GPT-4o, plus five qualitative applications.

**Headline numbers (arXiv v3 HTML, GPT-4o).**

| Benchmark | No RAG | BM25 | AutoCodeRover | CodexGraph |
|---|---|---|---|---|
| CrossCodeEval Lite Python, EM | 10.80 | 21.20 | 21.20 | 27.90 |
| CrossCodeEval Lite Python, ES | 59.36 | 66.18 | 61.92 | 67.98 |
| SWE-bench Lite, Pass@1 | 27.83 | n/a | 28.78 | 36.02 |
| EvoCodeBench, Pass@1 | n/a | n/a | 11.17 | 11.87 |

Token cost per instance (Table 2): on SWE-bench Lite, BM25 14.76k, AutoCodeRover 76.01k, CodexGraph 102.25k; on CrossCodeEval Lite, BM25 1.47k, AutoCodeRover 10.74k, CodexGraph 22.16k; on EvoCodeBench, AutoCodeRover 21.41k, CodexGraph 24.49k. Removing the translation agent cost GPT-4o 19.60 EM points and 11.62 ES points on CrossCodeEval Lite.

**Limitations.** The paper states CodexGraph "sometimes encounters loops where it fails to generate executable graph queries," which drives up token use. Python only. Schema completeness and indexing efficiency are listed as future work.

### 1.2 RepoGraph

**Citation.** Siru Ouyang, Wenhao Yu, Kaixin Ma, Zilin Xiao, Zhihan Zhang, Mengzhao Jia, Jiawei Han, Hongming Zhang, Dong Yu. "RepoGraph: Enhancing AI Software Engineering with Repository-level Code Graph." ICLR 2025. arXiv:2410.14684 (v2 Mar 2025).
URLs: https://arxiv.org/abs/2410.14684 ; https://arxiv.org/html/2410.14684

**Method.** Line-level graph: "each node in V corresponds to a line of code" with line_number, file_name, directory attributes, typed as "def" (definition) or "ref" (reference). Two edge families, invoke and contain. Built with tree-sitter ASTs. A two-step filter drops Python built-ins and third-party library relations. Retrieval returns "the k-hop ego-graphs with the search term in the centric," which are then either flattened or summarized into the prompt. Designed as a plug-in to existing scaffolds.

**Evaluation.** SWE-bench Lite with four base methods (RAG, Agentless, AutoCodeRover, SWE-agent), each with and without RepoGraph; CrossCodeEval Python subset (2,665 samples).

**Headline numbers.**

| Base method | Model | Without | With RepoGraph |
|---|---|---|---|
| RAG | GPT-4 | 2.67 | 5.33 |
| Agentless | GPT-4o | 27.33 | 29.67 |
| AutoCodeRover | GPT-4 | 19.00 | 21.33 |
| SWE-agent | GPT-4o | 18.33 | 20.33 |

The paper claims 29.67% was "state-of-the-art performance on the benchmark among all open-source methods" at the time. CrossCodeEval with GPT-4o: code-match EM 10.5 to 28.7, identifier match 16.8 to 36.0; with Deepseek-Coder, EM 10.2 to 19.7.

Cost (Table 2): Agentless $0.34 and 42,376 tokens versus Agentless+RepoGraph $0.39 and 47,323 tokens; AutoCodeRover $0.45 and 38,663 tokens versus $0.58 and 45,112; SWE-agent $2.53 and 498,346 tokens versus $2.69 and 518,792. Ego-graph ablation on Agentless: 1-hop flattened 29.67%, 2-hop flattened 26.00%, 1-hop summarized 28.33%. Flattened 1-hop context averaged 2310.7 tokens and 2-hop 10505.3 tokens. The average SWE-bench repository graph has "over 1,000 nodes and 25,000 edges."

**Limitations (stated).** "only explored proprietary LLMs, i.e., GPT-4 series"; "Experiments were only conducted on the Lite set due to the high cost"; "only explored Python in our main experiments."

### 1.3 LocAgent

**Citation.** Zhaoling Chen, Robert (Xiangru) Tang, Gangda Deng, Fang Wu, Jialong Wu, Zhiwei Jiang, Viktor Prasanna, Arman Cohan, Xingyao Wang. "LocAgent: Graph-Guided LLM Agents for Code Localization." Proceedings of ACL 2025, Volume 1: Long Papers, pp. 8697-8727. DOI 10.18653/v1/2025.acl-long.426. arXiv:2503.09089.
URLs: https://aclanthology.org/2025.acl-long.426/ ; https://arxiv.org/abs/2503.09089 ; https://arxiv.org/html/2503.09089

**Method.** Directed heterogeneous graph with node types directory, file, class, function and edge types contain, import, invoke, inherit, built by parsing "each Python file using the abstract syntax tree (AST)." A four-level sparse index sits on top: entity ID index, name dictionary, BM25 inverted index, and a content index. Three tools: SearchEntity (keyword search with fold/preview/full detail), TraverseGraph (type-aware BFS with direction, hop, and type filters, output as a tree), RetrieveEntity (full code by ID). Qwen2.5-7B and 32B were fine-tuned on Claude-3.5 trajectories.

**Evaluation.** SWE-bench Lite localization (274 instances), the new Loc-Bench (560 instances collected after Oct 2024: 242 bug, 150 feature, 29 security, 139 performance), and downstream SWE-bench Lite resolution via Agentless repair.

**Headline numbers (SWE-bench Lite, Table 4).**

| Method | Model | File Acc@5 | Module Acc@10 | Function Acc@10 |
|---|---|---|---|---|
| BM25 | none | 61.68 | 52.92 | 36.86 |
| CodeRankEmbed | none | 84.67 | 78.83 | 58.76 |
| Agentless | Claude-3.5 | 79.56 | 68.98 | 58.76 |
| OpenHands | Claude-3.5 | 90.15 | 83.58 | 70.07 |
| LocAgent | Qwen2.5-7B ft | 88.32 | 82.85 | 71.53 |
| LocAgent | Qwen2.5-32B ft | 92.70 | 87.23 | 77.01 |
| LocAgent | Claude-3.5 | 94.16 | 87.59 | 77.37 |

Ablation on the fine-tuned 7B model (Table 6), function Acc@10: full 71.53; without TraverseGraph 66.06; without RetrieveEntity 69.34; without SearchEntity 53.28; without BM25 index 60.22. Cost (Table 5): OpenHands Claude-3.5 13 rounds $0.79; LocAgent Claude-3.5 7 rounds $0.66; LocAgent Qwen2.5-32B ft 9 rounds $0.09, which the paper calls "approximately 86% reduction." Downstream SWE-bench Lite (Table 8): Agentless Claude-3.5 Pass@1 26.31 / Pass@10 33.58; LocAgent Claude-3.5 27.92 / 37.59. Loc-Bench with Claude-3.5: file Acc@10 86.07, module Acc@10 71.07, function Acc@15 60.71. Code Researcher (section 1.14) separately reports LocAgent's graph construction took 764 seconds on sympy.

**Limitations (stated).** Fine-tuning limited to Qwen-2.5-Coder with LoRA; only bug repair assessed downstream; training data "relied heavily on trajectories generated by Claude-3.5"; "evaluation focuses primarily on Python codebases"; only accuracy and NDCG metrics.

### 1.4 CodeRAG / GraphCodeAgent (dual graph)

**Citation.** Jia Li, Xianjie Shi, Kechi Zhang, Ge Li, Zhi Jin, Lei Li, Huangzhao Zhang, Jia Li, Fang Liu, Yuwei Zhang, Zhengwei Tao, Yihong Dong, Yuqi Zhu, Chongyang Tao. "GraphCodeAgent: Dual Graph-Guided LLM Agent for Retrieval-Augmented Repo-Level Code Generation." arXiv:2504.10046 (v1 Apr 2025, v2 Nov 2025). Venue not stated.
URLs: https://arxiv.org/abs/2504.10046 ; https://arxiv.org/html/2504.10046
Note: that the v1 title was "CodeRAG" with a "bigraph" is UNVERIFIED from my fetch; the current version is titled GraphCodeAgent. A different paper named CodeRAG (section 1.9) is not graph-based.

**Method.** Two graphs. The Requirement Graph has function, class, and method requirement nodes with parent-child and semantically-similar edges; requirement text is generated by DeepSeek-V2.5 where docstrings are missing and "Two PhD candidates majoring in computer science" verified it. The Structural-Semantic Code Graph has file, class, function, method nodes with import, contain, inherit, invoke, and semantic-similarity edges (cosine threshold 0.8 with stella_en_400M_v5), built with tree-sitter and stored in Neo4j. Agent tools: RGRetrieval, DualGraphMapping, SSCGTraverse, WebSearch, CodeTesting, in a ReAct loop.

**Evaluation.** DevEval (1,825 examples) and CoderEval, with GPT-4o, Gemini-1.5-Pro, QwQ-32B.

**Headline numbers (Pass@1).** DevEval, GPT-4o: no retrieval 17.24; BM25 27.07; dense RAG 40.43; RepoCoder 30.95; GraphCoder 36.44; CodeAgent 28.66; GraphCodeAgent 58.14. DevEval, Gemini-1.5-Pro: dense 39.34; GraphCodeAgent 54.74. CoderEval, GPT-4o: dense 38.26; GraphCodeAgent 53.91. Ablation (GPT-4o, DevEval): without SSCGTraverse 51.83; without CodeTesting 57.09; without WebSearch 57.85.

**Limitations.** LLM-generated requirements rely on "careful prompt engineering and human verification rather than gold-standard annotation"; the offline graph limits "rapidly evolving repositories"; gains were larger for GPT-4o than Gemini, suggesting model dependence.

### 1.5 GraphCoder

**Citation.** Wei Liu, Ailun Yu, Daoguang Zan, Bo Shen, Wei Zhang, Haiyan Zhao, Zhi Jin, Qianxiang Wang. "GraphCoder: Enhancing Repository-Level Code Completion via Coarse-to-fine Retrieval Based on Code Context Graph." ASE 2024, research track (presented 31 Oct 2024, Sacramento). arXiv:2406.07003 (v2 Sep 2024).
URLs: https://arxiv.org/abs/2406.07003 ; https://arxiv.org/html/2406.07003 ; https://conf.researchr.org/details/ase-2024/ase-2024-research/46/GraphCoder-Enhancing-Repository-Level-Code-Completion-via-Coarse-to-fine-Retrieval-B

**Method.** The code context graph is "a directed multi-graph" whose vertices are statements and whose edges are control-flow, data-dependence, and control-dependence, derived from AST analysis. A slicing algorithm collects control-flow neighbours within h hops and their dependence predecessors. Retrieval is coarse (Jaccard on bag-of-words over the slice) then fine (re-ranking by a "decay-with-distance subgraph edit distance" weighting edits by gamma^hops from the completion point).

**Evaluation.** RepoEval-Updated, 8,000 tasks in Python and Java at line and API level, with GPT-3.5 and other models.

**Headline numbers (Table 2, GPT-3.5).** Python line-level EM: no RAG 33.10, vanilla RAG 37.90, RepoCoder 38.20, GraphCoder 46.60. Java API-level EM: 37.95, 54.10, 56.05, 61.57. Averaged gains "+6.06 in code match and +6.23 in identifier match." Efficiency (Table 4): retrieval 1.08 s versus 4.73 s for vanilla RAG; 23,560 database entries versus 115,109; 102.4 MB versus 159.4 MB.

**Limitations.** Python and Java only; low-duplication repositories suffer "reduced recall rate during retrieval"; vanilla RAG baselines were custom-implemented; GraphCoder underperforms on about 5% of tasks because retrieved snippets lead to literal copying.

### 1.6 RepoHyper

**Citation.** Huy N. Phan, Hoang N. Phan, Tien N. Nguyen, Nghi D. Q. Bui. "RepoHyper: Search-Expand-Refine on Semantic Graphs for Repository-Level Code Completion." arXiv:2403.06095 (v4 Aug 2024). Venue not stated.
URLs: https://arxiv.org/abs/2403.06095 ; https://arxiv.org/html/2403.06095

**Method.** Repo-level Semantic Graph with function/method, class, and script nodes and import, invoke (caller/callee), ownership, encapsulation, and class-hierarchy edges. Built with tree-sitter for entities, PyCG for call edges, Python's ast module for imports. Retrieval: kNN anchor search over embeddings, BFS expansion (depth 4, up to 1,000 nodes, optionally pattern-filtered), then a 3-layer GraphSAGE link predictor.

**Headline numbers.** RepoBench-R Easy: acc@1 25.94 (UniXcoder) to 32.56, acc@3 59.69 to 68.91, "~26% relative increase." RepoBench-P with GPT-3.5-Turbo, cross-file setting: RepoCoder 48.73 EM / 57.62 CodeBLEU; RepoHyper-L2H 52.76 / 61.49.

**Limitations (stated).** Manually selected path types; RepoBench only; heavy compute.

### 1.7 CoCoMIC

**Citation.** Yangruibo Ding, Zijian Wang, Wasi Ahmad, Murali Krishna Ramanathan, Ramesh Nallapati, Parminder Bhatia, Dan Roth, Bing Xiang. "CoCoMIC: Code Completion by Jointly Modeling In-file and Cross-file Context." LREC-COLING 2024, pp. 3433-3445. arXiv:2212.10007.
URLs: https://aclanthology.org/2024.lrec-main.305/ ; https://arxiv.org/abs/2212.10007

**Method and numbers.** CCFinder builds a project-level context graph from import relations to retrieve cross-file entities, which are then jointly encoded with the in-file context. The abstract reports "a 33.94% relative increase in exact match and a 28.69% relative increase in identifier matching for code completion when the cross-file context is provided."

### 1.8 RepoCoder (similarity baseline)

**Citation.** Fengji Zhang, Bei Chen, Yue Zhang, Jacky Keung, Jin Liu, Daoguang Zan, Yi Mao, Jian-Guang Lou, Weizhu Chen. "RepoCoder: Repository-Level Code Completion Through Iterative Retrieval and Generation." EMNLP 2023 main. arXiv:2303.12570.
URL: https://arxiv.org/abs/2303.12570

Iterative similarity retrieval plus generation; introduced RepoEval. Reported to beat the in-file baseline "by over 10% in all settings." It is the standard non-graph baseline in GraphCoder, RepoHyper, and GraphCodeAgent.

### 1.9 RepoFuse, GRACE, CodeRAG (EMNLP 2025)

**RepoFuse.** Ming Liang, Xiaoheng Xie, Gehao Zhang, Xunjin Zheng, Peng Di, Wei Jiang, Hongwei Chen, Chengpeng Wang, Gang Fan. arXiv:2402.14323 (Feb 2024). https://arxiv.org/abs/2402.14323 . Fuses "analogy context" and "rationale context" with rank-truncated generation; reports "a 40.90% to 59.75% increase in exact match (EM) accuracy" on CrossCodeEval and "26.8% enhancement in inference speed." CGM (section 1.12) characterizes RepoFuse as flattening node context into text.

**GRACE.** Xingliang Wang, Baoyi Wang, Chen Zhi, Junxiao Han, Xinkui Zhao, Jianwei Yin, Shuiguang Deng. "GRACE: Graph-Guided Repository-Aware Code Completion through Hierarchical Code Fusion." arXiv:2509.05980 (Sep 2025). https://arxiv.org/abs/2509.05980 . Unifies file structure, AST, call graph, class hierarchy, and data-flow graphs with a GNN structural retriever and a graph-attention re-ranker; reports DeepSeek-V3 gains of "8.19% EM and 7.51% ES points."

**CodeRAG (name collision).** Sheng Zhang, Yifan Ding, Shuquan Lian, Shun Song, Hui Li. EMNLP 2025. arXiv:2509.16112. https://arxiv.org/abs/2509.16112 . Log-probability-guided query construction and multi-path retrieval for completion; not graph-based.

### 1.10 CodePlan

**Citation.** Ramakrishna Bairi, Atharv Sonwane, Aditya Kanade, Vageesh D C, Arun Iyer, Suresh Parthasarathy, Sriram Rajamani, B. Ashok, Shashank Shet. "CodePlan: Repository-level Coding using LLMs and Planning." FSE 2024 per the project README. arXiv:2309.12499. ACM DOI UNVERIFIED (dl.acm.org returned 403).
URLs: https://arxiv.org/abs/2309.12499 ; https://arxiv.org/html/2309.12499 ; https://github.com/microsoft/CodePlan

**Method.** Dependency graph with block-level nodes (imports, methods, classes, fields, statements) and seven relation families: syntactic (ParentOf/ChildOf/Construct), import, inheritance, override, invocation (Calls/CalledBy), instantiation, field use. File-local analysis plus inter-procedural class-hierarchy analysis, via tree-sitter for C# and Jedi for Python. The graph is updated incrementally after each LLM edit. Adaptive planning builds a plan graph of obligations and uses "change may-impact analysis" with escape analysis to discover newly affected blocks.

**Evaluation and numbers.** Package migration (C#) and temporal code edits (Python) across six repositories requiring changes to "between 2-97 files." CodePlan got "5/6 repositories to pass validity checks" versus "0/6" for an oracle-guided baseline using the build system or type checker.

**Limitations (stated).** No data-flow dependencies, no dynamic dispatch, no multi-threading dependencies; C# and Python only; requires an oracle.

### 1.11 Agentless (non-graph baseline)

**Citation.** Chunqiu Steven Xia, Yinlin Deng, Soren Dunn, Lingming Zhang. "Demystifying LLM-based Software Engineering Agents" (arXiv title "Agentless: ..."). FSE 2025 research papers, DOI 10.1145/3715754. arXiv:2407.01489 (v2 Oct 2024).
URLs: https://arxiv.org/abs/2407.01489 ; https://arxiv.org/html/2407.01489 ; https://conf.researchr.org/details/fse-2025/fse-2025-research-papers/85/Demystifying-LLM-based-Software-Engineering-Agents

**Method.** Three fixed phases: localization, repair, patch validation. Localization is hierarchical: a repository tree prompt plus embedding retrieval to choose files, a "skeleton format" of signatures to choose elements, then full code to choose edit lines.

**Headline numbers.** arXiv v2: 96/300 (32.00%) on SWE-bench Lite at $0.70 and 78,166 tokens per instance, versus SWE-agent Claude 3.5 23.00% at $1.62, AutoCodeRover GPT-4 19.00% at $0.45, OpenDevin Claude 3.5 26.67% at $1.14, Moatless GPT-4o 24.67% at $0.14. The FSE page states 32.67%. Localization ablation (ground truth contained in selected files): prompting alone 78.67%; embedding alone 67.67%; embedding with folder filter 70.33%; combined 81.67%. Skeleton format contained ground truth 58.33% versus 53.67% for complete files. SWE-bench Lite-S removes 51 problematic instances (4.3% with patches in the issue, 9.7% with exact solution steps, 5.0% misleading, 10.0% under-specified).

**Limitations (stated).** Possible training-data leakage; Lite only; only 94/213 reproduction tests validated the gold patch; "closed-source agents outperformed on problems with no location clues."

### 1.12 Code Graph Model (CGM)

**Citation.** Hongyuan Tao, Ying Zhang, Zhenhao Tang, Hongen Peng, Xukun Zhu, Bingchang Liu, Yingguang Yang, Ziyin Zhang, Zhaogui Xu, Haipeng Zhang, Linchao Zhu, Rui Wang, Hang Yu, Jianguo Li, Peng Di (Ant Group, ShanghaiTech). "Code Graph Model (CGM): A Graph-Integrated Large Language Model for Repository-Level Software Engineering Tasks." NeurIPS 2025 main track, DOI 10.52202/085713-0537. arXiv:2505.16901 (v4 Jun 2025).
URLs: https://proceedings.neurips.cc/paper_files/paper/2025/hash/178ae4ba29022eb7bf509c2e27bc8ab8-Abstract-Conference.html ; https://arxiv.org/abs/2505.16901 ; https://arxiv.org/html/2505.16901 ; https://arxiv.org/pdf/2505.16901v4

**Method.** Seven node types (REPO, PACKAGE, FILE, TEXTFILE, CLASS, FUNCTION, ATTRIBUTE) and five edge types (contains, calls, imports, extends, implements). Built by AST parsing plus "lightweight semantic analysis to resolve symbols," with class hierarchy analysis and over-approximation for dynamic calls; Python and Java. Node text is encoded by CodeT5+, projected by a two-layer MLP adapter into the LLM input space, and "the graph structure is incorporated into the LLM through the attention mask, allowing direct message passing only between neighboring nodes." The agentless Graph RAG pipeline is Rewriter (extractor and inferer), Retriever (string match plus semantic search, one-hop expansion), Reranker (top-10 by filename, top-5 by skeleton), Reader (CGM).

**Headline numbers.** SWE-bench Lite 43.00% with Qwen2.5-72B, "first among open weight models, second among methods with open-source systems, and eighth overall"; 12.33 points above Moatless+DeepSeek-V3 and 2.33 above Agentless+Claude-3.5-Sonnet. SWE-bench Verified 50.40%. SWE-bench-java Verified 14.29%. CGM-Multi on Lite 36.67%.

**Ablations (Appendix C.7, extracted from the PDF).** Table 10, SWE-bench Lite resolve rate: full CGM 43.00; w/o Rewriter 34.67; w/o Retriever 31.67; w/o Reranker 18.33; w/o all three (entire graph fed to Reader, truncated) 9.67; FlatGraph reader instead of CGM 5.33. Table 11, CrossCodeEval with a 7B backbone: replacing the graph-aware attention mask with a causal mask costs 8.61 EM points on Java and 5.56 on Python; removing subgraph-reconstruction pretraining costs 7.65 EM points; training the adapter alone over a frozen model gains 22.26 (Java) and 21.43 (Python). Appendix C.4.1: graph construction "usually takes 3 minutes or more depending on the complexity of the code repository" and is done offline.

**Limitations (Appendix E and implicit).** Python and Java only, schema untested on other paradigms; Java resolve rate far below Python; file-level recall on SWE-bench-java Verified about 60% versus 87% on Lite; the paper acknowledges closed-source agents still lead (DARS 47% on Lite at the time).

### 1.13 KGCompass

**Citation.** Boyang Yang, Jiadong Ren, Shunfu Jin, Yang Liu, Feng Liu, Bach Le, Haoye Tian. "KGCompass: Knowledge Graph Enhanced Repository-Level Software Repair." arXiv:2503.21710 (v2 Sep 2025). Venue not stated.
URL: https://arxiv.org/html/2503.21710v2

**Method.** A repository-aware knowledge graph with issue and pull-request nodes alongside file, class, and function nodes. Edges: containment, direct calls, imports and attribute references, and issue-to-code links mined with a "template-based regular expression extractor." Code structure from "static Abstract Syntax Tree (AST) analysis." A temporal filter keeps only artifacts predating each instance (removing 24.0% of mined artifacts). Candidate functions are cut from thousands to the top 20, and each candidate is passed to the repair prompt with an explicit "relationship_path" chain.

**Headline numbers (SWE-bench Lite).** With KGCompass: Claude-4 Sonnet 175/300 (58.3%), file acc 83.6%, function acc 56.0%, $0.20 per repair; Claude-3.5 Sonnet 137 (46.0%); DeepSeek-V3 110 (36.7%); Qwen2.5 Max 100 (33.3%). Pure-LLM baselines: Claude-4 116 (38.7%), Claude-3.5 106 (35.3%), DeepSeek-V3 51 (17.0%), Qwen2.5 Max 39 (13.0%). Relative gains 50.8%, 30.2%, 115.7%, 156.4%. Localization ablation (Table 5): KG-only 68.7% file / 40.4% function; LLM-only 81.0 / 53.1; hybrid 84.3 / 57.7. Entity-path ablation at temperature 0 (Table 6): Claude-4 resolved 91 without KG, 131 with KG but no paths, 142 with paths; Claude-3.5: 85, 102, 108. The paper reports "89.7% of successful fault locations require 2 or more hops."

**Limitations.** Python only; function-level accuracy well below file-level; about 17% of targets fall outside the 20 candidates; reproduction tests generated for only 67.7% of instances.

### 1.14 Code Researcher (non-graph, Microsoft)

**Citation.** Ramneet Singh, Sathvik Joel, Abhav Mehrotra, Nalin Wadhwa, Ramakrishna B Bairi, Aditya Kanade, Nagarajan Natarajan. "Code Researcher: Deep Research Agent for Large Systems Code and Commit History." arXiv:2506.11060 (v1 May 2025, v2 May 2026). Venue UNVERIFIED.
URLs: https://arxiv.org/abs/2506.11060 ; https://arxiv.org/html/2506.11060

**Method.** Not a graph. Tools are search_definition (ctags index), search_code (git grep -E), search_commits (git log -G and --grep), close_definition, done, with a memory-filtering step before synthesis. The paper reports the ctags index took 0.76 seconds on sympy versus 764 seconds for LocAgent's graph.

**Evaluation and numbers.** kBenchSyz, 200 validated Linux kernel crashes (75K files, 28M lines). Crash-resolution rate at Pass@5 with GPT-4o: SWE-agent 31.5, Agentless 31.0, Code Researcher 48.0; GPT-4o+o1 58.0; Gemini 2.5 Flash 67.0; Pass@10 with GPT-4o 54.0. Code Researcher explored 10.0 files per trajectory versus 1.33 for SWE-agent. Memory shrank from 21,557 to 7,797 tokens after filtering. Removing search_commits dropped CRR from 48% to 38%.

**Limitations (stated).** Crash resolution only; 50K token context cap; only 10 FFmpeg crashes; depends on reproducers and stack traces.

### 1.15 CoSIL

**Citation.** Zhonghao Jiang, Xiaoxue Ren, Meng Yan, Wei Jiang, Yong Li, Zhongxin Liu. "Issue Localization via LLM-Driven Iterative Code Graph Searching" (CoSIL). ASE 2025. arXiv:2503.22424 (v3 Oct 2025).
URLs: https://arxiv.org/abs/2503.22424 ; https://arxiv.org/html/2503.22424

**Method.** No pre-built index. The LLM constructs module call graphs and then function call graphs dynamically in textual form from imports and code, with a pruner to cut irrelevant directions and a reflection mechanism to fix formatting in long contexts.

**Headline numbers (Qwen2.5-Coder-32B).** SWE-bench Verified function Top-1: CoSIL 0.446, Agentless-FL 0.326, OrcaLoca 0.254, LocAgent 0.250; file Top-1: 0.640, 0.624, 0.612, 0.686. SWE-bench Lite function Top-1: 0.433 versus 0.247, 0.217, 0.103. Plugged into Agentless: Verified 27.40% to 30.00%, Lite 17.33% to 21.67%. Cost with GPT-4o per instance: CoSIL $0.341 and 0.14M tokens; Agentless-FL $0.026 and 10.6k tokens; LocAgent $4.004 and 2.07M tokens.

**Limitations (stated).** Iteration-count sensitivity; Python only; leakage risk; file/function match does not always track resolution.

### 1.16 RANGER

**Citation.** Pratik Shah, Rajat Ghosh, Aryan Singhal, Debojyoti Dutta. "RANGER: Repository-Level Agent for Graph-Enhanced Retrieval." arXiv:2509.25257 (Sep 2025). Venue not stated.
URLs: https://arxiv.org/abs/2509.25257 ; https://arxiv.org/html/2509.25257

**Method.** Neo4j graph with Module, Class, Function, Method, Field, GlobalVariable nodes and CONTAINS, HAS_METHOD, INHERITS, USES edges down to variable level. Each node carries LLM-generated descriptions and an embedding. Entity queries are translated to Cypher; natural-language queries use MCTS traversal with a bi-encoder for expansion and a cross-encoder for reward.

**Headline numbers.** CodeSearchNet NDCG@10 / Recall@10: RANGER 0.786 / 0.911 versus Qwen-3-8B embeddings 0.725 / 0.891. RepoQA: 0.741 / 0.890 versus 0.722 / 0.850. RepoBench Acc@5: 0.5471 versus 0.4940. CrossCodeEval EM with DeepSeek-Coder-7B: RANGER+BM25 36.27, BM25 28.57, CGM-Multi 33.88, RepoFuse 27.92.

**Limitations (stated).** "static offline repository graphs limit applicability to dynamic or rapidly evolving codebases"; MCTS latency "unsuitable for interactive developer workflows"; Python only; no construction-time numbers.

### 1.17 RepoNavigator ("One Tool Is Enough")

**Citation.** Zhaoxi Zhang, Yitong Duan, Yanzhi Zhang, Yiming Xu, Zhixiang Wang, Kun Liang, Weikang Li, Jiahui Liang, Deguo Xia, Jizhou Huang, Jiyan He, Yunfang Wu. arXiv:2512.20957 (v6 May 2026). Venue not stated.
URLs: https://arxiv.org/abs/2512.20957 ; https://arxiv.org/html/2512.20957

**Method.** One tool, jump-to-definition, implemented with Pyright (AST, LEGB scope analysis, MRO-based attribute resolution, import graph). Trained with GRPO from a base model without distillation.

**Headline numbers (SWE-bench Verified, function-level IoU).** RepoNavigator+GRPO: 7B 26.43%, 14B 26.84%, 32B 32.30%; RepoSearcher 32B 18.23%. Training-free 14B baselines: Agentless 12.28, LocAgent 12.32, CoSIL 12.12, RepoSearcher 10.60. Adding GetClass, GetFunc, GetStruc to jump dropped IoU from 24.28% to 13.71% (Table 4). Downstream resolution 15.03% versus 13.01% for LocAgent.

**Limitations (stated).** "monkey patches ... and dynamic imports can degrade the performance of the language server"; Python only; entry points are provided to the agent, which "deviate from the official SWE-bench standard."

### 1.18 ARISE

**Citation.** Shahd Seddik, Fahd Seddik, Amirrezza Esmaeili, Mahdieh Sadatbenis, Fatemeh Fard. "ARISE: A Repository-level Graph Representation and Toolset for Agentic Program Repair and Fault Localization." arXiv:2605.03117 (v2 Jul 2026).
URL: https://arxiv.org/abs/2605.03117

**Method and numbers.** Multi-granularity graph extended to statement nodes with intra-procedural definition-use edges; a data-flow slicing tool exposed through a three-tier API mountable on any tool-use scaffold. On SWE-bench Lite with Qwen2.5-Coder-32B-Instruct, ARISE on SWE-agent resolved "22.0% of issues (66/300), a 4.7 percentage-point gain over the unmodified SWE-agent baseline"; function Recall@1 "rising from 0.43 to 0.60" and line Recall@1 "from 0.26 to 0.41." Ablations attribute the gain "to the data-flow graph rather than the tool schema."

### 1.19 RPG-Encoder

**Citation.** Jane Luo, Chengyu Yin, Xin Zhang, Qingtao Li, Steven Liu, Yiming Huang, Jie Wu, Hao Liu, Yangyu Huang, Yu Kang, Fangkai Yang, Ying Xin, Scarlett Li. "Closing the Loop: Universal Repository Representation with RPG-Encoder." arXiv:2602.02084 (Feb 2026).
URL: https://arxiv.org/abs/2602.02084

Encodes code into a Repository Planning Graph combining "lifted semantic features with code dependencies," evolves it incrementally "reducing overhead by 95.7%," and reports "93.7% Acc@5" localization on SWE-bench Verified and more than 10 points over the best baseline on SWE-bench Live Lite. Detailed tables not fetched.

### 1.20 LARGER

**Citation.** Yuntong Hu, Tongli Su, Liang Zhao, Bowen Zhu, Hasibul Haque. "LARGER: Lexically Anchored Repository Graph Exploration and Retrieval." arXiv:2605.16352 (May 2026). Venue not stated.
URLs: https://arxiv.org/abs/2605.16352 ; https://arxiv.org/html/2605.16352

**Method.** Heterogeneous directed graph of directories, files, classes, functions with contains, imports, invokes, and code-to-test/doc edges, built by "multi-language AST-based construction" with 0-1 edge confidence and Leiden communities. Rather than separate graph tools, lexical matches from the agent's own grep are anchored to graph nodes and confidence-filtered K-hop neighbourhoods are shown alongside the grep results.

**Evaluation.** LocBench (560 issues), MuLocBench (1,100 multi-file issues, 46 repos), SWE-Atlas Codebase QA and Test Writing. Baselines: BM25, Agentless, SWE-agent, OpenHands, Codex, Claude Code, mini-swe-agent, CoSIL, LocAgent. "All in-house LLM-based methods use GPT-5.2" except Claude Code on Claude-Opus-4.6, flagged as "not directly comparable in absolute terms."

**Headline numbers.**

| Method | LocBench Acc@5 | LocBench Recall@5 | MuLocBench Acc@5 | MuLocBench Recall@5 |
|---|---|---|---|---|
| Claude Code (Opus 4.6) | 75.2 | 85.6 | 54.9 | 69.0 |
| Codex (GPT-5.2) | 74.1 | 82.8 | 50.0 | 65.1 |
| LARGER fixed (GPT-5.2) | 87.0 | 90.1 | 55.7 | 68.6 |
| LARGER tuned (GPT-5.2) | 89.1 | 92.3 | 60.0 | 72.0 |

SWE-Atlas: Codebase QA 29.83 (Codex) to 32.25; Test Writing 32.22 to 37.78. Cost per instance on MuLocBench: LARGER 353K tokens, 13.0 calls, about $0.44, 99.9 s mean; Codex 521.8K tokens, about $0.98, 139.9 s.

**Limitations.** No explicit limitations section. Gains on the multi-file benchmark are much smaller than on the single-file one; two hyperparameter regimes are reported; the Claude Code row uses a different model.

### 1.21 RepoAtlas

**Citation.** Yunxiang Zhang, Haiquan Wang, JiaWei Guo, Hanyang Xia, Yan Chen, Tong Chen, Zhang Zhiwei, Junchen Ye. "RepoAtlas: Guiding Coding Agents via Evolving Multimodal Repository Views." arXiv:2609.16936 (Sep 2026).
URLs: https://arxiv.org/abs/2609.16936 ; https://arxiv.org/html/2609.16936

**Method.** Typed directed code graph (modules, classes, callables, variables; containment, inheritance, calls, imports, overrides, symbol use) built with tree-sitter and PyCG. A select-project-refresh loop picks a task-relevant region under a fixed budget, renders it as both image and text, and refreshes when exploration state changes. Training-free.

**Headline numbers (SWE-bench Verified).**

| Model | mini-SWE-agent (no graph) | LocAgent | SeeRepo | RepoAtlas |
|---|---|---|---|---|
| Qwen3.6-35B-A3B resolve % | 60.8 | 61.5 | 60.8 | 63.1 |
| Qwen3.6-35B-A3B input tokens | 1010.6K | 1570.6K | 987.3K | 909.3K |
| MiMo-V2.5 resolve % | 65.4 | 66.1 | 66.9 | 68.0 |
| Kimi-K2.5 resolve % | 68.0 | 69.4 | 68.6 | 72.4 |
| Kimi-K2.5 input tokens | 693.0K | 984.7K | 660.1K | 630.3K |

The abstract summarizes this as "+2.4 points while reducing input tokens and model calls by 5.8% and 7.8%" relative to the strongest multimodal graph baseline.

**Limitations (stated).** "the statically constructed graph cannot capture dependencies established only at runtime," illustrated by sympy-17630.

### 1.22 Code Isn't Memory (structural index ablation)

**Citation.** Ishaan Bhola, Adithyan Krishnan, Sravanth Kurmala, Mukunda NS. "Code Isn't Memory: A Structural Codebase Index Inside a Coding Agent." arXiv:2606.22417 (Jun 2026). The authors appear affiliated with the SuperCoder harness under test.
URLs: https://arxiv.org/abs/2606.22417 ; https://arxiv.org/html/2606.22417

**Method.** The index combines chunk embeddings, a tree-sitter call-graph index of definitions, references, and call edges, and BM25 over identifiers, exposed as codebase_search and codebase_graph tools, with Merkle-diff incremental updates. Three arms: harness with index (SC-ON), same harness without (SC-OFF), and OpenCode as an agentic-grep comparator. Claude Opus 4.7 fixed; 91 instances from SWE-PolyBench Verified and SWE-bench Pro (34 Go, 20 Java, 37 Python); three seeds; leak-audited sandbox.

**Headline numbers.**

| Metric | SC-ON | SC-OFF | OpenCode | Wilcoxon p |
|---|---|---|---|---|
| Resolve % | 50.4 | 41.9 | 45.3 | 0.003 (ON vs OFF); 0.087 (ON vs OpenCode) |
| Localization acc@5 | 84.5 | 44.3 | 75.3 | <0.0001; 0.080 |
| Recall@5 | 0.611 | 0.330 | 0.601 | <0.0001 |
| $ per solved | 2.30 | 2.84 | 2.92 | |
| $ per cell | 1.15 | 1.19 | 1.32 | 0.73 (null) |
| Turns | 28.3 | 36.2 | 36.0 | <0.0001 |
| Tokens (k) | 10.1 | 11.1 | 14.0 | 0.027 |

**Limitations (stated).** Effective n of 75 to 80 so "resolve-level paired tests are under-powered"; JavaScript and TypeScript not run; all five provider-truncation exclusions fell on the index arms; issue-text realism gap; single model.

### 1.23 Aider repo map

**Sources.** https://aider.chat/docs/repomap.html ; https://aider.chat/2023/10/22/repomap.html ; https://raw.githubusercontent.com/Aider-AI/aider/main/aider/repomap.py

**Design.** Tree-sitter extracts definitions and references. The docs state the map is optimized "using a graph ranking algorithm, computed on a graph where each source file is a node and edges connect files which have dependencies," within a budget set by the map-tokens switch "which defaults to 1k tokens," and includes "only the most important identifiers, the ones which are most often referenced by other portions of the code." The source confirms NetworkX PageRank (nx.pagerank with weight="weight" and personalization and dangling set to chat files), edges from referencing file to defining file with weights multiplied by 50 for chat-file references, square-rooted reference counts, a 10x boost for long camelCase identifiers and 0.1x for underscore-prefixed names, and a binary search over tag count until the rendered map lands within 15% of the token budget.

**Evidence.** Neither page gives benchmark numbers; claims are qualitative ("The LLM can see classes, methods and function signatures from everywhere in the repo"). Agent Retrieval Bench (section 2.4) evaluates a "RepoMap" vectorless baseline that dominates trace-to-code retrieval.

### 1.24 Code property graphs and LLMs

**Yamaguchi, Golde, Arp, Rieck.** "Modeling and Discovering Vulnerabilities with Code Property Graphs." IEEE Symposium on Security and Privacy 2014, pp. 590-604, DOI 10.1109/SP.2014.44. Record fetched from https://researchr.org/publication/YamaguchiGAR14 (dblp and IEEE Xplore returned access-denied pages). Merges AST, CFG, and PDG into one graph queried by traversals; the "18 previously unknown vulnerabilities" figure comes from a search summary and is UNVERIFIED against a fetched page.

**LLMxCPG.** Ahmed Lekssays, Hamza Mouhcine, Khang Tran, Ting Yu, Issa Khalil. "LLMxCPG: Context-Aware Vulnerability Detection Through Code Property Graph-Guided Large Language Models." arXiv:2507.16585, "USENIX 2025 (accepted)" per the arXiv page. https://arxiv.org/abs/2507.16585 . CPG-based slices "reduce code size by 67.84 to 90.93% while preserving vulnerability-relevant context" and give "15-40% improvements in F1-score." Vulnerability detection, not agentic coding.

**CodeGRAG.** Kounianhua Du et al. arXiv:2405.02355 (v4 May 2025). https://arxiv.org/abs/2405.02355 ; https://arxiv.org/html/2405.02355 . Control- and data-flow "meta-graph" prompts for function-level generation: HumanEval-X Python Pass@1 77.44 versus 72.56 with GPT-3.5. Not repository-level.

### 1.25 Other graph systems and surveys

**Athale and Vaddina.** "Knowledge Graph Based Repository-Level Code Generation." arXiv:2505.14394 (May 2025); venue not stated on the arXiv page (an ICSE 2025 LLM4Code workshop talk of the same title appeared in search results, UNVERIFIED). https://arxiv.org/abs/2505.14394 ; https://arxiv.org/html/2505.14394 . Nodes File, Class, Method, Function, Attribute, Generated Description; AST structure plus LLM descriptions in Neo4j with full-text and vector indexes; hybrid retrieval with n-hop expansion. EvoCodeBench pass@1: 36.36 (Claude 3.5 Sonnet), 33.45 (GPT-4o), versus CodexGraph GPT-4o 36.02 and no-context GPT-4 7.27.

**Codebase-Memory.** Martin Vogel, Falk Meyer-Eschenbach, Severin Kohler, Elias Grünewald, Felix Balzer. arXiv:2603.27277 (Mar 2026). https://arxiv.org/abs/2603.27277 . Tree-sitter knowledge graph over MCP for 66 languages; "83% answer quality versus 92% for a file-exploration agent, at ten times fewer tokens and 2.1 times fewer tool calls."

**GREPO.** Juntong Wang et al. arXiv:2602.13921 (Feb 2026). https://arxiv.org/abs/2602.13921 . 86 Python repositories, 47,294 bug-fixing tasks as GNN-ready graphs; GNNs beat IR baselines.

**RepoMem.** Boshi Wang, Weijian Xu, Yunsheng Li, Mei Gao, Yujia Xie, Huan Sun, Dongdong Chen. "Improving Code Localization with Repository Memory." ICLR 2026. arXiv:2510.01003. https://arxiv.org/abs/2510.01003 ; https://arxiv.org/html/2510.01003 . Adds commit-history and file-summary memory tools to LocAgent; SWE-bench Verified Acc@5 with GPT-4o 71.6 to 76.5; SWE-bench Live 63.1 to 66.2. Memory reduced reliance on TraverseGraph; irrelevant memory "can pollute reasoning context."

**Survey.** Yicheng Tao, Yuante Li, Yao Qin, Yepang Liu. "Retrieval-Augmented Code Generation: A Survey with Focus on Repository-Level Approaches." arXiv:2510.04905 (v3 May 2026). https://arxiv.org/abs/2510.04905 . Organizes methods by retrieval substrate, control regime, and evaluation setting, explicitly separating "graph-based and non-graph-based retrieval paradigms" and "autonomous agent architectures."

**SWE-agent.** John Yang, Carlos E. Jimenez, Alexander Wettig, Kilian Lieret, Shunyu Yao, Karthik Narasimhan, Ofir Press. arXiv:2405.15793. https://arxiv.org/abs/2405.15793 ; https://arxiv.org/html/2405.15793 . Relevant for plain-search design: SWE-bench Lite ablation shows summarized search 18.0 versus iterative 12.0 and no search 15.7; a 100-line file viewer 18.0 versus 30 lines 14.3 and full file 12.7; linting 18.0 versus 15.0 without.

**OpenHands.** Xingyao Wang and 23 coauthors. ICLR 2025. arXiv:2407.16741. https://arxiv.org/abs/2407.16741 . Platform paper; appears as a baseline in LocAgent at 90.15% file Acc@5 with 13 rounds and $0.79 per instance.

---

## Part 2. Benchmarks and retrieval studies

### 2.1 SWE-bench retrieval baselines

Carlos E. Jimenez et al. "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" ICLR 2024. arXiv:2310.06770. https://arxiv.org/abs/2310.06770 ; https://arxiv.org/html/2310.06770 . BM25 recall of oracle files was 29.58% at 13k tokens, 44.41% at 27k, 51.06% at 50k, yet Claude 2 resolved 1.96% at 13k and 1.22% at 50k, and 4.8% with oracle files. "in almost half of the instances ... it retrieves none of the oracle files." More context hurt even as recall rose.

### 2.2 CodeRAG-Bench

Zora Zhiruo Wang, Akari Asai, Xinyan Velocity Yu, Frank F. Xu, Yiqing Xie, Graham Neubig, Daniel Fried. Findings of NAACL 2025, pp. 3199-3214, DOI 10.18653/v1/2025.findings-naacl.176. arXiv:2406.14497. https://aclanthology.org/2025.findings-naacl.176/ ; https://arxiv.org/html/2406.14497 . SWE-bench Lite NDCG@10: BM25 43.0, GIST-large 47.8, Voyage-code 29.1, OpenAI-03 43.3, SFR-Mistral 62.7. GPT-4o on SWE-bench Lite: 2.3% without retrieval, 21.7% best RAG with rerank, 30.7% with gold documents. Five documents was optimal; more added noise. Python only, vanilla pipelines.

### 2.3 CORE-Bench

Fuwei Zhang et al. EMNLP 2026 main. arXiv:2606.11864. https://arxiv.org/abs/2606.11864 . Over 180K queries and 106K broader-context labels built from code search tasks and SWE-bench instances; "a sharp drop from traditional code search to code retrieval in agentic coding settings"; supervised fine-tuning of embedders helps substantially.

### 2.4 Agent Retrieval Bench

Bowen Qin, Yi Xie. arXiv:2607.24882 (Jul 2026). https://arxiv.org/html/2607.24882v1 . 427 samples over 25 repositories, file-level, four task types (code2test, comment2context, trace2code, edit2ripple) plus abstention cases. Embedders (Qwen3-Embedding-4B/8B, Jina, pplx, nomic) versus BM25, path-aware TF-IDF, and a vectorless RepoMap using "path, symbol, reference, and repository graph signals." Winners vary by task; RepoMap dominates trace2code (0.2742 MRR, 0.8366 Recall@20) where Qwen3-4B gets 0.0827 MRR. RRF fusion of Qwen3-8B and RepoMap gave 0.2713 MRR, "+18% over best single." Logged agent trajectories "never touch any gold file on 35.2% of OpenAI samples and 27.2-29.3% of Codex samples." Does not measure end-to-end resolution.

### 2.5 CodeGrep

Wuya Chen, Yihao Yang, Yang Cao, Yue Lin. arXiv:2608.05886 (Aug 2026). https://arxiv.org/abs/2608.05886 . "a 30B OpenHands agent averages 23 rounds and 631K tokens per resolved issue." A 14B RL-trained grep/glob/read retriever feeding a frozen agent gave 27.0% versus 25.8% on SWE-bench Verified with 15% fewer rounds and 19% fewer tokens. Precision threshold: "BM25 with precision 0.375 degrades the agent, Jina with precision 0.445 is neutral, and CodeGrep with precision 0.677 crosses the threshold."

### 2.6 The Missing Complement

Zhexi Feng, Ruiyi Zhang, Yongbo Yang, Pengtao Xie. arXiv:2609.20050 (Sep 2026). https://arxiv.org/abs/2609.20050 . Reframes retrieval as recovering a jointly sufficient evidence set given agent state; SERBench of 500 states; 73.0% complete sets at five items versus 61.4% for Qwen3 embedding with reranking.

### 2.7 QA and long-context evaluations

SWE-QA: Weihan Peng et al., ACL 2026 Findings, arXiv:2509.14635, https://arxiv.org/abs/2509.14635 . 576 QA pairs from 11 repositories covering cross-file and multi-hop questions.
RepoQA: Jiawei Liu et al., arXiv:2406.06025, https://arxiv.org/abs/2406.06025 . 500 needle-function search tasks, 50 repos, 5 languages, 26 LLMs.
LongCodeBench: Stefano Rando et al., arXiv:2505.07897, https://arxiv.org/abs/2505.07897 . Drops "from 29% to 3% for Claude 3.5 Sonnet, or from 70.2% to 40% for Qwen2.5" as context grows.
DeepRepoQA: Weihan Peng et al., arXiv:2608.24221, https://arxiv.org/abs/2608.24221 . MCTS exploration on SWE-QA.
ReCUBE: Jiseung Hong, Benjamin G. Ascoli, Jinho D. Choi, arXiv:2603.25770, https://arxiv.org/abs/2603.25770 . GPT-5 reaches "only 37.57% strict pass rate in the full-context setting"; a caller-centric dependency-graph toolkit improves agents "up to 7.56%."
SWE Context Bench: Jiayuan Zhu et al., arXiv:2602.08316, https://arxiv.org/abs/2602.08316 . "unfiltered or incorrectly selected context provides limited or negative benefits."
LoCoEval: Yang Liu et al., arXiv:2603.06358, https://arxiv.org/abs/2603.06358 . Long-horizon conversational context management for repositories.

---

## Part 3. Industry positions and evaluations

### 3.1 Anthropic and Claude Code

Latent Space podcast, 7 May 2025, https://www.latent.space/p/claude-code . Boris Cherny: "very early versions of Claude actually used RAG. So we, like, indexed the code base, and I think we were just using Voyage ... Eventually, we landed on just agentic search as the way to do stuff. ... one is it outperformed everything. By a lot." He cites "this whole, like, indexing step ... the code drifts out of sync. And then there's security issues because this index has to live somewhere."

Pragmatic Engineer, 4 March 2026, https://newsletter.pragmaticengineer.com/p/building-claude-code-with-boris-cherny . "Claude Code's 'agentic search' is really just glob and grep, and it outperformed RAG." Alternatives tried included local vector databases and recursive model-based indexing; drawbacks named were stale indexes and permission complexity.

Claude Agent SDK post, 29 Sep 2025, https://claude.com/blog/building-agents-with-the-claude-agent-sdk . "Semantic search is usually faster than agentic search, but less accurate, more difficult to maintain, and less transparent." Recommends starting with agentic search.

Context engineering post, 29 Sep 2025, https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents . Defines context rot ("as the number of tokens in the context window increases, the model's ability to accurately recall information from that context decreases"), the "attention budget," "just in time" retrieval via "lightweight identifiers (file paths, stored queries, web links, etc.)," and compaction as summarizing and reinitiating the context window. No numbers are published in any of these.

### 3.2 Cursor

"Improving agent with semantic search," Stefan Heule, Emily Jia, Naman Jain, 6 Nov 2025, https://cursor.com/blog/semsearch . Internal Cursor Context Bench "focused on retrieving information in codebases with known correct answers." Against agents relying "solely on traditional search tools like grep," semantic search gave "on average 12.5% higher accuracy in answering questions (6.5%-23.5% depending on the model)." "agent code retention increases by 0.3% when semantic search is available. This effect increases to 2.6% on large codebases with 1,000 files or more." The embedder is trained to match LLM rankings of what would have helped in logged agent sessions.

### 3.3 Augment Code

"Auggie tops SWE-Bench Pro," https://www.augmentcode.com/blog/auggie-tops-swe-bench-pro . "Auggie scored 51.80%"; "solved 15 more problems than Cursor and 17 more than Claude Code out of 731 total," all on Claude Opus 4.5 in an "identical harness" run by Augment. Scale AI's own leaderboard baseline for Opus 4.5 with SWE-Agent was 45.89% (January 2026). Attributed to a Context Engine that "builds a semantic index of the full codebase." Vendor-run evaluation.

### 3.4 Sourcegraph Deep Search

https://sourcegraph.com/blog/deep-search (Matt Tanner, 1 May 2026) and https://sourcegraph.com/changelog/deep-search-evaluator (27 Apr 2026). Agentic loop over Code Search, Code Navigation (go-to-definition, find-references), git history, and file reads. Neither page reports accuracy numbers or comparisons; the only figure is anecdotal support-investigation time falling from "45 to 90 minutes" to "roughly five minutes."

### 3.5 Milvus / Zilliz

https://milvus.io/blog/why-im-against-claude-codes-grep-only-retrieval-it-just-burns-too-many-tokens.md (Cheney Zhang, 25 Aug 2025). Claims "Token usage dropped by over 40%, without any loss in recall" for a Claude Context MCP plugin versus grep. Vendor opinion piece with no methodology, repo sizes, or token counts.

### 3.6 Academic agentic-search versus index study

Rafiei Oskooei et al. (Yildiz Technical University; Intellica). "Deep Agentic Search for Repository-Level Code Question Answering: An Empirical Study." arXiv:2608.01507 (Aug 2026). https://arxiv.org/html/2608.01507v1 . On SWE-QA (720 questions, 15 Python repos) with Gemini 2.5 Flash, Gemini 2.5 Pro, Gemini 3 Flash, Qwen3-235B: a vector-index ReAct agent answered 65.2% pooled versus 46.2% for a planner-plus-subagent grep/find/glob "deep agentic" arm; per model, 48.4 vs 42.8, 54.2 vs 44.2, 89.3 vs 39.7, 68.8 vs 58.2. Cost per correct answer $0.32 versus $0.74. Qwen3-235B consumed about 34k versus about 761k input tokens per question. 41.8% of deep-agentic failures were silent planner-subagent hand-off breakdowns; non-terminating loops 13.5% versus 2.9%. Caveat: the agentic arm is a hierarchical two-agent design, not a single-loop agent like Claude Code; fixed snapshots so index staleness was not measured; Python only.

### 3.7 Context-window and compaction studies

Chroma, "Context Rot," Kelly Hong, Anton Troynikov, Jeff Huber, 14 Jul 2025, https://www.trychroma.com/research/context-rot . 18 models; "performance grows increasingly unreliable as input length grows."
Lost in the Middle: Nelson F. Liu et al., TACL 2023, arXiv:2307.03172, https://arxiv.org/abs/2307.03172 .
Sense and Sensitivity: Adam Štorek et al., ACL 2026, arXiv:2505.13353, https://arxiv.org/abs/2505.13353 . Semantic recall on code shows "median accuracy drops of 92.73%" as the snippet moves to the middle of the context.
The Limits of Long-Context Reasoning in Automated Bug Fixing: Ravi Raju et al., ICLR 2026 ICBINB workshop, arXiv:2602.16069, https://arxiv.org/abs/2602.16069 . Successful mini-SWE-agent trajectories "typically remain under 20k-30k tokens"; single-shot repair at 64k with perfect recall gives 7% for Qwen3-Coder-30B-A3B and 0% for GPT-5-nano.
The Complexity Trap: Tobias Lindenbauer et al., NeurIPS 2025 DL4C workshop, arXiv:2508.21433, https://arxiv.org/abs/2508.21433 . Observation masking "halves cost relative to the raw agent while matching, and sometimes slightly exceeding, the solve rate of LLM summarization"; a hybrid saves a further 7% and 11%.
What Context Does a Coding Agent Actually Need to Act?: Brian Sam-Bodden, arXiv:2607.09691, https://arxiv.org/abs/2607.09691 . With localization fixed by an oracle on SWE-bench Verified, UML skeletons of a file's remainder resolved no more issues than deleting it (N=70, McNemar p=0.75); compressed context matched whole files at "19K context tokens, not 94K"; "temperature-0 API inference flips ~9% of per-instance outcomes between byte-identical runs."

### 3.8 Leaderboard analyses

Matias Martinez, Xavier Franch. "Dissecting the SWE-Bench Leaderboards." arXiv:2506.17208 (v2 Aug 2025). https://arxiv.org/html/2506.17208v2 . 79 Lite and 99 Verified entries, 80 unique approaches; "no single architecture consistently achieves state-of-the-art performance." No count of graph-based entries. Which Verified entries use graphs beyond CGM's self-reported ranks is UNVERIFIED.
Fengshuo Liu et al. "Coding Agents Have Converged." ADMA 2026. arXiv:2609.17394. https://arxiv.org/abs/2609.17394 . 254 submissions; top two Verified entries both at 396/500; "Exact paired McNemar tests separate none of the 29 adjacent Verified top-thirty pairs at alpha=0.05"; within-model scaffold ranges "reach 29.8 percentage points."
Benjamin Rombaut. "Inside the Scaffold." arXiv:2604.03515. https://arxiv.org/abs/2604.03515 . Source-level taxonomy of 13 scaffolds; tool counts 0 to 37; seven compaction strategies.

---

## Part 4. Synthesis

### (i) When graphs measurably help, and when plain agentic search is as good

Three patterns recur across the fetched evidence.

First, graphs lift localization, and the effect grows with granularity. KGCompass found that "89.7% of successful fault locations require 2 or more hops" through its graph. ARISE raised function Recall@1 from 0.43 to 0.60 and line Recall@1 from 0.26 to 0.41 with data-flow slices. LARGER added about 13 Acc@5 points on single-file LocBench over Codex but only about 6 on multi-file MuLocBench. LocAgent reached 94.16% file Acc@5 against 90.15% for OpenHands and 79.56% for Agentless on the same model. CoSIL nearly doubled function Top-1 over Agentless-FL on SWE-bench Verified with a 32B open model.

Second, localization gains convert into resolution gains that are real but small. RepoGraph added 2.0 to 2.7 points across four scaffolds. ARISE added 4.7 points on SWE-agent. RepoAtlas added 2.3 to 4.4 points over a no-graph mini-SWE-agent while using fewer tokens. LocAgent added 4 points at Pass@10. CoSIL added 2.6 points on Verified. The one study holding a frontier model fixed against a grep agent found 50.4 versus 45.3 resolve at p = 0.087, which the authors themselves describe as not separated at conventional significance, and the leaderboard audit shows gaps of that size are inside the noise of adjacent entries. Sam-Bodden's finding that temperature-0 flips about 9% of instance outcomes reinforces that most reported graph gains need paired tests and seeds.

Third, plain agentic search is as good when the task is lexically anchored. Agentless's prompt-only localization contained the ground truth 78.67% of the time, and its authors concede closed-source agents did better only "on problems with no location clues." LocAgent's own ablation is the sharpest evidence: removing TraverseGraph cost 5.5 function-level points, removing the keyword SearchEntity tool cost 18.3, and removing the BM25 index cost 11.3. Lexical search mattered two to three times more than graph traversal. RepoNavigator shows that a single jump-to-definition tool, which is a one-hop graph query, beats richer graph toolkits, and that adding tools hurt (24.28% to 13.71% IoU). The regime where graphs win is where the needed context is reachable only through callers, data flow, or test links that grep cannot name: multi-hop dependency questions, cross-file edits, and trace-to-code retrieval, where the vectorless RepoMap baseline beat every embedder in Agent Retrieval Bench.

On read-only question answering, the picture is different: two independent evaluations (Cursor's Context Bench, the SWE-QA study) found embedding indexes beating grep-only agents by 12.5 and about 19 points respectively, while Anthropic's unpublished internal result went the other way. The SWE-QA study's agentic arm was a hierarchical planner-subagent design whose dominant failure mode was coordination, not search, so it does not settle the question for a single-loop agent.

### (ii) Static-analysis graphs versus LLM-extracted graphs

Nearly every system with measured gains builds its graph from deterministic static analysis: tree-sitter (RepoGraph, GraphCodeAgent, LARGER, RepoAtlas, Code Isn't Memory, Aider), Python AST (LocAgent, KGCompass, Athale and Vaddina), PyCG (RepoHyper, RepoAtlas), Jedi (CodePlan), Pyright (RepoNavigator), ctags (Code Researcher), or class hierarchy analysis (CGM). Construction cost ranges from under a second (ctags) through minutes (CGM "3 minutes or more") to 764 seconds (LocAgent on sympy). Precision is high but blind to behaviour fixed at runtime: RepoAtlas's sympy-17630 dependency "registered only at runtime," RepoNavigator's monkey patches and dynamic imports, CodePlan's absent dynamic dispatch.

LLM-extracted content appears as a layer on top of static structure rather than as the structure itself: node descriptions and embeddings (RANGER, Athale and Vaddina), requirement graphs (GraphCodeAgent, verified by two PhD students), file functionality summaries (RepoMem), and issue-to-code links mined by regex (KGCompass). Where an LLM builds the structural graph itself, CoSIL does it on demand per issue rather than as a persistent index, at 0.14M tokens per instance with GPT-4o versus 2.07M for LocAgent's full agent. KGCompass's hybrid ablation shows the pattern: KG-only localization 68.7% file coverage, LLM-only 81.0%, hybrid 84.3%.

Drift is the stated reason Anthropic dropped indexing ("the code drifts out of sync"), and RANGER and GraphCodeAgent both concede static offline graphs limit rapidly evolving codebases. The answers in the literature are incremental maintenance: CodePlan's per-edit edge recomputation, Code Isn't Memory's Merkle-diff updates, RPG-Encoder's 95.7% overhead reduction, and Aider's cheap full rebuild under a 1k-token budget.

### (iii) Graphs and the context-window budget

The evidence cuts both ways and the direction depends on whether the graph replaces exploration or is added to the prompt.

Graphs reduce tokens when they substitute for exploration turns. RepoAtlas used 630.3K input tokens against 693.0K for the plain agent and 984.7K for LocAgent on Kimi-K2.5, with fewer model calls. LARGER used 353K tokens and $0.44 against 521.8K and $0.98 for Codex on MuLocBench. The structural index cut turns from 36.2 to 28.3 and tokens from 14.0k to 10.1k against agentic grep. Codebase-Memory used "ten times fewer tokens" at a 9-point quality cost. Code Researcher's explicit memory filtering cut working context from 21,557 to 7,797 tokens and improved resolution. CodeGrep quantifies the mechanism: a 30B OpenHands agent spends 23 rounds and 631K tokens per resolved issue mostly on grep, glob, and view_file, and precise retrieval cuts rounds by 15% and tokens by 19%.

Graphs inflate context and hurt accuracy when their output is dumped. CodexGraph used 102.25K tokens per SWE-bench Lite instance against 76.01K for AutoCodeRover, partly from Cypher-generation loops. RepoGraph's 2-hop ego-graph at 10,505 tokens scored 26.00% against 29.67% for 1-hop at 2,311 tokens. CGM collapsed from 43.00% to 9.67% when the whole graph was fed to the reader without reranking, and the reranker alone was worth 24.67 points. CodeRAG-Bench found five documents optimal. SWE-bench's original BM25 results showed resolution falling as the budget rose from 13k to 50k tokens even as recall rose from 29.58% to 51.06%.

This matches the long-context studies: successful agent trajectories stay under 20k to 30k tokens, single-shot repair at 64k with perfect recall yields 7% or 0%, compressed context matches whole files at 19K instead of 94K tokens, semantic recall of code collapses mid-context, and simple observation masking halves cost without losing accuracy. The practical rule supported by the data is that the graph should rank and filter what enters the window, not be read into it.

### (iv) Known failure modes

Stale graphs: stated by RANGER, GraphCodeAgent, and Anthropic; measured nowhere, because every academic evaluation uses fixed snapshots (the SWE-QA study says so explicitly).
Runtime-only dependencies: RepoAtlas (sympy-17630 failed in every run), RepoNavigator (monkey patching, dynamic imports), CodePlan (no dynamic dispatch or data flow).
Over-retrieval and noise: RepoGraph 2-hop, CGM without reranker, CodeRAG-Bench beyond five documents, SWE Context Bench's "unfiltered or incorrectly selected context provides limited or negative benefits," and CodeGrep's precision threshold where BM25 at 0.375 made the agent worse than no retrieval.
Text-to-Cypher errors: CodexGraph's non-executable query loops and its 19.60 EM drop without a translator agent; RANGER routes only entity queries to Cypher for this reason.
Schema and tool-surface mismatch: adding three tools to a single jump tool cut IoU by more than ten points; SWE-agent's ACI ablation shows search output that is too long or too short both hurt; RepoNavigator argues success probability compounds multiplicatively across unfamiliar tool calls.
Copy-paste misguidance: GraphCoder loses on about 5% of tasks because a structurally similar snippet induces literal copying.
Coordination failures in hierarchical agents: 41.8% of deep-agentic failures in the SWE-QA study were silent planner-subagent hand-offs, and 91% of those returned fluent, ungrounded answers.
Memory pollution: RepoMem notes irrelevant retrieved history "can pollute reasoning context and distract."
Construction cost at scale: 764 seconds for LocAgent on one mid-size repo versus 0.76 seconds for ctags, which is why Code Researcher chose ctags for a 28M-line kernel.

### (v) Open gaps

No fetched study compares a static-analysis graph against a frontier single-loop grep agent such as Claude Code on SWE-bench Verified with the model held fixed and enough instances to separate a few points. The closest, Code Isn't Memory, had n = 75 to 80 and was inconclusive against agentic grep; LARGER's Claude Code row ran a different model from its own system.
Almost all graph results are Python. CGM's Java resolve rate was 14.29% against 43.00% for Python; Code Isn't Memory covered Go, Java, and Python but not JavaScript or TypeScript.
The QA evidence favouring embeddings (Cursor, SWE-QA study) and Anthropic's internal result favouring agentic search have not been reconciled by any public controlled study, and the one academic study used an architecturally different agentic arm.
Per-instance outcome noise of about 9% at temperature 0 and the non-separability of adjacent leaderboard entries mean most reported 2-to-5-point graph gains lack the paired tests and multiple seeds that only Code Isn't Memory and the ADMA audit provide.
Staleness has never been measured: every evaluation uses frozen repository snapshots.
Hybrids are consistently best but under-analysed: lexical plus graph (LARGER, LocAgent), embedding plus RepoMap (Agent Retrieval Bench, +18% MRR), KG plus LLM (KGCompass), and graph plus commit memory (RepoMem) each beat their components, but no benchmark isolates which component carries which task type.
Set-level sufficiency rather than per-passage relevance (The Missing Complement) and state-conditioned retrieval mid-trajectory are new framings with one paper each.

---

## Complete URL list

Fetched and cited:
- https://aclanthology.org/2025.naacl-long.7/
- https://arxiv.org/abs/2408.03910
- https://arxiv.org/html/2408.03910
- https://arxiv.org/abs/2410.14684
- https://arxiv.org/html/2410.14684
- https://aclanthology.org/2025.acl-long.426/
- https://arxiv.org/abs/2503.09089
- https://arxiv.org/html/2503.09089
- https://arxiv.org/abs/2504.10046
- https://arxiv.org/html/2504.10046
- https://arxiv.org/abs/2406.07003
- https://arxiv.org/html/2406.07003
- https://conf.researchr.org/details/ase-2024/ase-2024-research/46/GraphCoder-Enhancing-Repository-Level-Code-Completion-via-Coarse-to-fine-Retrieval-B
- https://arxiv.org/abs/2403.06095
- https://arxiv.org/html/2403.06095
- https://aclanthology.org/2024.lrec-main.305/
- https://arxiv.org/abs/2212.10007
- https://arxiv.org/abs/2303.12570
- https://arxiv.org/abs/2402.14323
- https://arxiv.org/abs/2509.05980
- https://arxiv.org/abs/2509.16112
- https://arxiv.org/abs/2309.12499
- https://arxiv.org/html/2309.12499
- https://github.com/microsoft/CodePlan
- https://arxiv.org/abs/2407.01489
- https://arxiv.org/html/2407.01489
- https://conf.researchr.org/details/fse-2025/fse-2025-research-papers/85/Demystifying-LLM-based-Software-Engineering-Agents
- https://proceedings.neurips.cc/paper_files/paper/2025/hash/178ae4ba29022eb7bf509c2e27bc8ab8-Abstract-Conference.html
- https://arxiv.org/abs/2505.16901
- https://arxiv.org/html/2505.16901
- https://arxiv.org/pdf/2505.16901v4
- https://arxiv.org/html/2503.21710v2
- https://arxiv.org/abs/2506.11060
- https://arxiv.org/html/2506.11060
- https://arxiv.org/abs/2503.22424
- https://arxiv.org/html/2503.22424
- https://arxiv.org/abs/2509.25257
- https://arxiv.org/html/2509.25257
- https://arxiv.org/abs/2512.20957
- https://arxiv.org/html/2512.20957
- https://arxiv.org/abs/2605.03117
- https://arxiv.org/abs/2602.02084
- https://arxiv.org/abs/2605.16352
- https://arxiv.org/html/2605.16352
- https://arxiv.org/abs/2609.16936
- https://arxiv.org/html/2609.16936
- https://arxiv.org/abs/2606.22417
- https://arxiv.org/html/2606.22417
- https://aider.chat/docs/repomap.html
- https://aider.chat/2023/10/22/repomap.html
- https://raw.githubusercontent.com/Aider-AI/aider/main/aider/repomap.py
- https://researchr.org/publication/YamaguchiGAR14
- https://arxiv.org/abs/2507.16585
- https://arxiv.org/abs/2405.02355
- https://arxiv.org/html/2405.02355
- https://arxiv.org/abs/2505.14394
- https://arxiv.org/html/2505.14394
- https://arxiv.org/abs/2603.27277
- https://arxiv.org/abs/2602.13921
- https://arxiv.org/abs/2510.01003
- https://arxiv.org/html/2510.01003
- https://arxiv.org/abs/2510.04905
- https://arxiv.org/abs/2405.15793
- https://arxiv.org/html/2405.15793
- https://arxiv.org/abs/2407.16741
- https://arxiv.org/abs/2310.06770
- https://arxiv.org/html/2310.06770
- https://aclanthology.org/2025.findings-naacl.176/
- https://arxiv.org/abs/2406.14497
- https://arxiv.org/html/2406.14497
- https://arxiv.org/abs/2606.11864
- https://arxiv.org/html/2607.24882v1
- https://arxiv.org/abs/2608.05886
- https://arxiv.org/abs/2609.20050
- https://arxiv.org/abs/2509.14635
- https://arxiv.org/abs/2406.06025
- https://arxiv.org/abs/2505.07897
- https://arxiv.org/abs/2608.24221
- https://arxiv.org/abs/2603.25770
- https://arxiv.org/abs/2602.08316
- https://arxiv.org/abs/2603.06358
- https://www.latent.space/p/claude-code
- https://newsletter.pragmaticengineer.com/p/building-claude-code-with-boris-cherny
- https://claude.com/blog/building-agents-with-the-claude-agent-sdk
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://cursor.com/blog/semsearch
- https://www.augmentcode.com/blog/auggie-tops-swe-bench-pro
- https://sourcegraph.com/blog/deep-search
- https://sourcegraph.com/changelog/deep-search-evaluator
- https://milvus.io/blog/why-im-against-claude-codes-grep-only-retrieval-it-just-burns-too-many-tokens.md
- https://arxiv.org/html/2608.01507v1
- https://www.trychroma.com/research/context-rot
- https://arxiv.org/abs/2307.03172
- https://arxiv.org/abs/2505.13353
- https://arxiv.org/abs/2602.16069
- https://arxiv.org/abs/2508.21433
- https://arxiv.org/abs/2607.09691
- https://arxiv.org/html/2506.17208v2
- https://arxiv.org/abs/2609.17394
- https://arxiv.org/abs/2604.03515

Fetched but not usable:
- https://dblp.uni-trier.de/rec/conf/sp/YamaguchiGAR14.html (access denied)
- https://ieeexplore.ieee.org/document/6956589 (empty response)
- https://dl.acm.org/doi/10.1145/3643757 (HTTP 403; CodePlan ACM DOI therefore UNVERIFIED)
- https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk (redirects to the claude.com post above)
