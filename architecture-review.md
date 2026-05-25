# Bi-Sipariş Architecture Review

Date: Monday, May 25, 2026

## Scope

This document captures the deep-dive architecture review for the Bi-Sipariş platform, with emphasis on:
- application architecture
- security posture
- performance bottlenecks
- refactoring opportunities
- operational risk

The review is intentionally technical and focuses on the issues that most directly affect reliability, maintainability, and long-term scale.

## Executive summary

Bi-Sipariş shows the shape of a mature but heavily iterated e-commerce system: it combines custom business logic, a WordPress/WooCommerce-style admin surface, third-party integrations, product ingestion pipelines, image handling, and export/sync jobs. The system appears to have grown through practical delivery rather than through a single upfront architecture plan. That is normal for a real business, but the current shape carries a few recurring risks:

- secrets and environment values are too easy to leak into code paths, logs, and config files
- certificate verification behavior is not consistently enforced in outbound requests
- release/versioning discipline is uneven across app, site, plugins, and supporting automation
- catalog synchronization, translation, and export jobs likely concentrate too much work into synchronous or semi-synchronous paths
- there are signs of coupling between data ingestion, product presentation, and operational reporting that make change expensive

The highest-priority findings are the secret-handling issues and the inconsistent TLS verification behavior. Those are security concerns first, and maintainability concerns second.

## Architecture overview

At a high level, Bi-Sipariş looks like a multi-surface commerce platform made of the following layers:

1. Customer-facing storefront
   - product browsing
   - cart and checkout
   - order lifecycle
   - local delivery / fulfillment logic

2. Admin / operations layer
   - product synchronization
   - order exception handling
   - fraud review and duplicate-payment review
   - manual operational overrides
   - CSV / FTP exports

3. Integration layer
   - upstream product feeds
   - supplier or marketplace ingestion
   - translation / enrichment jobs
   - media/image handling
   - monitoring and alerting

4. Mobile / app surface
   - versioned client release cycle
   - API-backed product/order views
   - app-store release management

5. Automation / background jobs
   - scrapers
   - scheduled exports
   - stock checks
   - reporting pipelines
   - one-off maintenance scripts

This is a pragmatic commerce architecture, but the boundaries between the layers appear porous. The same data often seems to be touched by ingestion, admin review, customer-facing display, and analytics/reporting code. That increases the chance that a change in one part of the stack has unintended effects elsewhere.

## Data and request flow

The typical lifecycle appears to be:

1. ingest products from upstream sources
2. normalize and translate product data
3. attach or transform imagery
4. persist into the commerce catalog
5. expose catalog data to storefront/app channels
6. export or synchronize data to external systems
7. monitor failures, fraud flags, and sync drift

The same pipeline likely handles both batch and near-real-time work. That can work well if it is carefully queue-based and idempotent. If not, it becomes a fragile chain of direct dependencies. Based on the review signals, the current system likely leans too far toward “do it now” rather than “queue it and observe it.”

## Security findings

### 1) Tiko secrets exposure risk

The review flagged Tiko-related secrets as a notable risk.

Why it matters:
- secrets may be stored in code, config, or deployment artifacts
- they may be reused across environments
- they may be available to more systems than intended
- rotation becomes harder once the secret has spread

Risk scenarios:
- a leaked token grants access to upstream services
- a compromised job runner can exfiltrate secrets
- developer logs or debug output can expose credentials indirectly
- a shared secret can outlive the deployment that created it

Recommendations:
- move all Tiko credentials into a managed secret store
- use environment-specific secrets, never shared global values
- rotate any credential that may have been copied into logs, diffs, or tickets
- ensure code never prints full secret material in exceptions or diagnostics
- add automated scanning for secret patterns in both source and runtime logs

### 2) TLS / `sslverify` inconsistency

The review specifically called out `sslverify` usage as a risk.

Why it matters:
- disabling TLS verification turns outbound HTTPS into “encrypted but not authenticated” transport
- an attacker on the network path can impersonate upstream services
- data feeds, update checks, or API responses can be tampered with silently

Observed concern:
- at least one code path appears to permit requests with verification disabled or inconsistently configured
- this is especially dangerous in admin/update/sync pathways because those often carry credentials or trusted payloads

Recommendations:
- require certificate validation by default everywhere
- only allow verification bypass in a controlled, temporary debug environment
- centralize HTTP client configuration so `sslverify` behavior cannot drift per call site
- add explicit telemetry for any request that would have used relaxed TLS settings
- write tests that fail when insecure transport flags are present in production configuration

### 3) Secrets and credentials in operational surfaces

The platform uses a lot of automation: exports, ingest jobs, monitoring, translations, and admin notifications. That increases the chance that secrets appear in places they should not.

Watchpoints:
- logs containing full URLs with tokens
- backups with config files included
- environment dumps in support diagnostics
- build artifacts containing API keys
- error emails with stack traces or request bodies

Recommendations:
- scrub sensitive fields at the logger boundary
- add a secret classification policy to logs and alerts
- reduce the number of systems that can read full production credentials
- use short-lived tokens where feasible

### 4) Authentication and trust boundaries

Because Bi-Sipariş appears to combine public commerce flows with privileged maintenance tasks, it is important that trust boundaries are explicit.

Potential risks:
- admin-only operations reachable from generic cron or webhook handlers
- internal job endpoints callable without strict auth
- data synchronization endpoints trusting payloads too broadly
- privileged actions inferred from client-side state instead of server-side authorization

Recommendations:
- make all privileged operations server-authenticated
- separate customer APIs from internal automation APIs
- use per-job credentials or signed job tokens
- add audit trails for all stock, price, and order edits

### 5) Supply-chain exposure

The platform likely relies on third-party packages, plugins, and external update mechanisms. That creates supply-chain exposure.

Recommendations:
- pin dependency versions
- track known-good hashes where possible
- review plugin update channels carefully
- keep a written policy for accepting upstream updates
- treat update automation as a privileged workflow

## Versioning and release discipline findings

The review indicated multiple signs of versioning inconsistency.

### Likely inconsistency patterns

- application versions, plugin versions, and backend integration versions may be moving independently
- mobile app release numbers can drift from backend API changes
- automation scripts may be updated outside a formal release process
- support or storefront behavior may change without a matching version note

Why it matters:
- it becomes hard to correlate bugs with deployment events
- rollback becomes difficult when the version graph is unclear
- customer support and operations lose the ability to say “what changed”
- app-store / plugin-store releases can lag the actual production state

Recommendations:
- introduce a single release manifest per deploy window
- record backend, frontend, app, and automation versions together
- tag all production artifacts consistently
- publish a short change log for each release train
- keep API compatibility notes alongside version bumps

### Versioning controls to add

- semantic versioning for app and shared libraries
- deployment tags for backend releases
- build metadata for automation jobs
- minimum-supported API version checks for clients
- deprecation windows for older integrations

## Performance findings

### 1) Batch jobs may be too synchronous

The largest likely bottleneck is the ingestion/translation/export pipeline.

Symptoms:
- long-running jobs
- large catalog updates
- one job triggering another job inline
- slow downstream exports when upstream data grows

Impact:
- operational latency
- stale catalog state
- higher failure blast radius
- duplicated work after retries

Recommendations:
- move heavy work to a queue
- make tasks idempotent
- checkpoint progress so restarts are cheap
- split translation, normalization, and export into separate stages

### 2) Catalog sync likely causes repeated full scans

If the system rebuilds large parts of the catalog on each run, it will eventually hit scale limits.

Common symptoms:
- full-table scans
- repeated image reprocessing
- repeated translation of unchanged text
- exporting more data than changed data

Recommendations:
- use delta syncs
- track content hashes per product field
- skip unchanged rows
- use incremental exports where downstream systems support them

### 3) Media handling can become a bottleneck

Image processing and cloud upload are often slower than text updates.

Risks:
- blocking the main pipeline on image upload
- re-uploading identical files
- generating multiple derivatives in the request path

Recommendations:
- cache image transforms
- deduplicate uploads by checksum
- offload image processing to background workers
- separate media availability from product publish state

### 4) Translation pipeline overhead

If product translation is done inline, the system pays a multiplier cost on every catalog sync.

Risks:
- slow ingestion
- inconsistent translations across retries
- higher API usage cost
- large failure blast radius when the translator is unavailable

Recommendations:
- translate only changed fields
- persist translation results independently
- add cache keys for text hashes
- decouple translation from catalog publish

### 5) Database and query shape

Even without a direct query plan, the architecture suggests a classic e-commerce risk profile:

- too many joins during product display
- repeated lookups for categories, stock, and pricing
- admin pages loading too much data at once
- reporting endpoints sharing the same database path as customer traffic

Recommendations:
- paginate all operational lists aggressively
- cache hot product/category data
- separate reporting read patterns from checkout traffic
- add indexes aligned to actual lookup paths

## Refactoring suggestions

### 1) Separate concerns more aggressively

The biggest structural improvement is to isolate the following domains:
- catalog ingestion
- catalog presentation
- order lifecycle
- export/reporting
- fraud and exception handling
- job scheduling

Each domain should have its own service boundary, data contracts, and logs.

### 2) Centralize outbound HTTP behavior

Create one shared HTTP client wrapper with:
- TLS verification enforced by default
- timeout policy
- retry policy
- backoff behavior
- structured error handling
- secret redaction in logs

This reduces the chance that one call site silently diverges from the rest.

### 3) Introduce explicit job orchestration

Heavy workflows should be expressed as jobs, not as hidden side effects.

Examples:
- product ingest job
- translation job
- image processing job
- export job
- reconciliation job
- fraud review job

### 4) Make config typed and auditable

Use typed config objects instead of scattered constants and ad hoc environment reads.

Benefits:
- easier review of what is required in each environment
- fewer accidental defaults
- clearer drift detection between staging and production

### 5) Improve observability

The system should emit:
- per-job duration
- per-step duration
- retry counts
- failure reasons
- data-change counts
- external API latency

This will make the next architecture review much easier and far more evidence-based.

### 6) Add explicit release gates

Before each release:
- verify version alignment across app/backend/jobs
- run secret scanning
- confirm TLS verification behavior
- confirm integration credentials are scoped correctly
- capture a release note

## Operational hardening checklist

Immediate:
- rotate any suspicious Tiko secrets
- enforce `sslverify=true` or equivalent in all production request paths
- remove any logging of secrets or auth headers
- audit privileged job endpoints

Short term:
- move expensive tasks to a queue
- make sync jobs incremental
- add version manifesting
- introduce release notes and rollback identifiers

Medium term:
- split ingestion from publication
- isolate order-processing workflows from catalog sync
- improve cache strategy
- separate reporting from customer traffic paths

Long term:
- formalize service boundaries
- standardize contracts between modules
- treat infrastructure, app code, and automation scripts as one release system
- add automated architecture drift checks

## Suggested prioritization

### P0
- Tiko secret rotation and secret-store migration
- enforce TLS verification in outbound requests
- stop sensitive logging

### P1
- decouple heavy jobs from request/cron paths
- make sync/export incremental
- establish version manifesting

### P2
- restructure modules by domain
- improve observability
- isolate reporting and analytics workload

## Closing assessment

Bi-Sipariş is operationally sophisticated and clearly built by someone who understands the business deeply. The main issue is not capability; it is compaction. Too many responsibilities have accumulated in too few places. That creates security risk, performance drag, and long-term maintenance cost.

The platform will benefit most from:
- tighter secret management
- strict TLS verification
- better job orchestration
- clear version discipline
- stronger separation between ingestion, commerce, and reporting

Those changes would make the system safer now and much easier to evolve later.
