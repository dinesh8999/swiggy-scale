# Incident Runbook: 5-Step Response Guide for SwiftEats Outages

**Target Audience:** On-call engineers (any seniority level - designed for first-time responders)  
**Execution Time:** 30 minutes from alert to full mitigation (vs. 45+ minutes without runbook)  
**Last Updated:** May 2024

---

## Table of Contents

1. [STEP 1 - DETECT](#step-1--detect-alert-thresholds)
2. [STEP 2 - TRIAGE](#step-2--triage-identify-root-failure-30-seconds)
3. [STEP 3 - RESPOND](#step-3--respond-component-specific-actions)
4. [STEP 4 - ROLLBACK](#step-4--rollback-failure-criteria)
5. [STEP 5 - POSTMORTEM](#step-5--postmortem-incident-review)

---

## STEP 1 - DETECT: Alert Thresholds

This section lists every alert that should fire during a major incident. If you are reading this runbook, at least one of these thresholds has been breached. **Do not proceed based on gut feeling - check the alerts that actually fired.**

### Primary Alerts (Act Immediately - Page On-Call)

#### Alert 1: ALB 5xx Error Rate Spike

**Signal:**
```
Metric: ALB HTTPCode_Target_5XX
Condition: >5% of requests returning 5xx for 2 consecutive minutes
CloudWatch Location: EC2 → Load Balancers → Target Health → Unhealthy Host Count
```

**What This Means:**
- API servers are returning errors faster than healthy responses
- Backend services are degraded or unreachable
- This alert almost always fires during a major incident

**Action:** Go to STEP 2 - TRIAGE

---

#### Alert 2: PostgreSQL Connections Exhausted

**Signal:**
```
Metric: RDS DatabaseConnections
Condition: >85 of max_connections (85/100) for 1 minute
Alternative: RDS FailedSQLServerAgentJobsCount increases
CloudWatch Location: RDS → Databases → (database name) → Connections
```

**What This Means:**
- DB pool is becoming saturated
- New connection requests will start failing
- This is Failure 1 from FAILURE-CASCADE.md

**Action:** Go to STEP 2 - TRIAGE → Check "DB Connection Pool" branch

---

#### Alert 3: EC2 Instance CPU Sustained High

**Signal:**
```
Metric: EC2 CPUUtilization
Condition: >80% on ALL instances for 3 minutes
CloudWatch Location: EC2 → Instances → (select running instances)
```

**What This Means:**
- Compute is saturated
- Auto-scaling may be triggered but new instances take 90 seconds to start
- Existing instances cannot process more requests

**Action:** Go to STEP 2 - TRIAGE → Check "Compute Saturation" branch

---

#### Alert 4: Redis Cache Hit Rate Drops

**Signal:**
```
Metric: ElastiCache CacheMisses vs CacheHits
Condition: Miss rate > 50% (or hits < 50% of requests)
CloudWatch Location: ElastiCache → Clusters → (redis-cluster) → Metrics
```

**What This Means:**
- Cache is not returning expected results
- Either cache was flushed, or memory evicted all keys
- Requests now hitting database (multiplying DB load)

**Action:** Go to STEP 2 - TRIAGE → Check "Cache Miss Spike" branch

---

#### Alert 5: SQS Queue Depth Exceeds Threshold

**Signal:**
```
Metric: SQS ApproximateNumberOfMessages
Condition: >50,000 messages in payment queue for 2 minutes
CloudWatch Location: SQS → Queues → payment-queue → ApproximateNumberOfMessages
```

**What This Means:**
- Payment processing is backlogged
- Orders are being placed but payments are not processing
- Either payment worker is down or payment gateway is rate-limiting

**Action:** Go to STEP 2 - TRIAGE → Check "Payment Queue Backup" branch

---

#### Alert 6: RDS Replication Lag

**Signal:**
```
Metric: RDS ReplicaLag
Condition: >10 seconds (warning at 30 seconds = critical)
CloudWatch Location: RDS → Databases → (read replica) → Replication lag
```

**What This Means:**
- Read replica is not keeping up with primary
- Queries to replica may return stale data
- Primary is under extreme write load

**Action:** Go to STEP 2 - TRIAGE → Check "DB Connection Pool" branch (primary is overloaded)

---

#### Alert 7: P99 Response Time Spike

**Signal:**
```
Metric: ALB TargetResponseTime (p99 percentile)
Condition: >2000ms (2 seconds) for 2 minutes
CloudWatch Location: ALB → (target group) → Monitoring → Target response time
```

**What This Means:**
- Users are experiencing timeouts and slow responses
- Not all requests are failing (would be 5xx)
- Something is queued or processing slowly

**Action:** Go to STEP 2 - TRIAGE (all symptoms can cause this)

---

### Secondary Alerts (Monitor, Escalate if Worsening)

#### Alert 8: Node.js Process Memory Usage High

**Signal:**
```
Metric: EC2 Memory Utilization via CloudWatch Agent
Condition: >85% on any instance
CloudWatch Location: Custom CloudWatch Metrics → Memory (requires CloudWatch agent)
```

**What This Means:**
- Instance is close to OOM crash
- If 100% is reached → process exits immediately

**Action:** Add more instances or restart the affected instance

---

#### Alert 9: CloudFront Error Rate

**Signal:**
```
Metric: CloudFront 4xx errors, 5xx errors
Condition: >1% of requests returning 4xx or 5xx
CloudWatch Location: CloudFront → Distributions → (distribution) → CloudWatch metrics
```

**What This Means:**
- CDN is having issues or origin (ALB) is returning errors
- Static assets may not be loading
- Likely cascading from ALB errors

**Action:** Usually resolves when ALB is fixed. See STEP 2.

---

#### Alert 10: Network Interface Saturation

**Signal:**
```
Metric: EC2 NetworkPacketsOut, NetworkBytesOut
Condition: >4.5 Gbps sustained (server limit is 5 Gbps on t3 instances)
CloudWatch Location: EC2 → Instances → (select instance) → Monitoring → Network
```

**What This Means:**
- Server is transmitting near maximum bandwidth
- Usually indicates image/static asset surge or DDoS
- New connections will timeout

**Action:** Check if CDN is properly configured. See ARCHITECTURE.md.

---

## STEP 2 - TRIAGE: Identify Root Failure (30 Seconds)

You have alerts. Now you need to determine which component is the root cause. **This is the most critical decision.** Wrong diagnosis = wrong fix = extended outage.

### Triage Decision Tree

**START HERE:** Check these dashboards in THIS ORDER. The first one showing red/orange is usually your root cause.

```
              INCIDENT DETECTED
                    │
                    ▼
    ┌───────────────────────────────────┐
    │  CHECK 1: RDS DatabaseConnections │
    │  (CloudWatch → RDS → Connections) │
    └───────────────────────────────────┘
         │               │
      RED/HIGH        NORMAL
        │               │
    ┌─────────┐      │
    │FAILURE 1│      ▼
    │DB Pool  │  ┌───────────────────────────────────┐
    │Exhausted   │  CHECK 2: EC2 CPUUtilization      │
    └─────────┘  │  (ALL instances at >80%?)         │
         │       └───────────────────────────────────┘
    GOTO 3a            │                  │
                    HIGH            NORMAL
                      │                  │
                  ┌─────────┐        ▼
                  │FAILURE 2│    ┌──────────────────────┐
                  │Compute  │    │CHECK 3: Redis Misses │
                  │Saturation   │(ElastiCache metrics) │
                  └─────────┘    └──────────────────────┘
                       │             │           │
                  GOTO 3b        HIGH/SPIKE   NORMAL
                                    │           │
                              ┌──────────┐    ▼
                              │FAILURE 4 │  ┌──────────────────┐
                              │Cache     │  │CHECK 4: SQS Queue│
                              │Invalidated  │(msg count > 50K?)│
                              └──────────┘  └──────────────────┘
                                   │            │           │
                              GOTO 3c      BACKLOG      NORMAL
                                           │               │
                                      ┌──────────┐       ▼
                                      │FAILURE 3 │  ┌─────────────────┐
                                      │Payment   │  │CHECK 5: Network │
                                      │Queue Down    │(bytes out >4.5G)│
                                      └──────────┘  └─────────────────┘
                                           │            │           │
                                      GOTO 3d       SATURATED   NORMAL
                                                       │
                                                  ┌──────────┐
                                                  │FAILURE 5 │
                                                  │NIC Satur.│
                                                  └──────────┘
                                                       │
                                                  GOTO 3e

                                    NOT A DB/COMPUTE/CACHE/QUEUE/NETWORK ISSUE?
                                    │
                                    ▼
                                ┌─────────────────────────────────────┐
                                │ ANOMALY: Check if external dependency│
                                │ - Payment gateway (Razorpay) down?  │
                                │ - CDN (CloudFront) returning errors? │
                                │ - Database upgrade happening?       │
                                │ - Recent deploy in last 2 hours?    │
                                └─────────────────────────────────────┘
```

### Triage Output

After completing the decision tree, you should have identified ONE of these:

- **FAILURE 1:** DB Connection Pool Exhausted → GOTO STEP 3a
- **FAILURE 2:** Compute Saturation (EC2 CPU) → GOTO STEP 3b
- **FAILURE 3:** Payment Queue Backup → GOTO STEP 3c (payment worker down)
- **FAILURE 4:** Cache Miss Spike → GOTO STEP 3d
- **FAILURE 5:** Network Interface Saturation → GOTO STEP 3e
- **External Dependency:** Payment gateway, CDN, or recent deploy → GOTO STEP 3f
- **Unknown:** No clear root cause visible → Escalate to @platform-team

---

## STEP 3 - RESPOND: Component-Specific Actions

Once you know the root cause, follow the exact steps for that failure type. **Read each command before executing it.** If anything seems wrong, ask in #oncall-support Slack channel before proceeding.

---

### 3a: PostgreSQL Connection Pool Exhausted

**Owner:** @dba-oncall  
**Severity:** CRITICAL  
**Expected MTTR:** 5-10 minutes

#### 3a-1: Confirm the Pool is Actually Exhausted

**In AWS Console:**

```
1. Open RDS → Databases → swiftEats-prod-primary
2. Look at "DB Connections" graph → should show ~95-100/100
3. Note the timestamp - connection exhaustion started when?
   (Use this for postmortem root cause analysis)
```

**Via CLI (if you have RDS access):**

```bash
aws rds describe-db-instances \
  --db-instance-identifier swiftEats-prod-primary \
  --region ap-south-1 | grep DBInstanceStatus

# Should show "available" (not "modifying", "rebooting", etc.)
```

#### 3a-2: Identify Stale Connections Holding the Pool

**Connect to the primary database** (requires VPN + DB password):

```sql
-- SSH to the bastion host first
ssh -i your-key.pem ec2-user@bastion.internal.swiftEats.com

-- From bastion, connect to PostgreSQL
psql -U postgres -h swiftEats-primary.internal -d swiftEats_prod

-- List all active connections and their state
SELECT 
  pid,
  usename,
  state,
  query,
  query_start,
  NOW() - query_start AS duration_seconds,
  client_addr
FROM pg_stat_activity
ORDER BY query_start ASC;

-- Count by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```

**What to look for:**
```
state = 'idle in transaction' → Connections stuck in a transaction, not releasing
state = 'active' → Currently executing queries
state = 'idle' → Idle, safe to close if query_start is very old

Example output:
 state           | count
─────────────────┼───────
 idle            |    30
 idle in transaction |   50  ← PROBLEM: These should be released
 active          |    18
 (null)          |     2
```

#### 3a-3: Terminate Stale Idle-In-Transaction Connections

**CAUTION:** Terminating a connection rolls back its transaction. Only do this for connections that have been idle for >30 seconds.

```sql
-- List idle-in-transaction connections older than 30 seconds
SELECT 
  pid,
  usename,
  state,
  query_start,
  NOW() - query_start AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < NOW() - INTERVAL '30 seconds'
ORDER BY query_start ASC;

-- If you see results like "UPDATE orders SET status = ..." from 5 minutes ago
-- These are safe to terminate. Kill them:

SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < NOW() - INTERVAL '30 seconds';

-- Check how many were killed
SELECT count(*) FROM pg_stat_activity;
-- Result should show fewer than before
```

#### 3a-4: Check if Pool is Now Available

```sql
SELECT count(*) as connection_count FROM pg_stat_activity;
-- Should return something like 45-60, NOT 95-100

-- If still high, check for long-running queries
SELECT 
  pid,
  usename,
  state,
  query,
  NOW() - query_start AS duration_seconds
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start ASC;

-- If a query has been running for >60 seconds, it might be stuck
-- Ask before terminating active queries (they might be legitimate slow queries)
```

#### 3a-5: Monitor Until Stable

**In AWS Console:**
```
1. Go back to RDS → Databases → DB Connections graph
2. Watch for 2 minutes
3. Should see connections drop from 100 to 40-50 range
4. Then stabilize or continue dropping as requests complete
```

**Success Criteria:**
- DB Connections: <80/100
- ALB 5xx error rate: Drops from 50%+ to <5% within 2 minutes
- P99 response time: Returns to <1000ms

**If not improving after 3 minutes:**
- Go to STEP 3b (Compute Saturation) - DB might be exhausted due to compute overload
- Or go to STEP 3c (Payment Queue) - Razorpay might be timing out, holding connections

#### 3a-6: Root Cause Follow-Up (Postmortem)

**Questions to answer:**
- Why did connections exhaust? (Connection leak, payment timeout, slow queries?)
- How long did it take to recover? (Measure from alert to <80% threshold)
- Did PgBouncer work as expected? (Check PgBouncer logs)

---

### 3b: Node.js Compute Saturation

**Owner:** @app-oncall  
**Severity:** CRITICAL  
**Expected MTTR:** 2-5 minutes

#### 3b-1: Confirm Compute is the Bottleneck

**In AWS Console:**
```
1. EC2 → Instances → (select all API instances)
2. Look at CPU Utilization graph
3. Are ALL instances at >80% CPU simultaneously?
   - YES → Compute is saturated, proceed to 3b-2
   - NO (some instances <70%) → Problem is elsewhere, go back to STEP 2
```

#### 3b-2: Trigger Auto-Scaling Manually (Faster Than Waiting)

**Option 1: Via AWS Console (Slowest, ~2 minutes):**

```
1. EC2 → Auto Scaling Groups → swiggy-app-asg
2. Under "Details" tab, find "Desired capacity"
3. Click "Edit"
4. Change from current value (e.g., 4) to +4 (e.g., 8)
5. Click "Update"
6. Wait 90 seconds for instances to launch
```

**Option 2: Via AWS CLI (Faster, ~30 seconds to execute):**

```bash
# Get current desired capacity
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiggy-app-asg \
  --region ap-south-1 \
  --query 'AutoScalingGroups[0].DesiredCapacity'

# Output: 4 (example)

# Increase capacity by 4
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swiggy-app-asg \
  --desired-capacity 8 \
  --region ap-south-1

# Verify change
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiggy-app-asg \
  --region ap-south-1 \
  --query 'AutoScalingGroups[0].DesiredCapacity'

# Output: 8 (confirmed)
```

#### 3b-3: Monitor New Instances Joining the Pool

**Watch the ALB target health:**

```
1. Go to EC2 → Load Balancers → swiggy-alb
2. Click "Target Groups" tab
3. Select the API target group
4. Under "Targets", you should see:
   - Healthy targets: should increase from 4 to 8 within 90 seconds
   - New targets: will show "Draining" temporarily, then "Healthy"
```

**Watch the ALB metrics:**

```
1. Stay on ALB page
2. Click "Monitoring" tab
3. Look at "HTTPCode_Target_5XX" graph
4. Should start decreasing within 2 minutes of new instances coming online
5. Should drop below 5% within 4 minutes
```

**Check CloudWatch dashboard:**
```
1. CloudWatch → Dashboards → swiggy-metrics
2. Look at "EC2 CPU Utilization" graph
3. Each new instance should show up as a line starting from 0%
4. CPU load should distribute evenly across all instances
5. No instance should be >70% after new instances join
```

#### 3b-4: Decide if Further Scaling is Needed

```
At T+4 minutes (new instances online):
  - Is ALB 5xx error rate still >10%?
    → YES: Add more instances (repeat 3b-2)
    → NO: Scaling successful, STOP

  - Is CPU on all instances still >80%?
    → YES: Requests are too heavy for this instance type. Escalate to @platform-team
    → NO: Normal, continue monitoring

  - Is P99 response time still >2000ms?
    → YES: Database is bottleneck, go to STEP 3a
    → NO: Compute scaling fixed it!
```

#### 3b-5: Success Criteria

- ALB 5xx rate: <5%
- EC2 CPU: <70% on all instances
- P99 latency: <1500ms
- ALB request rate: Evenly distributed across all instances (check target response times in ALB metrics)

#### 3b-6: Document for Postmortem

- What was the peak CPU reached? (measure from AWS console)
- How many instances did you scale to?
- How long from alert to recovery?
- Did ASG auto-scale trigger correctly or was manual scaling necessary?

---

### 3c: Payment Queue Backup (SQS Depth > 50K)

**Owner:** @payments-oncall  
**Severity:** HIGH  
**Expected MTTR:** 10-15 minutes

#### 3c-1: Check the Queue Status

**In AWS Console:**

```
1. SQS → Queues → payment-queue
2. Look at "Attributes" tab → ApproximateNumberOfMessages
3. If >50,000: Queue is backing up
4. Look at "Dead Letter Queue" tab → same check
```

#### 3c-2: Determine If Payment Worker is Dead

**Check ECS Task Status:**

```
1. ECS → Clusters → swiggy-workers
2. Click on "Services" → payment-worker-service
3. Look at "Tasks" section
4. Count "Running" tasks (should be ≥5, up to 50 at peak)
5. Count "Stopped" or "Failed" tasks (should be 0)

If Running tasks = 0 or <2:
  → Payment worker crashed, go to 3c-3
If Running tasks ≥5:
  → Worker is running but can't keep up, go to 3c-4
```

#### 3c-3: Restart the Payment Worker Service

**Via ECS Console:**

```
1. ECS → Clusters → swiggy-workers
2. Services → payment-worker-service
3. Click "Update service"
4. Check "Force new deployment" checkbox
5. Click "Update service"
6. Wait 2-3 minutes for new tasks to spin up
7. Verify "Running tasks" count increases
```

**Via AWS CLI:**

```bash
# Force new deployment
aws ecs update-service \
  --cluster swiggy-workers \
  --service payment-worker-service \
  --force-new-deployment \
  --region ap-south-1

# Monitor
aws ecs describe-services \
  --cluster swiggy-workers \
  --services payment-worker-service \
  --region ap-south-1 \
  --query 'services[0].[runningCount, pendingCount, desiredCount]'

# Expected output: [5, 2, 7] (5 running, 2 starting, 7 desired)
```

#### 3c-4: Check for External Dependency Issues

**If worker restarted but queue is still backing up:**

```sql
-- Connect to payment worker logs via CloudWatch
1. CloudWatch → Log Groups → /ecs/payment-worker
2. Look for error patterns in last 5 minutes:
   - "Razorpay 429" = Payment gateway rate-limiting us → Wait and retry
   - "Razorpay 5xx" = Payment gateway is down → Contact Razorpay support
   - "Connection timeout" = Network issue → Check VPC/security groups
   - "Invalid API key" = Configuration issue → Escalate to @platform-team
```

**Via CLI (faster):**

```bash
# Get last 100 lines of logs
aws logs tail /ecs/payment-worker --follow --since 5m
```

**Look for patterns like:**
```
ERROR: 429 Rate Limit Exceeded
ERROR: {"error": "Invalid API key"}
ERROR: ECONNREFUSED 10.0.0.1:443
```

#### 3c-5: Increase Payment Worker Capacity if Needed

If worker is running but queue is still >50K after 2 minutes:

```bash
# Increase desired task count
aws ecs update-service \
  --cluster swiggy-workers \
  --service payment-worker-service \
  --desired-count 20 \
  --region ap-south-1

# Wait 60 seconds, then check queue depth
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-south-1.amazonaws.com/YOUR_ACCOUNT_ID/payment-queue \
  --attribute-names ApproximateNumberOfMessages \
  --region ap-south-1

# Output: Messages in queue should start decreasing
```

#### 3c-6: If Payment Gateway is Down

**Do NOT disable payments** - instead, pause order placement:

```bash
# Add ALB rule to block POST /api/v1/orders
# In ALB console:
# 1. EC2 → Load Balancers → swiggy-alb
# 2. Listeners → HTTPS rule (port 443)
# 3. Add new rule: IF path = /api/v1/orders THEN return 503 Service Unavailable
# 4. Priority: set as highest (priority = 1)

# Or via CLI (more complex), we prefer console for this
```

**Notify users:**

```
In-app banner text:
"Payment gateway is temporarily unavailable. 
Order placement is paused. We're working to restore it.
Check back in 10 minutes."
```

#### 3c-7: Success Criteria

- Queue depth: <10,000 messages
- Payment worker running tasks: ≥5
- Error rate in payment logs: <1%
- Average SQS message processing time: <5 seconds

---

### 3d: Redis Cache Miss Spike

**Owner:** @app-oncall  
**Severity:** MEDIUM (becomes CRITICAL if paired with DB exhaustion)  
**Expected MTTR:** 5-15 minutes

#### 3d-1: Confirm Cache Issue

**In AWS Console:**

```
1. ElastiCache → Clusters → swiggy-redis
2. Click on cluster name
3. Look at "Cache Performance" graph
4. Check "Cache Misses" vs "Cache Hits" in last 5 minutes
5. Misses should be <5% normally, >50% = problem
```

#### 3d-2: Check What Caused the Flush

**Possibility 1: Someone ran FLUSHALL**

```
This happens when:
  - A recent deploy included a cache reset
  - Manual cache flush was triggered
  - A DBA ran maintenance commands

Check:
  1. Ask in #deployments Slack: "Was there a recent deploy?"
  2. If YES: This is expected, cache is rebuilding (will recover in 10-15 min)
  3. If NO: Proceed to 3d-3
```

**Possibility 2: Redis ran out of memory**

```sql
-- Check Redis memory usage
1. ElastiCache → Clusters → swiggy-redis
2. Look at "Evictions" metric in last hour
3. If Evictions > 0:
   → Redis deleted old keys to make room for new ones
   → This causes cache misses on keys that were evicted
```

#### 3d-3: Check for Evictions

**If cache memory is full:**

```
1. ElastiCache → Clusters → swiggy-redis → Monitoring
2. Look at "DatabaseMemoryUsagePercentage" graph
3. If >90%: Redis is full, evicting old keys
```

**Scale up the cache cluster:**

```bash
# Increase node count (adds replica, more memory)
aws elasticache create-cache-cluster \
  --cache-cluster-id swiggy-redis-new-node \
  --cache-node-type cache.r7g.large \
  --engine redis \
  --region ap-south-1

# Or in console:
# ElastiCache → Clusters → swiggy-redis
# Modify → Increase "Number of cache nodes" from 3 to 4
# This takes ~10 minutes, zero downtime with cluster mode enabled
```

#### 3d-4: Manual Cache Rebuild (Fast Route)

**If you need to rebuild cache immediately without waiting:**

```
This should only be done if:
  1. Recent deploy included code that accidentally flushed cache
  2. Cache invalidation is correct (no stale data will result)

Rebuild:
  1. Redeploy application with @app-oncall approval
  2. Application warups cache on startup (usually automatic)
  3. Check Redis CacheHits metric → should spike as cache re-populates
```

#### 3d-5: Monitor Cache Recovery

```
1. ElastiCache → Clusters → swiggy-redis → Monitoring
2. Watch "CacheHits" and "CacheMisses" over next 10 minutes
3. Misses should decrease as cache fills
4. Hits should increase

If after 15 minutes, hits are still <50%:
  → Possible code issue with cache key generation
  → Escalate to @app-oncall or @platform-team
```

#### 3d-6: Success Criteria

- Cache hit rate: >90%
- Cache misses: <10% of requests
- Evictions metric: 0 (new keys aren't being evicted)
- ALB 5xx rate: Returns to <5% (was elevated due to more DB queries)
- DB connections: Return to normal range (less load on DB)

---

### 3e: Network Interface Saturation (NIC >4.5 Gbps)

**Owner:** @infrastructure-oncall  
**Severity:** HIGH  
**Expected MTTR:** 5-10 minutes

#### 3e-1: Verify NIC is the Bottleneck

**In AWS Console:**

```
1. EC2 → Instances → select all API instances
2. Monitoring tab → check "Network" graphs
3. Look for NetworkBytesOut graph
4. If any instance shows >4.5 Gbps sustained: NIC is saturated
```

#### 3e-2: Determine What's Using the Bandwidth

**Common culprits:**

```
1. Images being served by API instead of CDN
   → Check if CloudFront is configured correctly
   → Look for image requests in ALB access logs

2. Database replicas receiving heavy write traffic
   → Check RDS replication lag
   → If >10 seconds: primary is overloaded

3. API returning unusually large responses
   → Check if JSON responses are bloated
   → Rare, but happens if nested queries aren't optimized
```

**Check ALB access logs:**

```
1. S3 → Bucket: alb-logs-swiggy
2. Look for 200-size column (response body size)
3. If average >100KB: Responses are large
4. If most requests are to /images/*: CDN configuration issue
```

#### 3e-3: Fix: Ensure CloudFront Cache Headers are Set

**Check API response headers:**

```bash
# Test a menu request
curl -I https://api.swigteats.com/api/v1/restaurants/123/menu

# Look for:
Cache-Control: public, max-age=300
  ↑ This tells CloudFront to cache for 5 minutes

# If missing, API servers are sending uncached responses
```

**If Cache-Control headers are missing:**

```
1. This is an application code issue
2. Redeploy API with correct cache headers (consult @app-oncall)
3. Once deployed, CloudFront will respect TTL
4. NIC usage should drop to 10-15% normal levels
```

#### 3e-4: Temporary Mitigation: Compress Large Responses

If deployment takes time, enable gzip immediately:

```
1. ALB → Listeners → HTTPS (port 443)
2. Edit listener rules
3. Add response header: Content-Encoding: gzip
4. ALB will compress responses on the fly (reduces by 80-90%)
5. Takes effect immediately
```

#### 3e-5: Success Criteria

- NetworkBytesOut on all instances: <2 Gbps (well below 5 Gbps limit)
- No more instance timeout errors
- ALB 5xx rate: <5%
- CloudFront origin requests: minimal (>90% hits at edge)

---

### 3f: External Dependency Failure

**Owner:** Varies (payment gateway, AWS, CDN provider)  
**Severity:** CRITICAL  
**Expected MTTR:** Depends on external vendor

#### 3f-1: Identify Which External Dependency

**Check these in order:**

```
1. Payment Gateway (Razorpay / PayU) Status
   → Check https://status.razorpay.com
   → Look for any ongoing incidents
   → If RED: Their service is down, not your problem
   
2. AWS Status Page
   → Check https://status.aws.amazon.com
   → Look for ap-south-1 (Mumbai) region
   → If RED: AWS infrastructure issue
   
3. CloudFront / CDN
   → Check CloudFront metrics for high error rate
   → Look for 5xx from origin errors
   
4. Recent Deploy?
   → Check Slack #deployments channel
   → "Any deploys in the last 30 minutes?"
   → If YES and incident started right after: Likely deploy-related bug
```

#### 3f-2: Payment Gateway Down

**If Razorpay is experiencing an outage:**

```
Action:
  1. Pause order placement via ALB rule (set 503 response to POST /orders)
  2. Notify users via in-app banner: "Payment processing temporarily unavailable"
  3. Check Razorpay status page for ETA
  4. Contact @razorpay-account-manager for priority support (if critical)
  5. Do NOT disable payments in database (order records might be corrupt)
  6. Re-enable orders only after confirmed Razorpay recovery

Expected wait: 5-30 minutes typically for payment gateway outages
```

#### 3f-3: AWS Service Failure

**If AWS service is down (rare, but happens):**

```
Example: RDS becomes unreachable
  
Action:
  1. Check https://status.aws.amazon.com for official status
  2. Do NOT attempt manual fixes (AWS is fixing on their end)
  3. Set ALB to return 503 Service Unavailable
  4. Notify users via banner
  5. Wait for AWS to recover (typically 5-60 minutes)
  6. Once recovered, system should auto-heal

This is one of the rare times you CANNOT fix the outage.
Document it for the postmortem.
```

#### 3f-4: Recent Deploy Causes

**If incident started within 2 minutes of a deploy:**

```
Action:
  1. Check what was deployed: Slack #deployments
     "What changed 2 minutes ago?"
  2. If clear culprit: Decide to rollback (STEP 4)
  3. If unclear: Check application error logs for new errors
  
Rollback command (if deploy caused it):
  See STEP 4 below
```

---

## STEP 4 - ROLLBACK: Failure Criteria and Rollback Procedure

### When to Rollback

Rollback is appropriate if ALL of these conditions are true:

```
1. 5xx error rate has been >20% for >5 minutes AND not improving
2. No active deploy in the last 120 minutes (not a new feature deploy)
3. Root cause is clearly an application issue (not database, not AWS infra)
4. Business impact is critical (users cannot place orders)
```

### When NOT to Rollback

DO NOT rollback if:

```
1. Problem is clearly external (payment gateway down, AWS outage)
2. Root cause is a database schema issue
   → Rolling back code won't fix schema corruption
3. Data migration is in-flight
   → Rolling back mid-migration causes data inconsistency
4. Scaling/cache fixes are working (give them 5 more minutes)
```

### Rollback Procedure

#### Step 1: Verify Rollback Criteria

```
Ask yourself:
  - Is the ALB 5xx rate >20% for 5+ minutes? YES/NO
  - Did we deploy in the last 2 hours? YES/NO
  - Do we know what in the deploy caused this? YES/NO
  
If any answer is NO, do NOT rollback.
Go back to STEP 3 and try different mitigations.
```

#### Step 2: Get Approval

**Before rolling back, ask in Slack:**

```
@app-oncall @platform-oncall: We're rolling back to previous build.
Reason: Deployed 5 min ago, caused 85% 5xx error rate, scaling fixes didn't work.
Rollback target: Image version PREVIOUS_BUILD_HASH
Estimated downtime: 2-3 minutes (brief service disruption during restart)

Any objections? (30 second pause)
```

#### Step 3: Identify Current vs Previous Build

**Get current version:**

```bash
aws ecs describe-services \
  --cluster swiggy-prod \
  --services api \
  --region ap-south-1 \
  --query 'services[0].taskDefinition'

# Output: arn:aws:ecs:ap-south-1:123456:task-definition/swiggy-api:47
# Current version is 47
```

**Get previous version:**

```bash
aws ecs describe-task-definition \
  --task-definition swiggy-api:46 \
  --region ap-south-1 \
  --query 'taskDefinition.[containerDefinitions[0].image, revision]'

# This shows the image from build 46
```

#### Step 4: Perform the Rollback

```bash
# Rollback to previous task definition version
aws ecs update-service \
  --cluster swiggy-prod \
  --service api \
  --task-definition swiggy-api:46 \
  --region ap-south-1

# Monitor the rollout
aws ecs describe-services \
  --cluster swiggy-prod \
  --services api \
  --region ap-south-1 \
  --query 'services[0].[runningCount, pendingCount, taskDefinition]'

# Example output:
# [4, 2, "arn:aws:ecs:ap-south-1:123456:task-definition/swiggy-api:46"]
# Means: 4 running, 2 starting/stopping, now on version 46
```

#### Step 5: Monitor Recovery

```
1. CloudWatch → ALB metrics
2. Watch HTTPCode_Target_5XX graph
3. Should drop from 80%+ back to <5% within 3-5 minutes
4. Once <5% for 2 minutes: Rollback successful

5. Check P99 latency → should return to normal (<1000ms)

6. Alert @on-call: "Rollback complete. System recovered."
```

### ⚠️ CRITICAL: Never Rollback Database Schema

```
NEVER run:
  aws rds stop-db-instance ...
  
Or attempt to downgrade PostgreSQL version.

If the issue is a database schema migration:
  - Database CANNOT be rolled back with code
  - Reverting code while new schema exists = application crashes
  - Solution: Roll forward (fix the code or migration)
  - Get @dba-oncall immediately
  
Database rollbacks require manual intervention and data recovery.
This is NOT a 3-minute fix. It's a 2-hour incident.
```

---

## STEP 5 - POSTMORTEM: Incident Review

Postmortem must be completed within 24 hours of incident resolution. It serves three purposes:

1. **Historical Record:** Future on-call engineers reference this to avoid the same mistake
2. **Root Cause Analysis:** Understanding the why, not just the what
3. **Action Items:** Prevent recurrence by fixing underlying issues

### Postmortem Template

**Copy this template and fill it out as you write.**

```markdown
# POSTMORTEM: [Incident Name]

## Executive Summary
[1-2 sentences about what happened and impact]

Example: "On May 10, 2024 at 8:05 PM IST, the payment processing system backed up 
due to a recent deployment that introduced a 5-second timeout in the order validation 
query. Customers couldn't place orders for 45 minutes, resulting in ~₹189 crore 
estimated revenue loss."

## Timeline (Include UTC timestamps)

Start by listing WHEN things happened, not WHY:

- **T+0s:** Event occurred (notification sent, traffic spike arrived)
- **T+X min:** First alert fired (exact time)
- **T+Y min:** On-call engineer paged
- **T+Z min:** Root cause identified
- **T+W min:** Mitigation applied
- **T+V min:** System fully recovered

Example:
- **T+0s:** 8:05 PM IST - Notification sent to 180M users
- **T+3s:** 8:05:03 PM - ALB 5xx error rate alert fires
- **T+4min:** 8:09 PM - Engineer wakes up, checks dashboard
- **T+7min:** 8:12 PM - Identified recent deploy at 8:00 PM
- **T+8min:** 8:13 PM - Initiated rollback
- **T+12min:** 8:17 PM - New version deployed
- **T+15min:** 8:20 PM - 5xx rate drops below 5%, orders resumed

## Root Cause

**The single deepest technical reason the system failed (not just a symptom):**

Example: "The order validation query changed from:
  SELECT * FROM promo_budget WHERE code = 'WORLD50'
to:
  SELECT * FROM promo_budget 
    JOIN users ON users.id = promo_budget.user_id 
    WHERE code = 'WORLD50' AND users.country = 'IN'

This join was not indexed, causing a full table scan on the users table.
Combined with 500K RPS, this query consumed 5+ seconds per execution,
exhausting the DB connection pool immediately."

NOT: "Database was slow" (too vague)
NOT: "Traffic spike" (that's the scenario, not the root cause)
NOT: "Insufficient capacity" (symptom, not root cause)
```

### Postmortem Sections

#### 1. Impact Assessment

```
Duration:             45 minutes (T+0 to T+45min)
Affected Systems:     Order placement, Payment processing
Users Impacted:       ~10,000,000 (attempted to place orders)
Orders Lost:          ~6,000,000 (estimated)
Revenue Lost:         ₹189 crore (at ₹4.2 crore/minute)
Chargeback Cases:     ~150,000 (estimated)
App Store Rating:     4.3 → 1.8 stars (recovered to 4.2 after 3 months)
Mean Time To Detect:  4 minutes (good)
Mean Time To Resolve: 15 minutes (acceptable, target <10)
```

#### 2. What Went Well (Positive Findings)

```
- Alerting fired correctly (ALB 5xx > 5% for 2 min threshold)
- Engineer was paged immediately
- ECS auto-scaling worked
  (When we added more containers, payments processed faster)
- Rollback procedure was simple and fast (3 minutes)
- Blameless postmortem culture meant engineers were honest about mistakes
```

#### 3. What Didn't Go Well (Action Items)

```
For each problem, assign owner and due date:

ITEM 1: Index Missing on promo_budget.user_id
  Problem: Recent deploy added JOIN on users table without index
           This isn't normally caught in dev/staging
  Owner: @db-team
  Action: Add index
           CREATE INDEX idx_promo_budget_user_id ON promo_budget(user_id);
  Due: May 12, 2024 (2 days)
  
ITEM 2: Deploy testing didn't catch slow query
  Problem: Query was slow (5s), but prod traffic is 41× staging traffic
           This is why it broke in prod and not staging
  Owner: @qa-team / @app-team
  Action: Load test deployments against realistic traffic volume (50K RPS)
          Use Artillery or Locust to simulate
  Due: May 15, 2024 (5 days)

ITEM 3: No Slow Query Alert
  Problem: We only alerted on error rate and CPU
           We didn't have an alert for p95 latency > 2 seconds
  Owner: @monitoring-team
  Action: Add CloudWatch alarm: p95 latency > 2000ms for 2 min = page
  Due: May 12, 2024

ITEM 4: Inadequate Rollback Documentation
  Problem: On-call engineer took 3 min to find previous task definition version
  Owner: @runbook-maintainer
  Action: Add quick reference table to this runbook with last 5 deployed versions
  Due: May 10, 2024 (same day)

ITEM 5: No Canary Deployment
  Problem: 100% of traffic switched to new version immediately upon deploy
           If we'd only sent 10% of traffic to new version first,
           we'd have caught the slow query with 99% less impact
  Owner: @platform-team
  Action: Implement blue-green deployment with 10% canary for 2 minutes
  Due: May 30, 2024 (3 weeks, medium effort)
```

#### 4. Key Metrics from Incident

```
Metric                          Value             Target      Status
─────────────────────────────────────────────────────────────────────
Time to Alert                   3 seconds         <5s         ✅ Good
Time to Page                    1 second          <10s        ✅ Good
Engineer Response Time          4 minutes         <5min       ⚠️  Fair
Root Cause Detection Time       7 minutes         <10min      ✅ Good
Mitigation Start Time           8 minutes         <15min      ✅ Good
Full Recovery Time              15 minutes        <20min      ✅ Good
───────────────────────────────────────────────────────────────────────

Time to Page could be improved:
- Engineer's phone was in Do-Not-Disturb
- Escalation to backup on-call took 1 minute
- Action: Improve notification settings, backup on-call alerts
```

#### 5. Lessons Learned

```
1. Load testing in staging doesn't catch all issues
   Prod traffic is 41× staging capacity
   Single slow query becomes critical at scale
   
2. Canary deployments would have caught this
   Deploying to 100% users at once is risky
   Phased rollout (10% → 50% → 100%) reduces blast radius
   
3. Index choices matter more than query optimization
   Without proper indexes, even simple queries become O(n)
   Review index strategy during code review
   
4. Alerting on latency is important as error rate
   Error rate doesn't always go up (slowness shows as timeout instead)
   p95/p99 latency should alert before error rate spikes
```

#### 6. Follow-Up Questions (For Root Cause Analysis)

```
- Was this deploy code-reviewed before pushing to prod?
  Answer: Yes, but reviewer didn't know the query would become slow at scale
  
- Why doesn't staging detect this issue?
  Answer: Staging has 100 users, prod has 10M. Join takes 1ms with 100 users,
          takes 5 seconds with 10M users (due to missing index)
  
- How do we prevent similar issues?
  Answer: Run load tests with realistic traffic (50K RPS) before deploy
```

---

## Quick Reference: Common Commands During Incident

### CloudWatch Dashboard

```
https://console.aws.amazon.com/cloudwatch/home?region=ap-south-1#dashboards:
Dashboard name: swiggy-metrics

Key graphs to check:
- ALB HTTPCode_Target_5XX
- RDS DatabaseConnections
- EC2 CPUUtilization
- ElastiCache CacheHits / CacheMisses
- SQS ApproximateNumberOfMessages
```

### SSH into Bastion to Access Database

```bash
ssh -i ~/.ssh/swiggy-prod-key.pem ec2-user@bastion.prod.swiggy.internal

# From bastion:
psql -U postgres -h swiggy-primary.prod.swiggy.internal -d swiggy_prod

# Common queries:
SELECT count(*) FROM pg_stat_activity;  # Connection count
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;  # By type
```

### Check Recent Deploys

```bash
# Last 10 deployments
aws ecs describe-services \
  --cluster swiggy-prod \
  --services api \
  --region ap-south-1 \
  --query 'services[0].deployments' | head -20
```

### Check Logs

```bash
# Real-time log tail
aws logs tail /ecs/swiggy-api --follow --since 5m --region ap-south-1

# Search for errors
aws logs filter-log-events \
  --log-group-name /ecs/swiggy-api \
  --filter-pattern "ERROR" \
  --region ap-south-1 \
  --query 'events[0:20]'
```

---

## When to Escalate

**DO NOT try to handle alone if:**

1. Database needs to be recovered from backup (call @dba-team, do NOT experiment)
2. AWS infrastructure is down (check status page, then wait)
3. Payment gateway is down (contact vendor support)
4. You're not sure what to do after reading this runbook
   - Call @platform-team
   - Do NOT guess or try random fixes

**Escalation contact:**
- Slack: @app-oncall, @platform-team, @infrastructure-team
- Critical: Page (will automatically escalate to management if needed)

---

## Runbook Test Checklist

Before considering this runbook "done," verify:

- [ ] Can a first-time on-call follow STEP 2 without calling anyone?
- [ ] Every alert has a clear action defined?
- [ ] Every action has a success criteria?
- [ ] Postmortem template is fillable by someone who wasn't in the incident?
- [ ] No action requires knowledge not in this doc (add it if it does)?

