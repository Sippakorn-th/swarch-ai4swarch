Agent: Codex; Model: not exposed.

# ADR-0002: Publish an order-placed event for independent post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Student (name unspecified)

## Context and problem

The student-provided requirement says that, after an order is placed, Online Boutique must send its confirmation through MailFast, replacing the mock emailservice logic; award points through a new loyaltyservice; and record an order-placed analytics event for marketing. Marketing expects additional actions next year: SMS, fraud scoring, and a recommendations update.

We must choose how to trigger and coordinate these actions as their number grows. Current service boundaries, protocols, storage, delivery guarantees, and deployment infrastructure are **Not evidenced in the provided files**: this zero-shot run uses the requirement alone. All mechanisms below are **proposed**, not descriptions of the existing system.

## Decision drivers

The following priority order is proposed for review; the requirement supplies no ranked quality goals or numeric targets.

- **Modifiability:** Add future actions without repeatedly changing the order-placement flow.
- **Reliability:** Avoid silently losing a required action and prevent duplicate business effects during retries.
- **Availability and performance:** Isolate order completion and other actions from a downstream outage, subject to agreement on acceptable completion delays.
- **Operability and cost:** Make failures visible while keeping additional infrastructure and operational work justified.

## Considered options

1. **A — Proposed synchronous orchestration:** The order-placement component calls the MailFast integration, loyaltyservice, and analytics destination and waits for their responses.
2. **B — Proposed asynchronous event choreography:** Reliably publish an order-placed event; independently deliver it to email, loyalty, and analytics consumers.
3. **C — Proposed asynchronous workflow orchestration:** Reliably start a durable workflow whose coordinator tracks and retries each post-order action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | Direct control flow; fewer new messaging components; immediate responses from each action. | Order response time and availability depend on downstream calls; partial success still needs recovery; every new action changes orchestration. |
| B | Independent consumers isolate failures and allow new subscribers; actions can progress separately. | Requires durable messaging, duplicate handling, and monitoring; completion is delayed; there is no single workflow owner tracking overall completion by default. |
| C | Explicit per-order progress and recovery; supports dependencies between actions. | Adds a durable coordinator and workflow state; new actions require workflow changes; coordinator operation and evolution add cost. |

## Decision

Choose **B, proposed asynchronous event choreography**, because expected growth in independent post-order actions favors modifiability and failure isolation. This proposal assumes that these actions may complete after the order response and have no required cross-action ordering. Stakeholders must confirm that assumption; a mandatory pre-completion fraud decision would require revisiting the flow.

Proposed design:

- The component that authoritatively records order placement produces a versioned order-placed event with a stable event ID, order ID, and correlation information. Its actual identity is unknown.
- Use a **proposed transactional outbox** to commit the order and event intent atomically, followed by a relay into durable messaging. This requires a compatible authoritative order store and transaction boundary, which are not evidenced. Confirm feasibility before implementation; do not substitute an unprotected record-then-publish sequence.
- Give each action its own durable subscription and acknowledgment state. Propose at-least-once delivery and acknowledge only after the action has completed or its outcome is durably recorded. No broker or protocol is selected here.
- Replace the mock email behavior with a proposed email consumer or adapter that sends confirmations through MailFast. Provider API, authentication, retry behavior, and idempotency support remain unknown.
- Build a proposed loyaltyservice consumer that durably records an event/order deduplication key atomically with awarding points. The points rule remains a business decision.
- Add a proposed analytics consumer that records the required marketing event in an agreed destination, using the event identity for deduplication where supported.
- Propose bounded retries with backoff, a failed-message queue, alerts, and controlled replay for failed actions. Duplicate suppression must cover business effects, not just message receipt. If MailFast accepts a request but its response is lost, reconcile its outcome or use provider-supported idempotency before resending; local deduplication alone cannot ensure exactly-once external sending.
- Propose a minimal event contract and access controls; put only approved data in events. Consumers obtain any additional authorized data through an agreed interface. Schema ownership and compatibility rules must be established before adding subscribers.

These mechanisms are design proposals, not verified capabilities. The decision remains Proposed pending resolution of the feasibility and business questions below.

## Consequences

- **Positive:** Future independent actions can subscribe without extending the producer's action list. (quality attribute: modifiability)
- **Positive:** A MailFast, loyalty, or analytics outage can leave its action pending while other actions continue. (quality attribute: availability)
- **Positive:** With the proposed reliable handoff, downstream execution need not extend the order response time. (quality attribute: performance)
- **Negative:** Customers and marketing may observe delayed completion, and the three actions can finish in different orders. (quality attribute: consistency)
- **Negative:** Durable messaging, outbox relay, monitoring, and replay require implementation and ongoing operation. (quality attributes: cost, operability)
- **Risks and mitigations:** Order/event dual-write failure can lose work; validate atomic outbox feasibility and test crash recovery. (quality attribute: reliability)
- **Risks and mitigations:** Redelivery or uncertain provider outcomes can duplicate emails, points, or analytics; use durable consumer idempotency and provider reconciliation where available, explicitly documenting remaining guarantees. (quality attributes: reliability, correctness)
- **Risks and mitigations:** Distributed progress can hide incomplete orders; correlate actions by order/event ID and expose pending, succeeded, and failed outcomes with alerts and replay controls. (quality attribute: observability)
- **Risks and mitigations:** Event payloads and retention can expose customer data; minimize fields and review subscriber access and retention. (quality attribute: security)

## Validation

All checks below are proposed acceptance activities; none has been executed in this run.

- Trace a placed order to a MailFast confirmation, one correct loyalty award, and one recorded marketing event using the agreed business definitions.
- Crash the producer between order persistence and publication, and crash the relay before and after acknowledgment; verify that committed orders retain recoverable event intent and aborted orders do not trigger actions.
- Stop each consumer and destination independently; verify other actions continue, pending work survives restart, and recovery completes the failed action.
- Redeliver events and interrupt consumers around side effects and acknowledgment; verify points are not awarded twice and validate email/analytics guarantees, including uncertain MailFast outcomes.
- Exercise retry exhaustion, alerting, and authorized replay; verify the affected order and action remain identifiable and replay does not duplicate completed effects.
- Add a trial subscriber to assess whether a future independent action can be introduced without modifying order-placement logic. Check event schema compatibility and access controls.
- Measure order response latency, event-to-action completion delay, backlog age, and failure rate under representative load. Agree workload and acceptance thresholds before testing; no numeric targets were provided.

## Open questions

- Which component and durable record define a successfully placed order? Can the order record and outbox intent share an atomic transaction? What persistence exists today?
- May the order response succeed while email, loyalty, or analytics remains pending? What completion deadlines, delivery guarantees, and recovery obligations apply to each action?
- Are these actions independent? Could future fraud scoring block order acceptance or require compensation after placement?
- What are the actual workload, latency and availability targets, retention requirements, and infrastructure budget? Who will operate messaging and handle failed actions?
- What does MailFast support for API access, authentication, idempotency, status reconciliation, rate limits, and retryable errors? What migration behavior prevents duplicate confirmations?
- How are points calculated, which customers are eligible, and what happens on cancellation or refund?
- What analytics destination, event schema, consent requirements, and duplicate-handling policy does marketing require?
- Which event fields are necessary and permitted, and how may consumers retrieve additional data? Who owns schema evolution, credentials, access, and retention?
- Which broker, protocols, deployment constraints, existing service contracts, and team capabilities should constrain implementation? These are **Not evidenced in the provided files**.
