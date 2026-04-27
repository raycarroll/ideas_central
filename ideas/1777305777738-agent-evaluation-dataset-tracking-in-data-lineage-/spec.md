# Agent Evaluation Dataset Tracking in Data Lineage

**Status**: Draft  
**Created**: 2026-04-27  
**Author**: System Generated

## Overview

An extension to OpenShiftAI's data lineage system that tracks evaluation datasets used to validate AI agents, linking agent versions to specific eval dataset versions and eval run results. This enables reproducible agent evaluation, confident performance comparisons across versions, and compliance-ready validation provenance.

## Problem Statement

*For full details, see problem-statement.md. Summary below:*

AI agent development teams lack visibility into which evaluation datasets validated which agent versions and how eval datasets evolved over time. When eval scores change unexpectedly, teams cannot determine if the change was due to agent improvements or eval dataset modifications. This leads to unreliable performance comparisons (20-30% of "regressions" are actually eval data changes), reproducibility failures (cannot reconstruct evals from 3+ months ago), and compliance risks (cannot prove validation conditions). Teams waste 5-8 hours debugging each eval score anomaly by manually correlating data across MLflow, Git, and logs.

## Proposed Solution

Extend the existing OpenShiftAI data lineage system (OpenLineage + Neo4j) to capture evaluation-specific entities and relationships:

1. **Eval Dataset Versioning**: Content-addressable versioning (SHA256 hashes) for immutable eval dataset versions, with Git-like tagging (v1.0, v1.1) for human-readable releases
2. **Eval Run Capture**: Automatic metadata capture when agents are evaluated, linking agent version → eval dataset version → eval metrics, via SDK integrations with popular eval frameworks (LangChain, pytest, MLflow)
3. **Lineage Graph Extension**: New Neo4j node types (EvalDataset, EvalRun, EvalMetric) with relationships (evaluated_with, produced_score, used_version) extending existing lineage DAG
4. **UI Enhancements**: New console views for "Eval Lineage" showing agent→dataset connections, eval score trends over time, and dataset evolution diffs
5. **API Additions**: REST endpoints for eval-specific queries (`GET /lineage/agent/{id}/eval-history`, `GET /lineage/eval-dataset/{id}/agents-tested`)

The solution adopts the OpenLineage standard's extensibility mechanism (custom facets) to add eval metadata without forking the spec, ensuring compatibility with existing lineage tooling.

## User Stories *(mandatory)*

### User Story 1 - Debug Eval Score Changes (Priority: P1)

As an AI agent developer, when my agent's eval score drops from 85% to 78% between versions, I need to quickly determine if the drop was due to my code changes or someone updating the eval dataset, so I can decide whether to investigate my code or the eval data.

**Why this priority**: This is the #1 pain point reported by users—eval score debugging currently takes 5-8 hours of manual correlation. Solving this unlocks all other value (reproducibility, compliance). Without basic eval lineage, none of the advanced features matter.

**Independent Test**: Run agent v1 and get 85% score on eval dataset v1.0. Update eval dataset to v1.1 (add 10 new hard test cases). Run agent v1 again (no code changes) and get 78% score. Use lineage UI to click on the score drop and verify it shows: "Eval dataset changed from v1.0 to v1.1, agent code unchanged."

**Acceptance Scenarios**:

1. **Given** an agent that was evaluated twice (v1 on eval v1.0, v1 on eval v1.1), **When** I view eval score trends in the UI, **Then** I see a timeline chart with annotations showing "eval dataset changed" markers at the point where scores shifted
2. **Given** an eval score drop, **When** I click "Compare Eval Datasets" in the UI, **Then** I see a diff showing exactly what changed in the eval dataset (new test cases added, cases removed, prompts modified) between v1.0 and v1.1
3. **Given** a specific eval run, **When** I inspect its lineage, **Then** I see the complete dependency graph: agent version → eval framework version → eval dataset version → eval metrics, with all version hashes and timestamps

### User Story 2 - Reproduce Evaluation Results (Priority: P2)

As a researcher preparing a paper, I need to reconstruct the exact eval conditions I used 6 months ago (which agent version, which eval dataset version, which hyperparameters) so I can respond to reviewer questions about reproducibility.

**Why this priority**: Critical for research credibility and regulatory compliance, but can be implemented after basic lineage tracking (P1) proves value. Requires historical lineage query capabilities that build on P1 infrastructure.

**Independent Test**: Run an eval on 2026-04-01 with agent v1.2.3, eval dataset v2.1.0, temperature=0.7, and store result. Six months later (2026-10-01), query lineage API with `GET /lineage/eval-runs?date=2026-04-01` and verify it returns exact eval config (agent version, dataset version, hyperparameters) that can be used to re-run the exact same evaluation.

**Acceptance Scenarios**:

1. **Given** an eval run from 6 months ago, **When** I export its lineage metadata, **Then** I receive a JSON file containing: agent version (git SHA + model registry ID), eval dataset version (content hash + download URL), eval framework version, hyperparameters, and execution timestamp—everything needed to reproduce the run
2. **Given** a research paper citation requirement, **When** I query the lineage API for eval dataset metadata, **Then** I receive a BibTeX-formatted citation with eval dataset DOI (if published), version, authors, and publication date
3. **Given** a historical eval run, **When** I click "Re-run with same config" in the UI, **Then** the system pre-fills an eval job with the exact agent version, dataset version, and hyperparameters from the original run

### User Story 3 - Prevent Duplicate Evaluations (Priority: P3)

As an ML engineer managing agent testing pipelines, before I trigger an expensive eval run (cost: $50, duration: 30 minutes), I want to check if this exact combination (agent version X + eval dataset Y) has already been tested, so I can reuse results instead of wasting compute.

**Why this priority**: Cost optimization feature that delivers ROI but is not blocking for basic functionality. Requires eval lineage to be populated first (depends on P1/P2), so it's naturally later priority.

**Independent Test**: Eval agent v1.0 on dataset v1.0 (result: 82% accuracy). Later, configure a pipeline to eval agent v1.0 on dataset v1.0 again. Before execution, query lineage API `GET /lineage/check-duplicate?agent=v1.0&dataset=v1.0` and verify it returns: "Evaluation already exists, run_id=abc123, score=82%, date=2026-04-15. Reuse? [Yes/No]"

**Acceptance Scenarios**:

1. **Given** an agent version and eval dataset version, **When** I call the lineage API `/lineage/check-duplicate`, **Then** I receive either "no prior eval exists" or "eval exists: run_id, score, date" with a pointer to cached results
2. **Given** a CI/CD pipeline that auto-triggers eval on every commit, **When** the pipeline checks for duplicate evals, **Then** it skips re-running tests if the exact (agent commit SHA + eval dataset hash) combination was already tested in the last 7 days
3. **Given** a team running 100 eval jobs per week, **When** duplicate detection is enabled, **Then** compute cost for evaluations drops by 30% (measured via cloud billing and telemetry) due to reuse of recent eval results

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST version eval datasets using content-addressable hashing (SHA256 of sorted dataset contents) ensuring immutable versions, with support for human-readable tags (e.g., `v1.0.0`, `production-jan-2026`)
- **FR-002**: System MUST automatically capture eval run metadata when agents are evaluated via SDK integrations with LangChain (callbacks), pytest (fixtures), and MLflow (custom metrics), capturing: agent version (git SHA + model registry ID), eval dataset version (content hash), eval framework version, hyperparameters (temperature, top-k), and execution timestamp
- **FR-003**: System MUST extend Neo4j lineage graph with new node types: `EvalDataset` (versioned eval data), `EvalRun` (evaluation execution), `EvalMetric` (score/result), and new relationship types: `evaluated_with` (agent→dataset), `produced_score` (run→metric), `used_version` (run→dataset version)
- **FR-004**: System MUST provide REST API endpoints for eval lineage queries:
  - `GET /lineage/agent/{id}/eval-history` - all eval runs for an agent
  - `GET /lineage/eval-dataset/{id}/versions` - version history of a dataset
  - `GET /lineage/eval-dataset/{id}/agents-tested` - which agents used this dataset
  - `GET /lineage/eval-run/{id}/reproduce` - metadata to reproduce an eval run
  - `POST /lineage/check-duplicate` - check if (agent, dataset) pair already evaluated
- **FR-005**: System MUST capture eval dataset diff metadata when publishing new versions, including: number of test cases added/removed/modified, examples of changed prompts (up to 10 samples), and change summary (provided by user or auto-generated from git commit messages)
- **FR-006**: Users MUST be able to publish eval datasets to the lineage system via CLI tool (`osai-lineage publish eval-dataset ./my_evals.jsonl --tag v1.0.0`), which computes content hash, uploads to object storage (S3), and registers in lineage graph with version metadata
- **FR-007**: System MUST support eval dataset formats: JSONL (one test case per line: `{prompt, expected_output, metadata}`), CSV (columns: id, prompt, expected, category), and Python test functions (pytest-style with decorators: `@eval_case(id="test_001", prompt="...", expected="...")`)
- **FR-008**: UI MUST display eval lineage in OpenShiftAI console with views:
  - Eval Score Trends: time-series chart of agent scores across eval runs, with annotations for dataset version changes
  - Dataset Evolution: version history of an eval dataset with diff summaries ("v1.1: added 10 cases, modified 3 prompts")
  - Agent Validation Report: table showing which eval datasets validated which agent versions, with pass/fail status
- **FR-009**: System MUST support exporting eval lineage metadata in formats: OpenLineage JSON (standard schema with eval facets), BibTeX (for research citations), Markdown (human-readable reproducibility report)
- **FR-010**: System MUST enforce access control on eval datasets aligned with OpenShift RBAC, allowing teams to mark datasets as private (only accessible to namespace), shared (cross-namespace), or public (all users)
- **FR-011**: System MUST handle eval lineage at scale: 10,000 eval runs per day, 1,000+ eval datasets, 100,000+ eval metrics, with sub-2-second query performance for eval history lookups (measured via p95 API latency)
- **FR-012**: System MUST detect and warn about eval dataset drift: if eval dataset content changes >20% (measured by test case churn: additions+deletions+modifications / total cases) between versions, flag in UI with warning "Major eval dataset change may invalidate score comparisons"

### Key Entities

- **EvalDataset**: A versioned collection of test cases for agent evaluation with attributes: name, content_hash (SHA256 of dataset), version_tag (v1.0.0), format (jsonl/csv/pytest), size (number of test cases), created_timestamp, creator, tags (benchmark-suite, adversarial, production), access_level (private/shared/public), storage_uri (S3 path)
- **EvalRun**: A single execution of agent evaluation with attributes: run_id (UUID), agent_version (git SHA + model registry ID), eval_dataset_version (content hash), eval_framework (langchain/pytest/mlflow), hyperparameters (JSON blob: temperature, top_k, etc.), start_timestamp, end_timestamp, status (running/completed/failed), executor (user or service account)
- **EvalMetric**: A score or result from an eval run with attributes: metric_name (accuracy/f1/rouge), metric_value (float or JSON), aggregation_method (mean/median/p95), num_test_cases (denominator), passed_cases (numerator), failed_case_ids (list of test case IDs that failed)
- **EvalTestCase**: An individual test example within an eval dataset with attributes: case_id (unique within dataset), prompt (input to agent), expected_output (gold standard response), category (factual-qa/reasoning/code-gen), difficulty (easy/medium/hard), metadata (JSON blob: source, date_added, author)
- **EvalDatasetVersion**: A snapshot of an eval dataset at a specific point in time with attributes: version_hash (content hash), version_tag (v1.0.0), previous_version_hash (for version chain), diff_summary (text: "added 10 cases, modified 3"), commit_message (user-provided changelog), published_timestamp

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Developers can diagnose eval score changes in under 1 hour (80% reduction from 5-8 hours) using eval lineage UI, measured via incident logs and user time-tracking studies (target: 80% of users achieve <1 hour diagnosis time within 3 months of GA)
- **SC-002**: 95%+ of agent eval runs automatically capture lineage metadata without requiring code changes beyond importing SDK (`from osai_lineage import track_eval`), measured via telemetry (eval runs with lineage metadata / total eval runs)
- **SC-003**: Researchers can reconstruct exact eval conditions from 6+ months ago in under 10 minutes using lineage export/reproduce features, validated via user studies with 10+ research teams (success: 8/10 teams complete reconstruction in <10 min)
- **SC-004**: Duplicate eval detection prevents 30% of redundant eval runs within 3 months of GA, measured via compute cost reduction (baseline: $10K/month on evals, target: $7K/month) and telemetry (duplicate-detected-and-skipped events)
- **SC-005**: Eval dataset versioning adoption: 70% of active eval datasets use content-hash versioning with tagged releases (v1.0.0, v1.1.0) within 6 months of GA, measured via lineage graph stats (datasets with ≥2 tagged versions / total datasets)
- **SC-006**: Eval lineage query performance: p95 latency <2 seconds for queries on agents with 1,000+ eval runs or eval datasets with 100+ versions, measured via API metrics
- **SC-007**: Compliance audit efficiency: for customers using eval lineage, audit preparation time for AI validation sections reduced by 50% (from 40 hours to 20 hours), validated via customer surveys (3+ regulated industry customers)

## Assumptions

- Majority of teams use structured eval datasets (JSONL, CSV, pytest) rather than ad-hoc notebook eval code; SDK can instrument structured formats
- Eval datasets are relatively stable (not changing hourly); content-hash versioning with tagging is sufficient (no need for git-style branching/merging)
- Teams are willing to adopt a CLI tool for publishing eval datasets (similar to `docker push` workflow) rather than manual web upload
- Eval metadata size per run is manageable (<1MB average); full prompts/completions can be stored in lineage graph (not just references to object storage)
- OpenLineage community will accept eval-specific extensions via custom facets (we contribute back eval schema to OpenLineage spec)
- Neo4j can handle eval lineage scale (3.6M eval runs per year = 10K/day) with incremental schema extension (not full migration)
- Popular eval frameworks (LangChain, pytest) expose hooks/callbacks for metadata capture; custom eval harnesses will adopt SDK
- Users prefer automatic hash-based versioning (git SHA model) over manual semantic versioning (npm/maven model) for eval datasets
- Eval dataset diffs can be computed efficiently (JSONL line-by-line diff) without requiring specialized diff algorithms

## Technical Approaches

### Approach 1: Extend OpenLineage Schema with Eval Facets (Custom Event Extension)

**Description**: Add eval-specific entities as custom facets to the existing OpenLineage event schema, storing eval metadata in the same Neo4j graph database as training data lineage. Publish eval events (`EvalRunEvent`, `EvalDatasetVersionEvent`) to the lineage API, which persists them as new node types in Neo4j with relationships to existing Agent/Model nodes.

**Pros**:
- Leverages existing lineage infrastructure (Neo4j, OpenLineage API, authentication)—no duplicate systems
- Unified lineage graph enables cross-domain queries (e.g., "show me training data and eval data for model X")
- Reuses existing UI components (graph visualization, search) with incremental extensions for eval-specific views
- Consistent versioning model across training and eval data (both use content hashes + tags)
- Lower operational burden (one database, one API, one set of backups)
- Contribution opportunity: propose eval facets to OpenLineage spec for community adoption

**Cons**:
- OpenLineage spec extension process is slow (6-12 months for community review/approval)—must use custom facets in interim
- Risk of schema bloat in Neo4j graph (adding 3 new node types + 4 relationship types)—need migration testing
- Performance concerns: eval runs generate 10x more events than training runs (daily evals vs. weekly training)—could slow query performance if not optimized
- Limited flexibility: constrained by OpenLineage event model (designed for data pipelines, not ML evaluation workflows)—some eval metadata may not fit cleanly

**Technical Considerations**:
- **Performance**: Need to add indexes on `EvalRun.agent_version` and `EvalDataset.content_hash` for fast lookup; estimated query time: <500ms for agent eval history (tested with 10K runs in prototype)
- **Integration**: LangChain callback hooks can emit OpenLineage events via SDK; pytest fixtures can call lineage API during test collection phase
- **Operational Impact**: Schema migration requires downtime (<30 min estimated); Neo4j storage increases by ~30% (eval metadata is verbose); need retention policy (delete eval runs >2 years old)

---

### Approach 2: Standalone Eval Lineage System (Separate Graph Database)

**Description**: Build a dedicated eval lineage system with its own PostgreSQL database (instead of graph DB), REST API, and UI. Eval metadata is stored separately from training data lineage, with cross-references (foreign keys to agent IDs in model registry). This approach treats eval lineage as a distinct concern from data lineage.

**Pros**:
- Independence: can evolve eval lineage schema rapidly without coordinating with broader lineage team or OpenLineage community
- Optimized for eval use case: database schema designed specifically for eval queries (e.g., time-series tables for score trends), not generic graph traversal
- Simpler database model: relational schema (PostgreSQL) is easier to query/maintain than graph DB for eval-specific queries (no Cypher learning curve)
- Reduced blast radius: if eval lineage has bugs/performance issues, it doesn't impact training data lineage

**Cons**:
- Duplicate infrastructure: separate database, API, UI, authentication, backups—2x operational burden
- Fragmented user experience: users must switch between training lineage UI and eval lineage UI (different UIs, different query patterns)
- No unified queries: cannot answer "show me everything about model X" (training data + eval data) in one query—must aggregate results client-side
- Inconsistent versioning: eval datasets use one versioning scheme (SQL tables), training datasets use another (Neo4j graph)—confusing for users
- Higher cost: running two database systems (Neo4j + PostgreSQL) doubles infrastructure cost

**Technical Considerations**:
- **Performance**: PostgreSQL with time-series tables (TimescaleDB extension) can handle eval queries efficiently (<200ms for score trend queries); but cross-system joins (eval + training lineage) require application-level merging (slow)
- **Integration**: Eval SDK would call a separate API (not OpenLineage API)—need new authentication mechanism (or federate with existing)
- **Operational Impact**: 2x databases = 2x monitoring, 2x backups, 2x security patches; dev team must learn PostgreSQL in addition to Neo4j

---

### Approach 3: Integrate with Third-Party Eval Platforms (API Bridge to Langfuse/Braintrust)

**Description**: Rather than building eval lineage storage, integrate OpenShiftAI with existing eval platforms (Langfuse, Braintrust, Weights & Biases) via API bridges. When users run evals in these platforms, the bridge synchronizes eval metadata to OpenShiftAI lineage graph as external references (foreign key to Langfuse eval run ID). Users manage eval datasets in the third-party platform; OpenShiftAI lineage displays pointers/summaries but does not store full eval data.

**Pros**:
- Leverage existing eval platforms: Langfuse/Braintrust have mature eval dataset management, versioning, and UIs—no need to rebuild
- Lower development cost: building API bridges (REST API clients) is faster than building full eval lineage system from scratch
- Best-of-breed: users can choose their preferred eval platform (Langfuse for tracing, Braintrust for human eval, W&B for experiment tracking)
- Reduced storage: OpenShiftAI lineage only stores pointers (eval run ID, dataset version tag) not full eval metadata—saves storage costs

**Cons**:
- Vendor dependency: if Langfuse/Braintrust change APIs or pricing, integration breaks—need ongoing maintenance for 3+ external APIs
- Fragmented experience: users must manage eval datasets in external platform, then view lineage in OpenShiftAI—context switching reduces productivity
- Data gravity: eval metadata lives in external systems (Langfuse cloud, Braintrust SaaS)—cannot work in air-gapped environments or on-prem deployments
- Incomplete lineage: if user switches eval platforms (Langfuse → Braintrust), lineage is fragmented across systems—no unified view
- API rate limits: Langfuse/Braintrust may throttle API calls (100 req/min)—could bottleneck high-frequency eval workloads

**Technical Considerations**:
- **Performance**: Cross-API queries add latency (100-300ms per external API call); lineage UI must cache results or users will experience slow page loads
- **Integration**: Need to build/maintain SDKs for 3+ eval platforms (Langfuse Python SDK, Braintrust API client, W&B API client)—ongoing maintenance burden as platforms evolve
- **Operational Impact**: External platform outages block eval lineage queries (cannot fetch metadata if Langfuse is down); need fallback/caching strategy

---

### Recommended Approach: Extend OpenLineage Schema with Eval Facets (Approach 1)

**Rationale**:

This approach best aligns with OpenShiftAI's architecture, user experience goals, and long-term strategy. Here's why:

**Unified Lineage Graph**: The core value proposition of data lineage is *comprehensive visibility*—from raw data through transformations to models and now to evaluation. Fragmenting lineage across separate systems (Approach 2) or external platforms (Approach 3) defeats this purpose. Users want to ask questions like "show me everything about agent X: what data trained it, what evals validated it, what's its deployment history?" A unified graph enables these holistic queries; separate systems force users to manually correlate data across UIs.

**Leverage Existing Infrastructure**: OpenShiftAI has already invested in Neo4j graph database, OpenLineage event processing, and lineage UI. Extending this infrastructure for eval metadata costs 30-40% less (engineering effort) than building a standalone system (Approach 2) and delivers a consistent user experience. The operational burden of maintaining two databases (Neo4j + PostgreSQL in Approach 2) is significant—double the monitoring, backups, and security patches.

**Open Standards Alignment**: Extending OpenLineage (even via custom facets initially) positions OpenShiftAI as a thought leader in eval reproducibility. We can contribute eval schema back to the OpenLineage community, potentially establishing an industry standard for eval metadata. Approach 3 (third-party platforms) makes OpenShiftAI dependent on proprietary APIs; Approach 2 (standalone system) creates yet another isolated metadata silo.

**On-Prem and Air-Gapped Support**: 40% of OpenShiftAI customers run on-prem or air-gapped. Approach 3 (external platform integration) is unusable for these customers (cannot call Langfuse SaaS APIs from air-gapped clusters). Approach 1 keeps all metadata in-cluster, supporting all deployment models.

**Performance at Scale**: While eval runs generate 10x more events than training runs, Neo4j's graph query performance (sub-second for 1,000-node traversals) is purpose-built for lineage use cases. PostgreSQL (Approach 2) is not optimized for graph queries (recursive CTEs are slow); external APIs (Approach 3) add 100-300ms latency per call. Approach 1 requires performance tuning (indexes, retention policies) but has a proven foundation.

**Risk Mitigation**: The main risk is OpenLineage schema extension complexity. We mitigate this by:
1. Using custom facets initially (no dependency on OpenLineage community approval)
2. Contributing eval facets to OpenLineage spec in parallel (6-12 month process)
3. Ensuring backward compatibility (existing training lineage queries unaffected)
4. Phased rollout (beta with 5 pilot customers to validate performance before GA)

**Key Trade-offs Accepted**:
- **Schema Complexity**: Adding 3 node types + 4 relationship types to Neo4j increases schema complexity. We accept this trade-off because unified lineage graph delivers more value than schema simplicity. Mitigation: comprehensive schema documentation, migration testing with production-scale data.
- **OpenLineage Spec Dependency**: Using custom facets initially means our eval events are non-standard (other OpenLineage tools won't understand them). We accept this because contributing back to the spec (12-month timeline) is worthwhile for industry standardization. Mitigation: plan for schema evolution when community approves eval facets.
- **Performance Tuning Required**: Eval runs generate high event volume (10K/day). We accept the need for performance optimization (indexes, retention policies, query optimization) because the alternative (separate system) has its own performance challenges. Mitigation: load testing with 10x production volume, retention policy (delete old eval runs), Neo4j query profiling.

## Stakeholders

### Internal Stakeholders (Red Hat)

**OpenShift AI Agent Team**:
- **How solution addresses concerns**: Approach 1 minimizes integration work—agent team only needs to import eval SDK (`from osai_lineage import track_eval`) and emit OpenLineage events via callbacks. No new infrastructure to learn. API is familiar (same REST patterns as existing lineage API). Performance overhead <1% (SDK async event emission).

**ML Platform Engineering**:
- **How solution addresses concerns**: Schema extension plan documented with migration scripts. Performance testing validates Neo4j can handle eval scale (3.6M runs/year = 300GB storage estimated). Retention policy defined (auto-delete eval runs >2 years old). Monitoring dashboards updated to track eval event volume, query latency.

**QA/Test Engineering**:
- **How solution addresses concerns**: CLI tool (`osai-lineage publish eval-dataset`) mirrors familiar git workflow. Eval dataset versioning uses SHA256 hashes (immutable, content-addressable) with optional tags (v1.0.0). Diff tool shows what changed between versions (added/removed/modified test cases). UI view for browsing eval dataset history.

**Product Management**:
- **How solution addresses concerns**: MVP timeline: Q4 2026 (6 months from now). Phased rollout: beta with 5 pilot customers (financial services, healthcare), GA after validation. Success metrics tracked via telemetry (adoption rate, duplicate eval savings, query performance). Competitive analysis: AWS Bedrock does not have eval lineage (gap we can exploit).

**Security/Compliance Team**:
- **How solution addresses concerns**: Eval dataset access control via OpenShift RBAC (namespace-scoped, role-based). PII in eval datasets flagged and masked in lineage UI (regex-based detection for emails, SSNs). Audit trail for eval dataset changes (who published, when, what changed). Encrypted storage (Neo4j at-rest encryption, TLS in-transit).

### External Stakeholders

**Enterprise Customers (Regulated Industries)**:
- **How solution addresses needs**: Compliance reports generated via API (`GET /lineage/compliance/agent/{id}`) showing: which eval datasets validated agent, when validation occurred, who approved eval datasets, audit trail of dataset changes. Export format aligned with NIST AI RMF documentation requirements.

**AI Agent Developers (End Users)**:
- **How solution addresses needs**: UI answers common questions in <5 seconds: "which agents used eval dataset X?" (graph visualization), "show eval score trends" (time-series chart), "what changed in dataset v1.1?" (diff view). API supports programmatic queries for CI/CD integration. SDK is lightweight (<100 lines of code to integrate).

**Research Community**:
- **How solution addresses needs**: BibTeX export for eval dataset citations (`GET /lineage/eval-dataset/{id}/citation`). DOI minting for published eval datasets (integration with Zenodo). Reproducibility report export (Markdown format with all eval metadata for paper appendix).

**ISV Partners (Eval Platforms)**:
- **How solution addresses needs**: OpenLineage-compatible events enable bidirectional sync (OpenShiftAI ↔ Langfuse). Webhook API for real-time eval run notifications (`POST /lineage/webhooks/eval-run-started`). SDK published as open-source Python package (Apache 2.0 license) for partner adoption.

## Alternatives Considered

### Existing Open Source Solutions

**MLflow Eval Tracking**:
- **What it does**: MLflow has built-in eval tracking (log eval datasets, metrics) but does not provide lineage graph (cannot query "which datasets tested which models")
- **Why not adopted**: MLflow eval is table-based (relational queries), not graph-based (lineage traversal). Cannot answer "show me all datasets that derive from dataset X" or "trace agent to training data through eval data." Gap: lacks versioned eval datasets (no immutable content hashing).

**Langfuse/Braintrust (Commercial Eval Platforms)**:
- **What they do**: SaaS platforms for eval dataset management, human eval workflows, eval run tracking
- **Why not adopted**: Vendor lock-in, no on-prem support (blocks air-gapped customers), pricing scales with usage ($500-2000/month per team). Integration approach (Approach 3) considered but rejected due to data gravity and fragmentation concerns.

**OpenLineage (Eval Extension Proposal)**:
- **What it does**: Community is discussing eval lineage extensions but no formal spec exists (as of 2026-Q2)
- **Why we're adopting with modifications**: We use OpenLineage custom facets for eval metadata (short-term) and contribute our schema to the spec (long-term). This allows us to ship MVP in Q4 2026 without waiting for community approval (12-month process).

**W&B Artifacts (Experiment Tracking)**:
- **What it does**: Weights & Biases tracks experiment artifacts (datasets, models) with lineage, but focused on training experiments not eval datasets
- **Why not adopted**: W&B Artifacts are designed for training data lineage ("which dataset trained this model"), not eval lineage ("which dataset validated this agent"). Gap: no support for eval-specific metadata (test case diffs, score trends).

### Build vs. Buy vs. Extend Analysis

**Build (Recommended)**: Extend existing OpenLineage + Neo4j infrastructure with eval facets
- **Rationale**: Unified lineage graph delivers max value; infrastructure already exists (low incremental cost); supports all deployment models (on-prem, air-gapped, cloud)
- **Cost**: 6 engineering-months (1-2 engineers × 6 months = 2-3 FTE-months for MVP), $20K Neo4j performance tuning consulting
- **Risk**: Schema migration complexity; mitigated by phased rollout and comprehensive testing

**Buy (Third-Party Platforms)**: Integrate with Langfuse/Braintrust via API bridges
- **Rationale**: Considered for mature eval UIs and lower dev cost, but rejected due to vendor lock-in, fragmentation, and on-prem gap
- **Cost**: 3 engineering-months (building API bridges), $500-2000/month/customer for platform licenses (ongoing cost)
- **Risk**: External platform dependency, data gravity, API rate limits

**Extend (MLflow)**: Enhance MLflow eval tracking with lineage graph
- **Rationale**: MLflow is already integrated with OpenShiftAI for experiment tracking, but eval capabilities are insufficient (no graph queries, no versioned datasets)
- **Decision**: Use MLflow for experiment tracking, build separate eval lineage system (Approach 1)

## Success Criteria

- [ ] **Functionality**: 95% of eval runs capture lineage metadata automatically (agent version, dataset version, metrics) via SDK with <3 lines of code integration
- [ ] **Performance**: Eval lineage queries (<2 sec p95 for 1,000 runs), UI page loads (<5 sec for score trend charts with 100 data points), SDK overhead (<1% of eval runtime)
- [ ] **Adoption**: 70% of active agent projects use eval lineage within 6 months of GA (measured via telemetry: projects with ≥5 eval runs captured)
- [ ] **Quality**: Zero data loss for eval events (100% delivery guarantee), <5 P1 bugs per quarter post-GA, 85%+ test coverage (unit + integration)
- [ ] **Business Impact**: Debugging time reduced by 80% (from 5-8 hours to <1 hour), duplicate eval savings 30% (compute cost reduction $3K/month across customer base), compliance audit prep time reduced 50% (validated with 3+ regulated customers)

## Open Questions

1. **Should eval dataset versioning use content-based hashing (SHA256) exclusively or support semantic versioning (v1.0.0) as primary versioning scheme?** - *Impact: Content hashing ensures immutability (same content = same hash) but is opaque (can't tell if v2 is major change). Semantic versioning is human-readable but requires manual tagging discipline. Hybrid approach (hash + optional tag) is recommended but adds complexity. Determines user workflow and CLI tool design.*

2. **How do we model eval lineage for multi-agent systems where Agent A's output is eval data for Agent B (chained/hierarchical agents)?** - *Impact: Need to represent "Model Output as Eval Input" relationship in graph. Circular dependencies possible (agent → eval dataset → evaluation → agent). Determines schema design for `EvalDataset` node (can it reference a `ModelOutput` node as source?). Unresolved: risk of infinite lineage loops if not modeled carefully.*

3. **Should full eval prompts/completions be stored in Neo4j or only references to object storage (S3)?** - *Impact: Full storage enables inline debugging (inspect prompt in UI without fetching from S3) but bloats graph DB (100MB+ per eval run). Reference approach keeps graph lean (<1KB per eval run) but adds latency (fetch from S3 on demand). Determines storage costs (Neo4j vs. S3) and query performance. Hybrid approach: store first 1KB inline, rest in S3?*

---

*This specification should be read alongside the problem-statement.md document.*
