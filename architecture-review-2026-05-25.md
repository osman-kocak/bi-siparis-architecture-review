# Bi-Sipariş Architecture Review

Review date: Monday, May 25, 2026

Repository reviewed: osman-kocak/bisiparisAPP
Comparison baseline: bi-siparis-app/AUDIT_REPORT.md dated 2026-03-16

## Executive summary

The latest codebase shows meaningful progress against the earlier audit. Several of the biggest launch-blocker class issues from the March report are now addressed in code:

- push notification ownership and token registration flows now enforce server-side ownership checks
- loyalty gamification flows now include idempotency keys and transient locks
- sort and recommendation endpoints now use batch fetches and cache layers instead of obvious N+1 patterns
- CORS handling in the WordPress mobile app manager has been tightened to a whitelist
- mobile networking now has retry, backoff, concurrency limiting, and improved maintenance detection
- KKTC phone normalization was corrected in the latest commit

That said, the repository still has a few high-impact architectural risks. The biggest remaining issues are:

1. recommendation API credentials are exposed to the mobile client bundle
2. payment/card data still traverses the app/backend boundary instead of staying inside a tokenized or hosted payment flow
3. the WordPress mobile app manager still eagerly loads a very large number of classes at bootstrap
4. recommendation tracking is still public-facing enough to be spoofed or poisoned, even with rate limiting

Overall assessment: the system is materially safer and faster than it was in the March audit, but two security issues and one bootstrap/scalability issue still deserve immediate attention.

## What changed since the previous analysis

Compared with the 2026-03-16 audit, the current tree shows evidence of the following improvements:

### Fixed or strongly improved

- Push registration no longer trusts a caller-supplied user_id; the current user comes from JWT auth only.
- Push read / read-all / history flows now verify notification ownership or device ownership.
- Spin wheel, daily bonus, and scratch card flows now use idempotency and transient locks to reduce duplicate reward risk.
- Sort endpoint product loading is now batched through wc_get_products, with cache priming.
- Recommendation endpoints now batch fetch product data and cache home/similar/cross-sell/cart responses.
- CORS origin handling now uses a whitelist instead of wildcard origin behavior.
- Mobile API client now uses explicit concurrency limits and retry behavior instead of uncontrolled parallel calls.
- Maintenance detection was made more conservative by batching failures before showing a maintenance state.
- KKTC phone handling was corrected in the latest commit.

### What is still structurally the same

- The platform is still a multi-surface commerce system with a large WordPress plugin layer, mobile app, and external recommendation service.
- The WordPress side still owns a lot of orchestration logic.
- The mobile app still depends on several backend contracts that are effectively treated as privileged infrastructure.
- The recommendation subsystem still uses client-side request signing.

## Current architecture snapshot

The repository now reads like a mature commerce platform with five major zones:

1. Mobile app
   - Expo Router app
   - Zustand + React Query state
   - SecureStore/MMKV split storage
   - recommendation, checkout, loyalty, push, and content services

2. WordPress application plugin layer
   - remote config
   - push notification APIs
   - stories/content/festival modules
   - checkout and loyalty integration surfaces
   - admin AJAX and REST endpoints

3. Recommendation stack
   - mobile client provider and API client
   - WP proxy endpoints
   - separate VPS-side recommendation service

4. Commerce checkout and payment flows
   - address management
   - totals calculation
   - payment methods
   - Tiko payment integration

5. Operational automation
   - cron hooks
   - export/reporting
   - sync and cache warmup jobs
   - marketing / lifecycle nudges

The architecture is functional and clearly product-driven, but still quite coupled. WordPress remains the orchestration center for many concerns that would be safer if they were isolated behind narrower boundaries.

## Remaining issues

### 1) Recommendation API secret is exposed in the mobile bundle

Evidence:
- `bi-siparis-app/src/config/env.ts` defines `RECOMMENDATION_API_KEY` and `RECOMMENDATION_API_SECRET` through `EXPO_PUBLIC_` environment variables.
- `bi-siparis-app/src/services/recommendation/api/ApiClient.ts` generates HMAC signatures on-device using that secret.

Why this matters:
- anything prefixed with `EXPO_PUBLIC_` is bundled into the client app and can be extracted by a determined user
- if the value is a true secret, it is no longer secret
- an attacker can forge signed requests, automate abuse, poison analytics/training data, or exhaust rate limits

Impact of fixing it:
- prevents credential extraction from the app bundle
- reduces API abuse and poisoning risk
- makes rate limiting and attribution trustworthy again
- keeps the recommendation stack in a proper trust boundary

Recommended direction:
- remove the secret from the client bundle entirely
- move request signing to server-side code, or use a short-lived token minted by the backend
- keep only a public identifier on the client if one is required
- rotate the current key material after the change

Severity: high

### 2) Card data still crosses the app/backend boundary

Evidence:
- `bi-siparis-app/src/services/payment.ts` still accepts `card_number`, `card_expiry`, `card_cvv`, and `card_name`.
- `bi-siparis-app/src/services/checkout.ts` passes those values to the backend create-order flow.

Why this matters:
- the app/backend path remains in PCI-sensitive territory
- even if the backend forwards data immediately, the infrastructure still processes raw card data
- breach impact is larger when card data is handled outside a hosted/tokenized payment flow

Impact of fixing it:
- reduces PCI scope significantly
- lowers breach blast radius
- simplifies compliance and audit requirements
- makes the payment architecture more resilient and easier to maintain

Recommended direction:
- use hosted fields, a payment SDK, or tokenized card entry so raw card data never reaches the application server
- keep the backend responsible only for tokens, payment intent IDs, or gateway references
- remove raw card fields from application-level request objects wherever possible

Severity: high

### 3) WordPress mobile app manager eagerly loads too much at bootstrap

Evidence:
- `wordpress-plugin/bi-siparis-mobile-app-manager.php` includes a very large number of classes with top-level `require_once` calls.
- This happens before route-specific usage is known.

Why this matters:
- every request pays the cost, even if only one small route is needed
- memory use and startup latency increase
- the plugin becomes harder to reason about and harder to split later

Impact of fixing it:
- reduces request bootstrap cost
- improves TTFB and PHP worker efficiency
- lowers memory pressure under load
- makes the plugin easier to evolve by feature area

Recommended direction:
- lazy-load class groups on the hooks that actually need them
- split admin-only, REST-only, cron-only, and content-only modules into smaller bootstrap paths
- keep the main plugin file as thin as possible

Severity: medium-high

### 4) Public recommendation tracking can still be spoofed

Evidence:
- `bi-siparis-app/src/services/recommendations.ts` posts tracking events from the client.
- `wordpress-plugin/includes/class-recommendations-rest-api.php` exposes `/recommendations/track` as a public endpoint and accepts `user_id` in the request body.
- There is IP rate limiting, but the endpoint is still easy to generate from outside the app.

Why this matters:
- analytics and recommendation data can be contaminated
- malicious or automated traffic can distort training signals
- rate limits help, but they do not establish identity or trust

Impact of fixing it:
- improves data quality for recommendations
- reduces model pollution and abuse
- makes event attribution more reliable

Recommended direction:
- bind tracking events to a signed session/device token rather than trusting arbitrary user_id values
- treat tracking as telemetry from a known device identity, not as a public write surface
- preserve rate limiting, but add identity validation before accepting events

Severity: medium

## Positive findings

The current codebase also contains several strong architectural choices:

- JWT refresh handling is centralized in the mobile API client.
- Secure storage is separated from fast local app state.
- Loyalty race conditions were treated seriously and now have idempotency/locking protection.
- Public read endpoints are mostly separated from authenticated write flows.
- Caching is now used in the sort and recommendation layers where it matters.
- The app now uses concurrency limiting to avoid request storms.
- Recent maintenance and retry logic shows a good understanding of shared-hosting behavior.

These are practical improvements that reduce day-to-day operational risk.

## Impact summary

If the remaining issues are fixed, the main benefits would be:

Security
- removes exposed secret material from the client
- reduces PCI and credential exposure
- lowers abuse and poisoning risk in telemetry/recommendations

Scalability
- lowers bootstrap cost in WordPress
- reduces unnecessary server work per request
- improves worker availability under load

Performance
- shorter PHP bootstrap time
- fewer wasted network retries
- lower chance of request spikes from client-side abuse or duplicate telemetry

Maintainability
- cleaner trust boundaries
- easier debugging and release management
- less coupling between payment, recommendation, and WordPress orchestration logic

## Priority order

P0
- remove recommendation secret from client bundle and move signing server-side
- remove raw card handling from app/backend request objects

P1
- lazy-load the WordPress mobile app manager bootstrap
- tighten recommendation tracking identity

P2
- continue splitting large orchestration classes into smaller domain modules
- keep reducing direct coupling between WordPress, mobile, and recommendation concerns

## Final assessment

The latest commit set clearly improves the platform. The biggest earlier issues around push authorization, loyalty duplication, and search/recommendation performance have been materially improved. The codebase is now in a much better state than the March audit.

The remaining work is mostly about hardening the trust boundary and reducing monolithic bootstrap cost. If the recommendation secret and card-data flow are fixed, the platform’s risk profile drops sharply. If the WordPress bootstrap is also slimmed down, the platform should become noticeably easier to operate at scale.
