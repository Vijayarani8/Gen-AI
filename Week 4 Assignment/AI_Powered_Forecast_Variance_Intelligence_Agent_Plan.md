# Plan: AI-Powered Variance Analysis Agent

## TL;DR

Build a 3-layer system (API, Database, Web) that enables enterprise-scale dataset comparison across Oracle environments with SQL pushdown processing and AI-driven business insights. Phase 1 focuses on core comparison engine; Phase 2 preps vector store infrastructure. Follow strict API-first development: build data models and services first, expose via FastAPI endpoints, then wire the Web UI to consume those endpoints.

---

## Steps

### LAYER 1: API (FastAPI Orchestrator)

#### 1.1 Core Service Classes

**ComparisonRequest** (Data Model)
- Fields: source_dataset, target_dataset, comparison_type, user_id, request_metadata
- Validation: dataset existence, schema compatibility checks
- Serialization: Pydantic model for request/response

**ComparisonService**
- `validate_datasets(source, target)` → bool
- `initiate_comparison(request)` → job_id
- `check_job_status(job_id)` → status, progress, metrics
- `cancel_job(job_id)` → confirmation
- Dependencies: VarianceEngine, JobOrchestrationService

**VarianceEngine** (SQL Pushdown Layer)
- `generate_comparison_sql(source_schema, target_schema, comparison_type)` → SQL string
- `execute_variance_detection(sql, db_connection)` → variance_metrics dict
- `classify_variance_types(metrics)` → {missing_source, missing_target, mismatches, perfect}
- `compute_statistics(source_rows, target_rows, mismatches)` → {counts, percentages, execution_time}
- Methods use FULL OUTER JOIN logic + dynamic column predicates

**JobOrchestrationService**
- `queue_comparison_job(request)` → job_id
- `manage_job_lifecycle(job_id, state_transitions)` 
- `trigger_llm_summarization(job_id, metrics)` after variance metrics ready
- `cleanup_temp_tables(job_id)` after job completion
- Integration: APScheduler for job queueing

**LLMIntegrationService**
- `format_metrics_for_llm(variance_metrics)` → structured prompt
- `call_enterprise_llm(formatted_input)` → raw_response
- `parse_llm_response(raw_response)` → {summary, root_cause, risk_level, actions}
- `store_summary_in_metadata(job_id, parsed_response)` 
- Constraint: Input is aggregates only, no raw data

**MetadataRepository** (Data Access Layer)
- `insert_job_master(job_details)` 
- `update_job_status(job_id, new_status)`
- `insert_job_stats(job_id, variance_metrics)`
- `insert_job_summary(job_id, llm_insights)`
- `get_job_history(user_id, filters)`
- `insert_audit_log(job_id, action, user_id, details)`

**AuthenticationService** (Minimal SSO)
- `validate_sso_token(token)` → user_id, roles
- `enforce_role_based_access(user_roles, resource)` → bool
- Middleware: token validation on all protected endpoints

#### 1.2 API Endpoints (FastAPI Router)

**POST /api/compare**
- Input: ComparisonRequest
- Logic: validate → create job → queue → return job_id
- Output: {job_id, status: PENDING}

**GET /api/status/{job_id}**
- Output: {status, progress_percent, execution_time_seconds, current_stage}

**GET /api/result/{job_id}**
- Output: VAR_JOB_STATS {source_rows, target_rows, missing_source, missing_target, mismatch_count, mismatch_percent}

**GET /api/summary/{job_id}**
- Output: VAR_JOB_SUMMARY {business_summary, root_cause_hypothesis, risk_classification, suggested_actions, confidence_score}

**POST /api/cancel/{job_id}**
- Logic: mark job as CANCELLED, cleanup in-progress resources
- Output: {status, confirmation}

**GET /api/download/{job_id}**
- Query param: ?format=csv|excel
- Output: file stream

**GET /api/datasets**
- Query params: filters, pagination
- Output: {available_datasets, schema_info, row_counts}

**GET /api/history**
- Query params: user_id, date_range, status_filter, pagination
- Output: paginated list of past comparisons

**GET /api/schemas**
- Query param: db_ref
- Output: {schema_names, table_names, column_metadata}

**GET /api/audit/logs**
- Query params: date_range, user_filter
- Output: audit trail entries with timestamps, actions, user IDs

---

### LAYER 2: Database (Oracle Metadata Repository)

#### 2.1 Core Tables (DDL)

**VAR_JOB_MASTER**
- PK: job_id (UUID)
- source_dataset_name, target_dataset_name
- source_db_ref, target_db_ref (connection identifiers)
- comparison_type (ENUM: SAME_SCHEMA, SAME_DB, CROSS_DB, CROSS_ENV)
- job_status (ENUM: PENDING, QUEUED, RUNNING, COMPLETED, SUMMARIZED, FAILED, CANCELLED)
- created_by, created_timestamp, started_timestamp, completed_timestamp
- request_metadata (JSON CLOB)
- Index: by job_id, created_by, created_timestamp

**VAR_JOB_STATS**
- PK: job_stats_id (UUID)
- FK: job_id → VAR_JOB_MASTER
- source_row_count, target_row_count
- missing_in_source_count, missing_in_target_count
- value_mismatch_count, perfect_match_count
- mismatch_percentage (NUMBER(5,2))
- execution_duration_seconds
- Index: by job_id

**VAR_JOB_SUMMARY**
- PK: summary_id (UUID)
- FK: job_id → VAR_JOB_MASTER
- business_summary (CLOB)
- root_cause_hypothesis (CLOB)
- risk_classification (ENUM: LOW, MEDIUM, HIGH)
- suggested_action (CLOB)
- confidence_score (NUMBER(3,2))
- generated_timestamp
- llm_model_version (VARCHAR)
- Index: by job_id

**VAR_COLUMN_VARIANCE** (Optional, for detailed analysis)
- PK: variance_id (UUID)
- FK: job_id → VAR_JOB_MASTER
- column_name
- source_value_sample, target_value_sample
- mismatch_count, variance_percentage
- Index: by job_id, column_name

**VAR_AUDIT_LOG**
- PK: audit_id (UUID)
- FK: job_id → VAR_JOB_MASTER
- action (VARCHAR: initiated, executing, completed, failed, cancelled)
- action_timestamp
- user_id
- details (JSON CLOB)
- Index: by job_id, action_timestamp, user_id

#### 2.2 Supporting Tables

**DATASET_REGISTRY**
- dataset_id (PK), dataset_name, db_ref, schema_name, table_name
- row_count_estimate, last_analyzed_date
- owner_user_id, created_timestamp

**DB_CONNECTION_CONFIG**
- db_ref (PK), db_instance_name, connection_string, host, port
- driver_type (oracledb | cx_Oracle)
- encrypted_credentials (vault reference)

**VECTOR_STORE_INDEX** (Phase 2 prep - not fully implemented)
- index_id (PK), job_id (FK)
- embedding_vector (VECTOR data type if Oracle AI Vector approved)
- similarity_threshold
- created_timestamp

#### 2.3 Indexes & Performance Tuning

- Index on VAR_JOB_MASTER(created_timestamp, job_status) for history queries
- Index on VAR_JOB_MASTER(created_by) for user-scoped filtering
- Index on VAR_JOB_STATS(job_id) for stat retrieval
- Composite index on VAR_AUDIT_LOG(job_id, action_timestamp)
- Consider partitioning VAR_JOB_MASTER by created_timestamp (monthly) for large historical volumes

---

### LAYER 3: Web UI (FastAPI Templates + HTML/CSS)

#### 3.1 Page Components

**Dashboard Page** (`/dashboard`)
- Layout: Header (user info) + Quick-start form + Recent jobs list
- Components:
  - `ComparisonQuickForm` - source/target dropdowns, submit button
  - `RecentJobsWidget` - table of last 5-10 comparisons with status badges
  - `SystemStatusWidget` - job queue health, LLM service status

**Compare Page** (`/compare`)
- Layout: Form pane (left) + Instruction panel (right)
- Components:
  - `DatasetSelector` - typeahead search for source dataset
  - `ComparisonTypeRadioGroup` - SAME_SCHEMA / SAME_DB / CROSS_DB / CROSS_ENV
  - `TargetDatasetSelector` - typeahead search
  - `AdvancedOptions` (collapsible) - batch processing, schedule option
  - `SubmitAndProgress` - submit button + real-time status polling

**Results Page** (`/results/{job_id}`)
- Layout: Metrics panel + Summary panel + Details accordion
- Components:
  - `VarianceMetricsCard` - row count comparison (numbers + bar chart)
  - `MismatchBreakdownChart` - pie/bar chart (missing_source, missing_target, mismatches, perfect)
  - `BusinessSummaryCard` - LLM-generated narrative text
  - `RootCausePanel` - hypothesis + confidence badge
  - `RiskClassificationBadge` - LOW/MEDIUM/HIGH with color coding
  - `SuggestedActionsPanel` - actionable recommendations
  - `DownloadButton` - CSV/Excel export
  - `ColumnVarianceAccordion` (expandable) - detailed column-level breakdown

**History Page** (`/history`)
- Layout: Filter bar + Paginated table
- Components:
  - `DateRangeFilter` - start/end date pickers
  - `StatusFilter` - multi-select (ALL, COMPLETED, FAILED, RUNNING)
  - `JobHistoryTable` - columns: job_id, source, target, status, created_date, actions
  - `PaginationControls` - prev/next, page size selector
  - `QuickActionButtons` - View Results, Re-run, Delete

**Details Page** (`/details/{job_id}`)
- Layout: Summary section + Execution metrics + Audit trail
- Components:
  - `JobMetadataPanel` - job_id, start/end times, duration, user
  - `ExecutionTimingsChart` - step-by-step execution duration breakdown
  - `ColumnVarianceTable` - sortable/filterable table of column mismatches
  - `AuditLogTimeline` - timeline view of job events (initiated → running → completed → summarized)
  - `SampleDataComparison` - side-by-side source/target samples for mismatches

**Admin Page** (`/admin`, SSO-gated)
- Layout: Tabbed interface (Datasets | Connections | LLM Config | Audit Logs)
- Components:
  - `DatasetManager` - form to add/edit/delete dataset registry
  - `DBConnectionForm` - form to add DB connection config (encrypted credentials)
  - `LLMModelSelector` - dropdown for active LLM model version
  - `AuditLogViewer` - searchable table of all audit entries with advanced filtering

#### 3.2 Shared UI Components

**Header/Navigation**
- Logo + Title
- User info (username, logout)
- Breadcrumb navigation
- Admin link (if user has admin role)

**LoadingSpinner** - for async operations
**StatusBadge** - color-coded status displays (PENDING, RUNNING, COMPLETED, FAILED, etc.)
**ErrorAlert** - error message dismissal panel
**ConfirmationModal** - for destructive actions (delete, cancel)
**PaginationWidget** - reusable pagination controls
**ExportDropdown** - CSV/Excel export options

#### 3.3 Frontend Architecture

**Templates Location**: `/app/templates/`
- `base.html` - master layout with header, nav, footer
- `dashboard.html`
- `compare.html`
- `results.html`
- `history.html`
- `details.html`
- `admin.html`

**Static Assets Location**: `/app/static/`
- `css/main.css` - global styles, responsive design
- `css/components.css` - component-specific styles
- `js/api-client.js` - fetch wrapper for backend endpoints
- `js/utils.js` - helper functions (date formatting, status color mapping)
- `js/polling.js` - job status polling logic

**Form Submission & Validation**
- Client-side HTML5 validation (required fields, dropdown constraints)
- Server-side validation in FastAPI endpoints
- Async polling for job status updates (50-100ms intervals)

#### 3.4 User Flows (Page Navigation)

```
Homepage (Dashboard)
  ↓
User clicks "New Comparison"
  ↓
Compare Page (select datasets, type, click submit)
  ↓
Automatic redirect to Results Page with job_id
  ↓
Polling status until COMPLETED/SUMMARIZED
  ↓
Display metrics + LLM summary + recommendations
  ↓
User can download, re-run, or view history
```

---

## Verification

**API Layer Testing**
- Unit tests for each Service class (ComparisonService, VarianceEngine, LLMIntegrationService)
- Integration tests for FastAPI endpoints using TestClient
- Mock Oracle connections + LLM responses for CI/CD
- Load test variance engine with 10M+ row simulation

**Database Layer Testing**
- Verify all tables created with correct schema, indexes, constraints
- Test data insertion/update/retrieval for each table
- Verify foreign key relationships and cascade behavior
- Query performance baseline on 1M+ historical records

**Web UI Testing**
- Browser compatibility (Chrome, Edge, Firefox)
- Form submission validation (required fields, type checking)
- Job status polling accuracy
- Results page rendering with various metric scenarios
- Responsive design on desktop/tablet/mobile

**End-to-End Flow**
- Submit comparison → monitor job → view results → download report
- Verify audit logs record all actions
- Test error handling (invalid datasets, connection failures, LLM timeouts)

---

## Decisions

- **API-First Approach**: All business logic in service classes; UI consumes HTTP API only. Enables future mobile/CLI clients.
- **SQL Pushdown**: No pandas DataFrames or Python-side joins; all comparison logic in Oracle SQL for 10M+ row scalability.
- **Minimal Auth Layer**: Basic SSO token validation; assume enterprise SSO provider (Okta/SAML) already available. Role enforcement at endpoint level via decorator middleware.
- **FastAPI Templates**: Server-side rendered HTML (Jinja2) over React SPA for simpler on-prem deployment, faster initial load, shared authentication session.
- **Async Job Processing**: APScheduler queues comparisons; FastAPI endpoints return job_id immediately and clients poll status. Prevents timeout on long-running SQL queries.
- **Vector Store (Phase 2 Partial)**: Include table schema (`VECTOR_STORE_INDEX`) and DB structure, but do NOT implement embedding generation or similarity search logic in Phase 1. Placeholder for Phase 2 extension.
- **Audit Everything**: All user actions logged to `VAR_AUDIT_LOG` for compliance and debugging.
