## Big Assumptions
- We cannot access other external APIs.
- “Health” for a CRE asset represents its performance relative to market averages across a defined set of metrics.
- This feature will operate within the existing Kubernetes ecosystem of the product.
- All data involved is public. Property data is not private and has no owner constraints.
- Current property data is stored in a single DB table (e.g. metrics are columns, suggested by JSON structure).
- Since this is not scaled yet, it should be easy to make DB changes at this time without breaking APIs

---

## Caching / Local Storage

### Notes
- Assume market data is uploaded once per month, on the 1st. We should confirm with the API provider how stable this schedule is and whether service disruptions could delay availability.
- If release timing is not guaranteed, implement a daily check until new data is detected.
- Assume uploads will never exceed monthly frequency and that one upload will occur each month.

- On-demand fetching vs background population:
  - On-demand improves efficiency and avoids unnecessary API calls.
  - However, first access each month may be slow.
  - If multiple asset managers load dashboards simultaneously, we risk duplicate upstream requests, increasing API cost and latency.
  - If first-load latency is unacceptable, pre-populating the cache becomes preferable.

- API pricing should influence architecture:
  - If API calls are inexpensive, proactive monthly background loading may be simpler and safer.
  - If expensive, lazy loading with safeguards may be justified.
  - We should avoid prematurely optimizing for API cost if it significantly increases system complexity.

- Background execution options:
  - Application-level timers introduce async complexity and persistent resource usage.
  - Kubernetes CronJobs are externally observable, ephemeral, retry-capable, and operationally clear.
  - CronJobs align better with the platform and reduce application-level complexity.

- Single job vs per-market jobs:
  - Per-market jobs improve isolation and observability for failures.
  - However, at 50+ markets, job sprawl may reduce clarity and increase operational noise.
  - A single job with internal market-level logging provides a cleaner operational model.

- Assume markets are in similar timezones.
  - Schedule jobs during off-peak hours (e.g., overnight U.S.) to reduce load and contention.

### Decision
Balance performance, operational simplicity, and API cost:

- Run a Kubernetes CronJob on the 1st of each month.
- Continue running daily until fresh data is detected.
- Execute overnight (U.S. timezone).
- Use a retry limit to prevent runaway API costs.
- Retrieve all markets in a single job execution.
- Skip markets that already have fresh data.
- After refreshing a market, recompute health scores for all properties within that market.

This approach prioritizes predictability and cost control while avoiding unnecessary architectural complexity.

### Process
K8s CronJob created  
→ Runs daily  
→ Checks each market for freshness  
→ Refreshes stale data  
→ Job completion triggers recomputation where necessary

---

## Health Score

### Notes
- Assume all metrics are numeric.
- We do not have access to full market-level raw distributions, limiting statistical techniques such as standard deviation or percentile-based scoring.
- Percentile calculations are not feasible without broader market data.

- Normalization:
  - Normalize each metric to a 0–100 scale.
  - Convert negative polarity metrics before comparison.
  - Score relative to market averages.
  - More sophisticated statistical techniques could improve accuracy but are not feasible given available data.

- Weighting:
  - Metrics are not equally important.
  - Assign explicit weights per metric.

- Confidence considerations:
  - Assume low / medium / high confidence levels exist.
  - Sample size, data quality, and confidence apply at the market level, not per metric.
  - Therefore, they cannot be used directly for per-metric weighting.
  - However, they can inform an overall confidence score for the computed health score.

- Missing data:
  - If a metric is missing for a given month, use the most recent available value.
  - Reduce its weight to prioritize freshness.
  - Stale data is typically more valuable than no data, provided its influence is discounted.

- Scope:
  - Only use fields present in the metrics API, market data API, and property data.

- Update strategies:
  - Scheduled jobs are too rigid for reacting to configuration changes.
  - A polling loop introduces unnecessary constant work unless made event-driven.
  - A Kubernetes controller provides a clean, event-driven mechanism.

- Technology choice:
  - Python is the most known language by the company
  - All k8s core components are written in Go, offering better support
  - Go is probably the best choice as Python offers poor controller support

### Decision
Implement a Kubernetes controller (or extend an existing one) that watches:

- Changes to the metrics ConfigMap hash.
- Completion of cache jobs.

Recompute health scores when:
- The ConfigMap hash changes (indicating algorithm updates).
- A cache job completes successfully and newer market data exists.

Health scores must store:
- The timestamp of the market data used for computation.
- The configuration hash used during computation.

If either differs from the current state, recompute.

This ensures:
- Event-driven execution.
- Eventual consistency.
- No unnecessary recomputation.
- Clear traceability between configuration, data version, and computed score.

---

## Serving Data

### Notes
- Consumers: dashboards, alerts, reports, and potentially other APIs.
- Serving over an API provides high availability and clean separation of concerns.

- Technology choice:
  - Golang may offer higher raw performance.
  - However, given team familiarity and development velocity, Python FastAPI may be more pragmatic.
  - Performance requirements for this use case are unlikely to demand a lower-level implementation initially.

- Architecture:
  - If an existing API serves a similar purpose, extending it may reduce operational overhead.
  - Otherwise, a dedicated service keeps microservice boundaries clean and responsibilities focused.

- Aggregation:
  - Aggregate region-based data would need to extract city from address string.
  - Aggregate time-based data most useful and already defined

### Decision
Implement a simple Python FastAPI service to expose health scores.

Endpoints:
- Get/List health score by property ID (time-based via query params).
- List health scores by market ID (time-based via query params).

The API should:
- Be well documented.
- Abstract underlying computation logic.
- Remain read-focused and lightweight.
- Allow possibility of returning both the current and an age-ranged list of health scores for each property.

---

## Impact of New Metrics

### Notes
- Newly introduced metrics should trigger an alert.
- Automatic inclusion without analysis risks degrading score integrity.
- A temporary low-weight auto-inclusion strategy was considered but rejected due to:
  - Polarity differences.
  - Unit inconsistencies.
  - Risk of unintended distortion.
- Options for supporting additional metrics:
  - DB column per metric (requires schema change for every new metric, some properties have empty null fields).
  - JSON fields (allows more complex dynamic schemas, but makes aggregation and management of data harder).
  - Separate table for metrics (more dynamic, no schema or code changes, but could result in a lot of tables).
- Configuration-driven metrics require strict validation and safe parameter handling (e.g., parameterized queries).

### Decision
Metrics are defined via configuration (e.g., ConfigMap), not hard-coded.

This enables:
- Algorithm updates without redeployment.
- Faster iteration.
- Safer controlled evolution of scoring logic.

The ConfigMap includes a configuration hash, allowing each health score to be associated with a specific scoring configuration.

The underlying table contains no sensitive data, and fields are accessible to avoid injection-related risks.

### Design
```
kind: ConfigMap
data:
    hash: XXXXXXXXXXX
    metrics:
        - db_column: current_occupancy_rate
          api_key: avg_occupancy_rate
          polarity: positive
          weight: 0.75
```

# Future Enhancements
- Support timezone grouping and split CronJobs to reduce peak database contention.
- Enable market-specific algorithms and weight sets.
- Replace ConfigMap with a Custom Resource:
  - Stronger state tracking.
  - Improved observability.
  - More expressive spec (e.g., per-market metadata, last updated times).
- If ownership constraints emerge:
  - Introduce owner foreign keys.
  - Enforce row-level access controls.
- Enhance public API filtering (e.g., filter by health score threshold for alerting use cases).
- Selective caching as we scale:
  - To prevent DB costs when storing hundreds of stale, unaccessed records.
  - Only cache resources that are frequently accessed locally.
  - Prefer caching current up to date property data over data that is many years old.
- Incrementally improve aggration strategy:
  - Take user/constumer feedback about how data is being used/accessed.
  - Only aggregate useful data.
  - Add additional aggregation for sought after data.
  - Consider adding city, state and market type aggregation, design leaving this option available.
- Analyze metric change frequency:
  - If change frequency is low, prefer simpler DB design with migrations required for new additions.
  - If rollback/removal of metrics is low, also prefer simpler DB design with migrations for changes.
