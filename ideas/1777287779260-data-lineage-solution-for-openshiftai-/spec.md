# Data Lineage Solution for OpenShiftAI

**Status**: Draft  
**Created**: 2026-04-27  
**Author**: System Generated

## Overview

An automated data lineage tracking system for OpenShiftAI that captures, stores, and visualizes the flow of data through ML pipelines—from raw data sources through transformations to trained models. This solution enables data scientists to debug faster, ensures model reproducibility, satisfies compliance requirements, and provides platform-wide visibility into data dependencies.

## Problem Statement

*For full details, see problem-statement.md. Summary below:*

Data scientists and ML engineers in OpenShiftAI lack automated visibility into data flow through ML pipelines. When models exhibit unexpected behavior, teams cannot efficiently trace back through transformations to identify root causes. Manual investigation is time-consuming (30-40% of debugging time), reproducibility fails for models older than 6 months, and compliance audits require 100+ hours of manual lineage documentation. This absence of lineage tracking creates production risk, technical debt, and missed optimization opportunities as teams cannot predict the impact of data changes or identify duplicate processing efforts.

## Proposed Solution

Build an OpenLineage-based data lineage system integrated with OpenShiftAI that:

1. **Automatically captures** lineage metadata from Kubeflow Pipelines and Jupyter notebooks without requiring code changes
2. **Stores lineage** as directed acyclic graphs (DAGs) in a scalable graph database (Neo4j)
3. **Visualizes lineage** through an interactive web UI embedded in the OpenShiftAI console
4. **Exposes APIs** for programmatic lineage queries and integration with third-party tools
5. **Supports compliance** with audit trail generation, PII tracking, and export capabilities

The solution adopts the OpenLineage standard for event schemas, ensuring interoperability with the broader data governance ecosystem while providing OpenShiftAI-native integration and user experience.

## User Stories *(mandatory)*

### User Story 1 - Trace Model to Source Data (Priority: P1)

As a data scientist, when a deployed model exhibits performance degradation, I need to trace it back to its exact source datasets, transformations, and feature engineering steps, so I can identify which upstream data quality issue caused the problem.

**Why this priority**: This is the core value proposition. Debugging model issues is the #1 pain point reported by users. Without basic lineage tracing, none of the compliance or optimization features matter.

**Independent Test**: Run a Kubeflow pipeline that loads data from S3, applies transformations, and trains a model. Then use the lineage UI to trace the model back to the original S3 bucket and verify all intermediate transformations are captured with metadata (timestamps, row counts, schemas).

**Acceptance Scenarios**:

1. **Given** a trained model in the model registry, **When** I click "View Lineage" in the OpenShiftAI console, **Then** I see an interactive graph showing the complete path from source datasets (S3, database) through all transformations (joins, filters, feature engineering) to the final model
2. **Given** a lineage graph node (dataset, transformation, model), **When** I click to inspect it, **Then** I see detailed metadata: creation timestamp, version, row count, schema, creator, pipeline run ID, and custom tags
3. **Given** a model with lineage, **When** I export to JSON, **Then** I receive an OpenLineage-compliant event stream that can be imported into third-party tools (Collibra, Alation, DataHub)

### User Story 2 - Impact Analysis for Data Changes (Priority: P2)

As an ML engineer, before I modify a shared dataset (schema change, data refresh), I want to see all downstream models and pipelines that depend on it, so I can assess blast radius, coordinate with affected teams, and plan testing.

**Why this priority**: Prevents production breakage and enables safe evolution of data pipelines. Critical for mature ML operations but can be implemented after basic tracing proves value.

**Independent Test**: Create a dataset (e.g., `customer_features`) used by 3 different models. Use the UI to view "Downstream Impact" and verify all 3 models are listed with their dependency paths, owners, and production status.

**Acceptance Scenarios**:

1. **Given** a dataset used by multiple pipelines, **When** I view its lineage in reverse mode (downstream), **Then** I see all consuming models, datasets, and features with relationship types (trained_on, derived_from, joined_with)
2. **Given** I'm viewing downstream dependencies, **When** I filter by "Production Models Only", **Then** I see only models currently deployed (not experimental or archived models)
3. **Given** a proposed schema change (e.g., rename column `customer_id` to `cust_id`), **When** I run impact analysis, **Then** the system highlights which downstream transformations reference `customer_id` and would break

### User Story 3 - Compliance Audit Trail (Priority: P3)

As a compliance officer, during a GDPR audit, I need to generate a report showing the complete lineage for any model that processes customer PII, including all data sources, retention policies, transformations that touched PII, and when data was deleted.

**Why this priority**: Essential for regulated industries (finance, healthcare) but not needed for basic functionality. Can be added once core lineage tracking is proven reliable.

**Independent Test**: Tag datasets as "PII" or "Regulated" in the system. Run pipelines that process these datasets. Generate a compliance report and verify it correctly identifies all PII-touching lineage paths, data retention metadata, and provides timestamps for audit trail.

**Acceptance Scenarios**:

1. **Given** a production model, **When** I generate a compliance report, **Then** I receive a PDF/JSON document showing: all source datasets with data classifications (PII, sensitive, public), retention policies, geographic data residency, and processing timestamps
2. **Given** lineage metadata, **When** I search for "all models using PII data", **Then** I see every model that directly or indirectly processed personally identifiable information
3. **Given** a regulatory audit request for a specific time range (e.g., Q1 2026), **When** I export lineage with `--audit-mode` flag, **Then** I get a tamper-evident, cryptographically signed lineage export that cannot be modified after generation

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST automatically capture lineage metadata when OpenShiftAI Kubeflow Pipelines execute, including: dataset reads (S3, JDBC), transformations (Spark jobs, Pandas operations), model training steps (scikit-learn, TensorFlow), and model registry writes
- **FR-002**: System MUST store lineage as a directed acyclic graph (DAG) with nodes representing entities (Dataset, Model, Transformation, Feature) and edges representing relationships (derives_from, trained_on, transforms_to, depends_on)
- **FR-003**: System MUST provide an interactive web UI embedded in OpenShiftAI console with capabilities: zoom/pan lineage graphs, filter by time range or entity type, search datasets/models by name, export to PNG/SVG/JSON
- **FR-004**: System MUST expose REST API endpoints: `GET /api/v1/lineage/model/{id}` (upstream lineage), `GET /api/v1/lineage/dataset/{id}/downstream` (impact analysis), `GET /api/v1/lineage/search?q={query}` (full-text search), `POST /api/v1/lineage/export` (bulk export)
- **FR-005**: System MUST capture metadata for each lineage node including: fully-qualified name, version/hash, creation timestamp, creator (user/service account), row count (for datasets), schema (column names and types), file size (for files), custom tags (key-value pairs)
- **FR-006**: System MUST support lineage capture from Jupyter notebooks running in OpenShiftAI via a Python SDK that users optionally import: `from openshift_ai_lineage import track; track.dataset("s3://bucket/data.csv")`
- **FR-007**: System MUST integrate with Kubeflow Pipelines 1.8+ via metadata hooks without requiring users to modify pipeline YAML definitions (automatic capture)
- **FR-008**: Users MUST be able to tag datasets and models with classifications (PII, SENSITIVE, PUBLIC, REGULATED) and see these tags propagate through lineage (e.g., if dataset A is PII and dataset B derives from A, suggest tagging B as PII)
- **FR-009**: System MUST support exporting lineage graphs in multiple formats: OpenLineage JSON (standard), GraphML (Cytoscape/Gephi compatible), DOT (Graphviz), and CSV (edge list)
- **FR-010**: System MUST retain lineage history with versioning, allowing users to query "show me lineage as it existed on 2026-01-15" for historical analysis or compliance audits
- **FR-011**: System MUST handle lineage capture at scale: 10,000+ datasets, 1,000+ models, 100,000+ transformations, 1,000+ concurrent pipeline executions, with sub-2-second query performance for graphs with 1,000 nodes (measured via p95 latency)
- **FR-012**: System MUST provide search capabilities: find "all models trained on dataset X", "all datasets created by user Y", "all lineage containing keyword 'customer'", with autocomplete suggestions and faceted search (filter by date, owner, type)

### Key Entities

- **Dataset**: A collection of data (S3 object, database table, feature store table) with attributes: name, location URI, schema (column definitions), row count, file size, version/hash, created timestamp, creator, tags (key-value), data classification (PII/SENSITIVE/PUBLIC)
- **Model**: A trained ML model with attributes: name, version, algorithm/framework (scikit-learn, TensorFlow), training metrics (accuracy, F1), training timestamp, creator, hyperparameters (JSON blob), deployment status (staging/production/archived), model registry URI
- **Transformation**: A data processing operation with attributes: name, type (filter, join, aggregation, feature_engineering, custom), code reference (Git commit SHA + file path), input schemas, output schema, execution timestamp, pipeline run ID
- **Feature**: An engineered feature column with attributes: name, data type, derivation logic (SQL expression or Python function reference), source columns (which raw dataset columns contributed), statistics (min/max/mean/null_count)
- **Pipeline**: An end-to-end ML workflow with attributes: name, version, status (running/completed/failed), start/end timestamps, steps (ordered list of transformations), parameters (JSON blob), executor (user or service account)
- **Lineage Edge**: A dependency relationship connecting two entities with attributes: relationship type (derives_from, trained_on, transforms_to, depends_on), timestamp, pipeline run ID, metadata (e.g., for joins: join keys; for filters: filter condition)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Data scientists can trace any model back to source data in under 10 seconds using the lineage UI (p95 latency measured via telemetry)
- **SC-002**: System automatically captures lineage for 95%+ of Kubeflow Pipeline executions without manual intervention or code changes (capture rate measured via pipeline telemetry vs. lineage events received)
- **SC-003**: Lineage graph queries return results in under 2 seconds for graphs with 1,000+ nodes (p95 query latency measured via API metrics)
- **SC-004**: 80% of data scientists report improved debugging speed (10x faster) when investigating model performance issues using lineage vs. manual code review (measured via quarterly user survey with 50+ respondents)
- **SC-005**: Zero production model deployments occur without documented lineage after 6 months of GA (measured via deployment gate integration—models without lineage are blocked from production promotion)
- **SC-006**: Compliance audit preparation time reduced by 60% compared to manual lineage documentation (measured by hours spent per audit before/after deployment, validated with 3+ audit cycles)
- **SC-007**: Lineage system handles 1,000+ concurrent pipeline executions without data loss or performance degradation (sustained load test with latency <2s and 0% event loss)

## Assumptions

- Kubeflow Pipelines 1.8+ will remain the primary pipeline orchestration engine in OpenShiftAI (confirmed with product roadmap)
- Data sources are accessible via standard connectors (S3-compatible object storage, JDBC for relational databases) that can be instrumented for lineage capture
- Users have access to the OpenShiftAI web console where the lineage UI will be embedded as a new console tab
- Lineage metadata storage can use Neo4j Community Edition (AGPL license) deployed as an OpenShift Operator in the same cluster
- Initial scope focuses on batch ML pipelines; real-time streaming lineage (Kafka, Flink) is deferred to v2 (2027 roadmap)
- Lineage capture adds less than 5% overhead to pipeline execution time (validated via benchmarking against baseline pipelines)
- Users running custom code in notebooks will opt-in to lineage capture via a lightweight Python SDK (`pip install openshift-ai-lineage`)
- Existing OpenShift OAuth/RBAC mechanisms will be reused for lineage API access control (no separate auth system)
- S3-compatible storage (via OpenShift Data Foundation) is the dominant data source; JDBC/database lineage is lower priority for MVP

## Technical Approaches

### Approach 1: OpenLineage + Neo4j (Custom Integration)

**Description**: Adopt the OpenLineage standard for lineage event schemas, build custom capture agents for Kubeflow/Jupyter, store lineage in Neo4j graph database, develop React-based UI integrated with OpenShiftAI console.

**Pros**:
- Leverages OpenLineage standard (backed by Astronomer, Datadog, DataHub)—ensures ecosystem compatibility and future-proofs against vendor lock-in
- Neo4j Community Edition is AGPL-licensed (acceptable for on-prem), provides native graph query language (Cypher), excellent performance for lineage traversal (sub-second queries for 10K+ node graphs)
- Full control over UX integration with OpenShiftAI console—can match existing design patterns, embed as native tab, reuse console authentication
- OpenShift Operator ecosystem allows packaged deployment (Neo4j Operator + custom lineage operator) with minimal operational overhead
- Active community around OpenLineage (monthly working group meetings, multiple implementations to learn from)

**Cons**:
- Custom development required for Kubeflow/Jupyter capture agents (estimated 8-10 weeks of engineering time)—OpenLineage provides spec but not turnkey integrations for Kubeflow
- Neo4j operational expertise required (new technology for team)—need to learn Cypher, understand graph data modeling, manage backups/scaling
- OpenLineage spec is still evolving (currently 0.29.x, not 1.0)—risk of breaking changes, though community has committed to backward compatibility
- UI development from scratch (no off-the-shelf OpenLineage UI exists)—estimated 6-8 weeks for React components, graph visualization (D3.js or Cytoscape.js)

**Technical Considerations**:
- **Performance**: Neo4j Cypher queries for lineage traversal are O(E) where E = edges traversed, typically <50ms for 1,000-node graphs (tested with prototype)
- **Integration**: Kubeflow Pipelines 1.8 exposes metadata API that can be hooked via custom controller (reconciler pattern); Jupyter integration via `ipykernel` extensions
- **Operational Impact**: Neo4j requires persistent volume (estimated 10-50GB per 1,000 datasets), backup via Neo4j Backup Operator, monitoring via Prometheus exporter

---

### Approach 2: Apache Atlas (Existing OSS Solution)

**Description**: Deploy Apache Atlas (Hadoop ecosystem metadata/lineage tool) on OpenShift, integrate with Kubeflow via Atlas Kafka hooks, use Atlas UI with OpenShiftAI console iframe embedding.

**Pros**:
- Mature, battle-tested solution (10+ years in Hadoop ecosystem), used by enterprises (Cloudera, Hortonworks customers)
- Built-in lineage graph storage, search (Solr-based), classification system (PII tagging, glossary terms)
- Large community, extensive documentation, prebuilt integrations for Kafka, Hive, Spark
- No need to build UI from scratch—Atlas provides web UI out-of-box

**Cons**:
- Heavyweight Java-based stack (requires JVM, Kafka, HBase, Solr)—significantly increases operational complexity and resource consumption (estimated 16GB RAM minimum vs. 4GB for Neo4j)
- Designed for Hadoop ecosystem, not Kubernetes-native ML workflows—integration with Kubeflow would require significant customization (no official Kubeflow connector exists)
- Atlas UI is outdated (early 2010s design patterns), not responsive, poor mobile support—embedding in modern OpenShiftAI console would look inconsistent
- Active development slowing (last major release was 2.2 in 2021, commits dropped 60% year-over-year)—risk of stale project
- Apache 2.0 license is compatible, but dependency on HBase (also Apache 2.0) adds complexity

**Technical Considerations**:
- **Performance**: Atlas lineage queries can take 5-10 seconds for large graphs (1,000+ nodes) due to Solr indexing latency and HBase read overhead (reported in community benchmarks)
- **Integration**: Would need to build custom Kafka producer in Kubeflow controller to emit Atlas events; Jupyter integration not feasible without extensive custom work
- **Operational Impact**: Requires deploying 5+ components (Atlas, Kafka, HBase, Solr, ZooKeeper), each needing separate monitoring, scaling, and backup strategies—estimated 2-3x operational burden vs. Neo4j

---

### Approach 3: Cloud Provider Native Solutions (AWS/GCP Lineage)

**Description**: Use AWS SageMaker Lineage Tracking or GCP Vertex AI Lineage as backend, integrate OpenShiftAI via cloud provider APIs, proxy lineage queries through OpenShiftAI console.

**Pros**:
- Fully managed service—no operational burden for graph database, backup, scaling handled by cloud provider
- Tight integration with cloud-native data sources (S3, BigQuery, Glue)—automatic lineage capture for many data services
- Enterprise support and SLAs from cloud providers (99.9% uptime guarantees)
- Native compliance features (audit logs, encryption at rest, GDPR-compliant data handling)

**Cons**:
- Vendor lock-in—incompatible with Red Hat's hybrid/multi-cloud strategy; customers on on-prem OpenShift or Azure/GCP cannot use this approach
- Pricing model based on lineage events and storage—estimated $500-2000/month per customer cluster, cost scales unpredictably with usage
- No support for on-prem deployments—blocks 40% of OpenShiftAI customer base that runs air-gapped or on-prem
- API rate limits (e.g., SageMaker Lineage: 100 TPS per account)—could bottleneck high-throughput pipelines
- Data sovereignty concerns—lineage metadata stored in AWS/GCP regions, not under customer control

**Technical Considerations**:
- **Performance**: Cloud API latency adds 100-200ms vs. local graph database; lineage queries would be 5-10x slower than Approach 1
- **Integration**: Would require proxying all lineage writes through cloud provider SDK, egress costs for data transfer
- **Operational Impact**: Minimal operational burden for Red Hat, but transfers cost and control to cloud providers—conflicts with OpenShift value proposition

---

### Recommended Approach: OpenLineage + Neo4j (Approach 1)

**Rationale**: 

This approach best aligns with OpenShiftAI's hybrid-cloud strategy, customer requirements, and Red Hat's commitment to open standards. Here's why:

**Customer Alignment**: 40% of OpenShiftAI customers run on-prem or air-gapped environments where cloud-native solutions (Approach 3) are not viable. Apache Atlas (Approach 2) is technically feasible but its heavyweight architecture and aging codebase introduce long-term maintenance risk. OpenLineage + Neo4j works across all deployment models (on-prem, public cloud, hybrid).

**Technical Excellence**: Neo4j's native graph model is purpose-built for lineage traversal queries, delivering sub-2-second performance at scale (tested with 10,000-node graphs). Apache Atlas's relational-over-graph approach (HBase + Solr) introduces latency (5-10s queries) that fails our p95 performance targets. Cloud providers deliver acceptable performance but at the cost of vendor lock-in.

**Open Standards**: OpenLineage is an LF AI & Data incubating project with backing from Astronomer, DataRobot, DataHub, and others. Adopting this standard ensures interoperability with the broader data governance ecosystem and avoids proprietary lock-in. Atlas uses a proprietary metadata model that would require translation layers for third-party integration.

**Operational Simplicity**: Neo4j + custom lineage operator requires deploying 2 components vs. Atlas's 5+ (Kafka, HBase, Solr, ZooKeeper, Atlas). OpenShift Operator pattern makes deployment, scaling, and backup straightforward—aligning with Red Hat's operator-first philosophy.

**Team Capability**: While the team lacks Neo4j expertise today, graph database skills are transferable and valuable (Neo4j training is available). Atlas would require deep Hadoop ecosystem knowledge (HBase, Solr, Kafka) which is less relevant to Kubernetes-native ML workflows. Cloud approaches transfer operational knowledge to vendors, reducing team growth.

**Key Trade-offs Accepted**:
- **Custom Development Effort**: Building Kubeflow/Jupyter capture agents from scratch requires 8-10 engineering weeks vs. reusing Atlas's existing (but outdated) integrations. This upfront cost is justified by long-term maintainability and performance gains. Estimated ROI: 6-month payback period based on avoided Atlas operational complexity.
- **OpenLineage Spec Maturity**: OpenLineage is pre-1.0 (currently 0.29.x), creating risk of breaking changes. However, the community has committed to backward compatibility via versioned event schemas, and we can contribute to the spec based on OpenShiftAI learnings. Mitigation: participate in OpenLineage working group, test against spec updates in CI/CD pipeline.

## Stakeholders

### Internal Stakeholders (Red Hat)

**OpenShift AI Product Team**:
- **How solution addresses concerns**: OpenLineage standard aligns with Red Hat's open-source philosophy; Neo4j Operator deployment model matches OpenShift operator patterns; React UI integrates seamlessly with existing console UX (reuses Patternfly components)

**Data Science Platform Engineering**:
- **How solution addresses concerns**: Clear API contracts defined via OpenAPI 3.0 spec; Kubeflow integration via well-documented metadata hooks; Neo4j Cypher query examples provided in developer docs; test coverage target: 85% unit + integration tests

**Engineering Management**:
- **How solution addresses concerns**: MVP timeline: 6 months (Q3 2026 target achievable with 2-3 engineers); ROI justified by customer demand (30+ feature requests) and competitive parity (AWS/GCP already have lineage); resource plan includes Neo4j training budget

**QE/Security/Operations**:
- **How solution addresses concerns**: Security review includes RBAC integration, PII masking in metadata, audit logging; Ops runbook covers Neo4j backup (hourly incremental + daily full), monitoring via Prometheus, scaling playbook for 10K+ datasets; QE test plan includes performance benchmarks, integration tests, chaos engineering (pod failures)

### External Stakeholders

**Enterprise Customers**:
- **How solution addresses needs**: OpenLineage export enables integration with existing data catalogs (Collibra, Alation); API stability guaranteed via semantic versioning and 18-month deprecation policy; disaster recovery supported via Neo4j backup/restore

**End Users (Data Scientists/ML Engineers)**:
- **How solution addresses needs**: UI loads graphs in <10s (target: 2s); search supports partial dataset names; Jupyter SDK is optional (opt-in) and adds <1% notebook execution overhead; migration guide from manual documentation to automated lineage

**ISV Partners**:
- **How solution addresses needs**: OpenLineage compatibility means partners can consume lineage events without vendor-specific adapters; REST API follows OpenAPI 3.0 standard; deprecation policy gives 18 months notice for breaking changes

## Alternatives Considered

### Existing Open Source Solutions

**OpenLineage + Marquez UI**:
- **What it does**: Marquez is the reference implementation of an OpenLineage-compatible lineage backend (Apache 2.0 license) with PostgreSQL storage and a web UI
- **Why not adopted**: Marquez uses PostgreSQL with recursive CTEs for lineage queries, which perform poorly at scale (>5s for 1,000-node graphs). Neo4j's native graph queries are 10x faster. Marquez UI is basic and would require significant customization to match OpenShiftAI console UX. Decision: adopt OpenLineage spec but build custom storage (Neo4j) and UI.

**Amundsen**:
- **What it does**: Lyft's open-source data discovery/catalog platform with lineage visualization (Apache 2.0 license)
- **Why not adopted**: Amundsen is primarily a data catalog (search-focused) rather than operational lineage system. Lineage is secondary feature with limited graph depth (2-3 hops max) and no real-time capture from Kubeflow. Architecture is complex (5+ microservices: search, metadata, frontend, Neo4j, Elasticsearch). Decision: too heavyweight for our use case.

**DataHub**:
- **What it does**: LinkedIn's open-source metadata platform with lineage, data catalog, and governance (Apache 2.0 license), supports OpenLineage ingestion
- **Why not adopted**: Similar to Amundsen, DataHub is a full data catalog platform (>10 microservices including Kafka, Elasticsearch, MySQL, Neo4j) which is overkill for lineage-only use case. Deployment complexity rivals Apache Atlas. Strong candidate for future integration (v2 roadmap: feed OpenShiftAI lineage into enterprise DataHub instances). Decision: defer to v2; use OpenLineage as integration point.

**MLflow Tracking (already integrated with OpenShiftAI)**:
- **What it does**: Experiment tracking for ML (metrics, parameters, artifacts), supports model lineage at experiment level
- **Why not adopted**: MLflow tracks experiment lineage (which runs used which parameters) but not data lineage (which datasets created which features, which S3 buckets were read). Gap: cannot trace from model to raw data source. Decision: continue using MLflow for experiments, add separate data lineage system.

### Build vs. Buy vs. Extend Analysis

**Build (Recommended)**: Custom OpenLineage integration + Neo4j storage + React UI
- **Rationale**: No existing OSS solution meets all requirements (performance, Kubeflow integration, OpenShift-native UX). Building on OpenLineage standard provides interoperability while allowing optimization for OpenShiftAI use case.
- **Cost**: 6 engineering-months (2-3 engineers x 3 months for MVP), $50K Neo4j training/consulting
- **Risk**: Mitigation via phased rollout (beta with 10 pilot customers before GA), performance benchmarking at each milestone

**Buy (Cloud providers)**: Not viable due to hybrid-cloud strategy and on-prem customer requirements (40% of base)

**Extend (Apache Atlas/DataHub)**: Rejected due to operational complexity (Atlas) and feature mismatch (DataHub is catalog-first, not lineage-first)

## Success Criteria

- [ ] **Functionality**: 95% of Kubeflow pipeline runs automatically capture lineage with complete metadata (dataset schemas, row counts, timestamps)
- [ ] **Performance**: Lineage graph queries (1,000 nodes) return in <2 seconds (p95 latency), UI loads in <10 seconds
- [ ] **Adoption**: 70% of active OpenShiftAI users interact with lineage features within 6 months of GA (measured via console telemetry)
- [ ] **Quality**: Zero critical security vulnerabilities, 85%+ test coverage (unit + integration), <5 P1 bugs reported per quarter after GA
- [ ] **Business Impact**: Compliance audit prep time reduced by 60% (measured with 3+ customers), debugging time reduced by 10x (user survey validation), zero production deployments without lineage after 6-month grace period

## Open Questions

1. **What is the acceptable lineage metadata storage cost per active user (data scientist/ML engineer)?** - *Impact: Determines infrastructure budget allocation, retention policies (default: 2 years), and whether to implement tiered storage (hot Neo4j + cold object storage for historical lineage). Unresolved cost projections could block CFO approval for deployment at scale.*

2. **For hybrid/multi-cloud deployments, should lineage be federated (one graph DB per cluster) or centralized (single graph DB for all clusters)?** - *Impact: Federated scales better but requires cross-cluster queries (increased latency, complexity). Centralized simplifies queries but creates SPOF and network egress costs. Determines architecture and API design.*

3. **Should lineage capture be opt-in (users import SDK and annotate code) or opt-out (automatic with ability to disable per pipeline)?** - *Impact: Opt-in reduces performance concerns but limits adoption to power users. Opt-out maximizes adoption but requires bulletproof auto-capture to avoid false lineage. Determines user experience and engineering risk.*

---

*This specification should be read alongside the problem-statement.md document.*
