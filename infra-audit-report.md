# Infrastructure Audit Report  
**Date:** 19/08/2025 (Tuesday)  
**Scope:** Banking Application Infrastructure (Cloud-based, Medium Scale - 1M to 2M userbase)  

---

## Executive Summary  
The infrastructure has been designed with layered security, logical separation, and secure control flows that align with modern cloud-native practices. This self-audit reviews the infrastructure against confidentiality, integrity, and availability principles. While the design covers critical controls such as authentication, encryption, and audit immutability, further refinements are identified in areas of resilience, monitoring, and dependency handling.  

---

## Observations  

### 1. Edge & Access  
The entry point to the system is shielded by a well-structured sequence of Route53, CloudFront, WAF, and API Gateway. This layered approach ensures that no direct access to the backend is possible without passing through multiple gates of filtering, caching, and validation. WAF rules enforce JSON schema validation and rate limiting, effectively reducing the attack surface from malformed payloads and brute-force attempts. Cognito MFA further strengthens the edge by ensuring that identities are validated w...

**To Work On:** While the current measures are robust, secret rotation in Cognito must be automated and tested periodically. Long-lived secrets are a prime attack vector if left unmanaged. JWT policies should also be validated with stricter expiry and rotation policies to minimize replay attacks. Expanding WAF rules to cover OWASP Top 10 attack classes (e.g., SQL injection, XSS) would improve resilience against common application-layer exploits. If left unattended, gaps here could lead to credential thef...

---

### 2. Application Tier  
The application tier is structured around ECS Fargate services running in private subnets with tightly scoped IAM roles. This separation ensures that sensitive workloads such as Payments and Ledger operate independently from less critical services like Notifications. Payment Worker is isolated to handle external payment rails, containing the blast radius of any third-party API compromise. Inter-service communication is routed through TLS, and IAM permissions are scoped per-task to uphold least privilege.  

**To Work On:** Chaos testing should be incorporated into ECS scaling events to validate service continuity under unpredictable spikes. Without such testing, assumptions about autoscaling reliability remain unverified. Furthermore, inter-service TLS enforcement must be validated — while TLS termination at the load balancer is in place, enforcing TLS for internal VPC traffic ensures that lateral movement within the network cannot compromise data flows. These measures reduce risks of service interruption a...

---

### 3. Data Layer  
The data layer exhibits multiple redundancies and safeguards. Aurora PostgreSQL separates write and read traffic, ensuring performance is not impacted by analytical queries. Its append-only ledger configuration guarantees financial records remain tamper-proof. Redis supports high-speed session management and caching, while DynamoDB underpins idempotency keys and outbox patterns, securing transaction ordering. S3 buckets with Object Lock ensure monthly statements and audit records remain immutable.  

**To Work On:** Failover testing in Aurora must be part of quarterly drills to confirm the ability to sustain outages with minimal downtime. DynamoDB TTL configurations should be validated to confirm expired idempotency keys are cleaned efficiently, preventing transaction collisions. Redis must be monitored for eviction rates; silent data loss here could lead to incorrect balance calculations or failed sessions, eroding trust. Without these checks, resilience remains assumed rather than demonstrated.  

---

### 4. Async & Events  
Asynchronous workflows are managed with SQS FIFO queues and SNS fan-out mechanisms. FIFO queues enforce strict ordering, ensuring no double-debits or race conditions in financial transfers. SNS decouples notifications, allowing services like email and SMS to be handled without blocking core transaction flows.  

**To Work On:** Dead-letter queues (DLQ) should be attached to SQS to capture poisoned or undeliverable messages. Without DLQs, failed messages may loop indefinitely, clogging the pipeline and delaying legitimate transfers. Additionally, SNS topic subscriptions should be audited regularly; redundant or misconfigured subscriptions may lead to duplicate notifications or wasted costs. Proactive refinement here prevents data inconsistency and unnecessary operational noise.  

---

### 5. Observability & Auditability  
Observability is enabled through CloudWatch logs, alarms, and metrics across the stack. CloudTrail captures all API actions, with records preserved in S3 buckets configured with Object Lock. This guarantees audit trails are immutable, a key requirement for both compliance and customer trust. Metrics on ECS services and database queries provide visibility into baseline performance trends.  

**To Work On:** CloudWatch anomaly detection should be expanded to identify transaction spikes or latency patterns that deviate from expected baselines. Without this, early indicators of fraud or denial-of-service activity could be missed. Additionally, CloudTrail logs should be periodically exported to cold storage systems for long-term archival. Such storage both reduces costs and strengthens regulatory alignment by safeguarding audit logs outside the operational blast radius.  

---

### 6. External Dependencies  
External systems such as UPI, IMPS, NEFT, and card networks are accessed exclusively through the Payment Worker, minimizing exposure of the core platform. Notifications are decoupled and delivered through managed providers, ensuring message delivery is reliable but independent of core ledger availability.  

**To Work On:** Failover strategies must be validated with secondary providers for both payment rails and notification services. Outages in a primary provider without pre-tested alternatives create single points of failure. SLA monitoring across all external APIs is also critical; without real-time visibility into upstream failures, cascading transaction delays could erode service reliability. These refinements will ensure the system withstands disruptions beyond its immediate control.  

---

## Key Flow Checks  

✔️ **Login Flow:** Client → API GW → Cognito → BFF → Redis → Aurora (audit log).  
Working: MFA validation and JWT issuance operate as designed, ensuring all traffic reaching backend services carries verified identity context. Short token lifetimes and enforced refresh token rotation remain areas to strengthen, mitigating token replay risks.  

✔️ **Account Overview:** Client → BFF → Accounts Service → Redis → Aurora (reader).  
Working: Cached reads accelerate account views without overloading Aurora. To work on: caching invalidation strategies should be tested to confirm stale balances are never displayed to customers. Incorrect balances, even temporarily, can significantly erode user trust.  

✔️ **Money Transfer:** Client → BFF → Payments Service → DynamoDB (idempotency) → SQS FIFO → Payment Worker → Payment Rails → Ledger → Aurora → SNS → Notifications.  
Working: Idempotency keys and FIFO queues ensure strict transaction ordering. To work on: stress-testing queue throughput and retry policies is essential to confirm transfers remain consistent under peak load. Without it, risks of delayed or duplicate notifications may appear.  

✔️ **Monthly Statements:** Scheduled ECS Job → Aurora → PDF → S3 (Object Lock) → Pre-signed URL to client.  
Working: Immutable storage guarantees regulatory-grade recordkeeping. To work on: checksum verification of generated PDFs adds another layer of assurance, confirming statements are not only immutable but also free of corruption during generation or retrieval.  

---

## Overall Assessment  
The infrastructure demonstrates strong adherence to secure, cloud-native principles. Layers of authentication, encryption, and immutability are well established, forming a trustworthy foundation for financial operations. However, to mature from designed security into proven resilience, continuous drills, failover tests, and chaos experiments are necessary. By addressing monitoring gaps, validating failover paths, and tightening token lifecycles, the platform can evolve into a fully production-ready banking application.

