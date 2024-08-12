# LHV Integration

## LHV service startup

* LHV Service is the Bank Payouts service that connects to LHV

```mermaid
sequenceDiagram

participant ecs as ECS Fargate
participant sqs as SQS
participant lhvs as LHV Service
participant lhvscddb as LHV Service Configuration DynamoDB
participant lhvsddb as LHV Service DynamoDB
participant lhvapi as LHV Payment Initiation API
participant lhvmapi as LHV Payment Initiation API

ecs->>lhvs: Start ECS task
lhvs->>lhvs: Set readiness check to Not Ready (default)
lhvs->>lhvs: Set ReadyForProcessing to false (default)
sqs->>lhvs: Receive CheckoutApprovalAccepted
lhvs->>sqs: SQS Messages put back on queue without processing or retry count incremented
loop
    ecs->>+lhvs: API call to readiness endpoint
    lhvs->>lhvscddb: Check connectivity (read a row)
    lhvs->>lhvsddb: Check connectivity (read a row)
    lhvs->>lhvapi: Check heartbeat endpoint returns 2xx
    lhvs->>lhvapi: Send message heartbeat (and ensure returns 2xx)
    lhvs->>lhvmapi: Check heartbeat message received for Message-Request-Id
    lhvs->>sqs: Check connectivity
    lhvs->>-ecs: Readiness response (200 or 500)
    alt Ready
        lhvs->>lhvs: Set to Ready
        lhvs->>lhvs: Exit loop
    else Not ready
        ecs->>ecs: Wait
    end
end

loop
    alt Can lock shard
        lhvs->>lhvscddb: Attempt to lock shard row for 30s (set LockedAt and LockExpiry), <br> using DynamoDB Conditional Expression
        loop 
            lhvs->>lhvs: Wait 20s
            lhvs->>lhvscddb: Lock shard for 30s (set LockedAt and LockExpiry), <br> using DynamoDB Conditional Expression
            lhvs->>lhvs: Set ReadyForProcessing to true
        end
    else Row cannot be locked
        lhvs->>lhvs: Wait 10s
    end
end
```
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />

## Payment Initiation

* `CheckoutApprovalAccepted` events written to DynamoDB
* Background process batches these and sends to LHV

```mermaid
sequenceDiagram

participant pc as Payment Controls
participant sns as SNS
participant sqs as SQS
participant lhvs as LHV Service
participant lhvscddb as LHV Service Configuration DynamoDB
participant lhvsddb as LHV Service DynamoDB
participant lhvapi as LHV Payment Initiation API

pc->>sns: Send CheckoutApprovalAccepted
sns->>sqs: Forward message
sqs->>lhvs: Receive CheckoutApprovalAccepted
alt ReadyForProcessing is true
    lhvs->>lhvsddb: Save event in DynamoDB <br> - One row per event<br> - Store with locked shard PartitionKey only<br> - PartitionKey = ShardId<br> - SortKey = PaymentId
    lhvs->>sqs: Complete message
else ReadyForProcessing is false
    lhvs->>sqs: Put message back on the queue<br> without incrementing retry count
end

loop
    alt Payouts exist
        loop Initiate payouts in LHV
            lhvs->>+lhvsddb: Get up to 100 payouts which are not InitiatedSuccessfully or are Retryable
            lhvsddb->>-lhvs: Payouts
            lhvs->>lhvs: Batch payouts into a single Payment Initiation request
            lhvs->>lhvs: Create new X-Idempotency-Key (GUID) or use existing on one payout
            lhvs->>lhvsddb: Store X-Idempotency-Key on payouts in DynamoDB
            loop Retry for 429, 5xx or HTTP timeout (Up to 5s with exponential back-off)
                lhvs->>+lhvapi: Attempt batch Payment Initiation with X-Idempotency-Key <br>(same key for all retries)
                lhvapi->>-lhvs: Payment Initiation response (or timeout)
            end
            alt 202 ACCEPTED
                lhvs->>lhvsddb: - Store Message-Request-Id (so we can find the response message)<br>- Mark as InitiatedSuccessfully with timestamp<br>- Store Message-Request-Id
            else HTTP Timeout or 5xx
                lhvs->>lhvsddb: Log warning
            end
        end
    else No payouts
        lhvs->>lhvs: Wait 5s
    end
end
```
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />

## Handling responses

## Graceful shutdown

## Ungraceful shutdown
