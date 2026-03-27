AI-Powered Variance Analysis Agent
Technical Architecture Design Document

Author: Vijayarani Nallachinnan
Version: 1.0
Environment: Enterprise On-Prem
Primary DB: Oracle
Target Users: Business & QA Teams
Processing Scale: 10M+ records per dataset

1. Executive Summary

The AI-Powered Variance Analysis Agent is designed to compare large datasets across Oracle environments and generate intelligent, business-readable variance insights.

The system enables:

Automated dataset comparison (same DB / cross DB / cross environment)

SQL pushdown processing for large-scale data (10M+ rows)

Intelligent variance classification

Natural language summaries for business users

API-first architecture for enterprise integration

On-prem secure deployment

This solution bridges technical variance detection with business-level interpretation.

2. Problem Statement

Current challenges:

Manual comparison of datasets across environments

Large data volume (>10M rows)

Business users unable to interpret raw variance outputs

Lack of automation and insight explanation

No AI-assisted root cause hypothesis generation

The goal is to build a scalable, intelligent, secure variance analysis agent.

3. High-Level Architecture
User (Business / QA)
        ↓
Minimal Web UI (FastAPI Templates)
        ↓
FastAPI Backend (Orchestrator Layer)
        ↓
-----------------------------------------
| Variance Engine (SQL Pushdown)       |
| LLM Insight Engine                   |
| Metadata Store                       |
| Vector Store (Optional)              |
-----------------------------------------
        ↓
Oracle Database(s)
4. Technology Stack
Layer	Technology	Reason
Backend	Python 3.11	Ecosystem + flexibility
API Framework	FastAPI	Async + API-first
Database	Oracle	Enterprise standard
Oracle Driver	oracledb / cx_Oracle	Native connectivity
Scheduling	APScheduler	Internal job scheduling
LLM	Enterprise Internal Endpoint	Secure AI inference
Vector Store	FAISS / Oracle AI Vector (if approved)	Context retrieval
Authentication	Enterprise SSO (if required)	Security
Deployment	On-Prem Linux Server	Data security
5. Core Components
5.1 API Orchestrator (FastAPI)

Responsibilities:

Accept comparison requests

Validate datasets

Trigger variance jobs

Call LLM for summarization

Store metadata

Expose job status endpoints

Provide downloadable results

Key Endpoints:

POST /compare

GET /status/{job_id}

GET /result/{job_id}

GET /summary/{job_id}

5.2 Variance Engine (SQL Pushdown Model)

Principle:
Heavy processing happens inside Oracle, not in Python memory.

Comparison Types Supported:

Same DB – Different Schemas

Same DB – Different Tables

Different DBs – DB Link

Different Environments

Comparison Strategy:

Step 1 – Primary Key Join
SELECT ...
FROM table_A a
FULL OUTER JOIN table_B b
ON a.pk = b.pk
Step 2 – Classification Logic

Missing in Source

Missing in Target

Value Mismatch

Perfect Match

Step 3 – Column-Level Delta Detection

Dynamic SQL generated based on metadata.

5.3 Metadata Repository

Stores:

Job ID

Dataset names

Execution timestamp

Row counts

Mismatch counts

Execution duration

Summary text

User ID

Status

Schema Example:

VAR_JOB_MASTER
VAR_JOB_STATS
VAR_JOB_SUMMARY
5.4 LLM Insight Engine

Purpose:
Convert raw variance metrics into business-readable explanations.

Input to LLM:

Row count differences

% mismatch

Column-wise variance

Missing records count

Historical comparison (optional)

Output:

Business summary

Possible root cause hypotheses

Risk classification

Suggested action

Example Output:

2.3% of records show financial value mismatches.
Likely caused by delayed ETL update in downstream environment.

LLM is invoked only after variance metrics are finalized.

5.5 Optional Vector Store (Phase 2)

Purpose:

Store historical job summaries

Retrieve similar past issues

Improve root cause suggestions

Options:

FAISS (local secure)

Oracle AI Vector (if enterprise-approved)

6. Data Flow

User submits dataset comparison

FastAPI validates metadata

SQL pushdown comparison executes in Oracle

Variance metrics stored in metadata tables

LLM generates business summary

Summary stored

User retrieves results

7. Performance Strategy

For 10M+ rows:

No pandas full dataset load

Use Oracle parallel hints

Use indexed primary keys

Process in batches if required

Materialized temp tables for intermediate joins

Collect statistics before execution

Expected Runtime:

10M rows: 3–8 minutes (indexed tables)

8. Security Architecture

On-prem deployment

No external LLM calls

Internal enterprise LLM endpoint only

Role-based access (Business vs QA)

No raw data sent to LLM (only aggregates)

Audit logging enabled

9. Deployment Model
Environment

Linux server

Gunicorn + Uvicorn workers

Internal HTTPS

Folder Structure
/app
    main.py
    routers/
    services/
        variance_engine.py
        llm_service.py
    db/
    models/
    scheduler/
10. Phased Implementation Plan
Phase 1 – Core Engine (4 Weeks)

Metadata schema

SQL pushdown comparison

API endpoints

Basic UI

Manual execution

Phase 2 – Intelligence Layer (3 Weeks)

LLM integration

Business summaries

Risk classification

Historical insights

Phase 3 – Automation & Enhancement (4 Weeks)

Scheduled comparisons

Vector memory

Auto root cause ranking

Email / notification integration

11. Future Roadmap

Cross-technology comparison (Oracle vs Snowflake)

Auto anomaly detection

Self-learning rule engine

Dashboard analytics

SLA breach prediction

Data quality scoring model

12. Risk Assessment
Risk	Mitigation
Full table scan	Index enforcement
LLM hallucination	Aggregate-only input
Memory overload	SQL pushdown
Cross-DB latency	Materialized staging
Unauthorized access	SSO integration
13. Success Metrics

80% reduction in manual comparison effort

60% faster defect triage

Business-readable reports

Reusable AI-assisted diagnostics

Standardized QA validation framework

14. Strategic Impact

This system:

Positions QA as an intelligence function

Moves from reactive defect detection to proactive risk analysis

Demonstrates AI-driven enterprise automation capability

Establishs reusable architecture for future AI agents