This implements a fully event‑driven ingestion pipeline for the Analytics Platform. When a CSV file is uploaded to the dev-analytic S3 bucket, an EventBridge rule triggers a Lambda function, which parses the file and writes user records into DynamoDB.

This section explains the architectural decisions behind:

•  Why EventBridge instead of native S3 notifications
•  Why DynamoDB instead of RDS
•  The error‑handling strategy used in the ingestion Lambda

Why EventBridge Instead of S3 Notifications

EventBridge and S3 Notifications both support reacting to S3 object uploads, but EventBridge offers several advantages that align with enterprise multi‑account governance.

1. Centralized, Multi‑Account Event Routing

EventBridge can route events across accounts and organizational units.  
S3 notifications cannot — they only deliver to resources in the same account.

For a multi‑account landing zone, EventBridge is the scalable choice.

2. No Tight Coupling to Downstream Services

S3 notifications require direct wiring to Lambda, SQS, or SNS.  
EventBridge decouples producers and consumers, enabling:

•  Multiple consumers per event
•  Future pipelines without modifying S3
•  Cleaner separation of duties

3. Rich Event Filtering

EventBridge supports JSON‑based event patterns:

{
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["dev-analytic"] },
    "object": { "key": [{ "suffix": ".csv" }] }
  }
}

S3 notifications only support prefix/suffix filters.


4. Reliability and Retry Behavior

EventBridge provides:

•  Built‑in retries
•  Dead‑letter queues
•  Event archives and replays

S3 notifications do not support event replay and have more limited retry semantics.

5. Governance and Auditability

EventBridge integrates with:

•  CloudTrail
•  CloudWatch metrics
•  Organization‑wide event buses

This aligns with enterprise governance patterns.

Conclusion:  
EventBridge is the correct choice for a scalable, multi‑account, event‑driven ingestion architecture

Why DynamoDB Instead of RDS

The ingestion pipeline stores user records in DynamoDB. This decision is based on operational, cost, and scalability considerations.

1. Serverless, Fully Managed, Zero Maintenance

DynamoDB requires:

•  No patching
•  No backups configuration
•  No scaling management
•  No connection pooling

RDS requires ongoing operational overhead.

2. Millisecond Latency at Any Scale

DynamoDB is designed for:

•  High‑velocity ingestion
•  Low‑latency reads
•  Horizontal scaling

RDS scales vertically and becomes a bottleneck under ingestion spikes.

3. Pay‑Per‑Request Pricing

DynamoDB’s on‑demand mode is ideal for:

•  Spiky workloads
•  Low‑traffic development environments
•  Cost‑efficient pipelines

RDS requires continuous instance uptime, even when idle.

4. Perfect Fit for Simple, Key‑Value Access Patterns

Your ingestion pipeline stores users by user_id.  
This is a classic DynamoDB access pattern:

•  Simple primary key
•  No complex joins
•  No relational constraints

RDS is only necessary when you need:

•  Multi‑table joins
•  Complex relational queries
•  Strong transactional guarantees

5. Native Integration with Lambda

DynamoDB’s SDK integration is lightweight and connectionless.  
RDS requires connection pooling and VPC networking.

Conclusion:  
DynamoDB is the ideal choice for a serverless ingestion pipeline with simple, high‑throughput access patterns.

Error Handling Strategy

The ingestion Lambda uses a defensive, production‑ready error‑handling strategy.

1. Try/Catch Around the Entire Processing Flow

Ensures any unexpected failure is logged and surfaced.

2. Per‑Row Error Isolation

If one row in the CSV is malformed:

•  The Lambda logs the error
•  Continues processing the remaining rows
•  Prevents partial ingestion failures

3. Structured Logging

Logs include:

•  File name
•  Bucket name
•  Row number
•  Error message

This makes CloudWatch debugging straightforward.

4. Idempotency by Primary Key

If the same CSV is uploaded twice:

•  DynamoDB overwrites the same user_id
•  No duplicate records
•  No need for deduplication logic

5. EventBridge Retry Behavior

If the Lambda fails:

•  EventBridge automatically retries
•  You can add a DLQ for poison events

6. Validation of Required Fields

Before inserting into DynamoDB, the Lambda checks:

•  user_id exists
•  email is present
•  membership_type is valid

Invalid rows are skipped with a warning.
