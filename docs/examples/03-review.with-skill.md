# Architecture Review — Django monolith on EC2 (~50k DAU)

**Reviewed:** Your described architecture (single Django app on one EC2 instance, single PostgreSQL with no replica, secrets in committed `settings.py`, no caching, synchronous third-party email call in the request path, no logging/metrics) · **Against:** Implied quality goals for a ~50k-DAU production web app — availability, security, performance, operability · **Date:** 2026-06-18

I'm doing a **Mode 3 architecture review** from the enterprise-architecture skill, grading against ISO/IEC 25010 quality attributes and EA principles rather than personal taste. Each finding carries evidence and a concrete fix. A note up front: you didn't state explicit quality goals (target availability, latency SLO, RPO/RTO, compliance scope), and at 50k DAU that absence is itself a finding — I've assumed commercially normal expectations for a system of this size.

## Verdict

**Needs revision.** The monolith itself is a perfectly reasonable choice at this scale — but the surrounding design has two production-breaking defects (committed secrets, single points of failure on the critical path) and a missing operability layer that means you'd be flying blind during the incidents this design makes likely. Don't scale further on it until the Criticals are closed.

To be clear about what is *not* wrong: a Django monolith for 50k DAU is a sound, even admirable, choice. The problems here are operational and security hygiene, not your decomposition strategy. Resist anyone who reads this review as "we must go microservices" — that would be solving a problem you don't have while ignoring the ones you do.

## Top priorities (fix first)

1. **Get secrets out of the repo and rotate them** — they are currently leaked to everyone with repo access and every fork/clone/CI log, forever in git history.
2. **Remove the single points of failure on the critical path** — one EC2 instance and one un-replicated Postgres mean any single failure (or a routine reboot) is a full outage with potential data loss.
3. **Move the synchronous email call out of the request path and add observability** — so requests stop being hostage to a third party, and so you can actually see what's happening when something breaks.

## Findings

### 🔴 Critical

- **Secrets committed in `settings.py`** — Database credentials, the email API key, and Django's `SECRET_KEY` live in version control. *Evidence:* secrets in `settings.py` committed to the repo. This is a `Security` breach by definition: anyone with repo read access (employees, contractors, CI, anyone who ever cloned or forked) has your production keys, and they persist in git history even after deletion. A leaked `SECRET_KEY` lets an attacker forge sessions and signed tokens; a leaked DB credential is direct data access. *Fix:* (1) Treat all committed secrets as compromised and **rotate every one now** — DB password, email API key, `SECRET_KEY`. Rotating `SECRET_KEY` invalidates existing sessions; plan for it. (2) Load secrets from environment variables or a secret manager (AWS Secrets Manager / SSM Parameter Store, since you're on EC2). (3) Purge from history (`git filter-repo` / BFG) and add `settings.py`'s secret-bearing parts to `.gitignore` via a `local_settings`/env split. (4) Add a pre-commit secret scanner (gitleaks/trufflehog) so this can't recur.

- **Single EC2 instance is a single point of failure on the critical path** — The entire application has no redundancy. *Evidence:* one EC2 instance serving all 50k DAU. This fails `Reliability/Availability`: any instance fault, AZ event, kernel panic, or even a routine OS patch reboot = a total outage. You also have no safe deploy path (every deploy is downtime). *Fix:* Run **≥2 app instances across ≥2 Availability Zones behind an Application Load Balancer**, in an Auto Scaling Group. This requires the app to be stateless (see the session/state note under Major). This is the single biggest availability win and is cheap relative to the outage cost at 50k DAU.

- **Single PostgreSQL instance, no replica — SPOF plus data-loss risk** — The database is both a SPOF and, depending on backups, a data-loss risk. *Evidence:* one Postgres instance, no replica. If the DB host dies you are down until you rebuild; if storage is lost and backups are absent/untested, data is gone (unbounded `RPO`). *Fix:* Move to **managed Postgres (Amazon RDS or Aurora) with Multi-AZ** for automatic failover, plus **automated backups and point-in-time recovery**, and **test a restore** (an untested backup is not a backup). Add a read replica only if/when read load justifies it — Multi-AZ standby is for failover, not read scaling. Define an explicit RPO/RTO so "good enough" is measurable.

### 🟠 Major

- **No logging, metrics, or tracing — the system is unobservable** — *Evidence:* "no logging or metrics." This fails `Operability`. With the reliability gaps above you *will* have incidents, and right now you'd have no signal: no error rates, no latency data, no audit trail, no way to do root-cause analysis or even know an outage is happening before users tell you. *Fix:* Add structured application logging shipped off the host (CloudWatch Logs / ELK), RED metrics (Request rate, Error rate, Duration) via CloudWatch or Prometheus + Grafana, error tracking (Sentry), and a health-check endpoint for the load balancer. Wire at least one alert (error rate / 5xx / DB connection failures) to a human. This is a prerequisite for safely operating everything else here.

- **Synchronous third-party email call inside the request path** — *Evidence:* "call a third-party email API synchronously inside the request path." This hurts `Performance efficiency` and `Reliability`: the user's request latency now includes the email provider's latency, and if the provider is slow or down, your requests hang, tie up worker threads/gunicorn workers, and can cascade into a full-app brownout — a third party can take *you* down. *Fix:* Make email **asynchronous** — enqueue a job (Celery + Redis/SQS, or Django-Q) and return immediately; a worker sends the email with retries, timeouts, and a dead-letter path. As an interim mitigation before the queue lands, at minimum set an aggressive timeout on the HTTP call so a hung provider can't pin your workers indefinitely.

- **No caching strategy** — *Evidence:* "there's no caching." At 50k DAU every read likely hits Postgres directly, which is a `Performance efficiency` and `Scalability` smell and adds load to your single (Critical) DB. *Fix:* Introduce a cache tier — **Redis/Memcached (ElastiCache)** — for hot reads, expensive queries, and (once you go multi-instance) **shared session storage**. Use Django's cache framework; start with per-view/queryset caching on the hottest paths identified by the metrics you're about to add. Don't cache blindly — measure first, then cache the proven hotspots.

- **No stated quality goals / SLOs** — *Evidence:* the request asks "is this design sound?" with no availability target, latency budget, RPO/RTO, or compliance scope. Per the review rubric, you can't assess fitness for purpose without them, and the team can't make the redundancy-vs-cost trade-offs above rationally. *Fix:* Write down target availability (e.g. 99.9%), p95 latency budget, RPO/RTO, and whether you handle PII/regulated data. This is a half-day exercise that makes every decision below objective.

- **App state likely blocks horizontal scaling** — *Evidence:* a single-instance design typically relies on local state (default DB/local-memory sessions, local file uploads, in-process caches). Going multi-instance (Critical #2) requires statelessness. *Fix:* Externalize sessions to Redis/DB, move uploaded media to S3, and ensure no instance-local state is assumed. Verify this explicitly before adding instances behind the load balancer.

### 🟡 Minor

- **Every deploy is an outage** — With one instance there is no rolling/blue-green path. This resolves naturally once you're multi-instance behind an ALB, but call it out: until then, deploys mean downtime. *Fix:* rolling deploys via the ASG once redundancy lands.
- **No documented network/security boundary** — Not described, so flagging the gap: ensure the DB is in a private subnet (not publicly reachable), security groups are least-privilege, and TLS terminates at the load balancer. *Fix:* confirm/establish these as part of the RDS + ALB migration.
- **No mention of automated tests or CI gates** — Not described; for a monolith this size, test coverage on critical paths is what makes the above changes safe to ship. *Fix:* confirm CI runs tests and the secret scanner on every PR.

### 🔵 Suggestions

- **Record the load-balanced + Multi-AZ datastore decision as an ADR.** These are significant, hard-to-reverse infrastructure choices; capture the context and trade-offs (Multi-AZ vs single, RDS vs Aurora, Celery vs SQS) so the next engineer understands the why. (Hand off to Mode 2 — ADR/MADR.)
- **Draw a C4 Container diagram of the target state** (ALB → app instances → RDS Multi-AZ, Redis cache, queue + email worker) so the to-be architecture is shared and reviewable. (Hand off to Mode 1.)
- **Stay a monolith.** Defer any service-decomposition conversation until a real scaling or team-boundary pressure forces it — not before. The fixes here keep the monolith healthy for well beyond 50k DAU.

## What's done well

- **A monolith is the right altitude for this stage.** One deployable, one codebase, no distributed-systems tax — exactly what a 50k-DAU product should run. You've avoided the very common over-engineering trap, and the issues above are all addable *without* re-architecting.
- **PostgreSQL is a solid, boring, correct datastore choice.** The problem is its topology (single, un-replicated), not the database.
- **The stack is conventional and well-understood**, which means every fix above is a standard, well-trodden pattern (ALB + ASG, RDS Multi-AZ, Celery, ElastiCache, Secrets Manager) with abundant Django-specific guidance — low execution risk.

## Recommended follow-ups

- **Rotate all committed secrets today** and move them to AWS Secrets Manager / SSM; purge git history; add gitleaks to pre-commit. (Immediate; Mode 3 → remediation)
- **Migrate DB to RDS/Aurora Multi-AZ with tested PITR backups**, then put the app behind an ALB across ≥2 AZs in an ASG. (First infra sprint; record as an ADR — Mode 2)
- **Introduce a job queue (Celery/SQS) and move email off the request path**; add Redis (ElastiCache) for cache + shared sessions. (Same window as going multi-instance)
- **Stand up observability (logs + RED metrics + Sentry + one alert) before the migrations**, so you can see them land safely.
- **Write down quality goals/SLOs (availability, latency, RPO/RTO, compliance)** to make the cost/redundancy trade-offs objective.
- **Produce a target-state C4 Container diagram** of the to-be architecture for the team. (Mode 1)
