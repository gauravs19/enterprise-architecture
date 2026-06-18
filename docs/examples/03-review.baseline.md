# Architecture Review — Django Monolith Web Application

**Reviewed:** Verbal description of deployment topology and key design choices (no diagrams or code provided) · **Against:** Implied quality goals for a ~50k DAU production web app — availability, security, performance, operability · **Date:** 2026-06-18

> This is a **Mode 3 (Review / assess)** pass from the enterprise-architecture toolkit. Findings are graded against ISO/IEC 25010 quality attributes and EA principles, each with evidence and a concrete fix — not personal taste. Where a fix implies follow-up work, I point to the relevant follow-up mode.

---

## Verdict

**Needs revision.** A monolith on a single instance is a perfectly reasonable *shape* for this scale, but the current design carries a security breach waiting to happen and several single points of failure on the critical path. The monolith is not the problem; the operational and security posture around it is. None of these are hard to fix, and you do **not** need to re-platform or microservice anything to fix them.

---

## Top priorities (fix first)

1. **Get secrets out of the repository and rotate them — today.** Committed credentials are an active, exploitable breach, not a future risk.
2. **Remove the synchronous third-party email call from the request path** — it couples your uptime and latency to a vendor you don't control.
3. **Add observability (logs + metrics + error tracking) and database backups.** Right now you are flying blind and have no proven recovery path if the single DB is lost.

---

## Findings

### 🔴 Critical

- **Secrets committed in `settings.py`** — Database passwords, the Django `SECRET_KEY`, and the email API key are in version control. *Evidence:* "Secrets live in settings.py committed to the repo." Anyone with repo access (and anyone who ever cloned it, including former staff) holds your production keys; the `SECRET_KEY` alone enables session forgery and tampering with signed data. Git history means deleting the line is **not** sufficient. *Fix:* (1) Rotate every secret now — DB password, `SECRET_KEY`, email API key. (2) Load config from environment variables or a secrets manager (AWS Secrets Manager / SSM Parameter Store, since you're on EC2) via `os.environ` / `django-environ`. (3) Purge history or, more pragmatically, treat all historical secrets as compromised and rely on rotation. (4) Add a pre-commit secret scanner (e.g. `gitleaks`). → *Record the config-management approach as an ADR (Mode 2).*

- **Single PostgreSQL instance with no replica and (apparently) no stated backup** — This is a single point of failure on your most irreplaceable asset: data. *Evidence:* "one PostgreSQL instance with no replica." An instance failure, a bad migration, or accidental deletion can mean permanent data loss for 50k users, with no defined RPO/RTO. *Fix:* (1) Enable automated backups + point-in-time recovery immediately (if self-managed, set up `pg_dump`/WAL archiving; strongly consider migrating to **RDS/Aurora**, which gives backups, PITR, and a managed failover replica with little ops burden). (2) Add a read-replica/standby for failover. (3) Document and *test* the restore procedure — an untested backup is not a backup.

### 🟠 Major

- **Synchronous third-party email call inside the request path** — Your request latency and availability are now hostage to an external vendor. *Evidence:* "we call a third-party email API synchronously inside the request path." If the provider is slow or down, your user-facing requests block, time out, and tie up Gunicorn/uWSGI workers — a vendor brownout can cascade into a full outage of *your* site. *Fix:* Move email sending to an asynchronous task queue (**Celery + Redis/SQS**, or Django's `django-q`/`dramatiq`). The request enqueues and returns; a worker sends with retries, timeouts, and a dead-letter path. This also unlocks retry-on-failure, which a synchronous call lacks.

- **No logging, metrics, or tracing** — The system is unobservable. *Evidence:* "No logging or metrics." You cannot detect an incident, debug a production error, measure latency, or know whether a fix worked. At 50k DAU this is operating blind. *Fix:* (1) Structured application logging shipped off-box (CloudWatch Logs / ELK). (2) Error tracking (**Sentry**) for exceptions. (3) Metrics — request rate, latency percentiles, error rate, DB connections, worker queue depth (CloudWatch / Prometheus + Grafana). (4) A `/health` endpoint and uptime check. → *Hand off to an observability design (Mode 2 doc).*

- **Single EC2 instance = application-tier SPOF and vertical-only scaling** — One instance means any host failure, deploy, or AZ event is a full outage, and you can only grow by buying a bigger box. *Evidence:* "single Django monolith on one EC2 instance." 50k DAU is well within a monolith's comfort zone, but not on *one* undifferentiated host with no redundancy. *Fix:* Run **2+ stateless app instances behind an Application Load Balancer across 2 AZs** (Auto Scaling Group). This is the single highest-leverage availability change and requires no code re-architecture — Django is already horizontally scalable provided sessions/state live in the DB or cache, not on local disk. Verify nothing is written to local filesystem (uploads → S3).

- **No caching layer** — Every request hits PostgreSQL, making the un-replicated DB both a SPOF *and* the throughput ceiling. *Evidence:* "There's no caching." *Fix:* Add **Redis/ElastiCache** for Django's cache framework (session store, per-view/fragment caching, expensive query results) and rate-limiting. This also becomes the broker/result backend for the Celery work above, so it's shared infrastructure, not a new silo.

### 🟡 Minor

- **No mention of a CDN / static-asset strategy** — Serving static and media files from the app instance wastes worker capacity. *Fix:* Static/media to **S3 + CloudFront**; use `whitenoise` only as a minimum baseline.
- **Deployment/release process unstated** — With one instance, a deploy is presumably an in-place restart, i.e. downtime per release. *Fix:* Once behind a load balancer, do rolling deploys; add a basic CI pipeline with the secret scanner and test run as gates.
- **No stated rate limiting / WAF** — A public app at this scale is a target for abuse and credential stuffing. *Fix:* Add rate limiting (via Redis) and consider AWS WAF in front of the ALB.

### 🔵 Suggestions

- **Keep the monolith.** Do not let this review push you toward microservices — at 50k DAU a well-run Django monolith is the *right* call (simplicity / YAGNI). Fix the operational envelope, not the architecture style.
- **Record the key decisions as ADRs (Mode 2):** secrets management, async email via Celery, and the move to RDS. The reasoning will save the next engineer a re-litigation.
- **Once observability exists, set explicit quality goals** — e.g. p95 latency target, monthly availability SLO, RPO/RTO. You currently have no stated targets to design against, which makes "is it sound?" partly unanswerable. Defining them is itself a worthwhile exercise.
- **Consider a container/context diagram (Mode 1)** showing user → ALB → app instances → Postgres / Redis / async workers / email provider. The missing pieces (queue, cache, replica) become obvious the moment they're drawn.

---

## What's done well

- **A monolith is the correct architectural style for this stage.** It's the simplest thing that can work, avoids distributed-systems complexity you don't need, and is easy to reason about. This is a genuine strength, not a default.
- **PostgreSQL is a sound, boring, scalable datastore choice** — the issue is its redundancy and backups, not the engine.
- **Being on EC2 (AWS) means every fix above is a managed service away** — RDS, ElastiCache, ALB/ASG, SQS, Secrets Manager, CloudWatch. You can reach a resilient posture without leaving the platform or rewriting application code.

---

## Recommended follow-ups

- **Rotate all secrets and externalize config** → then record the approach as an ADR (Mode 2).
- **Stand up async email (Celery + Redis/SQS)** → ADR for the messaging choice (Mode 2).
- **Migrate DB to RDS Multi-AZ with backups/PITR** → ADR (Mode 2); update the container diagram (Mode 1).
- **Add ALB + Auto Scaling Group across 2 AZs** → container diagram showing the redundant tier (Mode 1).
- **Instrument logging/metrics/error-tracking + health checks** → short observability design doc (Mode 2).
- **Define quality goals (latency SLO, availability SLO, RPO/RTO)** so future reviews have a target to measure against.

### Suggested sequencing
1. **Day 1 (security):** rotate secrets, move to env/secrets manager, add secret scanning.
2. **Week 1 (data durability):** enable DB backups + PITR; verify a restore.
3. **Weeks 2–3 (decoupling & visibility):** async email queue; logging, metrics, Sentry, health checks.
4. **Weeks 3–4 (availability & performance):** RDS Multi-AZ, ALB + ASG (2 AZs), Redis cache, CDN for static.
