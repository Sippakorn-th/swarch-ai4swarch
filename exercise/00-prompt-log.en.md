**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Sippakorn Thunyahan / 6631355721

**Agent(s) used:** Codex in the desktop app, with fresh subagents explicitly authorized by the student.

**Model(s):** Exact model identifiers were not exposed in the run outputs; no version is assumed.

**Plan / tier:** Not recorded.

**Provenance:** Runs 1-A, 1-B, 2-a, 2-b and 3 used separate agents with no parent conversation history. The coordinating chat knew the broader context, but that history was not sent to those agents. The final choice of a coordinator and its guardrails came from the student; Codex edited the wording at the student's explicit request. The student subsequently requested completion of the worksheets from actual files and the supplied group messages. The notes, comparison and reflection are labelled AI-assisted. Useful scores are artifact-based AI-assisted assessments, not independently supplied student ratings.

## Requests and outcomes

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A, 2026-10-26 | Codex; model not exposed | Follow prompts/1-recover-architecture.md; run A; evidence demo.proto. | Manifest A below. | Yes for the isolated analysis. | 3 | Correctly limits runtime conclusions; nine interfaces do not prove nine deployed containers. |
| 2 | 1-B, 2026-10-26 | Codex; model not exposed | Same task; run B; evidence demo.proto plus kubernetes-manifests/. | Manifest B below. | Yes for the isolated analysis. | 4 | Recovers configured topology and flags optional resources and unproven implementation languages. |
| 3 | 1-B follow-up | Coordinating Codex → run-B agent | Preserve configuration/execution distinction; keep optional instrumentation as gaps; match tables and diagram; do not invent languages/browser code. | No extra source scope was authorized or reported. | Yes; additional coaching is disclosed. | 4 | This was an extra request beyond the worksheet's baseline; results are not an untouched single-request comparison. |
| 4 | 2-a, 2026-10-26 | Codex; model not exposed | Follow prompts/2-write-adr.md; run a; context none; English; R-12 only. | AGENTS.md; prompts/2-write-adr.md; docs/adr/template.md. | Yes; no scenario/source/example. | 3 | Generic choreography proposal, with missing numerical constraints left as questions. |
| 5 | 2-b, 2026-10-26 | Codex; model not exposed | Same requirement; run b; scenario, example ADR and checkout main.go. | AGENTS.md; prompts/2-write-adr.md; docs/adr/template.md; docs/scenario.md; docs/adr/0001-use-redis-for-cart-storage.md; system/online-boutique/src/checkoutservice/main.go. | Yes for the isolated analysis. | 4 | RabbitMQ proposal uses actual constraints and identifies the missing order/outbox boundary. |
| 6 | Guided final decision | Coordinating Codex | Explain the full problem, ask questions, then edit from my answers. Student chose B, fast checkout/simple maintenance, concurrent pools, durable work, order_id idempotency and business sign-off. | Rules, template, scenario, 2b draft and checkout main.go; task-3 prompt for review setup. | Broader coaching context, not another controlled run. | 4 | Human decision supplied in chat; AI-assisted final wording is disclosed in the ADR and worksheet 02. |
| 7 | 3, 2026-10-26 | Fresh Codex reviewer; model not exposed | Follow prompts/3-review-adr.md; adr: staged final coordinator ADR; language English. | AGENTS.md; prompts/3-review-adr.md; docs/scenario.md; staged docs/adr/0002-post-order-processing.md. | Yes; no source code, prior drafts or stakeholder messages. | 4 | Staged ADR was installed byte-for-byte. Verdict: accept with changes. Review remains unchanged after the board. |
| 8 | Submission completion, 2026-10-27 | Coordinating Codex | Finish GitHub using chronological stakeholder screenshots and message-12.txt; student supplied ID 6631355721. | Current worksheets/README/rules, original outputs/final ADR, verification files below, prior provenance notes and the provided group record/screenshots. | Post-board compilation; deliberately outside the original experiment scopes. | 4 | Shared board text copied exactly. Five static dependencies checked with AI assistance. No hidden role-card folder was opened. |

## Exact task inputs

### Architecture A

~~~text
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
~~~

### Architecture B

~~~text
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
~~~

The additional run-B follow-up is recorded in row 3.

### R-12 supplied to both ADR runs

After an order is placed, Online Boutique must: (1) send the order confirmation email through MailFast, a new external email provider replacing the mock emailservice logic; (2) award loyalty points through a new loyaltyservice, to be built; (3) record an order-placed analytics event for marketing. Marketing expects more post-order actions next year: SMS, fraud scoring and a recommendations update. Only this requirement, not the scenario's numbers, was supplied to run a.

### ADR a

~~~text
Follow prompts/2-write-adr.md
run: a
context: none
language: English
requirement: [the R-12 text above]
~~~

### ADR b

~~~text
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
language: English
requirement: [the same R-12 text]
~~~

### Final AI review

~~~text
Follow prompts/3-review-adr.md
adr: /private/tmp/swarch-ai4swarch-20261026-final/docs/adr/0002-post-order-processing.md
language: English
~~~

That staged file's SHA-256 was 813dbc3f0dc9c0ca6ad49db151ce2201f157d009e4d698695cae4e1cdd3c48aa. The installed pre-board ADR matched it. The later board addendum is outside this review's scope.

## Files read: architecture agents

### Manifest A

- `AGENTS.md` — repository rules.
- `prompts/1-recover-architecture.md` — task instructions.
- `system/online-boutique/protos/demo.proto` — the only architecture evidence.

No other repository files, memory, internet sources, role cards, or upstream repository were read.

### Manifest B

- `AGENTS.md`
- `prompts/1-recover-architecture.md`
- `system/online-boutique/protos/demo.proto`
- `system/online-boutique/kubernetes-manifests/README.md`
- `system/online-boutique/kubernetes-manifests/adservice.yaml`
- `system/online-boutique/kubernetes-manifests/cartservice.yaml`
- `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`
- `system/online-boutique/kubernetes-manifests/currencyservice.yaml`
- `system/online-boutique/kubernetes-manifests/emailservice.yaml`
- `system/online-boutique/kubernetes-manifests/frontend.yaml`
- `system/online-boutique/kubernetes-manifests/kustomization.yaml`
- `system/online-boutique/kubernetes-manifests/loadgenerator.yaml`
- `system/online-boutique/kubernetes-manifests/paymentservice.yaml`
- `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml`
- `system/online-boutique/kubernetes-manifests/recommendationservice.yaml`
- `system/online-boutique/kubernetes-manifests/shippingservice.yaml`

## Coordination, setup and additional help

Before the experimental runs, the student asked for the assignment explanation, individual versus group fork/submission scope, GitHub Desktop fork/clone steps, confirmation of the fork URL, role advice, Architect duties and the extent of permitted assistance. The student authorized proceeding and explicitly authorized fresh agents. These were setup/coaching requests, not additional architecture-recovery evidence.

After the drafts, the student requested a step-by-step sharing/push guide, an ADR editing explanation, a simpler explanation, and a question-and-answer editing workflow. They then supplied the coordinator decision and guardrails. Codex applied those choices and arranged the isolated review. Before the latest completion request, the student had committed/pushed that ADR and the five original outputs; the remote/local starting commit was ebdb677aa28891a43d37ce6d4408d86adc1bb43b.

The parent read README.en.md; AGENTS.md; exercise/00–05 English worksheets; prompts/1-recover-architecture.md, prompts/2-write-adr.md and prompts/3-review-adr.md; docs/scenario.md; docs/adr/template.md; docs/adr/0001-use-redis-for-cart-storage.md; docs/adr/0002-post-order-processing.md; the five generated outputs; and system/online-boutique/src/checkoutservice/main.go. During final verification it additionally read these manifests: frontend.yaml, checkoutservice.yaml, paymentservice.yaml, shippingservice.yaml, emailservice.yaml, cartservice.yaml and kustomization.yaml under system/online-boutique/kubernetes-manifests/. It also read the two earlier temporary preparation/provenance notes, the user-provided message-12.txt and stakeholder screenshots. This file list describes coordination, not evidence supplied to each isolated run.

**Deviations and limits:** the parent used pwd/rg to locate the clone during setup, and approved one-time file-copy scripts to install generated documents outside the chat's writable workspace. It uses Git status/log/diff and commit/push operations for submission logistics. These terminal operations are disclosed; the five isolated analysis agents did not run terminal commands. The student explicitly asked for AI editing of the final ADR and completion of worksheets, despite the exercise's default human-editing instructions. No role-card files or original Online Boutique repository were read. Hidden business details entered only through post-board messages; no original output or AI review was regenerated with that information. Static checks are not runtime tests, and personal manual verification is not asserted.

## Shared meeting evidence and submission scope

The supplied group record names all five members, records accept with changes, five required-change categories and fifteen concerns. It is copied unchanged to 04-review-board.en.md. Screenshot times show stakeholder posts at 14:00/14:16 (security), 14:22 (SRE), 14:48 (PO) and 17:18 (compiled record); no exact meeting date or individual vote tally is supplied, so neither is invented. Actual legal approval, production experience, incident history and marketing promises are treated as fictional stakeholder-reported facts, not independently verified real-world facts. The record's three-week delivery window is the PO's scenario constraint, not a newly calculated calendar schedule.

All six worksheets are completed in English only. The Thai templates and optional bonus are left untouched. No implementation, load test, business/legal approval, or LMS submission is claimed.
