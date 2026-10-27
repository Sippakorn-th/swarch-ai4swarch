**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture

**Name / Student ID:** Sippakorn Thunyahan / 6631355721

**Assistance disclosure:** This worksheet was prepared with Codex assistance. The checks below were performed by Codex against the repository files; they are static evidence checks, not a claim that I manually inspected every file or ran the system. The original independent run outputs are unchanged.

## Results

- Run A: [outputs/1A-architecture.md](outputs/1A-architecture.md), using only demo.proto as architecture evidence.
- Run B: [outputs/1B-architecture.md](outputs/1B-architecture.md), adding the Kubernetes manifests.
- Run A inventories nine RPC interfaces but establishes no runtime containers or caller relationships. Run B recovers twelve supplied containers and seventeen configured relationships, with explicit inference/optional-component caveats.

## Verify Run B yourself (the important part)

The following five dependencies were checked against the actual files on 2026-10-27. A check confirms the stated configuration or source call; it does not prove successful production execution.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What was found |
|---|---|---|---|
| frontend → checkoutservice | frontend.yaml, CHECKOUT_SERVICE_ADDR; checkoutservice.yaml, grpc Service | ✅ Static configuration, Codex check | system/online-boutique/kubernetes-manifests/frontend.yaml:79–80 sets checkoutservice:5050. checkoutservice.yaml:88–90 exposes grpc on 5050. This confirms the configured target, not an executed frontend call. |
| checkoutservice → paymentservice | checkoutservice.yaml, PAYMENT_SERVICE_ADDR; paymentservice.yaml, grpc Service | ✅ Configuration and source, Codex check | checkoutservice.yaml:61–62 sets paymentservice:50051; paymentservice.yaml:79–81 exposes that port. system/online-boutique/src/checkoutservice/main.go:369–375, chargeCard, constructs PaymentServiceClient and calls Charge. |
| checkoutservice → shippingservice | checkoutservice.yaml, SHIPPING_SERVICE_ADDR; shippingservice.yaml, grpc Service | ✅ Configuration and source, Codex check | checkoutservice.yaml:59–60 sets shippingservice:50051; shippingservice.yaml:79–81 exposes that port. main.go:313–321 calls GetQuote and main.go:386 onward calls ShipOrder through ShippingServiceClient. |
| checkoutservice → emailservice | checkoutservice.yaml, EMAIL_SERVICE_ADDR; emailservice.yaml, grpc Service | ✅ Configuration and source, Codex check | checkoutservice.yaml:63–64 sets emailservice:5000. emailservice.yaml:81–83 maps Service port 5000 to container port 8080, so those different ports are intentional. main.go:379–383 calls SendOrderConfirmation. |
| checkoutservice → cartservice | checkoutservice.yaml, CART_SERVICE_ADDR; cartservice.yaml, grpc Service | ✅ Configuration and source, Codex check | checkoutservice.yaml:67–68 sets cartservice:7070; cartservice.yaml:80–82 exposes that port. main.go:324–335 calls GetCart and EmptyCart through CartServiceClient. |

The manifest paths in the table are under system/online-boutique/kubernetes-manifests/. The source checks were done after runs A/B for verification; the original isolated agents did not receive this extra source evidence.

**Wrong or missing in Run B:** No contradiction was found in these five checked dependencies. The graph remains a configuration reconstruction: it does not establish actual traffic, call order, successful requests or implementation languages. The loadgenerator has a supplied manifest, but kustomization.yaml:24 comments it out; it must not be assumed active in the default resource list. Redis uses emptyDir in cartservice.yaml:141, which does not establish durable storage across Pod replacement. These are limitations the output already identifies, not newly discovered running-system failures.

## Reflect (2–3 sentences each)

1. **Compare A and B:** A protobuf file tells us which RPC interfaces and message types exist, but not which processes are deployed or which process calls each interface. Adding manifests reveals configured service addresses, workload boundaries, ports and Redis. I would treat a diagram derived only from interfaces as a hypothesis until deployment or source evidence supports its connections.

2. **Most misleading mistake:** Presenting every configured or supplied component as definitely running would give a new team member a false picture of the system. The loadgenerator is a concrete example because its manifest exists while the default resource list comments it out. A configuration graph should label that distinction rather than look like a verified runtime trace.
