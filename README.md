# ai-agent-payment-system-architecture-
Enterprise-grade, secure, and highly available AWS architecture for an AI-driven financial payment system.

🚀 AI Agent Payment System - Enterprise AWS Architecture

A highly scalable, secure, and event-driven financial payment system designed to manage autonomous AI agent spending while strictly enforcing human oversight, financial isolation, and enterprise-grade security.

** Table of Contents

 1. Executive Summary & Core Strategy

 2. High-Level System Architecture (7 Layers)

 3. Detailed AWS Architecture & Workflows

 4. AWS Service Selection & Trade-offs
  
 5. Network & Security Design (VPC)
   
 6. Business Continuity & Disaster Recovery
 
 7. FinOps & Cost Optimization

 8. Key Engineering Decisions

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Executive Summary & Core Strategy

Integrating AI agents into financial workflows introduces severe risks regarding unauthorized spending and compromised credentials. This architecture solves that problem by implementing a "Zero Trust" AI Financial Control strategy.

<img width="1536" height="1024" alt="Exexutive Digram" src="https://github.com/user-attachments/assets/0c13ef41-81b1-496b-be6c-af54e7dd89f4" />

Figure 1: Executive Overview highlighting the "Zero Trust" strategy and the Four Pillars of Architecture.


🛡️ The Core Strategy: Blast Radius Containment

We never give the AI direct access to the User's main bank account. Instead, we use a Blast Radius Containment strategy:

* Human Admin (Root User): Controls the Main Bank Account, allocates funds to the AI, and approves high-risk actions.

* AI Agent (Automated Worker): Accesses an Isolated Wallet ONLY. It has strict daily spend limits and cannot touch main funds. If the AI is compromised, the financial damage is strictly contained to the allocated limit.

🏛️ The Four Pillars of Architecture

1. Enterprise-Grade Security: Bank-grade encryption (KMS), global traffic filtering (WAF/Shield), and audit-ready financial records.

2. Intelligent Risk & Fraud Management: A real-time "Risk Engine" (Redis) evaluates every transaction. If the AI attempts an unusual purchase, the system pauses and requires Human biometric approval.

3. Unbreakable Reliability: Queue-based processing ensures that even if 10,000 transactions happen at once, they are safely queued without data loss.

4. Business Continuity (BCP): Multi-Region "Warm Standby" architecture guarantees near-zero data loss (<1 second RPO) and system recovery in under 2 minutes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# High-Level System Architecture (7 Layers)

The system is built on a decoupled, layered architecture to ensure separation of concerns, security, and scalability.

<img width="949" height="1658" alt="High level Digram" src="https://github.com/user-attachments/assets/afbbb953-1763-44b8-a66e-44923611ebf9" />

Figure 2: The 7-Layer Architecture from Client Ingress to Observablity


Layer Breakdown & Technical Decisions:


1. Client Layer (Strict Role Separation)
   
 *Human Root User: Authenticates via Email + Password + Biometric. Has full system control.

 * AI Agent (Restricted): Authenticates via AaSN API Key. Strictly limited to "Spend Only" actions to minimize blast radius.
  

2. Edge Security & Resilience Layer
   
 * Traffic Flow: Route 53 (Geo Routing) ➔ AWS WAF (IP/DDoS Protection) ➔ Application Load Balancer.

 * Security Decision: TLS/HTTPS Termination happens at the Load Balancer, ensuring all data is Encrypted in Transit across the public internet.

 * Resilience Decision: ALB performs Health Checks every 30 seconds for Auto-healing (replacing failed instances) and Auto-scaling (handling traffic spikes).

 * VCP Decision: [US Region ACTIVE] ➔ failover ➔ [EU Region PASSIVE]. We use an Active-Passive model to keep DR costs low while maintaining rapid recovery.
  

3. Authentication & Authorization Layer
   
 * Auth Server (Cognito): Issues a short-lived JWT Access Token (15 min) and a secure Refresh Token (30 days). Includes token rotation and theft detection.

 * API Gateway: Validates JWT signature.

    ■ Rate Limiting Decision: We use a Redis Token Bucket applied AFTER authentication. Why? To prevent unauthenticated bots from consuming rate limit quotas and to apply stricter limits based on user identity.

    ■ Based Routing: Routes role="root" to Management Services. Strictly blocks role="ai_spend_only" from accessing admin endpoints (e.g., blocks /popup routes).


4. Application Layer (Role-Based Services & Risk Engine)
   
 * Management Service (Human Only): Handles Top-ups, AI Management,payment and Statement generation.
  
 * Payment Service (AI & Human):
  
  ■ Risk Engineering (Level 5 - Internal): Evaluates every transaction using Velocity Checks (Redis), Anomaly Detection, and Fraud Scoring.

  ■ Decisions: ALLOW ➔ Pass | BLOCK ➔ Reject. If CHALLENGE, triggers Step-Up Auth (Human biometric approval).

  ■ ACID Transaction: Executes Lock Row ➔ Deduct Funds ➔ Commit in MySQL to prevent race conditions.


* Top-up Service (Human Only - Inbound Funds):
  
 ■ Dedicated service for funding the AI Isolated Wallet. It manages the complex, asynchronous nature of external bank transfers.

 ■ Webhook Handler: Receives asynchronous callbacks from the Bank API. 

 ■ Security Decision: Strictly verifies TLS & Webhook Signatures on every callback to prevent malicious actors from spoofing bank responses to falsely credit accounts.

 ■ Polling Mechanism: Resiliency Decision: Because external banks can sometimes drop or delay webhooks, we implement a fallback Polling Handler that periodically checks the bank's status API to ensure no top-up is left in a "pending" state forever.

 ■ Bank Reconciliation: Consistency Decision: A dedicated Reconciliation Handler runs periodically to match our internal Aurora ledger with the actual bank statements. This guarantees 100% financial accuracy and automatically unblocks any transactions that got stuck due to network glitches.

 ■ Decision: By isolating these heavy, asynchronous bank integrations (Webhooks/Polling/Reconciliation) into the Top-up Service, we ensure they never cause thread exhaustion or latency in the high-frequency AI Payment Service.


5. Async Processing Layer (Zero Data Loss & Reconciliation)
   
 * AWS SQS (Durable, Disk-based Queue): Why SQS? If a background worker crashes mid-processing, SQS Worker Death Recovery (via Visibility Timeout) ensures the message returns to the queue for another worker to process. This guarantees zero data loss.

* Message Flow:

1. Payment Q ➔ 2. Update Q ➔ 3. Receipt Q ➔ 4. Top-up Q ➔ 5. Statement Q.
  
* Background Workers:
  
 ■ Worker (Every 30s): Crucial Consistency Decision. Runs a reconciliation flow every 30 seconds to match internal records with bank webhooks, auto-unblocking stuck payments.

 ■ Reliability Equipped with Circuit Breakers, Exponential Backoff, DynamoDB Idempotency and Dead Letter Queues (DLQ).

* Document Generation & Notifications (The Final Step):
  
 ■ Receipt Worker (For AI Payments): Picks up successful tasks from the Receipt Queue. It generates a PDF receipt, uploads it to Amazon S3 (encrypted at rest using AWS KMS), and notifies the AI Agent via WebSocket/Webhook that the payment is complete.

 ■ Statement Worker (For Human Top-ups): Picks up successful tasks from the Statement Queue. It generates a PDF statement, uploads it to Amazon S3 (encrypted via AWS KMS), and triggers Amazon SNS to send a success notification (Email/SMS) to the Human Admin.

 ■ Decision: PDF generation is CPU-intensive. By decoupling it into dedicated queues, we ensure the core financial transaction is marked "COMPLETED" instantly, while the heavy document processing happens asynchronously in the background.


7. Data Layer (Multi-Storage & Encryption)
   
 ■ Amazon Aurora MySQL: Source of Truth for Wallets/Transactions (ACID).

 ■ Amazon DynamoDB: High-speed Idempotency Keys (TTL: 7 days) to prevent double-charging during network retries.

 ■ Amazon ElastiCache (Redis): Sub-millisecond Rate Limiting and Fraud Score caching (TTL: 60 seconds to ensure fresh data).

 ■ AWS KMS (Key Management Service): Security Decision. Enforces Encrypt at Rest, Encrypt in Transit, and automatic Key 
Rotation for all sensitive financial data.

 ■ S3 / Object Store: Secure storage for Receipts, Audit Logs, and Top-up Statements.


8. Observability & Reliability
    
 ■ Monitoring: CloudWatch Dashboards and Centralized Logging (CloudWatch Logs / ELK Stack) for deep log analysis.

 ■ Alerting: PagerDuty/Slack integration for Critical Errors. Decision: Triggers immediate alert if DLQ > 10 messages, indicating a systemic failure.

 ■ Auto-Healing: Health checks every 30s ensure 99.99% uptime.

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Complete Top-Up Flow (Human Adds Money)

When a Human Admin initiates a fund transfer (e.g., "Top-up $500 from bank"), the system prioritizes security, idempotency, and resilience. We use a 17-step asynchronous flow to ensure zero data loss and perfect financial accuracy.

<img width="1224" height="1285" alt="Top Up Flow" src="https://github.com/user-attachments/assets/c919a52f-194b-4ae3-9a8c-c7d2c7a57366" />

Figure X: The secure, 17-step Top-Up flow featuring dual idempotency checks, circuit breakers, and automated receipt generation.


** Phase 1: API / Management Service (Synchronous Validation)

 ■ Step 1: Validate Request. The Management Service checks the JWT (role="root"), validates the amount (e.g., $500 > 0 and < $10,000), and verifies bank account details.

 ■ Step 2: Create Idempotency Key. The server generates a unique key (e.g., idem_topup_xyz789) to track this specific request.

 ■ Step 3: Check Idempotency Key (1st Check). We query DynamoDB. If the status is "COMPLETED", we return the original result immediately. If NOT FOUND, we continue.

 ■ Step 4: Push to Top-up Queue (SQS). We return a "status": "PROCESSING" response to the Human instantly, while pushing the payload to the durable SQS queue for background processing.

** Phase 2: Top-up Worker (Background Processing & Bank Integration)

 ■ Step 5: Check Idempotency Key (2nd Check). The worker queries DynamoDB again to ensure no duplicate processing occurred between Step 3 and now.

 ■ Step 6: Mark Idempotency = "IN_PROGRESS". We update DynamoDB with a 7-day TTL to lock this transaction.

 ■ Step 7: Create Transaction in MySQL. We insert a record with Status = 'PENDING'. Crucial Decision: The Wallet Balance remains UNCHANGED at this stage. We do not credit the funds until the bank confirms success.

 ■ Step 8: Check Circuit Breaker (Redis). We check if the external Bank API is healthy. If DOWN ➔ Fail fast. If UP ➔ Continue.

 ■ Step 9: Call External Bank API (With Resiliency). We execute the transfer.
On Failure: We use Exponential Backoff (Wait 1s 2s ➔ 4s). After 3 failures, we move to the Dead Letter Queue (DLQ) and mark status as "FAILED".

 ■ On Success: We proceed to Step 10.

** Phase 3: Transaction Finalization & Event Publishing

 ■ Step 10: Update Transaction Status = "SUCCESS". We update the MySQL record.

 ■ Step 11: Update Wallet Balance. We credit the amount to the AI Isolated Wallet, update the balance, and record the transaction history.

 ■ Step 12: Publish Event. We publish an event to Amazon SNS / EventBridge to notify other microservices and trigger analytics.

 ■ Step 13: Mark Idempotency = "COMPLETED". We update DynamoDB (TTL: 7 days) to finalize the lock.

 ■ Step 14: Delete from Processing Queue. We remove the message from SQS to confirm successful processing.

** Phase 4: Notifications, Receipts & Failure Handling

 ■ Step 15: Send Success Notification & Generate Receipt.
A background worker generates a PDF receipt, encrypts it using AWS KMS, and uploads it securely to Amazon S3.
Simultaneously, we trigger an Email/Notification to the Human Admin confirming the "Top-up Successful! $500".

 ■ Step 16: Move to DLQ (If Needed). If the process failed after 3 retries, the task is moved to the DLQ, error details are logged, and an alert is sent to the engineering team.

 ■ Step 17: End. The process is complete, and all systems are updated.


🔄 Bank Integration: Webhooks, Polling, and Reconciliation

Because external banks are notoriously unreliable with callbacks, the Top-up Service implements a multi-layered approach to guarantee funds are never lost:

 ■ Webhook Handler: Receives asynchronous callbacks from the Bank API. Strictly verifies TLS & Webhook Signatures to prevent spoofed callbacks.

 ■ Polling Mechanism: A fallback handler periodically checks the bank's status API to ensure no top-up is left in a "pending" state forever if a webhook is dropped.

 ■ Bank Reconciliation: A dedicated Ledger Worker runs every 30 seconds to match our internal Aurora ledger with the actual bank statements, automatically unblocking any stuck transactions.

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# AI Payment Flow (Restricted Agent Spending)

When the AI Agent initiates a payment (e.g., "Pay $50 to Amazon"), the system must execute the transaction instantly while strictly enforcing ACID compliance, preventing race conditions, and handling slow external merchant APIs. We achieve this using a 4-Phase Asynchronous Architecture.

<img width="1024" height="1536" alt="Payment Flow" src="https://github.com/user-attachments/assets/8956203e-2679-49d0-a6f7-b7b1fc1ca5e8" />

Figure X: The 4-Phase AI Payment Flow ensuring ACID compliance, preventing race conditions, and automating refunds on failure.


** Phase 1: Synchronous Path (Fast, Protects Wallet Balance)

This phase executes in milliseconds to lock the funds and return an immediate response to the AI, preventing it from overspending.

 ■ 1: API Gateway Validation. Validates the JWT and routes the request to the Payment Service.

 ■ 2: Idempotency Check. Checks DynamoDB for the Idempotency-Key (e.g., idem_abc123). If already "COMPLETED", returns the cached result to prevent double-charging.

 ■ 3 & 4: Policy & Balance Check. Validates the AI Spending Policy and checks the Wallet Balance in Redis (sub-millisecond latency).

 ■ Case: If balance < $50, routes to a Ledger Queue for logging and ends immediately.

 ■ Step 5: ACID Transaction (The Critical Lock). Executes a strict MySQL transaction:

  • BEGIN; SELECT FOR UPDATE (Lock Row); Re-check balance.

  • UPDATE wallets SET balance = balance - 50;

  • INSERT INTO transactions (status='PROCESSING'); COMMIT;

 Decision: SELECT FOR UPDATE physically locks the database row, preventing the AI from spending the same $50 twice in concurrent requests.

 ■ Step 6 & 7: Queue & Respond. Marks the key as "IN_PROGRESS" in DynamoDB (TTL: 7 days), pushes the task to Queue 1 (Payment Queue), and instantly returns "status": "PROCESSING" to the AI.


** Phase 2: Async Path 1 (External API Execution)

 ■ Step 8: Payment Worker. Picks up the task from Queue 1. It re-checks DynamoDB (safety net), checks the Circuit Breaker in Redis (to fail fast if Amazon's API is down), and calls the External Merchant API.

  • Retry Logic (Exponential Backoff): If the API fails, the worker waits 1s ➔ 2s ➔ 4s (with jitter) before retrying.

 ■ Step 9: Success Path. On success, the result is pushed to Queue 2 (Update Queue).

  • Decision: This extra queue protects MySQL from direct worker pressure, ensuring the database isn't hammered by sudden spikes of successful API callbacks.

 ■ Step 9b: Failure Path (Automated Refund). If the API fails 3 times, the task moves to the Dead Letter Queue (DLQ) for manual review. Crucially, this triggers an automated rollback: UPDATE wallets SET balance = balance + 50 to refund the locked funds back to the AI.


** Phase 3: Async Path 2 (Database Finalization)

 ■ Step 10: Background Worker. Picks up the success result from Queue 2.

  • Updates MySQL: UPDATE transactions SET status='COMPLETED' WHERE id='tx_789'.

  • Marks Idempotency = "COMPLETED" in DynamoDB.

  • Pushes the final task to Queue 3 (Receipt Queue).


** Phase 4: Async Path 3 (Receipt Generation & Notification)

 ■ Step 11: Receipt Worker. Picks up from Queue 3.
Generates a PDF Receipt.

  • Uploads the encrypted PDF to Amazon S3 (via AWS KMS).

  • Notifies the AI Agent via WebSocket / Webhook that the "Payment Completed!"
END.

<img width="1536" height="1024" alt="PAY " src="https://github.com/user-attachments/assets/e1699b26-0989-41a0-a9c1-63af91555981" />

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Detailed AWS Architecture & Execution Flows

The core of the system is divided into two distinct, highly resilient execution paths: the Human Top-Up Flow (Inbound Funds) and the AI Payment Flow (Outbound Funds). Both paths share a unified, secure ingress layer but diverge to optimize for their specific business requirements.

<img width="768" height="1677" alt="AWS Architecture Diagram  drawio" src="https://github.com/user-attachments/assets/1227ed9d-f54e-48a6-9d48-cbf30be799e1" />

Figure 3: Complete AWS Architecture showing the dual-path execution, async processing, and observability stack.

🛡️ 1. Unified Ingress & Authentication Layer

Before any business logic is executed, all traffic passes through a strict security and authentication gauntlet:
Edge Routing: Traffic enters via Amazon Route 53 and passes through AWS WAF to block malicious IP addresses and DDoS attacks.

 ■ TLS Termination: The Application Load Balancer (ALB) terminates SSL/TLS using AWS ACM, ensuring all data is encrypted in transit.

 ■ Identity Management (Amazon Cognito):

  • Human: Authenticates via Biometric + Password verification.

  • AI Agent: Authenticates via restricted AaSN API Key validation.

  • Issues short-lived JWTs and Refresh Tokens.

 ■ API Gateway: Validates the JWT signature and expiry. Crucially, it enforces User Rate Limiting using a Redis Token Bucket to prevent API abuse before the request reaches the microservices.

💰 2. The Human Top-Up Flow (Inbound Funds)

Optimized for security, external bank integration, and human notification.

1. Idempotency Check: The Management Service checks DynamoDB for an existing Idempotency Key. If "COMPLETED", it skips processing to prevent duplicate top-ups.
   
2. Queueing: The request is pushed to the SQS Top-up Queue, and a "PROCESSING" status is instantly returned to the Human user.
   
3. Top-up Worker (Lambda):
   
 • Marks the Idempotency Key as "IN_PROGRESS" in DynamoDB.

 • Creates a PENDING transaction in Aurora MySQL.

 • Checks the Circuit Breaker in ElastiCache (Redis). If the bank is healthy, it calls the External Bank API via the NAT Gateway.

 • Resiliency: Implements exponential backoff retries (1s ➔ 2s ➔ 4s). On the 3rd failure, it pushes to the SQS Dead Letter Queue (DLQ).

4. Database Finalization: Upon bank success, Aurora MySQL credits the wallet (balance = balance + 500) and updates the transaction to "COMPLETED". It also checks the Ledger Queue to auto-unblock any stuck payments.
   
5. Statement Generation: The task moves to the SQS Statement Queue. A Statement Worker (Lambda) generates a PDF receipt, encrypts it using AWS KMS, and uploads it to Amazon S3.
   
6. Notification: Finally, Amazon SNS sends a success notification (Email/SMS) to the Human user.

🤖 3. The AI Payment Flow (Outbound Funds)

Optimized for high-frequency, low-latency transactions and strict ACID compliance.

1. Policy & Balance Check: The Payment Service checks DynamoDB for idempotency, then uses ElastiCache (Redis) to validate the AI Spending Policy and check the Wallet Balance in sub-milliseconds.
   
2. ACID Transaction: To prevent race conditions, Aurora MySQL executes a strict SELECT FOR UPDATE (Row Lock), deducts the funds (balance = balance - 50), and inserts a "PROCESSING" transaction record.
   
3. Queueing: The Idempotency Key is marked "IN_PROGRESS" in DynamoDB (TTL: 7 days). The task is pushed to the SQS Payment Queue, and "PROCESSING" is returned to the AI Agent.
   
4. Payment Worker (Lambda): Picks up the queue, checks the Redis Circuit Breaker, and calls the External Merchant API (with the same 1s➔2s➔4s retry logic).
   
5.The "Buffer" Queue: On success, the result is pushed to the SQS Update Queue.
   
   • Architectural Decision: This intermediate queue protects Aurora MySQL from direct worker pressure, preventing database thread exhaustion during traffic spikes.
 
6. Background Worker (Lambda): Picks up from the Update Queue, updates Aurora MySQL status to "COMPLETED", and marks the DynamoDB Idempotency Key as "COMPLETED".
   
7. Receipt Generation: The task moves to the SQS Receipt Queue. A Receipt Worker (Lambda) generates the PDF, securely uploads it to Amazon S3 via AWS KMS, and triggers Amazon SNS to notify the AI Agent via Webhook.

   
🔍 4. Observability & Reliability Stack

To ensure 99.99% uptime and rapid incident response, the architecture is wrapped in a comprehensive observability layer:

 ■ Auto-Healing & Auto-Scaling: ALB performs health checks every 30 seconds, automatically replacing failed instances and scaling compute resources during traffic spikes.

 ■ Centralized Logging: AWS X-Ray provides distributed tracing, while logs are aggregated in CloudWatch Logs for deep analysis.

 ■ Critical Alerting: CloudWatch Alarms are integrated with PagerDuty/Slack. If the SQS DLQ exceeds 10 messages, an immediate critical alert is fired to the engineering team.

 ■ Event-Driven Routing: Amazon EventBridge routes system events to trigger downstream microservices and analytics pipelines seamlessly.

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Network & Security Architecture (VPC Design)

Security is not an afterthought; it is the foundation of this architecture. We implemented a rigorous Defense-in-Depth strategy across multiple layers—from the public edge to the isolated database vault. The network is split across two Availability Zones (AZs) for high availability, utilizing strict CIDR segmentation (10.0.0.0/16).

<img width="1672" height="941" alt=" AWS VPC Architecture Diagram" src="https://github.com/user-attachments/assets/c0fe65fe-7cef-427d-b7a6-df030ce470ef" />

Figure X: US Two-AZ High Availability VPC Design featuring strict Public/Private subnet separation, NACLs, Security Group chaining, and outbound traffic inspection.

** Step 1: Edge Security & Ingress Flow
  
Before traffic even reaches our VPC, it passes through a multi-layered AWS Edge security gauntlet:

 1. Amazon Route 53: Resolves DNS to the Application Load Balancer IP.
   
 2. AWS Shield: Provides automatic, always-on Layer 3/4 DDoS protection at the edge.
   
 3. AWS WAF (Web Application Firewall): Inspects Layer 7 traffic, blocking malicious payloads (SQLi, XSS) and bad bots before they enter the network.
   
 4. Internet Gateway (IGW): The bridge allowing public traffic to enter the VPC.
   
 5. VPC Flow Logs: Captures all IP traffic metadata entering and leaving the VPC, shipping it securely to Amazon S3 for continuous security auditing and forensic analysis.

** Step 2: Public Subnet (The DMZ) - 10.0.0.0/24 & 10.0.3.0/24

The Public Subnet houses internet-facing resources. It is heavily restricted using both NACLs and Security Groups.

 ■ Network ACL (NACL): Acts as the stateless subnet firewall.

  • Inbound: Explicitly allows only Port 443 (HTTPS) from the internet.
  
  • Outbound: Allows all traffic on ephemeral ports (0.0.0.0/0 - 65535) to allow return traffic from the ALB.
 
■ Application Load Balancer (ALB): Protected by SG_ALB, which strictly allows Inbound Port 443 ONLY from the AWS WAF Prefix List.

■ NAT Gateway: Allows private instances to reach the internet.

   • Crucial AWS Detail: NAT Gateways do not support Security Groups. They are secured entirely by the Subnet's NACL and the strict outbound rules of the Private Compute Security Groups.
 
■ AWS Network Firewall: Placed in the public subnet to perform Deep Packet Inspection (DPI) on all outbound traffic, ensuring no data exfiltration occurs.

■ Route Table: Routes internet-bound traffic (0.0.0.0/0) to the IGW, and keeps internal VPC traffic (10.0.0.0/16) local.

** Step 3: Private Compute Subnet (Application Layer) - 10.0.1.0/24 & 10.0.4.0/24

This subnet hosts the core business logic (ECS/Lambda). It has no direct internet access.

■ Network ACL (NACL): Denies all inbound traffic from the internet. Only allows inbound traffic from the Public Subnet CIDR.

■ Security Group Chaining (SG_Compute):

  • Inbound: Allows Ports 80/443 ONLY from SG_ALB. (We reference the Security Group ID, not an IP address, for dynamic scaling).

  • Outbound: Strictly limited to TCP Port 443 ONLY to the specific IP address of the Third-Party Bank API via the NAT Gateway. This prevents compromised instances from communicating with rogue external servers.

 ■ Route Table Mechanics (The "Interview Trap"):

  • 10.0.0.0/16 ➔ Local: Used for internal communication with the Data Subnet. This bypasses the NAT Gateway entirely, keeping internal traffic fast, secure, and free.

  • 0.0.0.0/0 ➔ NAT Gateway (10.0.0.x): Used ONLY for outbound internet traffic to the Bank API.

** Step 4: Private Data Subnet (The Vault) - 10.0.2.0/24 & 10.0.5.0/24

This subnet houses the crown jewels: Amazon Aurora (MySQL) and ElastiCache (Redis). It is strictly isolated.

 ■ Network ACL (NACL): Denies ALL inbound traffic from the internet. Only allows inbound Port 3306 (MySQL) from the Compute Subnet CIDR.

 ■ Security Group (SG_Data):

  • Inbound: Allows Ports 3306/6379 ONLY from SG_Compute.

  • Outbound: DENY ALL. The database is never allowed to initiate outbound connections.

 ■ Route Table (Strict Isolation): Contains ONLY the Local Route (10.0.0.0/16 ➔ Local).

  • NO Internet Route. NO NAT Gateway Route. The database is physically incapable of reaching the internet.

** Step 5: High Availability & Inter-AZ Communication

 ■ AZ-2 Mirroring: The exact same secure structure is mirrored in us-east-1b using sequential CIDRs (10.0.3.0/24, 10.0.4.0/24, 10.0.5.0/24). Aurora and Redis run as Read Replicas in AZ-2.

 ■ Internal Routing: AZ-1 and AZ-2 communicate securely via the VPC's internal backbone using the 10.0.0.0/16 ➔ Local route.

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Business Continuity & Disaster Recovery 

In the financial sector, downtime and data loss are unacceptable. To ensure the system survives a complete regional outage (e.g., us-east-1 goes dark), I designed a Multi-Region Active-Passive (Warm Standby) architecture. This strategy provides near-instant data recovery while keeping DR infrastructure costs significantly lower than a fully active Multi-Region setup.

<img width="1536" height="1024" alt="AWS Multi-Region Disaster Recovery Architecture" src="https://github.com/user-attachments/assets/6cd90d6c-44f6-4532-a72c-2f59a71085b3" />

Figure X: Multi-Region Warm Standby Architecture featuring automated Route 53 failover and sub-second data replication.

1. Traffic Routing & Failover (Amazon Route 53)

  ■ Health Checks: Route 53 performs rigorous health checks every 30 seconds against the Primary Region's endpoints.

  ■ Automated Failover: If the health checks fail (indicating a regional outage), Route 53 automatically updates the DNS records to route 100% of User and AI Agent traffic to the Passive DR Region (eu-west-1).

🟢 2. Primary Region: us-east-1 (Active)
 
This region handles 100% of the production traffic under normal operating conditions.

 ■ Compute: ECS / Lambda runs at Full Scale to handle peak transaction loads.

 ■ Data Stores: Hosts the Aurora MySQL Writer, DynamoDB Primary Table, S3 Primary Bucket, and ElastiCache (Redis) Primary node.

🔵 3. DR Region: eu-west-1 (Passive / Warm Standby)
 
This region sits ready to take over instantly, but runs at a minimal footprint to optimize costs.

  ■ Compute (Warm Standby): Instead of running at full scale, we maintain a minimum of 1 ECS/Lambda instance always running. This keeps the environment "warm" (code deployed, connections pooled) so it can auto-scale to full capacity within seconds of a failover, without paying for idle compute.

  ■ Data Stores: Hosts the Aurora Global Database (Read Replica), DynamoDB Global Table (Replica), S3 (Cross-Region Replicated), and ElastiCache (Warm-up).

🔄 4. Data Replication Strategy (Zero Data Loss)

To achieve our strict SLA targets, every data layer utilizes AWS's native cross-region replication capabilities:

  ■ Amazon Aurora Global Database: Replicates transaction logs from the us-east-1 Writer to the eu-west-1 Read Replica with sub-second latency.

  ■ DynamoDB Global Tables: Provides multi-active, cross-region replication for our Idempotency Keys and transaction records.

 ■ S3 Cross-Region Replication (CRR): Automatically copies encrypted PDF receipts and audit logs to the DR bucket.

 ■ Redis Global Datastore: Replicates the in-memory cache (Rate limits, Fraud scores) to the DR region to ensure the Risk Engine functions immediately upon failover.


5. Recovery Metrics (SLA Targets)
   
This architecture is engineered to meet the strictest financial industry SLAs:

■ RPO (Recovery Point Objective): < 1 Second**

 • Definition: How much data might we lose in a disaster?
 
 • How we achieve it: Because Aurora Global DB and DynamoDB Global Tables replicate data continuously at the storage layer, our data loss window is less than one second.
 
■ RTO (Recovery Time Objective): < 2 Minutes

 • Definition: How long does it take to restore service?
 
 • How we achieve it: Route 53 DNS Failover takes less than 60 seconds to propagate. Combined with the Auto-Scaling of the "Warm" ECS/Lambda instances in the DR region, the system is fully operational in under 2 minutes..

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# FinOps & Cost Architecture (100K Users)

Building a secure financial system is only half the battle; running it profitably is the other half. For a user base of 100,000 active users, this architecture is engineered for extreme cost-efficiency, resulting in a total monthly infrastructure cost of just $2,155.

<img width="1536" height="1024" alt="AWS AI Payment FinOps Dashboard" src="https://github.com/user-attachments/assets/9fe15fd2-859a-4cac-a99d-c3190a452d3a" />

Figure X: FinOps Dashboard showing Unit Economics, Cost Distribution, and Optimization Strategies.


📊 Unit Economics (Monthly)

■ Total Monthly Cost: $2,155

■ Cost Per User: $0.021

■ Cost Per Transaction: $0.005

■ Industry Average Cost: $0.050 

■ SAVINGS: 58% CHEAPER than industry standards.


🥧 Cost Distribution

■ 44% Data Layer (~$945): Moderate cost. Aurora MySQL is the primary cost driver due to Multi-AZ provisioning.

■ 30% Network & Security (~$650): High cost. Driven by premium security features like the AWS Network Firewall.

■ 24% Compute Layer (~$510): Excellent cost efficiency. Serverless components scale to zero.

■ 2% Storage & Observability (~$50): Excellent. Very low cost for logs and receipts.

Detailed Cost Breakdown


1. Compute Layer (~$510 / 24%) - ✅ Excellent (Serverless scales to zero)

 • API Gateway: 3M requests/mo @ $3.50/M = $10

 • AWS Lambda: 6M invocations (512MB, 500ms avg) = $50

 • ECS Fargate: 4 Tasks (2 vCPU, 4GB RAM) running 24/7 across 2 AZs = $450

 

2. Data Layer (~$945 / 44%) - ⚠️ Moderate (Aurora is the biggest cost driver)
   
 • Aurora MySQL: 2x db.r6g.large (Multi-AZ) + 100GB Storage + I/O = $720

 • Amazon DynamoDB: 100GB Storage (On-Demand) for Idempotency/Keys = $75

 • ElastiCache Redis: 2x cache.t3.medium nodes (Multi-AZ) for Fraud/Cache = $150

 

3. Network & Security (~$650 / 30%) - 🚨 High (Premium security features)
   
 • Network Firewall: 2 Endpoints (1 per AZ) @ $0.395/hr = $576

 • NAT Gateways: 2 Gateways fixed fee ($32x2) + 10GB data processing = $65

 • AWS WAF & Shield: Web ACL + 3M requests = $8

 • Route 53: Hosted Zone + 3M DNS queries = $2

 

4. Storage & Observability (~$50 / 2%) - ✅ Excellent (Very low cost)
   
 • Amazon S3: 30GB (PDF Receipts + VPC Flow Logs) = $1

 • CloudWatch: 50GB Log ingestion + Alarms = $35

 • AWS X-Ray: 3M Traces for distributed tracing = $15

 

️** FinOps Optimization Plan

How to save 40% and reduce the monthly cost to ~$1,300/mo.

Quick Wins (Immediate Implementation)

1. Aurora Reserved Instances (1-Year RI): Commit to 1-year usage for the database to save ~30% on DB costs. (-$216/mo)

2. S3 Lifecycle Policies: Automatically move PDF receipts older than 90 days to Glacier Instant Retrieval. (-$10/mo)

3. CloudWatch Log Retention: Auto-expire VPC and Application logs after 30 days to stop paying for infinite storage. (-$15/mo)
   


⚙️ Architectural Shifts (Long-term Strategy)

4. Evaluate Network Firewall: If compliance allows, replace the $576/mo Network Firewall with VPC Endpoints for the Bank API. (-$500/mo)
   
5. Aurora Serverless v2: Switch to Serverless v2 to automatically scale down to 0.5 ACUs during off-peak hours.
   
6. DynamoDB Auto-Scaling: Ensure strict On-Demand mode is configured to avoid paying for idle provisioned capacity.

 # The 6 Key Engineering Decisions (The "Why" Behind the Architecture)
 
 
1. Zero-Trust AI Financial Isolation (Blast Radius Containment)
   
  • The Decision: We never give the AI Agent direct access to the main corporate bank account. Instead, we created an Isolated AI Wallet with strict daily spend limits.
 
  • The Problem Solved: AI models can hallucinate, be prompt-injected, or compromised. If an AI goes rogue, it shouldn't be able to drain the company's main funds.
 
  • The Impact: Limits the financial blast radius strictly to the allocated funds in the isolated wallet, ensuring absolute financial safety.
  
 
2. Asynchronous 3-Queue Processing Pattern (Zero Data Loss)
   
 • The Decision: We decoupled the core transaction from external API calls using three distinct SQS queues: Payment Q ➔ Update Q Receipt Q.

 • The Problem Solved: External Bank and Merchant APIs are slow, prone to timeouts, and can drop webhooks. If we processed these synchronously, a slow API would block the system, cause thread exhaustion, and lose transactions if a worker crashed.

 • The Impact: Guarantees Zero Data Loss. If a worker crashes, SQS's Visibility Timeout ensures the message returns to the queue for another worker. The intermediate "Update Queue" protects the core MySQL database from direct worker pressure.


3. Strict Financial Consistency (Idempotency & Row Locking)
   
 • The Decision: We implemented DynamoDB Idempotency Keys (with 7-day TTL) for network retries, and MySQL SELECT FOR UPDATE (Row-Level Locking) for concurrent requests.
 
 • The Problem Solved: In financial systems, network glitches cause duplicate requests. Without idempotency, a user gets double-charged. Without row locking, an AI could spend the same $50 twice simultaneously (a race condition).

 • The Impact: Guarantees ACID compliance and prevents double-charging or overspending, even under high-concurrency AI traffic.


4. Defense-in-Depth Network Routing (Local vs. NAT)
  
  • The Decision: Internal communication between the Compute and Data subnets uses the VPC Local Route (10.0.0.0/16 ➔ local), completely bypassing the NAT Gateway. The NAT Gateway is strictly reserved for outbound internet traffic to the Bank API.
  
  • The Problem Solved: Routing internal database traffic through a NAT Gateway is slow, insecure, and incurs unnecessary AWS data processing charges. Furthermore, the database must be completely air-gapped from the internet.
  
  • The Impact: Keeps internal traffic on the secure, free AWS backbone. The Private Data Subnet has NO Internet Route, making the database physically incapable of being breached from the outside.
  
  
5. Strict Outbound Data Exfiltration Prevention
   
  • The Decision: The SG_Compute Security Group outbound rule is restricted to TCP Port 443 ONLY to the specific IP address of the Third-Party Bank API.
  
  • The Problem Solved: If a hacker compromises a Lambda function or ECS container, they will try to "phone home" and exfiltrate database credentials to their own rogue server.
  
  • The Impact: Even if the compute layer is compromised, the Security Group drops the packet because the destination IP doesn't match the Bank's IP. Combined with the AWS Network Firewall, this creates an impenetrable outbound boundary.
  
  
6. Cost-Optimized Multi-Region Warm Standby (BCP)
   
 • The Decision: We chose a Warm Standby architecture (Active-Passive) over a Multi-Active setup, keeping only 1 minimum compute instance running in the DR region (eu-west-1), while relying on Aurora Global DB and DynamoDB Global Tables for data.
 
 • The Problem Solved: Running a fully active second region doubles the infrastructure cost. Running a "Pilot Light" (cold) region takes too long to boot up during a disaster.
 
 • The Impact: Achieves an RPO of < 1 Second (near-zero data loss) and an RTO of < 2 Minutes (rapid recovery), while keeping the DR infrastructure costs significantly lower than a fully active setup.


# License & Attribution

Designed and Architected by Faisal Ahmed Ome

Connect with me on Linkedin
