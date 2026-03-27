# Design Guidelines: AI-Powered Variance Analysis Agent

**Principle**: Keep everything simple. Build in small, testable pieces. Avoid complex architecture.

---

## Core Philosophy

- **Simplicity First**: Choose straightforward solutions over elegant abstractions.
- **Incremental Development**: Ship small, working pieces before adding complexity.
- **Testability by Design**: Every component should be independently testable with minimal mocking.
- **No Premature Optimization**: Build for correctness first, optimize only when proven necessary.

---

## API Layer - Simplicity Guidelines

### 1.1 Service Classes Should Be Thin

**Do:**
- Single responsibility per service (one job, one class)
- Keep methods focused (5-10 lines of logic per method)
- Pure functions where possible (same input = same output, no hidden side effects)

**Don't:**
- Mix database, business logic, and orchestration in one class
- Create base classes or abstract patterns unless there are 3+ identical implementations
- Build generic, reusable frameworks before you have the specific use case

**Example Pattern:**
```
ComparisonService
  ├─ validate_datasets() - calls DatasetValidator
  ├─ initiate_comparison() - creates record, returns job_id
  └─ check_job_status() - reads DB, returns status

VarianceEngine
  ├─ generate_sql() - builds SQL string (pure function)
  ├─ execute_comparison() - runs SQL, collects metrics
  └─ classify_results() - categorizes mismatches
```

### 1.2 API Endpoints Should Be Thin Too

**Do:**
- Endpoint logic: validate input → call service → return result
- Keep business logic in services, not in endpoints
- Return the same data structure as the service produces

**Don't:**
- Add conditional logic, loops, or complex transformations in endpoints
- Create custom response wrappers until you have 5+ endpoints with identical patterns
- Build middleware for things that can be done in services

**Max Endpoint Size**: 10-15 lines of code (validation + service call + response)

### 1.3 Avoid Inheritance & Generic Patterns

**Do:**
- Write the same 5 lines of code twice if needed
- Use composition: service A calls service B

**Don't:**
- Create abstract base classes to reduce duplication
- Build generic "comparison" or "processor" frameworks
- Over-engineer for Phase 2 features until Phase 2 starts

### 1.4 Dependency Injection: Keep It Simple

**Do:**
- Pass dependencies as constructor parameters
- Use a simple dependency container (no magic framework autowiring)

**Don't:**
- Use complex DI frameworks with decorators/annotations
- Create singletons or global state
- Hide dependencies behind factory methods

**Example:**
```python
class ComparisonService:
    def __init__(self, variance_engine, job_repo, llm_service):
        self.variance_engine = variance_engine
        self.job_repo = job_repo
        self.llm_service = llm_service
```

### 1.5 Error Handling: Explicit Over Hidden

**Do:**
- Catch specific exceptions, log, return error to user
- Return meaningful error messages to API caller
- Let unexpected errors bubble up (they should fail tests)

**Don't:**
- Swallow exceptions silently
- Return generic "something went wrong" messages
- Create custom exception hierarchies with 8+ exception types

---

## Database Layer - Simplicity Guidelines

### 2.1 Tables Should Be Flat, Not Normalized to Death

**Do:**
- 5-7 core tables with clear purposes
- Add columns if it simplifies queries
- Store JSON in CLOB for variable data (request_metadata)

**Don't:**
- Create junction tables for every relationship
- Normalize until you have 20+ tables
- Use complex domain models that require multiple joins

### 2.2 SQL Should Be Readable, Not Clever

**Do:**
- Write SQL that a SQL developer can understand in 2 minutes
- Add comments for non-obvious logic
- Use simple FULL OUTER JOIN for variance detection

**Don't:**
- Use window functions, CTEs, and subqueries unless absolutely necessary
- Write dynamic SQL generators that produce unpredictable queries
- Optimize queries before you've proven they're slow

**SQL Gen Pattern:**
```python
def generate_comparison_sql(source_schema, target_schema):
    return f"""
    SELECT 
        a.pk, b.pk,
        CASE WHEN a.pk IS NULL THEN 'MISSING_IN_SOURCE'
             WHEN b.pk IS NULL THEN 'MISSING_IN_TARGET'
             ELSE 'MATCH' END as variance_type
    FROM {source_schema}.table_a a
    FULL OUTER JOIN {target_schema}.table_b b 
        ON a.pk = b.pk
    """
```

### 2.3 No Fancy Schema Tricks

**Do:**
- Use standard data types (VARCHAR, NUMBER, CLOB, TIMESTAMP)
- Explicit NOT NULL constraints where needed
- Simple indexes on foreign keys and frequently filtered columns

**Don't:**
- Use Oracle-specific features (flashback, partitioning) until required
- Create complex constraints or triggers
- Partition data before you have millions of rows

### 2.4 Incremental Data Models

**Phase 1:**
- VAR_JOB_MASTER
- VAR_JOB_STATS
- VAR_JOB_SUMMARY
- VAR_AUDIT_LOG

**Phase 2 (if needed):**
- VAR_COLUMN_VARIANCE
- VECTOR_STORE_INDEX

Don't create Phase 2 tables in Phase 1.

---

## Web UI Layer - Simplicity Guidelines

### 3.1 No Frontend Framework Complexity

**Do:**
- Use FastAPI templates (Jinja2) for server-side rendering
- Vanilla JavaScript (fetch API) for async operations
- Bootstrap or simple CSS for styling

**Don't:**
- Use React, Vue, or other SPAs
- Build client-side state management (Redux, Vuex)
- Create reusable UI component libraries

### 3.2 Each Page Should Be Self-Contained

**Do:**
- One HTML template = one page
- API calls inline in template or simple JS file
- Pass all needed data from backend to template (no client-side rendering)

**Don't:**
- Share state between pages
- Build complex page-to-page navigation logic
- Create a custom router

**Example Structure:**
```
/app/templates/dashboard.html
  - calls GET /api/recent-jobs in JavaScript block
  - displays results immediately when page loads
```

### 3.3 Forms: Simple and Direct

**Do:**
- HTML5 form validation (required, type checks)
- POST to backend, backend redirects on success
- Show validation errors in simple alert or span

**Don't:**
- Live field validation with server roundtrips
- Complex form state management
- Multi-step wizards

### 3.4 Polling: Not Elegant, But Works

**Do:**
- Simple setInterval() to poll job status every 2 seconds
- Update page content with response data
- Stop polling when job is complete

**Don't:**
- WebSocket connections
- Complex event systems
- Real-time sync frameworks

**Example:**
```javascript
function pollJobStatus(jobId) {
  const interval = setInterval(async () => {
    const res = await fetch(`/api/status/${jobId}`);
    const status = await res.json();
    document.getElementById('status').innerText = status.status;
    
    if (status.status === 'COMPLETED') {
      clearInterval(interval);
    }
  }, 2000);
}
```

### 3.5 UI Complexity Budget

**Do:**
- 6 pages total (dashboard, compare, results, history, details, admin)
- 5-7 reusable components (header, footer, status badge, alert, table)
- CSS: ~500 lines total (simple grid + flexbox, no animations)

**Don't:**
- 10+ pages
- Component libraries
- CSS-in-JS or preprocessors

---

## Build & Test Strategy - Keep It Simple

### 4.1 Testing Pyramid: Simple at the Base

**Unit Tests (60%)**
- Test each service method in isolation
- Mock: database, LLM service
- Example: `test_variance_engine_classify_types.py`

**Integration Tests (30%)**
- Test service → database layer
- Test service → LLM layer
- Use test database (separate Oracle schema)

**End-to-End Tests (10%)**
- Full workflow: submit comparison → get results
- Run in staging environment only

**Don't:**
- Aim for 100% code coverage
- Start with E2E tests
- Mock at too many levels

### 4.2 Incremental Build Order

**Week 1-2: Core Database + Services**
1. Create VAR_JOB_MASTER table
2. MetadataRepository class (insert/read job records)
3. VarianceEngine class (generate SQL, execute, collect metrics)
4. Unit tests for each

**Week 3-4: API Orchestration**
1. ComparisonService class
2. JobOrchestrationService class
3. POST /api/compare endpoint
4. GET /api/status/{job_id} endpoint
5. Integration tests

**Week 5-6: Web UI**
1. Base template + header/footer
2. Dashboard page + quick form
3. Results page (mock API response first)
4. Real API integration

**Week 7-8: Refinement**
1. History page
2. Admin page (basic dataset config)
3. Error handling + edge cases
4. Performance tuning

### 4.3 No Premature Optimization

**Do:**
- Build it, make it work
- Measure performance
- Optimize only if profiling shows a bottleneck

**Don't:**
- Use connection pooling before you have concurrent requests
- Add caching before you've proven data retrieval is slow
- Optimize SQL before running EXPLAIN PLAN

---

## Deployment Strategy - Keep It Manual

### 5.1 Minimal Infrastructure

**Do:**
- Single Linux server (on-prem)
- Gunicorn + Uvicorn for Python
- Oracle JDBC driver (already enterprise standard)
- Rsync or simple git pull for deployment

**Don't:**
- Docker/Kubernetes
- Configuration management tools (Ansible, Terraform)
- Blue-green deployments

### 5.2 Configuration Management: Plain Files

**Do:**
- `config.yaml` or `.env` file for settings
- Environment variables for secrets
- Simple if/else in code for environment-specific behavior

**Don't:**
- Configuration frameworks
- Dynamic config servers
- Feature flags for everything

### 5.3 Monitoring: Logs and Dashboards

**Do:**
- Write to application logs (stdout + file)
- Query logs directly for debugging
- Simple table of jobs + status in admin UI

**Don't:**
- APM tools (Datadog, New Relic)
- Distributed tracing
- Complex alerting rules

---

## Architecture Diagram: Simple

```
User (Browser)
    ↓
FastAPI (Jinja2 Templates)
    ↓
┌─ ComparisonService
│    ├─ VarianceEngine (SQL Builder)
│    ├─ JobOrchestrationService
│    └─ LLMIntegrationService
├─ MetadataRepository (SQL Queries)
└─ AuthenticationService (Token Verify)
    ↓
Oracle Database
    ├─ VAR_JOB_MASTER
    ├─ VAR_JOB_STATS
    ├─ VAR_JOB_SUMMARY
    └─ VAR_AUDIT_LOG
```

**That's it.** No message queues, no caches, no event buses.

---

## Decision Log: Why Simple?

| Decision | Why Simple | Tradeoff |
|----------|-----------|----------|
| Jinja2 templates, not React | Faster to build, easier to deploy on-prem | Less interactive UI, page reloads |
| Polling for job status, not WebSockets | No need for bidirectional connection | Slight delay in UI update (2-sec poll) |
| Single service, flat database | Easier to understand and modify | May need refactoring if scope doubles |
| No caching layer | Simplifies debugging, less state to manage | Queries may be slightly slower |
| Inline JavaScript, no build step | Instant deployment, everyone can edit HTML | Not suitable for large frontend teams |
| Basic error handling, not middleware | Services handle their own errors | Repeated error handling code |

---

## Checklist: Is This Design Simple Enough?

- [ ] Can I explain each class in one paragraph?
- [ ] Can I draw the data flow on a whiteboard in 2 minutes?
- [ ] Do I have fewer than 10 service classes?
- [ ] Can I run unit tests without Docker/containers?
- [ ] Is the largest method fewer than 30 lines?
- [ ] Can a new developer understand the codebase in 4 hours?
- [ ] Is there a clear path to test each component in isolation?
- [ ] Do I avoid abstraction layers that aren't needed yet?

If you answer "no" to any of these, simplify.

---

## When to Add Complexity

**Only add complexity when:**

1. **You've shipped the MVP** and it works
2. **You have a specific problem** (not a hypothetical one)
3. **You've measured it** (profiling shows the bottleneck)
4. **It will solve the problem** (not just "be nice to have")

**Example**: "The results page takes 30 seconds to load because SQL queries are slow" → add caching. Not: "We might have caching in the future, let's build a generic cache layer."

---

## Summary

- **Start with the simplest working solution**
- **Measure before optimizing**
- **Refactor when you see the pattern repeated 3+ times**
- **Ship early, iterate based on real feedback**
- **Avoid frameworks, build what you need**

**The best architecture is one you can explain to a junior developer in an hour.**
