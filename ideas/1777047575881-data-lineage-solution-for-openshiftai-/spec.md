# Data Lineage Solution for OpenShiftAI

**Status**: Draft  
**Created**: 2026-04-24  
**Author**: System Generated

## Overview

A comprehensive data lineage tracking system for OpenShiftAI that automatically captures, stores, and visualizes the flow of data through ML pipelines, from raw data sources through transformations to final model outputs. This solution enables data scientists and ML engineers to understand data provenance, debug pipeline issues, ensure reproducibility, and meet compliance requirements.

## Problem Statement

Data scientists and ML engineers working in OpenShiftAI currently lack visibility into how data flows through their ML pipelines. This affects multiple stakeholder groups:

**Data Scientists** struggle to reproduce models because they cannot trace back which exact data sources and transformations produced a specific model. When model performance degrades, they waste hours manually investigating which upstream data changes caused the issue.

**ML Engineers** face risks when updating datasets or features, as they cannot assess downstream impact on production models. One schema change can break multiple dependent pipelines without warning.

**Compliance Officers** cannot demonstrate data governance and lineage for regulatory requirements (GDPR, healthcare, financial). Manual documentation is error-prone and quickly becomes outdated.

**Platform Administrators** lack observability into data dependencies across the platform, making it difficult to plan migrations, upgrades, or deprecations.

Without automated lineage tracking, teams resort to manual documentation which doesn't scale as pipelines grow in complexity, creating technical debt and compliance risk.

## Proposed Solution

Build an automated data lineage tracking system integrated with OpenShiftAI that:

1. **Automatic Capture**: Instruments OpenShiftAI pipelines, notebooks, and data connections to automatically capture lineage metadata without requiring code changes
2. **Lineage Graph Storage**: Persists lineage relationships in a graph database optimized for traversal and querying
3. **Visual Explorer**: Provides interactive UI to visualize lineage graphs, trace data flow upstream/downstream, and inspect metadata
4. **Metadata API**: Exposes REST API for programmatic access to lineage information
5. **Integration Points**: Connects with OpenShiftAI pipelines, Jupyter notebooks, data sources (S3, databases), and model registry

The solution will capture lineage at multiple levels:
- Dataset lineage (which datasets were used to create new datasets)
- Feature lineage (which raw data columns became which engineered features)
- Model lineage (which datasets/features trained which models)
- Pipeline lineage (complete end-to-end flow from source to deployed model)

## User Stories *(mandatory)*

### User Story 1 - Trace Model to Source Data (Priority: P1)

As a data scientist, I need to trace any deployed model back to its exact source datasets and transformations, so I can reproduce the model or investigate data quality issues.

**Why this priority**: This is the core value proposition - without basic lineage tracing, none of the other features matter. This enables reproducibility and debugging, the two most common use cases.

**Independent Test**: Can be fully tested by running a simple pipeline (load data → transform → train model), then using the UI to trace the model back to the original data source and verify all intermediate steps are captured.

**Acceptance Scenarios**:

1. **Given** a trained model in the model registry, **When** I select "View Lineage" in the UI, **Then** I see a graph showing the complete path from source datasets through all transformations to the final model
2. **Given** a lineage graph for a model, **When** I click on any node (dataset, transformation, feature), **Then** I see detailed metadata including timestamps, versions, row counts, and schemas
3. **Given** a model with lineage, **When** I export the lineage as JSON, **Then** I receive a complete machine-readable representation of all dependencies

### User Story 2 - Impact Analysis for Data Changes (Priority: P2)

As an ML engineer, when I need to modify or update a dataset, I want to see all downstream models and pipelines that depend on it, so I can assess the impact and plan testing.

**Why this priority**: This prevents breaking changes and enables safe evolution of data pipelines. Critical for production ML systems but can be implemented after basic tracing is working.

**Independent Test**: Create a dataset used by 3 different models, then use the UI to view "Downstream Impact" and verify all 3 models are listed with their dependency paths.

**Acceptance Scenarios**:

1. **Given** a dataset used by multiple pipelines, **When** I view its lineage, **Then** I see all downstream models, datasets, and features that depend on it
2. **Given** I'm viewing downstream dependencies, **When** I filter by "Production Models Only", **Then** I see only models currently deployed to production
3. **Given** a proposed schema change, **When** I run impact analysis, **Then** the system highlights which downstream consumers would be affected

### User Story 3 - Compliance Audit Trail (Priority: P3)

As a compliance officer, I need to generate audit reports showing the complete lineage for any model used in regulated decisions, including data sources, retention policies, and PII handling.

**Why this priority**: Essential for regulated industries but not needed for basic functionality. Can be added once core lineage tracking is proven.

**Independent Test**: Tag certain datasets as "PII" or "Regulated", run pipelines using them, then generate an audit report and verify it correctly identifies all PII paths and retention policies.

**Acceptance Scenarios**:

1. **Given** a production model, **When** I generate a compliance report, **Then** I receive a document showing all data sources, their classifications (PII, sensitive, public), and retention policies
2. **Given** lineage metadata, **When** I search for "models using PII data", **Then** I see all models that processed personally identifiable information
3. **Given** a regulatory audit request, **When** I export lineage for a specific time range, **Then** I get a tamper-evident record of all data flows during that period

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST automatically capture lineage metadata when OpenShiftAI pipelines execute, including dataset reads, transformations, and model training steps
- **FR-002**: System MUST store lineage as a directed acyclic graph (DAG) with nodes representing datasets/models/transformations and edges representing dependencies
- **FR-003**: Users MUST be able to visualize lineage graphs in a web UI with zoom, pan, filter, and search capabilities
- **FR-004**: System MUST provide REST API endpoints to query lineage programmatically (e.g., GET /lineage/model/{id}, GET /lineage/downstream/{dataset-id})
- **FR-005**: System MUST capture metadata for each lineage node including: timestamp, version, schema, row count, file size, creator, and custom tags
- **FR-006**: System MUST support lineage capture from Jupyter notebooks running in OpenShiftAI without requiring code modifications
- **FR-007**: System MUST integrate with OpenShiftAI Data Science Pipelines (Kubeflow Pipelines) to auto-capture lineage from pipeline runs
- **FR-008**: Users MUST be able to tag datasets and models with classifications (PII, sensitive, public, regulated) and see these tags propagate through lineage
- **FR-009**: System MUST support exporting lineage graphs as JSON, GraphML, or DOT format for external analysis
- **FR-010**: System MUST retain lineage history with versioning, allowing users to view lineage "as of" a specific date
- **FR-011**: System MUST handle lineage capture at scale (10,000+ datasets, 1,000+ models, 100,000+ transformations) with sub-second query performance
- **FR-012**: System MUST provide lineage search allowing users to find "all models using dataset X" or "all datasets created from source Y"

### Key Entities

- **Dataset**: A collection of data (file, table, feature store) with attributes: name, location, schema, row count, size, version, created timestamp, creator, tags
- **Model**: A trained ML model with attributes: name, version, algorithm, metrics, training timestamp, creator, hyperparameters, deployment status
- **Transformation**: A data processing step with attributes: name, type (filter, join, aggregation, feature engineering), code reference, input schemas, output schema
- **Pipeline**: An end-to-end ML workflow with attributes: name, version, status, start/end timestamps, steps, parameters
- **Lineage Edge**: A dependency relationship connecting two entities with attributes: type (derives_from, trained_on, transforms_to), timestamp, pipeline run ID

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Data scientists can trace any model back to source data in under 10 seconds using the lineage UI
- **SC-002**: System automatically captures lineage for 95%+ of pipeline executions without manual intervention or code changes
- **SC-003**: Lineage graph queries return results in under 2 seconds for graphs with 1,000+ nodes
- **SC-004**: 80% of data scientists report improved debugging speed when investigating model performance issues (measured via survey)
- **SC-005**: Zero production model deployments occur without documented lineage (measured via deployment gate checks)
- **SC-006**: Compliance audit preparation time reduced by 60% compared to manual lineage documentation (measured by hours spent per audit)
- **SC-007**: Lineage system handles 1,000+ concurrent pipeline executions without data loss or performance degradation

## Assumptions

- OpenShiftAI pipelines use Kubeflow Pipelines as the orchestration engine, allowing us to hook into pipeline execution events
- Data sources are accessible via standard connectors (S3, JDBC, HTTP) that we can instrument for lineage capture
- Users have access to the OpenShiftAI web console where the lineage UI will be embedded
- Lineage metadata storage can use a dedicated graph database (e.g., Neo4j) or triple store deployed in the OpenShift cluster
- Initial scope focuses on batch pipelines; real-time streaming lineage is deferred to v2
- Lineage capture adds less than 5% overhead to pipeline execution time
- Users running custom code in notebooks can opt-in to lineage capture via a lightweight Python SDK
- Existing OpenShiftAI authentication/authorization mechanisms will be reused for lineage API access

## High-level Technical Approach

**Architecture Components**:

1. **Lineage Capture Agents**:
   - Kubeflow Pipeline interceptor that hooks into pipeline DAG execution
   - Jupyter notebook kernel extension that tracks data reads/writes
   - Python SDK for manual lineage annotation in custom scripts

2. **Lineage Store**:
   - Graph database (Neo4j or AWS Neptune) for storing lineage DAG
   - Time-series metadata store for versioning and history
   - Object storage for large metadata payloads (schemas, sample data)

3. **Lineage API**:
   - REST API for querying lineage (GraphQL for complex graph traversals)
   - Event streaming (Kafka/NATS) for real-time lineage updates
   - Authentication via OpenShiftAI OAuth integration

4. **Lineage UI**:
   - React-based web UI embedded in OpenShiftAI console
   - Interactive graph visualization using D3.js or Cytoscape.js
   - Search, filter, and export capabilities

**Integration Points**:
- Hook into Kubeflow Pipeline controller to capture pipeline metadata
- Instrument OpenShift Data Foundation (ODF) access for dataset tracking
- Integrate with model registry API to link models to lineage
- Connect to OpenShift logging for audit trail correlation

**Data Flow**:
1. Pipeline executes → Capture agent sends lineage events to API
2. API validates and stores lineage in graph database
3. UI polls API or receives websocket updates for real-time visualization
4. Users query lineage via UI or API for analysis/compliance

## Stakeholders

### Primary End Users

- **Data Scientists**: Need lineage for debugging model issues, understanding data provenance, and reproducing experiments. Key use case: tracing model performance degradation to upstream data quality issues.

- **ML Engineers**: Require impact analysis before making data/pipeline changes, dependency mapping for production systems. Key use case: assessing blast radius before modifying a shared dataset.

- **Compliance Officers**: Need audit trails, PII tracking, and regulatory compliance reporting. Key use case: generating lineage reports for GDPR/HIPAA audits.

### Internal Stakeholders

- **OpenShift AI Product Team**: Platform owners who will integrate lineage features into the OpenShiftAI console and ensure it scales with the platform.

- **Platform Administrators**: Responsible for deploying and operating the lineage infrastructure, monitoring performance, and managing retention policies.

- **Security Team**: Concerned with access controls, audit logging, and ensuring lineage metadata doesn't leak sensitive information.

### Related Red Hat Products

- **OpenShift Data Foundation (ODF)**: Lineage system will integrate with ODF to track data stored in object storage (S3-compatible).

- **OpenShift Pipelines (Tekton)**: Potential future integration for capturing lineage from CI/CD pipelines that prepare training data.

- **Red Hat OpenShift**: The underlying platform that hosts OpenShiftAI and the lineage infrastructure.

- **Red Hat Ansible Automation Platform**: May consume lineage metadata for automating data governance workflows.

## Alternatives Considered

### Open Source Solutions

1. **Apache Atlas**:
   - Pros: Mature, Hadoop ecosystem integration, metadata management
   - Cons: Heavy Java dependency, designed for Hadoop not Kubernetes/ML
   - Decision: Too heavyweight for OpenShiftAI, not ML-focused

2. **OpenLineage**:
   - Pros: Emerging standard, lightweight, event-driven, growing community
   - Cons: Still early, limited UI tooling, requires custom integration work
   - Decision: **Recommended foundation** - adopt OpenLineage spec for lineage events, build custom UI/storage on top

3. **MLflow Tracking**:
   - Pros: ML-native, tracks experiments and models
   - Cons: Limited dataset lineage, weak graph visualization
   - Decision: Complement but don't replace - use MLflow for experiment tracking, new system for full lineage

4. **Amundsen/DataHub**:
   - Pros: Data catalog + lineage, strong metadata management
   - Cons: More focused on discovery than operational lineage, complex deployment
   - Decision: Consider for future catalog integration, not core lineage solution

5. **AWS/GCP Native Solutions** (SageMaker Lineage, Vertex AI Lineage):
   - Pros: Cloud-integrated, managed services
   - Cons: Vendor lock-in, doesn't work in on-prem OpenShift environments
   - Decision: Not applicable for OpenShiftAI multi-cloud/hybrid strategy

**Recommendation**: Build on OpenLineage standard for lineage event schema, use Neo4j for graph storage, develop custom UI integrated with OpenShiftAI console. Reuse OpenShift operators for deployment/lifecycle management.

## Open Questions

1. **Lineage Capture Granularity**: Should we track lineage at the row/column level (e.g., "feature X derived from columns A, B in dataset Y") or only at the dataset/model level? Row-level provides more insight but generates significantly more metadata and compute overhead.

2. **Cross-Cluster Lineage**: How do we handle lineage when data flows between different OpenShift clusters or external systems (e.g., data ingested from on-prem database, processed in cloud OpenShiftAI, model deployed to edge)? Do we need federated lineage tracking or accept gaps in multi-cluster scenarios?

3. **Lineage Retention Policy**: How long should historical lineage be retained? Options: retain forever (expensive storage, unlimited audit trail), retain for compliance period (e.g., 7 years for regulated industries), or retain based on model lifecycle (delete lineage when model is retired)?

---

*Generated by Idea Workflow System*
