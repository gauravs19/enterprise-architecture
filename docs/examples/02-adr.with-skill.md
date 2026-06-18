I'll write this up as an **Architecture Decision Record** using the **MADR (Markdown Any Decision Records)** format — the right tool here, since you're making one architecturally significant, hard-to-reverse choice and the value lies in capturing *why* and the *trade-offs you weighed*, not just the answer. (This maps to Mode 2 — "decisions as code" — in the enterprise-architecture toolkit.)

Save this as `docs/decisions/0001-messaging-system-for-order-events.md` in your repo so it's diff-able and lives next to the code it governs.

---

# 0001. Use AWS SNS/SQS for order events

- Status: proposed
- Date: 2026-06-18
- Deciders: <fill in — e.g. Backend Lead, Platform/SRE Lead, Eng Manager>
- Tags: messaging, order-events, eventing

## Context and Problem Statement

We need a messaging backbone to publish and consume **order events** (order created, paid, fulfilled, cancelled, etc.) between services. We expect a sustained throughput of **~10k events/sec**, we want the ability to **replay** past events (to rebuild read models, onboard new consumers, and recover from consumer bugs), and the team has **no Kafka operational experience**. Which messaging system should we adopt: Apache Kafka, RabbitMQ, or AWS SNS/SQS?

## Decision Drivers

- **Throughput**: must comfortably sustain ~10k events/sec with headroom for growth and bursts.
- **Event replay**: consumers must be able to re-read historical events from an arbitrary point, not just process-and-discard.
- **Operational burden / team skills**: the team has no Kafka ops experience; we want to minimize cluster operations (broker tuning, partition rebalancing, ZooKeeper/KRaft, storage, upgrades).
- **Time-to-value**: get to production safely without a multi-month platform-engineering investment.
- **Ordering & delivery semantics**: order lifecycle events for a given order must be processed in order; we need at-least-once delivery and idempotent consumers.
- **Cost predictability** at the expected volume.
- **Reversibility**: prefer a choice that doesn't lock us out of moving later (see "More Information").

## Considered Options

1. **Apache Kafka** (self-managed or via Amazon MSK / Confluent Cloud)
2. **RabbitMQ** (with the streams plugin for replay)
3. **AWS SNS + SQS** (fan-out), optionally with **EventBridge Archive & Replay** or **Kinesis Data Streams** where native replay is required

## Decision Outcome

**Chosen option: "AWS SNS/SQS (fan-out), with a managed log for replay where needed."**

Rationale tied to the drivers: it best matches a team with **no Kafka ops experience** while still meeting **~10k events/sec** and giving us a credible **replay** story without us operating brokers. SNS topic → SQS queue fan-out is a fully managed, pay-per-use pattern with effectively no capacity planning, per-queue scaling, and a dead-letter queue (DLQ) story out of the box. SNS/SQS alone does **not** provide log-style replay (SQS deletes messages on consumption), so we pair it with a managed, replayable log for the streams that need it:

- **EventBridge** with **Archive & Replay** for event routing where replay windows are coarse-grained and operational, **or**
- **Amazon Kinesis Data Streams** (managed, log-based, replay within retention up to 365 days) for the order-event stream that downstream services rebuild state from.

Net: we get Kafka-like *durable-log replay* on the one stream that needs it, via a managed service, and cheap managed fan-out for everything else — with no broker fleet to run.

> If a single concrete shape is wanted: **publish order events to an SNS FIFO topic for fan-out to per-consumer SQS FIFO queues** (MessageGroupId = orderId gives per-order ordering), **and in parallel write the canonical event stream to Kinesis Data Streams** as the replayable system of record. Consumers that need only live processing read SQS; consumers that need to rebuild/replay read Kinesis.

### Consequences

- **Good**: Minimal operational burden — no clusters, brokers, partitions, or upgrades to manage; directly fits the team's current skills.
- **Good**: Elastic and managed at ~10k events/sec; SQS scales horizontally with effectively no tuning, Kinesis scales by shards (~10k events/sec ≈ on the order of 10 shards at 1 MB/s or 1k records/s per shard — size to real payload).
- **Good**: Fast time-to-value; IAM-based security, DLQs, and AWS-native observability (CloudWatch) come for free.
- **Good**: Pay-per-use; no idle cluster cost at low volume.
- **Bad**: Replay is **not** uniform — SQS has none, so replay depends on the Kinesis/EventBridge-Archive layer; this adds a second mechanism and some architectural complexity vs. Kafka's "one log does everything."
- **Bad**: **Vendor lock-in** to AWS; SNS/SQS/Kinesis APIs are proprietary. Mitigate by keeping publish/subscribe behind a thin internal interface.
- **Bad**: Per-message pricing can exceed a self-run broker at very high sustained volume; revisit cost if throughput grows well beyond 10k/sec sustained.
- **Bad**: FIFO throughput has limits and higher cost than standard; confirm SNS/SQS FIFO quotas cover 10k/sec for your message-group cardinality, or use high-throughput FIFO / standard-with-idempotency.
- **Neutral**: Requires idempotent consumers and an explicit ordering strategy (MessageGroupId / partition key = orderId) regardless of broker.
- **Neutral**: Decision is anchored to AWS; if the org later goes multi-cloud or hits Kafka-scale, supersede this ADR (we are not blocked from doing so — see below).

## Pros and Cons of the Options

### Apache Kafka (self-managed or MSK / Confluent Cloud)

- **Good**: Best-in-class for high throughput and the gold standard for **log-based replay** — consumers seek to any offset within retention; one mechanism does fan-out *and* replay.
- **Good**: Strong ordering within a partition; rich ecosystem (Kafka Streams, Connect, Schema Registry).
- **Good**: Managed options (MSK, Confluent Cloud) remove some ops; Confluent Cloud in particular is fully serverless.
- **Bad**: **Self-managed Kafka is a poor fit given zero team ops experience** — partition strategy, rebalancing, broker/storage sizing, and upgrades are real, ongoing work and a common source of outages.
- **Bad**: Even MSK leaves meaningful operational responsibility (sizing, scaling, monitoring, version upgrades); Confluent Cloud reduces this but adds cost and a third-party vendor.
- **Bad**: Heaviest learning curve of the three; highest time-to-value for this team.

### RabbitMQ (with streams plugin)

- **Good**: Mature, flexible routing (exchanges/bindings), straightforward mental model for classic work queues; lower ops complexity than Kafka.
- **Good**: The **streams** feature (RabbitMQ 3.9+) adds a replayable, append-only log, partially closing the replay gap; Amazon MQ for RabbitMQ offers a managed option.
- **Bad**: Classic queues are **consume-and-acknowledge**, not a replayable log — replay is a bolt-on (streams) rather than the core model, and less battle-tested at scale than Kafka's log or Kinesis.
- **Bad**: ~10k events/sec is achievable but pushes classic-queue tuning (flow control, memory/disk alarms, mirrored/quorum queues); throughput is sensitive to message size and fan-out width.
- **Bad**: Still a broker (or cluster) to operate; managed RabbitMQ narrows but doesn't eliminate this. Replay maturity and operational fit are both weaker than the chosen option for *this* team.

### AWS SNS/SQS (chosen)

- **Good**: Lowest operational burden — best match for a team with no Kafka experience; managed, elastic, pay-per-use; native DLQs, IAM, CloudWatch.
- **Good**: Clean fan-out (one SNS topic → many SQS queues); FIFO variants give ordering + exactly-once-ish processing per message group.
- **Bad**: **No native log replay** in SNS/SQS — must add Kinesis Data Streams or EventBridge Archive & Replay for the replay requirement, which is the main architectural cost of this choice.
- **Bad**: AWS lock-in and per-message cost scaling; FIFO throughput quotas need verification against 10k/sec.

## More Information

- **Replay design note**: Treat replay as a first-class requirement, not an afterthought. Pick one of: (a) Kinesis Data Streams as the replayable system-of-record with SNS/SQS for live fan-out, or (b) EventBridge Archive & Replay if your routing is event-bus-shaped and replay is operational/coarse. Document the chosen retention window (e.g., 7–30 days, or up to 365 with Kinesis extended retention) — replay can only go back as far as retention.
- **Idempotency & ordering**: Require idempotent consumers and use `orderId` as the FIFO MessageGroupId / Kinesis partition key so all events for one order are ordered and land on one shard/group.
- **Reversibility / exit**: Hide the broker behind a thin internal publish/subscribe interface so a future migration to Kafka (if we hit Kafka-scale or go multi-cloud) is a single adapter swap, not a rewrite. If we make that move, **supersede this ADR** with a new one and set this one to `superseded by ADR-XXXX`.
- **Before accepting**: (1) confirm SNS/SQS **FIFO throughput quotas** cover 10k/sec at your message-group cardinality; (2) run a short **load + cost spike** at ~10k events/sec with realistic payload sizes to validate the chosen replay layer's shard/cost sizing; attach results here.
- **Link this from arc42 §9 (Architecture Decisions)** once accepted, and add it to `docs/decisions/README.md` (the ADR index).
- Related ADRs: <none yet>

---

**Next steps in this skill if useful:**
- I can draw the **C4 Container diagram** showing producers → SNS/SQS + Kinesis → consumers (Mode 1), so the replay path is visible rather than described.
- I can generate the matching **arc42 §9** stub and the `docs/decisions/README.md` index entry to keep one source of truth.
