# Decision: Messaging System for Order Events

## Summary / Recommendation

**Recommendation: Adopt AWS SNS/SQS for order events now, with a deliberate path to migrate to a managed Kafka (Amazon MSK or Confluent Cloud) if and when durable, long-window event replay becomes a hard, recurring requirement.**

If event replay is a firm, non-negotiable Day-1 requirement (not just "nice to have"), then go straight to **managed Kafka (Amazon MSK Serverless or Confluent Cloud)** instead — but do *not* run Kafka yourself given the team has zero Kafka ops experience.

RabbitMQ is not recommended for this use case.

---

## Context

We need a messaging backbone for **order events** (e.g. `OrderPlaced`, `OrderPaid`, `OrderShipped`, `OrderCancelled`) consumed by multiple downstream services (fulfillment, billing, analytics, notifications).

Key constraints and requirements driving this decision:

| Driver | Detail | Weight |
|---|---|---|
| Throughput | ~10,000 events/sec sustained | High |
| Event replay | Re-process historical events (new consumers, bug recovery, backfills) | High |
| Team experience | **No Kafka operational experience** | High |
| Operational burden | Small team; prefer managed over self-hosted | High |
| Multiple consumers | Several independent services need the same events | Medium |
| Ordering | Per-order ordering likely matters (state transitions) | Medium |
| Cost | Reasonable and predictable | Medium |

---

## Options Considered

### Option A — Apache Kafka

**What it is:** A distributed, partitioned, replicated commit log. Events are retained on disk for a configurable window (or indefinitely / compacted), and consumers track their own offset.

**Pros**
- **Best-in-class for event replay.** This is Kafka's defining feature. Consumers can reset offsets and re-read from any point in the retention window. New consumer groups can read the full history independently.
- Comfortably handles 10k events/sec — Kafka routinely runs at hundreds of thousands to millions of events/sec.
- Native fan-out: many consumer groups read the same topic without duplicating storage.
- Strong per-partition ordering (partition by `orderId` → all events for an order stay ordered).
- Rich ecosystem (Kafka Connect, Kafka Streams, ksqlDB, schema registry).

**Cons**
- **Operational complexity is the headline risk.** Self-managed Kafka requires expertise in brokers, partitions, replication, ISR, rebalancing, retention tuning, and (historically) ZooKeeper/KRaft. The team **has none of this experience.**
- Self-hosting Kafka with a green team is a recipe for outages and on-call pain.
- Higher baseline cost and conceptual overhead than a queue.

**Mitigation:** Use a **managed** offering rather than self-hosting:
- **Amazon MSK / MSK Serverless** — AWS-managed brokers; MSK Serverless removes capacity planning.
- **Confluent Cloud** — fully managed, includes Schema Registry, connectors, RBAC; lowest ops burden of the Kafka options.

Managed Kafka neutralizes most of the "no ops experience" objection, leaving only the (still real) conceptual learning curve.

---

### Option B — RabbitMQ

**What it is:** A traditional message broker (AMQP) built around queues, exchanges, and routing. A message is typically delivered and then **removed** from the queue once acknowledged.

**Pros**
- Mature, well-understood, flexible routing (topic/direct/fanout exchanges).
- Lower conceptual overhead than Kafka for classic task-queue patterns.
- Good for command/RPC-style and work-distribution workloads.

**Cons**
- **Not designed for event replay.** RabbitMQ is a queue, not a log — once a message is consumed and acked, it's gone. Replay requires bolt-on patterns (re-publishing, stream plugins, external store) that fight the tool's grain.
- RabbitMQ **Streams** (3.9+) do add log-like retention and replay, but this is newer, less proven at scale for this pattern, and you'd still be self-operating it (or paying for CloudAMQP) — losing the simplicity argument.
- 10k events/sec is achievable but pushes RabbitMQ harder than Kafka, especially with persistence and mirrored/quorum queues; throughput degrades as queues grow deep.
- Still requires cluster operations (mirroring, quorum queues, partition handling) — not zero-ops.

**Verdict:** RabbitMQ is the wrong shape for an event-replay requirement. We'd be choosing a queue and then re-implementing a log on top of it.

---

### Option C — AWS SNS + SQS

**What it is:** SNS (pub/sub topic) fans out messages to multiple SQS queues; each consuming service owns its own SQS queue. Fully managed, serverless, pay-per-use.

**Pros**
- **Zero operational burden** — no brokers, no clusters, no patching, no capacity planning. This directly addresses "no Kafka ops experience."
- **Scales to 10k events/sec easily.** Standard SQS has effectively unlimited throughput; SNS handles the fan-out. (FIFO has limits — see below.)
- Native multi-consumer fan-out via the SNS→multiple-SQS pattern; each consumer gets its own independently-managed queue and dead-letter queue.
- Deep AWS integration (Lambda, EventBridge, IAM, CloudWatch) — fast to build and operate.
- Pay-per-message pricing; cheap to start, predictable.
- Built-in DLQs, retries, visibility timeouts.

**Cons**
- **Weak/limited event replay.** SQS is a queue — once consumed and deleted, messages are gone. Max retention is **14 days**, and even within that window there's no offset-based "re-read from timestamp T" like Kafka. You cannot easily spin up a brand-new consumer and have it read all history.
- **Throughput vs. ordering trade-off:**
  - **Standard** SNS/SQS: at-least-once delivery, **best-effort ordering** (not guaranteed), virtually unlimited throughput.
  - **FIFO** SNS/SQS: exactly-once, strict ordering per message-group, but throughput is capped (≈3,000 msg/sec per topic without batching, up to ~30,000/sec with batching and high-throughput mode). At 10k/sec FIFO, you must partition carefully by `orderId` as the message-group key and rely on batching/high-throughput mode.
- No native log/streaming semantics (no consumer offsets, no compaction).

**Replay mitigation:** Pair SNS/SQS with an **event store / archive** — e.g. SNS → Kinesis Data Firehose → S3, or fan out to an audit queue that persists to S3/DynamoDB. Replay = re-publish from the archive. This covers *backfill/recovery* replay but is clunkier than Kafka's native offset reset, and not real-time-seekable.

---

## Decision Matrix

Scoring 1 (poor) to 5 (excellent), weighted by the drivers above.

| Criterion | Weight | Self-managed Kafka | Managed Kafka (MSK/Confluent) | RabbitMQ | AWS SNS/SQS |
|---|---|---|---|---|---|
| Event replay | High (×3) | 5 | 5 | 2 | 2 |
| 10k events/sec | High (×3) | 5 | 5 | 3 | 5 (Standard) / 3 (FIFO) |
| Low ops burden (no Kafka exp.) | High (×3) | 1 | 4 | 3 | 5 |
| Multi-consumer fan-out | Med (×2) | 5 | 5 | 4 | 5 |
| Ordering guarantees | Med (×2) | 5 | 5 | 4 | 3 (Std) / 5 (FIFO) |
| Cost predictability | Med (×2) | 3 | 3 | 4 | 4 |
| Time-to-first-value | Med (×2) | 2 | 3 | 4 | 5 |
| **Weighted total** | | **57** | **70** | **52** | **64** |

> Managed Kafka scores highest because it satisfies the two "High" weight conflicting requirements simultaneously: native replay *and* low ops. SNS/SQS comes a close second by maximizing operational simplicity but loses points on replay. Self-managed Kafka is penalized hard by the team's lack of ops experience.

---

## Recommendation in Detail

The two genuinely viable paths are **AWS SNS/SQS** and **managed Kafka**. The choice hinges on **how hard the event-replay requirement really is.**

**Choose AWS SNS/SQS if:**
- "Event replay" mostly means *recovery and backfill* (re-process the last few days after a bug, or seed a new consumer from an archive), not arbitrary point-in-time seeking.
- You value shipping fast with near-zero ops above all.
- You're committed to AWS.
- → Implement the **SNS → per-service SQS** fan-out, add **DLQs** on every queue, and add an **S3 event archive** (via Firehose) for replay-by-re-publish.

**Choose managed Kafka (MSK Serverless or Confluent Cloud) if:**
- Event replay is a **first-class, recurring product requirement** — new analytics consumers reading full history, frequent reprocessing, long retention, log compaction.
- You're willing to invest in a moderate learning curve in exchange for native replay semantics.
- → **Do not self-host.** Use the managed service so the team never touches broker operations.

**Given the stated emphasis on event replay,** if replay is truly a firm requirement, lean toward **managed Kafka (MSK Serverless / Confluent Cloud)**. If replay is more of a safety-net/backfill concern, **SNS/SQS + S3 archive** is the lower-risk, faster, cheaper choice that the team can operate today.

**RabbitMQ is rejected** — it's a queue being asked to behave like a log, which fights its design.

---

## Consequences

**If we go SNS/SQS:**
- ✅ Live quickly, minimal ops, easy AWS integration, cheap to start.
- ⚠️ Replay is limited to the archive-and-republish pattern; build the S3 archive from Day 1 or you'll regret it.
- ⚠️ Decide Standard vs. FIFO early. At 10k/sec with per-order ordering, plan FIFO with `orderId` message-group key + batching + high-throughput mode, and load-test it.
- ⚠️ Vendor lock-in to AWS.

**If we go managed Kafka:**
- ✅ Native replay, generous retention, strong ordering, headroom far beyond 10k/sec.
- ⚠️ Team must learn Kafka concepts (topics, partitions, consumer groups, offsets) even if not operating brokers.
- ⚠️ Higher and somewhat less predictable cost than starting on SQS.
- ✅ Adopt a **schema registry** (Confluent / Glue) and define partitioning by `orderId` up front.

---

## Open Questions / Next Steps

1. **Pin down "event replay."** Get the team to define it concretely: *who* replays, *how far back*, *how often*, *real-time seek or batch backfill?* This single answer decides SNS/SQS vs. Kafka.
2. **Confirm ordering requirements** — strict per-order ordering, or is best-effort acceptable?
3. **Confirm cloud commitment** — all-in on AWS, or multi-cloud / portability concerns? (Confluent Cloud is more portable; MSK and SNS/SQS lock to AWS.)
4. **Run a 2-week spike / load test** on the leading candidate at ~10k events/sec with realistic payloads and consumer counts before committing.
5. **Define the event schema and versioning strategy** regardless of broker choice.

---

## TL;DR

- **No Kafka ops experience → never self-host Kafka.** Use managed if you pick Kafka.
- **RabbitMQ:** rejected — wrong tool for replay.
- **Replay is firm & recurring → managed Kafka (MSK Serverless / Confluent Cloud).**
- **Replay is mostly backfill/recovery → AWS SNS/SQS + S3 archive** (fastest, simplest, cheapest to operate).
- 10k events/sec is comfortable for managed Kafka and for SNS/SQS Standard; watch FIFO throughput limits if strict ordering is needed.
