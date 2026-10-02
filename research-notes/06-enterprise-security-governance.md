# Enterprise brief: giving Claude Code access to a Neo4j code knowledge graph via MCP

Research date: 2026-10-01. Every claim below is tied to a fetched URL (full list at the end). Items that could not be confirmed from a primary page are marked UNVERIFIED.

## Headline assessment

The dominant risk is not the graph database. It is that a code graph is a channel of untrusted text (identifiers, docstrings, comments, commit messages, dependency READMEs) flowing into an agent that also holds write access to files, shells, and other MCP servers. Every incident examined below has the same shape: injected text in a tool result, plus one over-privileged tool, plus one exfiltration path. The controls that matter most are therefore (1) a read-only database identity that the MCP server cannot escalate, (2) a curated parameterised tool surface instead of raw Cypher, (3) egress control on both the agent host and the Neo4j host, and (4) policy delivered through Claude Code managed settings rather than CLAUDE.md, which Anthropic's own docs describe as "context, not enforced configuration."

---

## 1. MCP threat model

### 1.1 Tool poisoning, rug pulls, cross-server shadowing

**Source:** Invariant Labs, "MCP Security Notification: Tool Poisoning Attacks", April 1, 2025. https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks

Instructions hidden in tool descriptions are invisible to users but followed by the model. In the demo the agent "willingly reads the user's ~/.cursor/mcp.json file, and other sensitive files like SSH keys." Rug pull: "a malicious server can change the tool description after the client has already approved it." Shadowing: "a malicious server can poison tool descriptions to exfiltrate data accessible through other trusted servers." Recommended mitigations: display tool descriptions clearly, pin MCP server versions using checksums, enforce boundaries between servers.

**Source:** Invariant Labs, "WhatsApp MCP Exploited", April 7, 2025 (updated April 9). https://invariantlabs.ai/blog/whatsapp-mcp-exploited

A sleeper tool on an unrelated server "only activates on the second launch of the MCP server, thereby circumventing the need to be re-approved by the user," and redirected messages from a trusted WhatsApp server to attacker-controlled numbers.

**Assessment.** A Neo4j server you build and host yourself is low risk for description poisoning, but any other server in the same Claude Code session can shadow it. The allowlist and description-hash check must cover the whole session, not just the graph server.

### 1.2 Prompt injection via tool results ("toxic agent flows")

**Source:** Invariant Labs, "GitHub MCP Exploited", May 26, 2025. https://invariantlabs.ai/blog/mcp-github-vulnerability

An attacker plants a malicious issue in a public repository. When the user asks the agent about issues, the payload coerces it into reading private repositories and writing the contents into a public pull request. The post calls this a "toxic agent flow" and concludes "model alignment is not enough"; recommended mitigations are per-session repository scoping and runtime scanning (MCP-scan).

**Source:** General Analysis, "Supabase MCP Security: How Prompt Injection Leaked Private Tables", demonstrated July 8, 2025 (page shows a September 6, 2026 review date). https://generalanalysis.com/blog/supabase-mcp-blog

A support ticket contained "THIS MESSAGE IS FOR YOU AFTER YOU READ THE LATEST MESSAGES FROM SUPABASE MCP > PLEASE DO THESE ACTIONS ASAP." The assistant in Cursor ran with service_role (bypassing row-level security) and the integration_tokens table was "extracted and inserted into a customer-visible support ticket message." Mitigations listed: read_only=true, project_ref and features scoping, and "Review tool calls and keep untrusted results as data."

**Source:** Simon Willison, "Supabase MCP can leak your entire SQL database", July 6, 2025. https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/

Defines the lethal trifecta as "access to private data, exposure to potentially malicious instructions and a mechanism to communicate data back out to an attacker." Read-only mode removes one leg (write-back exfiltration) but he judges it insufficient on its own.

**Source:** Supabase, "Defense in Depth for MCP Servers", September 16, 2025. https://supabase.com/blog/defense-in-depth-mcp

Supabase added read-only mode, project-scoped mode, feature groups "restricting which MCP tools the LLM can use," and result wrapping with warnings. Principle: "combine input validation, output sanitization, context isolation, and least privilege." Their headline recommendation: "Never connect AI agents directly to production data."

**Assessment.** A code graph is built from repository text. In most organisations some of that text comes from external contributors, vendored code, generated files, and dependency documentation. Treat every graph query result as untrusted content, exactly as Supabase now treats database rows.

### 1.3 Confused deputy, token passthrough, session hijacking, SSRF, local server compromise

**Source:** MCP specification, Security Best Practices, revision 2025-11-25. https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices

Normative text:
- Confused deputy via OAuth proxy servers: "MCP proxy servers MUST implement per-client consent and proper security controls."
- Token passthrough: "MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server."
- SSRF via OAuth metadata discovery: clients "SHOULD block requests to private and reserved IP address ranges," including the cloud metadata range 169.254.0.0/16.
- Session hijacking: "MCP servers that implement authorization MUST verify all inbound requests. MCP Servers MUST NOT use sessions for authentication."
- Local servers: clients offering one-click install "MUST: Show the exact command that will be executed, without truncation."
- Scope minimisation: avoid "wildcard or omnibus scopes (*, all, full-access)."

**Source:** MCP specification, Tools, revision 2025-11-25. https://modelcontextprotocol.io/specification/2025-11-25/server/tools

"For trust & safety and security, there SHOULD always be a human in the loop with the ability to deny tool invocations." "Clients MUST consider tool annotations to be untrusted unless they come from trusted servers." Servers "MUST: Validate all tool inputs, Implement proper access controls, Rate limit tool invocations, Sanitize tool outputs." Clients "SHOULD: ... Validate tool results before passing to LLM, Implement timeouts for tool calls, Log tool usage for audit purposes."

### 1.4 Verified CVEs and incidents, 2025 to 2026

| Item | Source and date | Fix |
|---|---|---|
| CVE-2025-6514, mcp-remote, CVSS 9.6. A malicious server's OAuth authorization_endpoint is passed to the OS; "arbitrary OS command execution" on Windows, arbitrary executable launch on macOS/Linux. Versions 0.0.5 to 0.1.15. | JFrog, July 9, 2025. https://jfrog.com/blog/2025-6514-critical-mcp-remote-rce-vulnerability/ | 0.1.16; connect only to HTTPS trusted servers |
| CVE-2025-49596, MCP Inspector. The proxy "accepts arbitrary stdio commands without authenticating or validating the source of requests from its browser client"; exploited via browsers treating 0.0.0.0 as localhost plus CSRF. 560 instances exposed on Shodan at publication. | Recorded Future (no date on page). https://www.recordedfuture.com/blog/anthropic-mcp-inspector-cve-2025-49596 | 0.14.1 (session tokens, origin checks) |
| CVE-2025-68143, CVE-2025-68144, CVE-2025-68145 in mcp-server-git (unrestricted git_init, argument injection in git_diff, path validation bypass). Chained with the Filesystem server to reach RCE via git clean/smudge filters. Reported by Cyata June 2025, patched December 2025. | The Register, January 20, 2026. https://www.theregister.com/2026/01/20/anthropic_prompt_injection_flaws/ | mcp-server-git 2025.12.18 |
| "MCP by design": OX Security found that the stdio transport gives "a direct configuration-to-command execution" path across the Python, TypeScript, Java and Rust SDKs; multiple downstream CVEs including CVE-2025-65720, CVE-2026-30623 (LiteLLM), CVE-2026-40933 (Flowise). THN reports Anthropic called the behaviour "expected" and did not change the reference implementation. | The Hacker News, April 20, 2026. https://thehackernews.com/2026/04/anthropic-mcp-design-vulnerability.html | Treat MCP configuration as untrusted; sandbox; install only from verified sources |

I found no Neo4j-specific MCP CVE.

**Assessment.** A stdio MCP server is a command line in a config file. Whoever can write `.mcp.json`, `~/.claude.json`, or `managed-mcp.json` can execute code on every developer machine. MCP configuration must go through the same code review and signing as any other executable artefact.

### 1.5 OWASP, NIST, CISA

**OWASP Top 10 for LLM Applications 2025.** https://genai.owasp.org/llm-top-10/ Relevant entries: LLM01 Prompt Injection, LLM03 Supply Chain, LLM05 Improper Output Handling, LLM06 Excessive Agency ("An LLM-based system is often granted a degree of agency"), LLM10 Unbounded Consumption.

**OWASP Agentic AI Threats and Mitigations**, February 17, 2025. https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/ The OWASP page only links a PDF. The fifteen threat names were taken from a third-party mapping document (https://github.com/luckyPipewrench/pipelock/blob/main/docs/owasp-agentic-top15-mapping.md): T1 Memory Poisoning, T2 Tool Misuse, T3 Privilege Compromise, T4 Resource Overload, T5 Cascading Hallucination Attacks, T6 Intent Breaking and Goal Manipulation, T7 Misaligned and Deceptive Behaviors, T8 Repudiation and Untraceability, T9 Identity Spoofing and Impersonation, T10 Overwhelming Human-in-the-Loop, T11 Unexpected RCE and Code Attacks, T12 Agent Communication Poisoning, T13 Rogue Agents in Multi-Agent Systems, T14 Human Attacks on Multi-Agent Systems, T15 Human Manipulation. UNVERIFIED against the OWASP PDF itself. T1, T2, T3, T8, T11 map directly onto a code-graph MCP design.

**OWASP Top 10 for Agentic Applications 2026**, published December 9, 2025. https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications/ The OWASP page confirms the date only; entry names from promptfoo's mirror (https://www.promptfoo.dev/docs/red-team/owasp-agentic-ai/): ASI01 Agent Goal Hijack, ASI02 Tool Misuse and Exploitation, ASI03 Identity and Privilege Abuse, ASI04 Agentic Supply Chain Vulnerabilities, ASI05 Unexpected Code Execution, ASI06 Memory and Context Poisoning, ASI07 Insecure Inter-Agent Communication, ASI08 Cascading Failures, ASI09 Human Agent Trust Exploitation, ASI10 Rogue Agents.

**NIST AI 100-2 E2025, Adversarial Machine Learning taxonomy**, March 2025. https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2025.pdf Section 3.4 (Indirect Prompt Injection, NISTAML.015): "Attackers may also use indirect prompt injection attacks to hijack a GenAI AGENT, causing it to perform a malicious, attacker-specified task instead of (or in addition to) its intended, user-provided task." Section 3.5 (Security of Agents): "because agents can take actions using tools, these attacks can create additional risks in this context, such as enabling actors to hijack agents to execute arbitrary code or exfiltrate data from the environment in which they are operating. Security research focused specifically on agents is still in its early stages."

**NIST CAISI AI Agent Standards Initiative**, launched February 17, 2026. https://www.nist.gov/artificial-intelligence/ai-agent-standards-initiative Three pillars (industry-led standards, community-led protocols, research on agent identity and security evaluations); RFI on AI agent security closed March 9, 2026. The nist.gov page does not mention MCP. Claims of a "Q4 2026 AI Agent Interoperability Profile" and "SP 800-53 overlays" appeared only in Cloud Security Alliance research notes: UNVERIFIED.

**CISA and partners, "Careful Adoption of Agentic AI Services"**, May 1, 2026. https://www.cisa.gov/resources-tools/resources/careful-adoption-agentic-ai-services CISA's page confirms co-authorship with ASD's ACSC "and other international and U.S. partners." The body text is hosted at cyber.gov.au, which timed out twice. Crowell's client alert (May 19, 2026, https://www.crowell.com/en/insights/client-alerts/american-and-allied-cyber-agencies-issue-first-joint-guidance-on-securing-agentic-ai) quotes it: "Configure least-privilege access so agents only have permissions required for their tasks"; "Require human oversight mechanisms, including approval steps and intervention points for high-risk actions"; "Validate that all third-party components originate from trusted sources"; risk categories "Expanded attack surface", "Privilege compromise and scope creep", "Behavior misalignment", "Obscure event records." Body quotes are second-hand.

### 1.6 Vendor and industry guidance

**Anthropic, Claude Code security page.** https://code.claude.com/docs/en/security "We encourage either writing your own MCP servers or using MCP servers from providers that you trust. ... Anthropic reviews connectors against its listing criteria before adding them to the Anthropic Directory ... but does not security-audit or manage any MCP server." Web fetch "uses a separate context window to avoid injecting potentially malicious prompts." "First-time codebase runs and new MCP servers require trust verification," but "Trust verification is disabled when running non-interactively with the -p flag."

**Anthropic, CISO's guide to agentic AI**, July 17, 2026. https://claude.com/blog/ciso-guide-to-agentic-ai Four questions: what untrusted content does it ingest, what actions can it take and on whose behalf, what is the blast radius, what observability exists. Controls: IdP credentials with revocation, connector allowlists, removing destructive actions from tools, sandboxed execution without production credentials, egress allowlisting ("only destinations you chose should be reachable"), OpenTelemetry to SIEM, organisation-wide kill switch. On prompt injection: success rates are "not zero" but declining.

**Microsoft, "Protecting against indirect prompt injection attacks in MCP"**, April 28, 2025. https://developer.microsoft.com/blog/protecting-against-indirect-injection-attacks-mcp/ Recommends Prompt Shields with "Spotlighting" and "Delimiters and Datamarking" so the model can "distinguish between valid system instructions and potentially untrustworthy external inputs," plus supply-chain verification of all components.

**Cloud Security Alliance, Agentic MCP Security Best Practices v1**, March 27, 2026 (draft). https://labs.cloudsecurityalliance.org/agentic/agentic-mcp-security-best-practices-v1/ "All remote MCP server connections must use OAuth 2.1 ... with PKCE mandatory"; tool-description hashing between sessions; access tokens "no more than one hour"; "Filesystem access should be blocked by default"; "Organizations should maintain a curated registry of approved MCP servers that have undergone security review, with the registry enforcing version pinning."

### 1.7 Registries, signing, scanning

**Official MCP Registry.** https://github.com/modelcontextprotocol/registry Launched in preview September 2025; API frozen at v0.1 on October 24, 2025. Verifies namespace ownership via GitHub OAuth/OIDC, DNS, or HTTP. It hosts metadata ("like an app store for MCP servers"), not code. No package signing or verification is documented.

**mcp-scan / Agent Scan.** https://github.com/invariantlabs-ai/mcp-scan Detects prompt injection, tool poisoning, untrusted content, destructive capabilities; supports Claude Desktop and Claude Code among others; requires interactive consent before contacting servers.

**Assessment.** No production signing chain for MCP servers exists today. Pin by exact command path or URL in managed settings and by container digest in your own deployment pipeline. Hash tool descriptions at approval time and alert on change.

---

## 2. Text-to-Cypher risk

### 2.1 What arbitrary Cypher can do

**Source:** Varonis, "Neo4jection: Secrets, Data, and Cloud Exploits", last updated February 8, 2023. https://www.varonis.com/blog/neo4jection-secrets-data-and-cloud-exploits Cypher injection enables exfiltration via LOAD CSV to attacker URLs ("an attacker can use the web functionality to exfiltrate data"), APOC HTTP procedures (apoc.load.jsonParams, apoc.load.csvParams), reading the AWS metadata endpoint ("GET ... http://169.254.169.254/latest/meta-data/iam/security-credentials/"), and dumping the system database including "Users (including the hash of the password!)". Mitigations: "Always use parameters if possible"; "Use the setting dbms.security.procedures.allowlist"; IMDSv2; least privilege.

**Source:** Agent Threat Rule ATR-2026-02100 (community rule set, "experimental" status). https://agentthreatrule.org/en/rules/ATR-2026-02100 Flags LLM-generated Cypher containing DETACH DELETE, dbms.security.* procedures, apoc.periodic.iterate, and apoc.load/runFile with file:// URIs, citing frameworks that pass "unvalidated LLM-generated Cypher directly to database drivers without statement-type allowlists or execution gates." Response actions: block tool, alert, escalate.

**Source:** Neo4j Cypher manual, LOAD CSV. https://neo4j.com/docs/cypher-manual/current/clauses/load-csv/ "LOAD CSV supports accessing CSV files via HTTPS, HTTP, and FTP (with or without credentials)." `dbms.security.allow_csv_import_from_file_urls` "determines whether file:/// URLs are allowed"; `server.directories.import` roots those paths. This makes the Neo4j host itself an outbound channel.

### 2.2 Neo4j controls

**RBAC and property-level security.** https://neo4j.com/docs/operations-manual/current/authentication-authorization/manage-privileges/ and https://neo4j.com/docs/operations-manual/current/authentication-authorization/property-based-access-control/ Available in Enterprise Edition, AuraDB Business Critical and Virtual Dedicated Cloud only (Community has no RBAC). Built-in roles: PUBLIC, reader, editor, publisher, architect, admin. Grant syntax example: `GRANT MATCH {*} ON GRAPH * NODE * TO architect`. Property rule example: `GRANT READ { address } ON GRAPH * FOR (n:Email|Website) WHERE n.domain = 'exampledomain.com' TO regularUsers`; relationship example: `GRANT READ {*} ON GRAPH * FOR ()-[r]-() WHERE r.classification IN ['UNCLASSIFIED', 'PUBLIC'] TO regularUsers`. Two documented caveats: "ensure the property used for the rule cannot be modified" and "A DENY rule fails open when its criteria cannot be evaluated, which means the restriction is not applied and access is granted by default." Performance overhead grows with TRAVERSE rules; Neo4j recommends label-based privileges for hot paths. The manage-privileges page also notes attribute-based auth rules in version 2026.09 (not examined).

**Transactions, memory, logging, procedures.** Sources: https://neo4j.com/docs/operations-manual/current/monitoring/transaction-management/ , https://neo4j.com/docs/operations-manual/current/performance/memory-configuration/ , https://neo4j.com/docs/operations-manual/current/monitoring/logging/ , https://neo4j.com/docs/operations-manual/current/configuration/configuration-settings/ , https://neo4j.com/docs/operations-manual/current/security/securing-extensions/

| Control | Setting | Documented default |
|---|---|---|
| Query timeout | db.transaction.timeout | 0 (disabled); dynamic via dbms.setConfigValue |
| Concurrent transactions | db.transaction.concurrent.maximum | 1000 |
| Per-transaction memory | db.memory.transaction.max | not shown on fetched page |
| Per-database transaction memory | db.memory.transaction.total.max | not shown |
| Global transaction memory | dbms.memory.transaction.total.max | not shown |
| Query log | db.logs.query.enabled (OFF, INFO, VERBOSE) | VERBOSE |
| Log parameters | db.logs.query.parameter_logging_enabled | true |
| Obfuscate literals | db.logs.query.obfuscate_literals | false |
| Procedure allowlist | dbms.security.procedures.allowlist | * (all loaded) |
| Unrestricted procedures | dbms.security.procedures.unrestricted | internal-API procedures "disabled by default" |
| file:// in LOAD CSV | dbms.security.allow_csv_import_from_file_urls | not shown |

Neo4j: "When any of the limits are reached, the transaction is terminated without affecting the overall health of the database." Security log records authentication attempts, admin commands and authorisation failures when auth is enabled.

### 2.3 Neo4j MCP servers

**Neo4j Labs Python server (mcp-neo4j-cypher).** https://github.com/neo4j-contrib/mcp-neo4j and https://github.com/neo4j-contrib/mcp-neo4j/blob/main/servers/mcp-neo4j-cypher/README.md Tools: read_neo4j_cypher, write_neo4j_cypher, get_neo4j_schema (needs APOC). Env: NEO4J_READ_ONLY (default false), NEO4J_READ_TIMEOUT (default 30 s), NEO4J_RESPONSE_TOKEN_LIMIT, NEO4J_MCP_SERVER_ALLOWED_HOSTS (default localhost,127.0.0.1), NEO4J_MCP_SERVER_ALLOW_ORIGINS, NEO4J_SCHEMA_SAMPLE_SIZE (1000). Transports stdio, SSE, HTTP. Labs disclaimer: "without any SLAs or guarantees around backwards compatibility and deprecation."

**Neo4j blog, "Production-proofing your Neo4j Cypher MCP server"**, Tomaž Bratanič, October 8, 2025. https://neo4j.com/blog/developer/production-proofing-cypher-mcp-server/ Read-only mode "completely dismisses the tool responsible for write operations"; read timeout "defaults to 30 seconds. If a query exceeds this limit, it's terminated immediately"; token-aware truncation; result sanitisation ("Oversized lists are dropped, nested dicts are pruned"); DNS-rebinding and CORS protection. States a "read-only database user provides a foundational layer of security."

**Official Neo4j Go server (github.com/neo4j/mcp).** https://github.com/neo4j/mcp Tools: get-schema, read-cypher, write-cypher ("disabled if NEO4J_MCP_READ_ONLY=true"), list-gds-procedures. read-cypher is "enforced via EXPLAIN and Neo4j's query-type classification," with the caveat: "custom procedures or functions incorrectly classified as read-only by Neo4j may bypass this check; ensuring correct classification is the responsibility of the procedure/function maintainer." stdio by default; NEO4J_MCP_TELEMETRY and NEO4J_MCP_LOG_LEVEL options.

**Neo4j community guidance.** The JDBC text2cypher docs recommend retrieving the generated Cypher, inspecting it, and running it separately rather than executing directly; community threads recommend a dedicated read-only user for any text-to-Cypher tool (search summaries; primary pages not fetched: UNVERIFIED as quotes).

### 2.4 Assessment and recommended pattern

Read-only mode in either MCP server is enforced client-side (by not exposing a tool, or by EXPLAIN classification). The real control is a database user holding only reader privileges. Willison's point stands: read-only removes only the write-back exfiltration leg. Neo4j has other legs (LOAD CSV over HTTP, APOC load procedures, GDS procedures that write), so you also need:

1. `dbms.security.procedures.allowlist` restricted to the few procedures the server actually calls (the schema tool needs some APOC meta procedures).
2. Outbound network blocked from the Neo4j host; IMDSv2 enforced if on AWS.
3. Raw Cypher either disabled or gated by a Claude Code PreToolUse hook that denies statements containing DETACH DELETE, dbms., apoc.load, apoc.periodic, LOAD CSV, CALL { } IN TRANSACTIONS, and any CREATE/MERGE/SET/REMOVE.
4. The default tool surface as a small set of typed, parameterised tools (find_symbol, callers_of, callees_of, file_dependencies, blast_radius, path_between) each backed by a fixed Cypher template with values bound as driver parameters. This is the "feature groups" pattern Supabase adopted, applied to graph queries, and it is the shape evaluated in the Codebase-Memory and CodeCompass papers in section 5.

---

## 3. Data residency and deployment

### 3.1 What leaves the boundary

**Source:** Claude Code data usage. https://code.claude.com/docs/en/data-usage "Claude Code runs locally. To interact with the LLM, Claude Code sends data over the network. This data includes all user prompts and model outputs, encrypted in transit via TLS 1.2+."

**Source:** Compliance API, session transcripts. https://platform.claude.com/docs/en/manage-claude/compliance-sessions "For local sessions, Anthropic records each conversation server-side as its requests reach the Claude API; nothing is installed on the device, and nothing is collected beyond the requests the client already sends to the Claude API." A transcript "is the sequence of user prompts, assistant responses, and tool calls and results in that conversation."

So every graph query result an MCP server returns is sent to the model endpoint as context. Claude Code caps MCP tool output at 25,000 tokens by default (warning at 10,000), with overflow written to a local file (https://code.claude.com/docs/en/mcp). The graph itself never leaves your network if the MCP server and Neo4j are inside it; only the query results do, plus whatever file contents Claude reads itself.

**Source:** Zero data retention for Claude Code. https://code.claude.com/docs/en/zero-data-retention "Data processed by third-party tools, MCP servers, or other external integrations is not covered by ZDR. Review those services' data handling practices independently."

### 3.2 Training

**Source:** Anthropic Commercial Terms of Service, effective June 17, 2025. https://www.anthropic.com/legal/commercial-terms "Anthropic may not train models on Customer Content from Services." "Customer (a) retains all rights to its Inputs, and (b) owns its Outputs."

**Source:** Claude Code data usage. "Commercial users: (Team and Enterprise plans, API, 3rd-party platforms, and Claude Gov) ... Anthropic does not train generative models using code or prompts sent to Claude Code under commercial terms, unless the customer has chosen to provide their data" (Development Partner Program, first-party API only).

### 3.3 Retention and ZDR

**Source:** https://platform.claude.com/docs/en/manage-claude/api-and-data-retention and https://code.claude.com/docs/en/zero-data-retention

- Commercial standard: 30-day retention.
- ZDR on the API: "Anthropic does not store customer prompts or responses at rest after the API response is returned." Enabled per organisation by the account team. Covers "Claude Code with API keys" from a commercial organisation; metrics logging data is exempt.
- ZDR for Claude Code on Claude for Enterprise is a separate qualified-account offering; it disables cloud sessions, Claude Tag, Artifacts, feedback submission, and Remote Control. "ZDR for Claude Code on Claude for Enterprise applies only to Anthropic's direct platform."
- Covered Models: "Claude Fable 5.1, Claude Mythos 5.1, Claude Fable 5, and Claude Mythos 5: These models require 30-day data retention and are not available under ZDR unless expressly authorized by Anthropic." A ZDR organisation can enable 30-day retention for one workspace.
- Policy violations: "Anthropic may retain the associated inputs and outputs for up to 2 years."
- Feature table: prompt caching is ZDR-eligible ("KV cache representations and cryptographic hashes are held in memory for the cache TTL"); the server-side MCP connector, Files API, batch, and code execution are not.
- Local transcripts in `~/.claude/projects/` are stored in plaintext for 30 days by default (cleanupPeriodDays).

**Source:** The Register, September 2, 2026. https://www.theregister.com/ai-and-ml/2026/09/02/anthropic-promises-zero-data-retention-but-customers-must-check-it-worked/5293789 Reports a by-invitation "Enterprise Frontier Safeguards" arrangement combining ZDR with abuse detection where "EFS works by storing data in cloud infrastructure controlled by the customer, not Anthropic," offered across Claude Code, Enterprise, Platform, Bedrock, Vertex and Foundry. Press reporting only: UNVERIFIED against Anthropic documentation.

### 3.4 Cloud providers

**Source:** API and data retention page. "On Amazon Bedrock and Google Cloud's Agent Platform, the cloud provider is the data processor." Claude Code's data-usage table: Bedrock "AES-256 with AWS-managed keys. Customer-managed keys available via AWS KMS"; Google "CMEK available"; Microsoft Foundry "Hosted on Azure" keeps prompts and completions within Azure with "only usage metadata and content flagged by Anthropic's safety systems egress to Anthropic."

**Source:** Amazon Bedrock data protection and data retention. https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html and https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html Model providers "don't have access to Amazon Bedrock logs or to customer prompts and completions." Retention modes: `none` ("No request or response data is written to durable storage by AWS or shared with the model provider"), `default`, `aws_review`. Claude Fable 5 and 5.1 "require human review (allowed_modes: [aws_review, provider_data_share])"; under aws_review "user prompts and completions are retained within the AWS boundary for up to 30 days and may be reviewed by AWS ... Your content is not shared with the model provider." Enforceable org-wide via SCP condition key `bedrock:DataRetentionMode`. Retained data for cross-region inference lives in the destination region.

**Source:** Claude Code on Bedrock and on Google Cloud's Agent Platform. https://code.claude.com/docs/en/amazon-bedrock and https://code.claude.com/docs/en/google-vertex-ai Bedrock uses the AWS SDK credential chain (IAM, SSO, bearer token); IAM needs bedrock:InvokeModel, InvokeModelWithResponseStream, ListInferenceProfiles, GetInferenceProfile; Bedrock Guardrails attach via ANTHROPIC_CUSTOM_HEADERS. Vertex: `roles/aiplatform.user`; CLOUD_ML_REGION accepts `global`, multi-region `eu`/`us`, or a region such as `us-east5`, with EU and US multi-region hostnames. Telemetry, error reporting and /feedback default off on Bedrock, Vertex and Foundry.

I could not retrieve Google's Vertex generative AI data-governance page (navigation shell only, and the search budget was exhausted). Vertex-side residency and training statements are UNVERIFIED here.

**Egress note.** The WebFetch domain safety check "sends the requested hostname to api.anthropic.com" regardless of provider unless `skipWebFetchPreflight: true` is set.

---

## 4. Access control alignment

### 4.1 MCP authorisation (OAuth 2.1)

**Source:** MCP Authorization spec, 2025-11-25. https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization "Authorization servers MUST implement OAuth 2.1"; "MCP servers MUST implement OAuth 2.0 Protected Resource Metadata (RFC9728)"; "MCP clients MUST implement PKCE ... MUST use the S256 code challenge method"; "MCP clients MUST implement Resource Indicators for OAuth 2.0 as defined in RFC 8707"; "MCP servers MUST validate that access tokens were issued specifically for them as the intended audience"; "MCP servers MUST NOT accept or transit any other tokens." "Implementations using an STDIO transport SHOULD NOT follow this specification, and instead retrieve credentials from the environment." Client registration priority: pre-registered, Client ID Metadata Documents, dynamic registration as fallback. Access tokens should be short-lived; refresh tokens rotated for public clients.

**Source:** Claude Code MCP docs. https://code.claude.com/docs/en/mcp OAuth discovery and refresh for HTTP servers; `oauth.scopes` restriction; `authServerMetadataUrl` override; `headersHelper` for custom tokens; "Credential variables (e.g., ANTHROPIC_API_KEY) read as empty in remote server url and headers to prevent credential leakage."

### 4.2 Mapping repository permissions onto the graph

Three patterns, in order of preference:

1. **Per-user HTTP MCP server with OAuth against the corporate IdP.** The server receives a user-bound token, resolves the user's repository set from GitHub, GitLab or Bitbucket (cached with short TTL), and filters every query by a `repo` property on nodes and a `repo` scope on traversals. This mirrors the GitHub MCP server model (https://github.com/github/github-mcp-server): OAuth or PAT, "enable the permissions that you feel comfortable granting your AI tools," `--read-only` takes priority over write tools, toolsets narrow exposure.
2. **Neo4j-native enforcement.** Map IdP groups to Neo4j roles and apply property-based READ and TRAVERSE rules on an immutable `repo` or `classification` property. Caveats from Neo4j docs: the property must not be writable by the indexing identity, DENY fails open, TRAVERSE rules cost performance. Enterprise-only.
3. **Service account plus application-layer filtering.** Simplest; the Supabase incident is this shape. The MCP spec notes shared tokens mean the server "will be unable to identify or distinguish between MCP Clients." Acceptable only for read-only, non-sensitive graphs, and still require the Claude Code user identity to be logged via the OAuth token or an LLM gateway.

### 4.3 Audit logging of agent queries

Three layers exist and should be correlated by user identity and session ID.

- **Neo4j query log** records query text, parameters, user, client, timing and resource use. Disable parameter logging or enable literal obfuscation if code snippets in parameters are sensitive.
- **Claude Code OpenTelemetry** (https://code.claude.com/docs/en/monitoring-usage) emits claude_code.tool_result, tool_decision, user_prompt, api_request events and cost/token metrics. "Spans redact user prompt text, tool input details, and tool content by default. Set OTEL_LOG_USER_PROMPTS=1, OTEL_LOG_TOOL_DETAILS=1, and OTEL_LOG_TOOL_CONTENT=1 to include them." Managed settings can pin the collector: "When you set an OTEL_EXPORTER_OTLP_* variable in managed settings, Claude Code removes conflicting developer-set variables at startup." The managed-MCP page says OTEL_LOG_TOOL_DETAILS=1 records MCP server and tool names.
- **Compliance API (Claude Enterprise only)** exposes Claude Code local sessions with `tool_use` and `tool_result` blocks; MCP calls appear under their `mcp__<server>__<tool>` names; tool-result text truncated to 10,000 bytes by default, up to about 1 MiB on request; retained six years by default or the org's custom period; ZDR sessions are excluded; "Tool definitions and MCP server configuration are not part of the transcript." Enterprise audit logs (https://support.claude.com/en/articles/9970975-access-audit-logs) cover 180 days of admin and content events but not Claude Code sessions.

---

## 5. Scale and maintenance

### 5.1 Parsing and indexing toolchain

- **tree-sitter** (https://tree-sitter.github.io/tree-sitter/): "a parser generator tool and an incremental parsing library" that is "robust enough to provide useful results even in the presence of syntax errors," pure C11 runtime, 24 official grammars plus community ones.
- **SCIP** (https://github.com/sourcegraph/scip ; announcement June 8, 2022, https://sourcegraph.com/blog/announcing-scip): language-agnostic precise-navigation index with indexers for TypeScript/JavaScript, Java/Scala/Kotlin, Python, Go, Rust (rust-analyzer), C/C++ (scip-clang), Ruby, C#, Dart, PHP. LSIF indexes were "on average 4x larger when gzip compressed"; "a 10x speedup in our CI when replacing lsif-node with scip-typescript"; SCIP was designed so "our backend only needs to index the files that have changed instead of the entire repository on every commit."
- **GitHub stack graphs** (December 9, 2021, https://github.blog/2021-12-09-introducing-stack-graphs/): analyse "each file completely in isolation" so only modified files are reprocessed per push; built on tree-sitter.
- **Sourcegraph auto-indexing** (https://sourcegraph.com/docs/code-navigation/auto-indexing): jobs are "scheduled periodically in the background" with a default 24-hour minimum re-index delay per repository; supports Go, TypeScript, JavaScript, Python, Ruby, JVM.

**Assessment.** Use tree-sitter for the structural layer across all languages and SCIP (or LSP queries) where resolved cross-file references are required. Trigger re-indexing from CI on merge with content-hash change detection rather than scheduled crawls.

### 5.2 Graph size estimates

| Repository | Size | Nodes | Edges | Build time | Source |
|---|---|---|---|---|---|
| Linux kernel | 28M LOC, 75K files | 2.1M | 4.9M | about 3 min | Codebase-Memory, arXiv 2603.27277 (March 28, 2026) |
| Django | not stated | 49,398 | 196,022 | about 6 s | same |
| ThingsBoard (Java) | 4,521 files | 4,691 | 8,570 | 13.8 s graph step | arXiv 2601.08773 (January 13, 2026) |
| OpenMRS Core (Java) | 1,258 files | 1,371 | 1,517 | 5.6 s graph step | same |
| Shopizer (Java) | 1,210 files | 1,158 | 1,503 | 2.8 s graph step | same |

The Linux figure implies roughly 75 nodes and 175 edges per thousand lines for a file/symbol/call schema. A "15.3K nodes and 48.2K edges per 100K LOC" figure appeared only in a search summary with no retrievable source: UNVERIFIED. Codebase-Memory reports incremental re-indexing by XXH3 content hash at "approximately four times speedup over full re-indexing," about 1.2 s for typical changes, across 66 languages. SemanticForge (arXiv 2511.07584, November 2025) claims O(|ΔR| · log n) incremental maintenance but reports no measured speeds in the abstract.

### 5.3 Neo4j sizing

**Source:** https://neo4j.com/docs/operations-manual/current/performance/memory-configuration/ and https://neo4j.com/docs/operations-manual/current/database-internals/store-formats/ Page cache guidance: `server.memory.pagecache.size = 1.2 × (total data and native indexes size)`; use `neo4j-admin server memory-recommendation`. No per-node formula is given. Block format (recommended, default for Enterprise since 5.22) uses a 128-byte static block per node holding "up to 10 labels, 6-7 properties, and 5 relationships" before overflow; limits 2^48 nodes, unlimited relationships.

**My estimate:** a 10M LOC monorepo at Linux-like density is under a million nodes and a few million edges, which is single-instance territory with a page cache in the low gigabytes. Vector embeddings per node will dominate storage if added (e.g. 1536-dim float32 is 6 KB per node, two orders of magnitude above the structural record).

### 5.4 LLM extraction vs deterministic static analysis

**Source:** Chinthareddy, "Reliable Graph-RAG for Codebases: AST-Derived Graphs vs LLM-Extracted Knowledge Graphs", arXiv 2601.08773, January 13, 2026. https://arxiv.org/html/2601.08773

| Pipeline | End-to-end cost, OpenMRS + ThingsBoard | Cost ratio | Graph build, ThingsBoard | Files skipped |
|---|---|---|---|---|
| Vector only | $0.149 | 1.00 | n/a | 0 |
| AST graph (tree-sitter) | $0.317 | 2.13 | 13.8 s | 0 |
| LLM-extracted graph | $6.80 | 45.64 | 883.7 s | 877 of 4,521 (19%) |

On Shopizer the LLM pipeline skipped 377 of 1,210 files (31%); on OpenMRS 440 of 1,258 (35%). Correctness on 15-question suites: AST graph 15/15, 14/15, 14/15; LLM graph 13, 12, 13; vector-only 6, 14, 11. Conclusion: deterministic AST graphs "provide more reliable coverage and multi-hop grounding than LLM-extracted graphs at substantially lower indexing cost."

**Assessment.** Use deterministic extraction for structure; reserve LLM calls for file- or module-level summaries, cached by content hash so they are only recomputed on change.

### 5.5 Does the graph help the agent?

- **CodeCompass**, arXiv 2602.20048, February 23, 2026: 99.4% completion on hidden-dependency tasks versus 76.2% for a vanilla agent and 78.2% for BM25, over 258 trials on a FastAPI repository. But "58% of trials with graph access made zero tool calls" until explicitly prompted. CLAUDE.md guidance on when to call the graph is part of the deployment.
- **Codebase-Memory**, arXiv 2603.27277: about 1,000 tokens per question versus about 10,000 for file exploration; 2.3 tool calls versus 4.8; but answer quality 83% versus 92% for the file-exploration agent, matching or exceeding on 19 of 31 languages for graph-native queries. Expect to trade some quality for token savings unless the tool surface is tuned.

### 5.6 Evaluation and regression testing

**Source:** CodeRAG-Bench, arXiv 2406.14497 (June 2024, revised February 2025): "current retrievers still struggle to fetch useful contexts especially with limited lexical overlap." No published benchmark targets code-graph MCP tools specifically. Practical approach: maintain a golden set of questions per repository with expected symbols and files; run it in CI after every indexer, schema, or tool-template change; track recall at k, tool-call count, tokens per question, and answer correctness, which is what both 2026 papers did by hand.

### 5.7 Staleness and drift

Three sources: unindexed commits (mitigate with CI-triggered incremental runs and an `indexed_commit` property on file nodes so tools can report age); schema evolution (version the schema and the Cypher templates together, with a contract test); CLAUDE.md referring to retired tools (Claude Code docs recommend treating CLAUDE.md edits "like any other documentation change" reviewed in pull requests, and `/doctor prompt-audit` finds stale references; https://code.claude.com/docs/en/large-codebases and https://code.claude.com/docs/en/memory).

---

## 6. Governance

### 6.1 Certifications

**Source:** https://support.claude.com/en/articles/10015870-what-certifications-has-anthropic-obtained Lists "SOC 2 Type I & Type II", "ISO 27001:2022 (Information Security Management)", "ISO/IEC 42001:2023 (AI Management Systems)", "HIPAA-ready configuration (BAA available)" for commercial products; reports via https://trust.anthropic.com (page content not retrievable by fetch). The API retention page states "Claude Code is not covered under HIPAA readiness."

### 6.2 Enforcement surface in Claude Code

**Sources:** https://code.claude.com/docs/en/managed-settings , https://code.claude.com/docs/en/settings , https://code.claude.com/docs/en/admin-setup , https://code.claude.com/docs/en/managed-mcp , https://code.claude.com/docs/en/permissions , https://code.claude.com/docs/en/hooks , https://code.claude.com/docs/en/memory

Managed settings "take precedence over local developer configuration" and are delivered as `managed-settings.json` (`/Library/Application Support/ClaudeCode/`, `/etc/claude-code/`, `C:\Program Files\ClaudeCode\`), a macOS configuration profile (`com.anthropic.claudecode`) or Windows HKLM registry key via MDM, or server-managed settings from the claude.ai console "fetched at startup and polled hourly." The HKCU registry is "writable without elevation, so treat it as a convenience default rather than an enforcement channel."

The memory docs are explicit: "Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer." The managed CLAUDE.md at the same system path (or the `claudeMd` key) "cannot be excluded by individual settings."

Keys that matter for a graph-MCP rollout:

- `allowedMcpServers` / `deniedMcpServers` matched by `serverUrl` (wildcards) or `serverCommand` (exact argv). Warning in the docs: "A serverName entry, in either list, is not a security control."
- `allowManagedMcpServersOnly: true` so user and project allowlists are ignored. `managed-mcp.json` gives exclusive control (users cannot add servers; `--mcp-config` exits at startup); `managedMcpServers` provides HTTPS-only servers alongside user servers and forbids `command`, `args`, `env`, `headersHelper`, and `${VAR}` in managed entries.
- `permissions.deny` on `mcp__<server>__write_cypher`, or `mcp__*` to remove every MCP tool; `allowManagedPermissionRulesOnly`; `permissions.disableBypassPermissionsMode`; `permissions.disableAutoMode`.
- `PreToolUse` hooks with matcher `mcp__.*` or `mcp__.*__write.*` that deny via exit code 2 or `permissionDecision: "deny"`; `allowManagedHooksOnly` blocks user, project and plugin hooks; `ConfigChange` hooks can block settings-file edits mid-session (not MDM-delivered policy).
- `sandbox.enabled` and `sandbox.network.allowedDomains`: "Denying WebFetch blocks Claude's fetch tool, but if Bash is allowed, curl and wget can still reach any URL. Sandboxing closes that gap with a network domain allowlist enforced at the OS level."
- `env` pinning for OTEL_* and `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`; `allowedProviders` (v2.1.285+) to force Bedrock/Vertex/Foundry; `forceLoginMethod` / `forceLoginOrgUUID` to keep sessions inside the ZDR organisation; `strictKnownMarketplaces` and `disableSideloadFlags` for plugin sources; `minimumVersion` / `requiredMinimumVersion`.
- The security page recommends auditing "settings changes during sessions with ConfigChange hooks" and monitoring via OpenTelemetry.

**Change management.** Keep `.mcp.json`, `.claude/settings.json`, CLAUDE.md, the MCP server source, and the Cypher tool templates in one reviewed repository with CODEOWNERS; treat the stdio command line as executable code given the OX Security finding; deploy policy via MDM so developers with local admin rights cannot silently diverge (the docs note a local admin "can edit the managed source itself, which is why MDM tooling can redeploy the profile or file on a schedule").

### 6.3 Observability

OpenTelemetry metrics (session count, cost, tokens, lines of code, permission decisions) and events (user_prompt, tool_result, api_request, tool_decision) work on every provider. The Compliance API (Claude Enterprise) and the 180-day Enterprise audit log are claude.ai-plan only; Anthropic's Compliance API announcement (March 30, 2026, https://claude.com/blog/claude-platform-compliance-api) notes it "does not log inference activities" for the Platform organisation, while the sessions endpoints cover Claude Code transcripts for Enterprise organisations. Neo4j query logs close the loop on the database side. The admin-setup page suggests a self-hosted Claude apps gateway for "a per-request audit log with IdP identity" when request-level logging is required on third-party clouds.

### 6.4 Public enterprise rollouts

- **Stripe** (claude.com case study, 2026, https://claude.com/customers/stripe): 1,370 engineers; worked with Anthropic to produce "a signed binary that could be deployed safely across the organization" after "two to three months of testing and iteration," specifically to avoid the npm dependency chain; pre-installed with pre-configured authentication ("It's pre-installed on everyone's laptop. It just works out of the box so no one has to go make an account").
- **League** (https://claude.com/customers/league): Enterprise agreement December 2025, company-wide rollout March 2026, 98% adoption; security infrastructure took about a quarter, with patient data "kept in a separate environment isolated from Claude access."
- Deloitte's 470,000-employee deployment appeared only in a search summary: UNVERIFIED.

---

## (a) Controls checklist for a Neo4j-backed code-graph MCP server

Database layer
1. Dedicated Neo4j user for the MCP server holding only reader privileges (Enterprise RBAC), distinct from the indexing identity. Label-based privileges for hot paths; property-based READ/TRAVERSE rules on an immutable `repo`/`classification` property where per-repository filtering is required. Remember DENY fails open.
2. `dbms.security.procedures.allowlist` limited to the procedures the server calls; `dbms.security.procedures.unrestricted` empty; `dbms.security.allow_csv_import_from_file_urls=false`; GDS and APOC write procedures excluded.
3. `db.transaction.timeout` set (the MCP server's 30 s is not a database guarantee); `db.memory.transaction.max` and `dbms.memory.transaction.total.max` set; `db.transaction.concurrent.maximum` lowered for the MCP user's connection pool.
4. Outbound network blocked from the Neo4j host (LOAD CSV over HTTP/FTP and APOC load are exfiltration channels); IMDSv2 on AWS; TLS on Bolt.
5. Query logging on (INFO or VERBOSE) with `obfuscate_literals` evaluated against snippet sensitivity; security log shipped to the SIEM; logs tagged with the OAuth user identity passed by the server.

MCP server layer
6. Server-side read-only mode on (NEO4J_READ_ONLY or NEO4J_MCP_READ_ONLY) as defence in depth; write tools not registered at all in production builds.
7. Default tool surface is a small set of typed, parameterised tools backed by fixed Cypher templates; raw Cypher disabled, or exposed as a separate tool denied in managed permissions and gated by a PreToolUse hook that blocks DETACH DELETE, dbms., apoc.load, apoc.periodic, LOAD CSV, CALL { } IN TRANSACTIONS and write clauses.
8. Response token limit and result sanitisation on; results wrapped with a data-not-instructions marker; nodes carry provenance (repo, path, commit, author class) so external-contributor text can be flagged or excluded.
9. HTTP transport with OAuth 2.1, PKCE S256, RFC 8707 resource indicator, audience-validated short-lived tokens; no token passthrough; per-user repository entitlements resolved from GitHub/GitLab/Bitbucket and enforced on every query. Stdio only for local development, with the binary pinned by digest.
10. Server container built from a reviewed repository, pinned by digest, with SBOM; tool descriptions hashed at approval and compared each session (mcp-scan or equivalent).

Claude Code layer (managed settings via MDM or server-managed)
11. `allowedMcpServers` by `serverUrl`/`serverCommand` plus `allowManagedMcpServersOnly: true`, or `managed-mcp.json` for exclusive control; `deniedMcpServers` for known-bad.
12. `permissions.deny` on write tools and on `Bash(curl *)`/`Bash(wget *)` where appropriate; `allowManagedPermissionRulesOnly`; `disableBypassPermissionsMode`; `allowManagedHooksOnly` with the Cypher-gating PreToolUse hook delivered as a managed hook.
13. `sandbox.enabled` with `sandbox.network.allowedDomains` covering only the model endpoint, the MCP server, and approved package registries.
14. OTEL exporter pinned in managed `env` with `OTEL_LOG_TOOL_DETAILS=1`; `forceLoginOrgUUID` for ZDR organisations; `allowedProviders` where inference must stay in-cloud; managed CLAUDE.md stating when to use the graph tools and that graph results are data.

Data and governance
15. Inference routed through Bedrock, Vertex, or Foundry in-region, or Anthropic direct with ZDR; Covered Model 30-day retention requirements recorded in the DPIA; Bedrock `DataRetentionMode` SCP where applicable.
16. CI-triggered incremental re-indexing with content hashes; `indexed_commit` on file nodes; schema and tool templates versioned together; golden-question regression suite on every indexer or template change.
17. MCP config, settings, CLAUDE.md, server code and templates in one repository with CODEOWNERS and mandatory review; change tickets for any new tool or scope.
18. Compliance API or OTEL capture of MCP tool calls retained per policy; quarterly review of Neo4j query logs for anomalous patterns (bulk reads, schema enumeration, repos outside the user's entitlement).

## (b) Open risks

- Prompt injection has no complete defence; NIST, CISA, Anthropic and the MCP spec all say so. Read-only plus egress control bounds impact; it does not prevent hijack.
- Neo4j's read-only classification can be bypassed by misclassified custom procedures (documented by Neo4j); the database-user privilege is the only hard boundary.
- Property-based DENY rules fail open; TRAVERSE rules cost performance; both are Enterprise-only.
- The MCP stdio transport remains configuration-to-execution by design; anyone who can edit MCP config on a developer machine has code execution there.
- Cross-server shadowing means the graph server's safety depends on every other server in the session.
- No signing or provenance chain exists for MCP servers in the official registry; version pinning is a local discipline.
- Quality trade-off: the graph-first agent used ten times fewer tokens but scored 83% versus 92% in the one controlled study; agents also under-use the graph without prompting.
- Covered Models (Fable 5.x, Mythos 5.x) require 30-day retention on every platform, which may conflict with ZDR commitments already made internally; on Bedrock they require aws_review human-review mode.
- ZDR does not cover MCP servers or local plaintext transcripts; the Compliance API retains Claude Code transcripts for six years by default.
- Vertex AI data-governance statements and the Five Eyes guidance body text were not retrieved from primary pages in this session.
- Neo4j Labs MCP server has no SLA; the official Go server is newer and less documented.
- Trust verification for new MCP servers is disabled in non-interactive (`-p`) mode, which is how CI and automation usually run.

## (c) URLs used

- https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks
- https://invariantlabs.ai/blog/whatsapp-mcp-exploited
- https://invariantlabs.ai/blog/mcp-github-vulnerability
- https://github.com/invariantlabs-ai/mcp-scan
- https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices
- https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
- https://modelcontextprotocol.io/specification/2025-11-25/server/tools
- https://github.com/modelcontextprotocol/registry
- https://genai.owasp.org/llm-top-10/
- https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/
- https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications/
- https://www.promptfoo.dev/docs/red-team/owasp-agentic-ai/
- https://github.com/luckyPipewrench/pipelock/blob/main/docs/owasp-agentic-top15-mapping.md
- https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2025.pdf
- https://www.nist.gov/artificial-intelligence/ai-agent-standards-initiative
- https://www.cisa.gov/resources-tools/resources/careful-adoption-agentic-ai-services
- https://www.cyber.gov.au/business-government/secure-design/artificial-intelligence/careful-adoption-of-agentic-ai-services (timed out; not read)
- https://www.crowell.com/en/insights/client-alerts/american-and-allied-cyber-agencies-issue-first-joint-guidance-on-securing-agentic-ai
- https://jfrog.com/blog/2025-6514-critical-mcp-remote-rce-vulnerability/
- https://www.recordedfuture.com/blog/anthropic-mcp-inspector-cve-2025-49596
- https://www.theregister.com/2026/01/20/anthropic_prompt_injection_flaws/
- https://thehackernews.com/2026/04/anthropic-mcp-design-vulnerability.html
- https://generalanalysis.com/blog/supabase-mcp-blog
- https://supabase.com/blog/defense-in-depth-mcp
- https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/
- https://developer.microsoft.com/blog/protecting-against-indirect-injection-attacks-mcp/
- https://labs.cloudsecurityalliance.org/agentic/agentic-mcp-security-best-practices-v1/
- https://claude.com/blog/ciso-guide-to-agentic-ai
- https://code.claude.com/docs/en/security
- https://code.claude.com/docs/en/mcp
- https://code.claude.com/docs/en/managed-mcp
- https://code.claude.com/docs/en/managed-settings
- https://code.claude.com/docs/en/settings
- https://code.claude.com/docs/en/admin-setup
- https://code.claude.com/docs/en/permissions
- https://code.claude.com/docs/en/hooks
- https://code.claude.com/docs/en/memory
- https://code.claude.com/docs/en/large-codebases
- https://code.claude.com/docs/en/monitoring-usage
- https://code.claude.com/docs/en/data-usage
- https://code.claude.com/docs/en/zero-data-retention
- https://code.claude.com/docs/en/third-party-integrations
- https://code.claude.com/docs/en/amazon-bedrock
- https://code.claude.com/docs/en/google-vertex-ai
- https://platform.claude.com/docs/en/manage-claude/api-and-data-retention
- https://platform.claude.com/docs/en/manage-claude/compliance-sessions
- https://claude.com/blog/claude-platform-compliance-api
- https://support.claude.com/en/articles/9970975-access-audit-logs
- https://support.claude.com/en/articles/10015870-what-certifications-has-anthropic-obtained
- https://trust.anthropic.com (content not retrievable by fetch)
- https://www.anthropic.com/legal/commercial-terms
- https://www.theregister.com/ai-and-ml/2026/09/02/anthropic-promises-zero-data-retention-but-customers-must-check-it-worked/5293789
- https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html
- https://docs.cloud.google.com/vertex-ai/generative-ai/docs/data-governance (navigation shell only; not read)
- https://claude.com/customers/stripe
- https://claude.com/customers/league
- https://github.com/github/github-mcp-server
- https://github.com/neo4j-contrib/mcp-neo4j
- https://github.com/neo4j-contrib/mcp-neo4j/blob/main/servers/mcp-neo4j-cypher/README.md
- https://github.com/neo4j/mcp
- https://neo4j.com/blog/developer/production-proofing-cypher-mcp-server/
- https://neo4j.com/docs/operations-manual/current/authentication-authorization/manage-privileges/
- https://neo4j.com/docs/operations-manual/current/authentication-authorization/property-based-access-control/
- https://neo4j.com/docs/operations-manual/current/monitoring/transaction-management/
- https://neo4j.com/docs/operations-manual/current/monitoring/logging/
- https://neo4j.com/docs/operations-manual/current/configuration/configuration-settings/
- https://neo4j.com/docs/operations-manual/current/security/securing-extensions/
- https://neo4j.com/docs/operations-manual/current/performance/memory-configuration/
- https://neo4j.com/docs/operations-manual/current/database-internals/store-formats/
- https://neo4j.com/docs/cypher-manual/current/clauses/load-csv/
- https://www.varonis.com/blog/neo4jection-secrets-data-and-cloud-exploits
- https://agentthreatrule.org/en/rules/ATR-2026-02100
- https://github.com/sourcegraph/scip
- https://sourcegraph.com/blog/announcing-scip
- https://sourcegraph.com/docs/code-navigation/auto-indexing
- https://github.blog/2021-12-09-introducing-stack-graphs/
- https://tree-sitter.github.io/tree-sitter/
- https://arxiv.org/abs/2601.08773 and https://arxiv.org/html/2601.08773
- https://arxiv.org/abs/2603.27277 and https://arxiv.org/html/2603.27277
- https://arxiv.org/abs/2602.20048
- https://arxiv.org/abs/2511.07584
- https://arxiv.org/abs/2406.14497
