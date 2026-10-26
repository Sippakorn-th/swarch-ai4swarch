Agent: Codex; model: not exposed.

# ADR-0002: Publish durable order events for independent post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Unspecified; the student is responsible for the final decision.

## Context and problem

R-12 requires MailFast confirmation emails, loyalty points through a new `loyaltyservice`, and an order-placed analytics record. Marketing expects SMS, fraud scoring and recommendations updates next year (`docs/scenario.md:7–14`).

Today, `checkoutservice.PlaceOrder` charges the card, ships the order, attempts to empty the cart, builds the result and waits for confirmation email before returning. Email failure only logs a warning; payment and shipping failures return errors (`system/online-boutique/src/checkoutservice/main.go:230–280`). The email helper calls `EmailService.SendOrderConfirmation` through gRPC (same file, `sendOrderConfirmation`, lines 379–383). The scenario identifies the existing email logic as a mock; its implementation was not provided.

Scenario constraints (`docs/scenario.md:24–30`):

| Constraint | Fact and implication |
|---|---|
| Checkout performance | p95 ≤ 500 ms; today approximately 300 ms. MailFast's p95 1.5 s and p99 8 s make waiting for it unsuitable. Quantiles cannot simply be added; validate the full request path. |
| Peak traffic and email capacity | 200 orders/s versus 50 emails/s. **[Inferred]** Assuming one email per order, even without failures, sustained peak traffic adds at least 150 queued emails/s. |
| MailFast reliability | Approximately 0.5% of calls fail; retries and failure visibility are needed. |
| Loyalty | Award within 1 minute, exactly once with no double points. |
| Availability | Orders must never fail because email, loyalty or analytics is down. |
| Delivery capability | Four developers, strong Go/gRPC skills; two have used RabbitMQ. Kubernetes already runs the system; open-source, self-hosted components are preferred. |

A durable order store, transaction boundary, broker and loyalty ledger are **Not evidenced in the provided files**. The examined `PlaceOrder` creates an order UUID and returns an in-memory result (`main.go:233–236,265–279`); this does not establish durable order recording. ADR-0001 treats cart loss as tolerable before payment and selects Redis for carts (`docs/adr/0001-use-redis-for-cart-storage.md:11,38–45`); that durability trade-off must not be assumed suitable for post-payment events.

## Decision drivers

1. **Availability and reliability:** consumer outages must not fail checkout or silently discard required work (`docs/scenario.md:9–12,28`).
2. **Performance:** keep `PlaceOrder` p95 ≤ 500 ms at 200 orders/s (same file, lines 24–25).
3. **Correctness and timeliness:** prevent duplicate points and complete loyalty within 1 minute (same file, line 27).
4. **Modifiability:** add future actions without extending checkout's downstream call sequence (same file, line 14).
5. **Operability and cost:** fit the four-person team's experience and self-hosted Kubernetes preference (same file, lines 29–30).

## Considered options

1. Call MailFast, loyalty and analytics synchronously from checkout, retaining warning-only handling for post-order failures.
2. Enqueue a durable workflow job; one post-order orchestrator coordinates all actions and tracks each action's completion.
3. Publish a durable `OrderPlaced` event through self-hosted RabbitMQ, with an independent durable queue and consumer for each action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous calls | Familiar request flow; immediate outcomes; few new components. | Slow email remains on checkout's critical path; ignoring failures loses required work without another durable mechanism; each new action changes checkout. |
| Durable workflow | Fast checkout after durable handoff; central visibility and explicit per-action recovery; familiar Go orchestration. | Requires durable workflow state and recovery; adding actions changes the coordinator; careless sequential processing lets email backlog delay loyalty. |
| RabbitMQ event fan-out | Independent action capacity and failure handling; future subscribers need no new checkout call; aligns with stated RabbitMQ experience. | Adds broker and durable publication operations; duplicate delivery needs application safeguards; cross-consumer tracking and schema evolution take work. |

## Decision

Choose **option 3**, subject to validation below. All components and mechanisms in this decision are **proposed**, not claims about the current deployment.

- Define a versioned `OrderPlaced` contract with stable event ID, order ID, customer reference, completion time and the minimum data each action needs. Do not publish credit-card details. Determine the email data boundary before implementation.
- Establish durable order completion and an outbox entry in one local transaction. A relay publishes the outbox event to a durable RabbitMQ exchange and records successful publication after broker confirmation. Retry publication safely. Broker unavailability leaves pending work in the outbox; checkout does not wait for consumers.
- **Implementation prerequisite:** the current code provides no evidenced order transaction into which an outbox can simply be inserted. Define durable order/workflow state and recovery around the existing remote payment/shipping calls. A crash after shipping but before recording completion must be recoverable without losing the event or repeating financial/shipping effects. An outbox alone does not make those remote calls atomic. This boundary is unresolved, so reliable delivery is not yet demonstrated.
- Bind separate durable email, loyalty and analytics queues. Use persistent messages, consumer acknowledgements after successful durable processing, bounded retry with backoff, and a dead-letter path with alerts and controlled replay. Configure broker replication and durable storage; queue durability alone is insufficient for host loss.
- Replace the mock email logic with a MailFast adapter consumed asynchronously. Apply a shared limit across all email workers of at most 50 calls/s, including retries. Timeout ambiguous calls and use provider idempotency if supported; duplicate-email protection is otherwise unresolved. Loyalty and analytics continue independently of email backlog.
- Build `loyaltyservice` with a durable ledger. In one local transaction, enforce uniqueness for the order's award and apply the points balance change; acknowledge afterward. Redelivery after a crash becomes a no-op. This targets **exactly-once business effects**, not exactly-once broker delivery. Deduplicate analytics by stable event/order ID when recording the marketing event.

This choice prioritizes independent extension and isolation over central workflow coordination. RabbitMQ selection is supported by team experience and hosting constraints (`docs/scenario.md:29–30`); the queue, persistence and service designs above remain proposals.

## Consequences

- **Positive:** downstream service outages and MailFast latency leave the checkout request path (quality attributes: availability, performance).
- **Positive:** new actions subscribe independently; separate queues protect loyalty from email congestion (modifiability, scalability).
- **Positive:** durable publication plus transactional loyalty deduplication supports recovery without double points (reliability, correctness), conditional on the unresolved completion boundary.
- **Negative:** broker, outbox storage, ledger, retry handling and monitoring increase operational work for four developers (cost, maintainability).
- **Negative:** post-order results become eventually consistent. At peak traffic, email backlog necessarily grows; more workers cannot overcome MailFast's stated limit (timeliness, capacity).
- **Risks and mitigations:** the 1-minute loyalty deadline cannot be guaranteed during an unbounded loyalty outage. Alert on oldest pending loyalty work well before the deadline; test recovery capacity and obtain an explicit outage policy (availability versus timeliness).
- **Risks and mitigations:** crashes around payment, shipping, completion recording and publication can lose or repeat work without durable recovery. Review this sequence and inject failures before implementation acceptance (reliability). New durable storage also adds an availability dependency; its failure policy must be decided.
- **Risks and mitigations:** poison messages, replay and prolonged outages can exhaust storage or repeat effects. Quarantine failures, monitor backlog age/size, preserve deduplication records for the agreed replay period, and rehearse restoration (operability, correctness).

## Validation

- Load-test end-to-end `PlaceOrder` at 200 orders/s; require p95 ≤ 500 ms with simulated MailFast p95 1.5 s, p99 8 s, 0.5% failures and 50 calls/s maximum. Record queue growth and recovery, not just checkout latency.
- Disable each post-order consumer/provider and the broker separately. Verify successful orders remain accepted under the agreed storage policy and pending work survives restarts, then drains after recovery.
- Inject crashes before/after completion recording, publication confirmation, loyalty ledger commit and acknowledgement. Verify every durably completed order is eventually published, analytics is recorded, and repeated delivery/replay awards points once.
- At peak load and agreed outage/recovery conditions, measure order-completion-to-loyalty-commit latency against 1 minute. Verify email backlog cannot delay loyalty. Review storage saturation and dead-letter recovery.
- Add one representative future subscriber without changing checkout's action calls; inspect event compatibility and access to customer data. These are proposed checks; none have been run for this ADR.

## Open questions

- What defines an order as placed, where will durable completion/outbox state live, and how will payment/shipping recovery and client retries be made safe? Their idempotency contracts are **Not evidenced in the provided files**.
- What happens when the completion/outbox store is unavailable or full? What durability, replication, retention and recovery targets are required?
- How long does the 200 orders/s peak last? What email delay is acceptable, and can MailFast capacity increase? Without these facts, backlog sizing and drain time cannot be fixed.
- What are MailFast's API/authentication, timeout, rate-limit and idempotency contracts? Provider acceptance versus actual delivery semantics are **Not evidenced in the provided files**.
- What are the loyalty calculation, customer identity, refund/reversal rules and outage exception policy for the 1-minute deadline? How much replay history must deduplication retain?
- What analytics sink, event fields, privacy/access rules, retention and delivery deadline does marketing require?
- Who owns the new services and broker operations, and what resources/budget are available? The student must resolve these questions and make the final decision; no approval is asserted.
