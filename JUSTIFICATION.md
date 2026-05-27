# Justification

### Serving Pattern: Online with Asynchronous Batch Fallback

We have selected an **Online Serving Pattern** backed by an asynchronous batch queue for edge cases. Content moderation requires immediate action to protect platform integrity; allowing unsafe images to persist while waiting for a nightly batch process would expose the platform to severe brand damage and legal liability.

Our choice is strictly tied to the operational realities of user content uploads. The critical path requires a real-time HTTP response immediately following an image upload to determine if the content can be instantly visible or must be withheld. However, to handle highly ambiguous inputs, the architecture introduces a **hybrid online/batch fallback**. When the online inference service returns a low-confidence score, the request safely escapes the synchronous loop. The system returns a pending state to the user while routing the image payload via a streaming event into an asynchronous queue for manual triage, balancing real-time responsiveness with human accuracy.

### Inference Infrastructure: Cloud Deployment

Inference is executed entirely within a centralized **Cloud Architecture**.

Given Scenario C’s specific traffic profile—averaging 2 uploads per second with a peak of 20 uploads per second—running an on-device edge filter introduces unnecessary client-side synchronization and engineering complexity. A centralized cloud infrastructure easily absorbs this volume. Furthermore, because mistakes carry heavy legal weight, a cloud deployment allows the system to host larger, highly expressive deep learning models (such as Vision Transformers) and maintain a unified, securely audited database of feature embeddings, maximizing classification accuracy and ensuring compliance.

### Operational Trade-offs & SLA Targets

To ensure optimal performance under heavy load, we are optimizing for **Latency** and **Cost**, while setting **Throughput** as our bound constraint (the budget).

* **Optimization 1: Latency (p95 < 600 ms):** Per the platform guidelines, the absolute maximum p95 latency budget is **600 ms**. Because mistakes carry severe legal consequences, we utilize this window to run deep visual feature extraction, compute high-quality embeddings, and cross-reference embedding spaces without causing noticeable platform lag for the uploading user.
* **Optimization 2: Cost (\$0.005 per image):** Given the relatively low traffic volume, we minimize infrastructure overhead by using auto-scaling, serverless cloud container configurations. By implementing vector embedding caching to instantly catch duplicate viral uploads, we bound our target operational cost to **$0.005 per uploaded image**.
* **The Budget: Throughput (sustained 2 RPS, peak 20 RPS):** Throughput is our rigid constraint. The architecture is engineered to handle a baseline load of **2 Requests Per Second (RPS)** and scale automatically to handle sudden viral traffic surges of up to **20 RPS** without dropping packets or exceeding our 600 ms latency SLA.

---

### Metric Reference Table

| Target Metric | Target Level | Constraint Type | Scenario Strategy |
| --- | --- | --- | --- |
| **System Latency (p95)** | $< 600\text{ ms}$ | Optimization | Matches the explicit platform latency SLA budget |
| **Unit Cost** | $\$0.005\text{ / image}$ | Optimization | Reduced via serverless scaling and embedding caching |
| **Baseline Throughput** | $2\text{ RPS}$ | Rigid Budget | Built around average platform upload volumes |
| **Peak Throughput** | $20\text{ RPS}$ | Rigid Budget | Elastic ceiling designed to absorb peak user traffic |

---

### Resiliency & Fallback Strategy

When a model fails or becomes entirely unavailable, the system defaults to a strict **Fail-Safe/Fail-Closed** posture. Because mistakes on this user-generated content platform carry severe legal weight, a Fail-Open configuration cannot be permitted under any risk tier.

If the `Moderation Inference Service` throws a 5xx error or times out beyond a 400 ms internal threshold, the API Gateway immediately trips a circuit breaker to protect downstream dependencies. Instead of breaking the user experience or risking a compliance breach, the system flags the image with a `System_Timeout` metadata tag, securely writes the raw image payload straight to `Object Storage`, and places a pointer inside the `Human Review Queue`. To the end-user, the content is set to a "Temporarily Hidden - Processing" state. This ensures that a total outage of the machine learning infrastructure cannot leak toxic content to the platform, while human moderators act as the manual dampener to drain the backlogged queue in batches once services recover.