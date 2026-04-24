# Data Lineage Capture Solution for OpenShiftAI

**Status**: Draft  
**Created**: 2026-04-24  
**Author**: User via expand-idea skill

## Overview

A data lineage capture and visualization system for OpenShiftAI that automatically tracks the complete lifecycle of data used in machine learning pipelines—from raw data sources through transformations, feature engineering, model training, and model deployment. This solution enables ML teams to understand data provenance, ensure compliance, debug model issues, and maintain reproducibility across their ML workflows.

## Problem Statement

Organizations using OpenShiftAI for ML/AI workloads face significant challenges:

1. **Lack of Visibility**: Data scientists and ML engineers cannot easily trace which datasets, features, and transformations contributed to a specific model version
2. **Compliance Risk**: Audit trails for model decisions are incomplete, making it difficult to meet regulatory requirements (GDPR, AI Act, industry-specific regulations)
3. **Debugging Complexity**: When models underperform or drift, teams struggle to identify upstream data quality issues or transformation errors
4. **Reproducibility Issues**: Re-training models or reproducing results is difficult without clear lineage tracking of data versions and processing steps
5. **Siloed Information**: Lineage information exists in fragments across notebooks, pipeline definitions, and logs, but is not centralized or queryable

## Proposed Solution

Build an integrated data lineage capture system that:

- **Automatically captures** lineage metadata from OpenShiftAI workloads (notebooks, pipelines, data science projects)
- **Tracks relationships** between datasets, features, transformations, experiments, and models
- **Provides visualization** of lineage graphs showing upstream and downstream dependencies
- **Enables querying** to answer questions like "Which models used this dataset?" or "What data sources contributed to this prediction?"
- **Integrates seamlessly** with existing OpenShiftAI components (Data Science Pipelines, Model Registry, Workbenches)
- **Supports compliance** by maintaining audit trails and impact analysis

## User Stories *(mandatory)*

### User Story 1 - Track Dataset-to-Model Lineage (Priority: P1)

As a **data scientist**, I need to see which datasets and versions were used to train a specific model, so that I can reproduce experiments and debug model issues.

**Why this priority**: Core value proposition - without basic lineage tracking from datasets to models, the system provides no value. This is the foundation for all other capabilities.

**Independent Test**: Can be fully tested by training a model in an OpenShiftAI notebook, then querying the lineage system to retrieve the complete list of input datasets with versions. Delivers immediate value for reproducibility.

**Acceptance Scenarios**:

1. **Given** a model has been trained using multiple datasets, **When** I query the lineage system with the model ID, **Then** I see a complete list of all input datasets including versions and access timestamps
2. **Given** I am viewing a model in the Model Registry, **When** I click "View Lineage", **Then** I see a visual graph showing all upstream datasets and transformations
3. **Given** a dataset version has been used by multiple models, **When** I query lineage for that dataset, **Then** I see all downstream models that consumed it

### User Story 2 - Visualize Transformation Pipelines (Priority: P1)

As an **ML engineer**, I need to visualize the complete chain of transformations from raw data to trained model, so that I can identify where data quality issues are introduced.

**Why this priority**: Critical for debugging and optimization - transformations are where most data quality issues occur. Independent of advanced features like impact analysis.

**Independent Test**: Can be fully tested by running a Data Science Pipeline with multiple transformation steps, then viewing the lineage graph showing each transformation node with inputs/outputs. Delivers value for pipeline debugging.

**Acceptance Scenarios**:

1. **Given** a pipeline has multiple transformation steps, **When** I view the lineage graph, **Then** each transformation appears as a distinct node with clear input/output relationships
2. **Given** a transformation step reads from and writes to different datasets, **When** I hover over the transformation node, **Then** I see metadata including code version, execution time, and row counts
3. **Given** a transformation failed during execution, **When** I view the lineage graph, **Then** the failed node is visually highlighted with error details

### User Story 3 - Generate Compliance Reports (Priority: P2)

As a **compliance officer**, I need to generate audit reports showing the complete lineage of data used in model decisions, so that I can demonstrate regulatory compliance.

**Why this priority**: Important for regulated industries but not required for basic system functionality. Can be delivered after core lineage tracking is working.

**Independent Test**: Can be fully tested by selecting a model and exporting a lineage report in PDF/JSON format that includes all datasets, transformations, and timestamps. Delivers compliance value independently.

**Acceptance Scenarios**:

1. **Given** a model is used for production decisions, **When** I request an audit report, **Then** I receive a document showing complete lineage from source data to model predictions
2. **Given** data retention policies exist, **When** I generate a report, **Then** it includes timestamps showing data was collected within allowed retention windows
3. **Given** sensitive data fields were used, **When** I view the lineage, **Then** I can see which transformations applied privacy protection (anonymization, aggregation)

### User Story 4 - Impact Analysis for Data Changes (Priority: P2)

As a **data engineer**, I need to know which models and pipelines will be affected by a data schema change, so that I can plan updates and communicate impact to teams.

**Why this priority**: Valuable for change management but depends on having comprehensive lineage tracking already in place. Secondary to core capture capabilities.

**Independent Test**: Can be fully tested by selecting a dataset and running an impact analysis query that returns all downstream models and pipelines. Delivers value for change planning independently.

**Acceptance Scenarios**:

1. **Given** a dataset schema is about to change, **When** I run impact analysis, **Then** I see all models and pipelines that consume this dataset
2. **Given** multiple versions of a dataset exist, **When** I compare them in the lineage view, **Then** I see which models use which version
3. **Given** a dataset is deprecated, **When** I mark it for retirement, **Then** I receive warnings about active dependencies

### User Story 5 - Integrate with External Data Catalogs (Priority: P3)

As a **platform engineer**, I need the lineage system to integrate with our existing data catalog (e.g., Apache Atlas, DataHub), so that lineage information is consistent across the organization.

**Why this priority**: Nice-to-have integration for organizations with existing data governance tools. Not required for standalone value.

**Independent Test**: Can be fully tested by configuring integration with a data catalog, then verifying that lineage metadata is synced bidirectionally. Delivers value for unified governance independently.

**Acceptance Scenarios**:

1. **Given** integration with external catalog is configured, **When** lineage is captured in OpenShiftAI, **Then** it is automatically synced to the external system
2. **Given** dataset metadata exists in external catalog, **When** it is used in OpenShiftAI, **Then** the lineage system enriches it with catalog information
3. **Given** lineage queries are made, **When** data spans both systems, **Then** results include unified view across internal and external sources

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST automatically capture lineage metadata when datasets are read or written in OpenShiftAI Workbenches (Jupyter notebooks)
- **FR-002**: System MUST track lineage for Data Science Pipelines, recording each pipeline step's inputs and outputs
- **FR-003**: System MUST integrate with OpenShift AI Model Registry to link models to their training data lineage
- **FR-004**: System MUST maintain version information for datasets, showing lineage differences across versions
- **FR-005**: System MUST provide a visual lineage graph showing relationships between datasets, transformations, and models
- **FR-006**: System MUST support querying lineage in both directions (upstream: "what created this?" and downstream: "what uses this?")
- **FR-007**: System MUST capture metadata including timestamps, user identity, code versions, and execution environment
- **FR-008**: System MUST persist lineage data in a queryable storage backend (e.g., graph database or relational database)
- **FR-009**: System MUST provide a REST API for programmatic access to lineage information
- **FR-010**: System MUST support exporting lineage reports in multiple formats (JSON, CSV, PDF) for compliance purposes
- **FR-011**: System MUST integrate with OpenShift AI's authentication and authorization mechanisms (RBAC)
- **FR-012**: Users MUST be able to manually annotate lineage with business context or data quality notes
- **FR-013**: System MUST track column-level lineage [NEEDS CLARIFICATION: Should lineage track individual columns/features, or just dataset-level? Column-level is more valuable but significantly increases complexity and storage]
- **FR-014**: System MUST retain lineage history for [NEEDS CLARIFICATION: How long should lineage be retained? Forever, or with time-based archival? Impacts storage and compliance]
- **FR-015**: System MUST support lineage capture for external data sources [NEEDS CLARIFICATION: Which external sources should be supported? S3, databases, API endpoints, streaming sources? Scope impacts integration effort]

### Key Entities *(include if feature involves data)*

- **Dataset**: Represents a collection of data (file, table, object storage location). Attributes: name, version, location, schema, size, creation timestamp, owner
- **Transformation**: Represents a processing step that reads inputs and produces outputs. Attributes: code reference, execution time, status, parameters, input/output mappings
- **Model**: Represents a trained ML model. Attributes: model ID, version, framework, training timestamp, performance metrics. Relationships: derived from Datasets via Transformations
- **Pipeline**: Represents an orchestrated workflow. Attributes: pipeline definition, execution ID, status. Relationships: contains multiple Transformations
- **Feature**: Represents an engineered feature derived from raw data. Attributes: name, data type, calculation logic. Relationships: derived from Dataset columns via Transformations
- **Execution**: Represents a single run of code (notebook, pipeline step). Attributes: start/end time, user, environment, status. Relationships: consumes Datasets, produces Datasets or Models
- **LineageEdge**: Represents a directional relationship between entities. Attributes: source entity, target entity, relationship type (reads, writes, derives), timestamp

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Data scientists can trace complete lineage from any model back to source datasets in under 5 seconds via the UI
- **SC-002**: System automatically captures lineage for 95%+ of OpenShiftAI workloads without requiring code changes from users
- **SC-003**: Lineage graphs for typical ML pipelines (5-10 steps) render in under 2 seconds
- **SC-004**: Users can export compliance-ready lineage reports for any model in under 30 seconds
- **SC-005**: System handles lineage tracking for at least 10,000 pipeline executions per day without performance degradation
- **SC-006**: Reduce time to debug data-related model issues by 40% (measured by time from issue identification to root cause)
- **SC-007**: 80% of surveyed users agree that lineage visualization helps them understand their ML workflows better
- **SC-008**: Zero manual intervention required for lineage capture during normal OpenShiftAI usage
- **SC-009**: Lineage metadata storage grows at a predictable rate (< 10MB per 100 pipeline executions)
- **SC-010**: API response time for lineage queries is < 500ms for 95th percentile

## Assumptions

- Users are working within OpenShift AI (Red Hat OpenShift AI or upstream Open Data Hub) environment
- Primary ML frameworks are Python-based (scikit-learn, TensorFlow, PyTorch, XGBoost) with some support for R
- Datasets are primarily stored in S3-compatible object storage, with some relational databases and file systems
- Data Science Pipelines are based on Kubeflow Pipelines / Tekton
- Users have basic familiarity with lineage concepts and understand the value proposition
- Organizations are willing to accept small performance overhead (< 5%) for automatic lineage capture
- Lineage data is not classified as highly sensitive and can be stored in the cluster (separate from the actual data)
- Integration with OpenShift AI's existing RBAC model is sufficient (no custom authorization required)
- Initial version will focus on batch/pipeline workloads; real-time streaming lineage is out of scope for v1
- Organizations have capacity to run additional infrastructure components (database, API server) in their OpenShift cluster

## High-level Technical Approach

### Architecture Components

1. **Lineage Capture Agents**:
   - Lightweight Python SDK/library that users can import (or is auto-injected) into notebooks
   - Kubeflow Pipeline interceptors that hook into pipeline execution to capture step-level lineage
   - Integration with OpenShift AI Workbench to automatically instrument notebook kernels

2. **Lineage Storage Backend**:
   - Graph database (Neo4j or Apache AGE on PostgreSQL) for efficient traversal of lineage relationships
   - Metadata store for entity attributes and execution context
   - Time-series retention policy for managing growth

3. **Lineage API Server**:
   - REST API for querying lineage (GraphQL could be considered for flexible queries)
   - Integration with OpenShift OAuth for authentication
   - Support for both interactive queries and bulk export

4. **Lineage UI/Visualization**:
   - Web-based UI integrated into OpenShift AI dashboard
   - Interactive graph visualization using libraries like D3.js or Cytoscape.js
   - Search and filtering capabilities

5. **Integration Points**:
   - OpenShift AI Model Registry integration for linking models to lineage
   - Data Science Pipelines webhooks/listeners for automatic capture
   - Optional: External data catalog connectors (Apache Atlas, DataHub)

### Capture Methodology

- **Implicit Capture**: Automatically detect data reads/writes by hooking into common libraries (pandas, spark, database connectors)
- **Explicit Capture**: Provide decorators/context managers for users to explicitly mark lineage boundaries
- **Pipeline Integration**: Leverage existing pipeline DAGs to infer lineage structure

### Privacy & Security

- Lineage metadata is separate from actual data content
- RBAC ensures users only see lineage for resources they have access to
- Optional: Encrypt lineage data at rest and in transit

## Stakeholders & Target Users

### End Users
- **Data Scientists**: Primary users who need to understand data provenance for their models
- **ML Engineers**: Users who build and maintain ML pipelines and need debugging capabilities
- **MLOps Teams**: Users responsible for model governance and lifecycle management
- **Compliance Officers**: Users who need to generate audit reports and ensure regulatory compliance
- **Data Engineers**: Users who manage data infrastructure and need impact analysis for changes

### Internal Red Hat Products & Integrations
- **Red Hat OpenShift AI**: Primary platform where this solution will be deployed
- **OpenShift Data Science Model Registry**: Integration point for linking models to lineage
- **OpenShift Data Science Pipelines (Kubeflow/Tekton)**: Integration point for pipeline lineage capture
- **OpenShift OAuth**: Authentication and authorization integration
- **Red Hat OpenShift Logging/Observability**: Potential integration for lineage event streaming
- **Red Hat Developer Hub**: Potential integration for exposing lineage in developer portal

## Alternatives Considered

### Existing Open Source Solutions

1. **Apache Atlas**:
   - Comprehensive data governance platform with lineage capabilities
   - **Pros**: Mature, supports multiple data sources, Hadoop ecosystem integration
   - **Cons**: Heavy infrastructure requirements, complex setup, not optimized for ML workflows, weak Kubernetes integration
   - **Assessment**: Could be integrated as an external catalog, but not ideal as primary solution for OpenShiftAI

2. **DataHub (LinkedIn)**:
   - Modern data catalog with lineage tracking
   - **Pros**: Modern architecture, good UI, growing community, supports ML metadata
   - **Cons**: Requires significant infrastructure (Kafka, Elasticsearch, MySQL), may be over-featured for OpenShiftAI-only use case
   - **Assessment**: Strong candidate for organizations wanting unified catalog across platforms; may be too heavy for OpenShiftAI-only deployments

3. **MLflow Tracking**:
   - Experiment tracking with some lineage capabilities
   - **Pros**: Purpose-built for ML, simple to use, captures metrics and parameters
   - **Cons**: Limited data lineage (focuses on experiments, not full data pipelines), no graph visualization, weak multi-user support
   - **Assessment**: Complementary tool but doesn't solve data lineage problem comprehensively

4. **Marquez (OpenLineage)**:
   - Built on OpenLineage standard for lineage metadata
   - **Pros**: Standard-based, good for cross-platform lineage, active community
   - **Cons**: Requires instrumentation of data tools, limited ML-specific features, separate infrastructure
   - **Assessment**: Promising for standardization but requires significant integration work

5. **Pachyderm**:
   - Data versioning and lineage for data pipelines
   - **Pros**: Strong data versioning, good lineage tracking, Kubernetes-native
   - **Cons**: Requires adopting Pachyderm's pipeline model, may not integrate well with existing Kubeflow Pipelines
   - **Assessment**: Good for greenfield deployments but too disruptive for existing OpenShiftAI users

6. **Amundsen (Lyft)**:
   - Data discovery and metadata management
   - **Pros**: Good search and discovery, integrates with various sources
   - **Cons**: Lineage capabilities are limited, focus is on discovery not governance
   - **Assessment**: Better for data discovery than lineage tracking

### Recommendation

Build a **lightweight, OpenShiftAI-native solution** with optional integration hooks for existing catalogs (Atlas, DataHub). Rationale:
- OpenShiftAI users need minimal friction and Kubernetes-native deployment
- Existing solutions are either too heavy (Atlas, DataHub) or too limited (MLflow)
- A focused solution can provide better UX by integrating directly with OpenShiftAI components
- Organizations with existing governance platforms can still integrate via APIs/connectors

## Open Questions

1. **Column-Level Lineage Scope**: Should v1 include column/feature-level lineage tracking, or start with dataset-level only? Column-level provides more value but increases complexity significantly.

2. **Retention Policy**: What is the appropriate retention period for lineage metadata? Should there be tiered storage (hot/warm/cold) based on age?

3. **External Data Sources**: Which external data source types should be prioritized for lineage capture? (S3, relational databases, streaming sources like Kafka, API endpoints, etc.)

4. **Performance Trade-offs**: What is the acceptable performance overhead for automatic lineage capture? Should there be a "detailed" vs. "lightweight" capture mode?

5. **Multi-Cluster Lineage**: Should the solution support lineage tracking across multiple OpenShift clusters, or assume single-cluster deployment?

6. **Real-Time Streaming**: Should v1 include lineage for real-time/streaming ML workloads, or focus exclusively on batch pipelines?

7. **Code Lineage**: Should the system track which code versions (notebooks, scripts, container images) were used in transformations, or focus only on data lineage?

8. **Cost Model**: For cloud deployments, what is the expected cost of running the lineage infrastructure (storage, compute) and how does it scale with usage?

---

*Generated by Idea Workflow System*
