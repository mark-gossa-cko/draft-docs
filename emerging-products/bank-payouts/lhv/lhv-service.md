# LHV Integration

## LHV service startup

* LHV Service is the Bank Payouts service that connects to LHV

```mermaid
sequenceDiagram

participant ecs as ECS Fargate
participant sqs as SQS
participant lhvb as LHV Bridge
participant lhvbcddb as LHV Bridge Configuration DynamoDB
participant lhvbddb as LHV Bridge DynamoDB
participant lhvapi as LHV Payment Initiation API
participant lhvmapi as LHV Messages API

ecs->>lhvb: Start ECS task
lhvb->>lhvb: Set readiness check to Not Ready (default)
lhvb->>lhvb: Set ReadyForProcessing to false (default)
sqs->>lhvb: Receive CheckoutApprovalAccepted
lhvb->>sqs: SQS Messages put back on queue without processing or retry count incremented
loop
    ecs->>+lhvb: API call to readiness endpoint
    lhvb->>lhvbcddb: Check connectivity (read a row)
    lhvb->>lhvbddb: Check connectivity (read a row)
    lhvb->>lhvapi: Check heartbeat endpoint returns 2xx
    lhvb->>lhvapi: Send message heartbeat (and ensure returns 2xx)
    lhvb->>lhvmapi: Check heartbeat message received for Message-Request-Id
    lhvb->>sqs: Check connectivity
    lhvb->>-ecs: Readiness response (200 or 500)
    alt Ready
        lhvb->>lhvb: Set to Ready
        lhvb->>lhvb: Exit loop
    else Not ready
        ecs->>ecs: Wait
    end
end

loop
    alt Can lock shard
        lhvb->>lhvbcddb: Attempt to lock shard row for 30s (set LockedAt and LockExpiry), <br> using DynamoDB Conditional Expression
        loop 
            lhvb->>lhvb: Wait 20s
            lhvb->>lhvbcddb: Lock shard for 30s (set LockedAt and LockExpiry), <br> using DynamoDB Conditional Expression
            lhvb->>lhvb: Set ReadyForProcessing to true
        end
    else Row cannot be locked
        lhvb->>lhvb: Wait 10s
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
participant lhvb as LHV Service
participant lhvbcddb as LHV Service Configuration DynamoDB
participant lhvbddb as LHV Service DynamoDB
participant lhvapi as LHV Payment Initiation API

pc->>sns: Send CheckoutApprovalAccepted
sns->>sqs: Forward message
sqs->>lhvb: Receive CheckoutApprovalAccepted
alt ReadyForProcessing is true
    lhvb->>lhvbddb: Save event in DynamoDB <br> - One row per event<br> - Store with locked shard PartitionKey only<br> - PartitionKey = ShardId<br> - SortKey = PaymentId
    lhvb->>sqs: Complete message
else ReadyForProcessing is false
    lhvb->>sqs: Put message back on the queue<br> without incrementing retry count
end

loop
    alt Payouts exist
        loop Initiate payouts in LHV
            lhvb->>+lhvbddb: Get up to 100 payouts which are not InitiatedSuccessfully or are Retryable
            lhvbddb->>-lhvb: Payouts
            lhvb->>lhvb: Batch payouts into a single Payment Initiation request
            lhvb->>lhvb: Create new X-Idempotency-Key (GUID) or use existing on one payout
            lhvb->>lhvbddb: Store X-Idempotency-Key on payouts in DynamoDB
            loop Retry for 429, 5xx or HTTP timeout (Up to 5s with exponential back-off)
                lhvb->>+lhvapi: Attempt batch Payment Initiation with X-Idempotency-Key <br>(same key for all retries)
                lhvapi->>-lhvb: Payment Initiation response (or timeout)
            end
            alt 202 ACCEPTED
                lhvb->>lhvbddb: - Store Message-Request-Id (so we can find the response message)<br>- Mark as InitiatedSuccessfully with timestamp<br>- Store Message-Request-Id
            else HTTP Timeout or 5xx
                lhvb->>lhvb: Log warning
            end
        end
    else No payouts
        lhvb->>lhvb: Wait 5s
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
