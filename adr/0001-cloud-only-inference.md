# ADR 0001: Cloud-Only Inference Layer

## Context

We must moderate uploads within a 600 ms p95 window at low volumes (2–20 RPS). Because mistakes carry severe legal weight, we require heavy, maximum-accuracy vision models and an instantly synchronized blocklist cache.

## Decision

I chose a centralized, cloud-only inference architecture using serverless containers and a vector cache. This eliminates client-side complexity, prioritizing maximum classification accuracy and strict audit control over distributed edge compute.

## Alternatives rejected

* **Hybrid Edge-to-Cloud:** Rejected because a 20 RPS peak is easily handled by minimal cloud compute; over-the-air model syncing and mobile optimization add unnecessary overhead.
* **Edge-Only Inference:** Rejected because quantized mobile models lack the parameter size and semantic expressiveness needed to safely handle legally sensitive edge cases.

## Consequences

* **Positive:** Allows running uncompressed, enterprise-grade Vision Transformers with a single, instantly updated duplicate-image blocklist.
* **Positive:** Simplifies observability and drift detection since 100% of telemetry passes through one cloud boundary.
* **Negative:** Weak connections can exhaust the 600 ms latency budget on image ingestion before cloud inference even begins.

## Revisit if

* Traffic increases by multiple orders of magnitude (e.g., >1,000 RPS), where cloud egress fees and concurrent GPU costs outweigh cloud-only engineering simplicity.