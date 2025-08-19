
# Medium-Sized Secure Banking App (AWS)

A **cloud-native banking application** designed for around **1–2 million users**.  
The system emphasizes **security, correctness, and predictable scaling** while remaining cost-effective and manageable by a medium-sized team.  

*check secure_banking_app.png for the architectural diagram*

---

## User Login & Access

**Flow:**  
Client → Route53 → CloudFront → WAF → API Gateway → Cognito (MFA, JWT) → BFF  

**Working:**  
- Traffic is resolved via Route53 and cached at CloudFront.  
- WAF inspects and filters requests (rate-limits, bad IPs, malformed payloads).  
- API Gateway validates JWTs issued by Cognito, enforcing MFA.  
- Only validated requests reach the Backend-for-Frontend (BFF).  

✔️ Authentication and throttling happen before any app service is touched.  

---

## Viewing Accounts

**Flow:**  
Client → BFF → Accounts Service → Redis (cache) → Aurora Reader (if cache miss) → Response → Logs → S3 (audit)  

**Working:**  
- Accounts Service first checks Redis for balances or recent activity.  
- On cache miss, it queries an Aurora read-replica.  
- The response is returned and also cached with a short TTL for subsequent requests.  
- Access logs go into CloudWatch; immutable audit records are stored in S3 with Object Lock.  

✔️ Fast responses for the user, immutable access history for compliance.  

---

## Money Transfers

**Flow:**  
Client (with Idempotency-Key) → API Gateway → BFF → Payments Service → DynamoDB (idempotency) → SQS FIFO → Payments Worker → Payment Rails → Ledger Service → Aurora (append-only entries) → SNS → Notifications → Email/SMS  

**Working:**  
- The client provides an Idempotency-Key to guard against retries.  
- Payments Service stores/validates the key in DynamoDB and enqueues the request into an SQS FIFO queue. Grouping by account ensures ordered processing.  
- A worker consumes the queue, interacts with external payment rails (with retries and circuit breakers).  
- On settlement, the worker calls Ledger Service, which appends balanced debit and credit rows to Aurora.  
- The ledger event is pushed to SNS, triggering Notifications Service to inform the customer.  

✔️ Exactly-once effect for transfers, even under retries or worker crashes.  
✔️ Strict ordering per account prevents race conditions.  

---

## Statements

**Flow:**  
EventBridge (monthly trigger) → Statement Job → Aurora (transactions) → PDF Generation → S3 (versioned, Object Lock) → Pre-signed URL → Client  

**Working:**  
- A scheduled job queries Aurora for monthly transactions.  
- Statements are generated as PDFs and stored in S3 with versioning and Object Lock for immutability.  
- Customers fetch them through short-lived pre-signed URLs from the Accounts Service.  

✔️ Statements are tamper-proof and safely retrievable years later.  

---

## Notifications

**Flow:**  
Ledger Event → SNS → Notifications Service → Email/SMS Provider → Customer  

**Working:**  
- Ledger or scheduled events are published to SNS.  
- Notifications Service consumes the event and pushes messages to email or SMS providers.  
- Delivery results are logged for operational visibility.  

✔️ Customers receive near real-time updates about their account activities.  

---

## Audit & Monitoring

**Flow:**  
Application Logs → CloudWatch  
AWS API Actions → CloudTrail  
Critical Events → S3 (Object Lock)  

**Working:**  
- CloudWatch handles runtime logs, metrics, and alarms (p95 latency, error rates, queue depth, replica lag).  
- CloudTrail records every AWS API call for traceability.  
- Critical audit logs (ledger entries, login failures, transfers) are sent to S3 buckets configured with Object Lock to prevent tampering.  

✔️ Operations are observable in real time; compliance logs remain immutable.  

---

## Scaling Characteristics

**Working:**  
- Reads are served from Redis, fallback to Aurora readers.  
- Writes flow through FIFO queues into Aurora, preserving ordering and consistency.  
- Workers autoscale with SQS queue depth.  
- Aurora scales with one writer and multiple readers, with RDS Proxy smoothing connection storms.  
- Redis provides sub-millisecond lookups for sessions and rate-limits.  

✔️ Predictable scaling under ~1k reads/sec and ~100 writes/sec.  

---

## Reliability in Failures

**Working:**  
- If a payment rail is down → transfers are marked `PENDING` and retried later.  
- If a worker fails → message is redelivered. DynamoDB idempotency prevents duplicate processing.  
- If Aurora writer fails → replicas take over; readers remain available.  
- Audit and statement buckets cannot be altered, ensuring post-incident investigations have trustworthy data.  

✔️ Ledger integrity and auditability remain intact even during partial failures.  

---

## Key Design Guarantees

✔️ Simple but secure — core paths (auth, ledger, audit, DR) are covered.  
✔️ Idempotent & ordered — no duplicate debits, no out-of-sequence transfers.  
✔️ Immutable audit — regulators and customers can rely on historical accuracy.  
✔️ Cloud-native & cost-aware — ECS Fargate, Aurora, Redis, DynamoDB, and S3 form a balanced stack.  
