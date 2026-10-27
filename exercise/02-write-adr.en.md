**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement

**Name / Student ID:** Sippakorn Thunyahan / 6631355721

**Assistance disclosure:** The comparison was drafted with Codex assistance from the original outputs. I supplied the final architectural choice and guardrails in conversation; Codex edited the wording at my explicit request. This is AI-assisted editing, not unaided manual authorship.

## Results

- [Run a — requirement only](outputs/2a-adr.md).
- [Run b — example and context](outputs/2b-adr.md).
- [Final ADR](../docs/adr/0002-post-order-processing.md).

Both runs used fresh agents with English output. Run a read the task/rules and ADR template, but no scenario, source or example. Run b additionally read docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md and system/online-boutique/src/checkoutservice/main.go.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It treats workload, latency and provider limits as open questions because only R-12 was supplied. | Yes: checkout p95 ≤ 500 ms, 200 orders/s, MailFast p95 1.5 s/p99 8 s, 0.5% failures, 50 emails/s, and loyalty within one minute. |
| Options considered (list them) | Synchronous orchestration; asynchronous event choreography; durable workflow orchestration. | Synchronous calls; one durable workflow coordinator; RabbitMQ fan-out with independent action queues. |
| Handles exactly-once loyalty and MailFast's rate limit? How? | Proposes atomic award/deduplication and asks about provider rate limits; cannot select the supplied numerical limit. | Distinguishes exactly-once business effects from redelivery, proposes a unique award record in the points transaction, caps MailFast requests including retries, and identifies at least 150 emails/s of backlog growth during sustained peak under one email/order. |
| Anything invented (a number, tool or fact not in the scenario)? | No fabricated scenario numbers identified. Outbox, durable messaging and deduplication are labelled proposals with feasibility questions. | RabbitMQ, outbox, ledger and retry mechanisms are design proposals, not established infrastructure. The durable order transaction remains an explicitly unevidenced prerequisite. The backlog calculation is a stated inference, not a supplied fact. |

The extra context makes run b more specific, especially about the checkout sequence and missing durability boundary. It does not prove the design meets the targets or that the proposed components already exist.

## Take ownership: your final ADR

I chose **B: one durable background coordinator**, prioritizing fast checkout and simple maintenance for the fictional four-developer Go team. I accepted weaker independent deployment/scaling than separate RabbitMQ consumers. My classroom group has five members; it is separate from the scenario's team size.

| # | What changed from the AI's draft | Why |
|---|---|---|
| 1 | Selected the durable coordinator instead of RabbitMQ fan-out. | My stated priorities were fast checkout and fewer separate action services to deploy, monitor and debug. |
| 2 | Required separate bounded goroutine pools and ready-work selection for email, loyalty and analytics. | Slow/rate-limited email must not block loyalty or occupy all execution capacity. |
| 3 | Required durable order/action records before acknowledging successful checkout, with crash recovery around payment/shipping. | Background execution must not lose work after a crash; a final database write alone cannot make remote effects atomic. |
| 4 | Required order_id as the loyalty idempotency key, enforced atomically with the points mutation. | Passing a key alone is insufficient; retries must never award points twice. |
| 5 | Required a shared MailFast request cap of 50/s including retries and overlapping coordinator instances. | Local worker concurrency must not bypass the provider limit. |
| 6 | Left delayed-email acceptance and the loyalty-outage exception subject to explicit business sign-off. | Keeping checkout available cannot silently override the supplied one-minute loyalty requirement. |
| 7 | Kept the 500 ms checkout goal as an acceptance test rather than a claimed result. | Asynchronous execution still adds durable writes and possible resource contention. |

**After the board:** the shared decision was accept with changes. The final ADR includes a dated addendum recording the five required change categories, while the original AI review remains the pre-board evidence. Legal approval, immediate-points agreement, operational readiness and successful testing are not asserted.
