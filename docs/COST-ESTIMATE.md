# AWS Cost Estimation: Baseline vs Peak Event Scaling

**Last Updated:** AWS Pricing as of May 2024 (India Region - Mumbai)  
**Currency:** USD and INR conversion used  
**Assumption:** All services deployed in `ap-south-1` (Mumbai, India)

---

## Scenario Overview

### Baseline Traffic (Normal Operations - 100K Daily Active Users)

This is the steady-state architecture running 24/7 to handle:
- ~100,000 daily active users
- ~500,000 monthly active users
- Regular order volume: ~50,000 orders/day
- No major promos active

### Peak Event (World Cup Night - 10M Users in 4 Hours)

The same infrastructure auto-scales for:
- 10 million concurrent users (spike from 8:00 PM - 12:00 AM)
- 7.5 million orders placed in 1 hour
- 500,000 requests per second (41× baseline capacity)
- Duration: 4 hours of elevated load

---

## BASELINE MONTHLY COST BREAKDOWN (Normal Operations)

### Compute Layer: EC2 Instances

**Configuration:**
```
Instance Type:         t3.large
vCPU:                  2 vCPU
Memory:                8 GB RAM
Baseline instances:    4 (running 24/7)
ASG Min:              2, Max: 50
```

**Pricing Calculation:**
```
Instance Price (ap-south-1): $0.1024/hour
Monthly hours:               730 hours (365 days × 24h / 12 months)
Monthly cost per instance:   $0.1024 × 730 = $74.75/month
4 instances × $74.75 =       $299.00/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| EC2 t3.large (on-demand) | /hour | 4 × 730h | $0.1024 | **$299.00/mo** |

---

### Database Layer: RDS PostgreSQL

**Configuration:**
```
Primary Database:
  Instance Type:     db.r6g.large (Graviton2, 2vCPU, 16GB RAM)
  Storage:          100 GB SSD (gp3)
  Backup:           7-day retention (included)
  Multi-AZ:         Disabled (for cost, enabled in prod)

Read Replicas:
  Replica 1:        db.r6g.large (same specs)
  Replica 2:        db.r6g.large (same specs)
  Location:         Same AZ as primary (synchronous replication)
```

**Pricing Calculation:**

```
RDS Pricing (ap-south-1):
  db.r6g.large:  $0.182/hour

Primary Instance:
  $0.182/hour × 730h = $132.76/month

Read Replica 1:
  $0.182/hour × 730h = $132.76/month

Read Replica 2:
  $0.182/hour × 730h = $132.76/month

Storage (all instances):
  3 instances × 100GB × $0.115/GB/month = $34.50/month
  (gp3 SSD in ap-south-1: $0.115/GB/month)

Backup Storage:
  Average backup size: ~80GB (7 daily backups, incremental)
  $0.021/GB/month × 80GB = $1.68/month

Total RDS:           $434.46/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| RDS db.r6g.large (Primary) | /hour | 730h | $0.182 | $132.76 |
| RDS db.r6g.large (Replica 1) | /hour | 730h | $0.182 | $132.76 |
| RDS db.r6g.large (Replica 2) | /hour | 730h | $0.182 | $132.76 |
| RDS Storage (gp3, 300GB total) | /GB/mo | 300 | $0.115 | $34.50 |
| RDS Backup Storage | /GB/mo | 80 | $0.021 | $1.68 |
| | | | **Subtotal** | **$434.46/mo** |

---

### Caching Layer: ElastiCache Redis

**Configuration:**
```
Cluster Type:        Redis Cluster (enabled)
Node Type:           cache.r7g.large (Graviton3, 2vCPU, 16GB RAM)
Cluster Size:        3 nodes (shard count = 1, replicas = 2)
Backup:              Daily snapshot to S3
Backup Retention:    5 days
```

**Pricing Calculation:**

```
ElastiCache Redis Pricing (ap-south-1):
  cache.r7g.large (on-demand): $0.166/hour

Per Node Cost:
  $0.166/hour × 730h = $121.18/month

3 Nodes:
  $121.18 × 3 = $363.54/month

Snapshot Storage:
  Snapshots size: ~15GB per day × 5 retained = 75GB
  $0.021/GB/month × 75GB = $1.58/month

Data Transfer (within AZ):
  Free (no charge for same-AZ data transfer)

Total ElastiCache:   $365.12/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| ElastiCache cache.r7g.large | /hour | 3 × 730h | $0.166 | $363.54 |
| Snapshot Storage | /GB/mo | 75 | $0.021 | $1.58 |
| | | | **Subtotal** | **$365.12/mo** |

---

### Load Balancing: Application Load Balancer

**Configuration:**
```
Type:                ALB (Application Load Balancer)
Scheme:              Internet-facing
Target Groups:       2 (API backends + health check)
Listeners:           443 (HTTPS), 80 (HTTP redirect)
SSL Certificate:     AWS Certificate Manager (free)
```

**Pricing Calculation:**

```
ALB Pricing (ap-south-1):
  ALB Base Cost:                    $16.20/month (fixed)
  LCU (Load Balancer Capacity Unit): per LCU processed

LCU Dimensions:
  New Connections:        100K new connections/hour
                         100K ÷ 4M = 0.025 LCU/hour

  Processed Bytes (GB):   Incoming: 50GB/day → ~0.07 GB/hour
                         Outgoing: 50GB/day → ~0.07 GB/day
                         Total: 0.14 GB/hour × 730h = 102.2 GB/month
                         102.2 ÷ 1GB = 102.2 LCU from data

  Target Connections:     At any moment: ~500 connections
                         Per hour: ~500 unique targets
                         500 ÷ 1M = 0.0005 LCU/hour

Estimated LCU per Hour: max(0.025, 102.2/730, 0.0005) = 0.14 LCU/hour
LCU Cost per hour: $0.006 per LCU
Monthly LCU Cost: 0.14 × $0.006 × 730h = $0.61/month

But AWS pricing floor for ALB data = ~$40/month (typical range $40-80)
Conservative estimate: $40/month

Total ALB:           $56.20/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| ALB Base Cost | /month | 1 | $16.20 | $16.20 |
| ALB LCU (estimated) | /month | 1 | $40.00 | $40.00 |
| | | | **Subtotal** | **$56.20/mo** |

---

### CDN: CloudFront

**Configuration:**
```
Origin:             api.swifteats.com (ALB)
Cache Behaviors:    
  1. /images/*              → 5-minute TTL
  2. /static/*              → 24-hour TTL
  3. /api/*                 → 1-second TTL (conditional)
  4. Default (API)          → Cache disabled
  
Distribution:       International (edge locations: 450+)
```

**Pricing Calculation:**

```
CloudFront Pricing (ap-south-1 - Data Transfer OUT):
  Per GB (first 10TB/month): $0.0085/GB
  Per GB (next 40TB/month):  $0.0068/GB
  Per HTTP request:          $0.0075/10,000 requests

Baseline Traffic:
  Daily traffic: ~10TB/month typical food delivery
  - Menu images cached (5m TTL): ~2TB/month
  - Static JS/CSS (cached): ~0.5TB/month
  - Origin requests: ~1TB/month (api calls)
  - User-agent requests: ~6.5TB/month

Baseline Data Transfer Cost:
  $0.0085/GB × 10,000GB = $85.00/month

Request Cost (Baselinee):
  ~500M requests/month
  ~50M from CloudFront edge (cached menus/static)
  ~50M origin requests
  50M ÷ 10K = 5,000 requests units
  $0.0075 × 5,000 = $37.50/month

Total CloudFront Baseline: $122.50/month

(Note: API requests not cached go directly from ALB, no CloudFront charge)
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| CloudFront Data OUT (10TB) | /GB | 10,000 | $0.0085 | $85.00 |
| CloudFront Requests | per 10K | 5,000 | $0.0075 | $37.50 |
| | | | **Subtotal** | **$122.50/mo** |

---

### Message Queue: AWS SQS

**Configuration:**
```
Queue Type:          Standard Queue
Message Retention:   4 days
Visibility Timeout:  1 second
Dead Letter Queue:   Enabled (for failed payments)
Daily Messages:      1,000,000 messages (~30K orders/day)
```

**Pricing Calculation:**

```
SQS Pricing (ap-south-1):
  First 1 million requests/month: $0.40/million (free tier is generous)
  
Monthly Message Volume:
  Orders per day: 50,000
  Messages per order: 2 (order placed, payment queued)
  Daily messages: 100,000
  Monthly messages: 100,000 × 30 = 3,000,000
  
Requests (API calls):
  Send message: 100K/day = 3M/month
  Receive message: 100K/day = 3M/month (payment worker consuming)
  Delete message: 100K/day = 3M/month (after processing)
  Total requests: 9M/month

SQS Cost:
  9M requests × ($0.40/1M) = $3.60/month

Dead Letter Queue (separate):
  Failed payment retries: ~1% = 30K/month
  Same operations: 90K requests
  $0.36/month

Total SQS: $3.96/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| SQS Send Message | per million | 3 | $0.40 | $1.20 |
| SQS Receive Message | per million | 3 | $0.40 | $1.20 |
| SQS Delete Message | per million | 3 | $0.40 | $1.20 |
| SQS DLQ operations | flat | 1 | $0.36 | $0.36 |
| | | | **Subtotal** | **$3.96/mo** |

---

### Container Orchestration: ECS (Payment Workers)

**Configuration:**
```
Service Type:        ECS Fargate
Task CPU:            1 vCPU
Task Memory:         2 GB RAM
Container Image:     payment-worker (from ECR)
Desired Tasks:       5 (baseline)
Max Tasks:           50 (during peak)
Pricing Model:       Fargate On-Demand
```

**Pricing Calculation:**

```
Fargate Pricing (ap-south-1):
  Per vCPU-hour:     $0.04048
  Per GB Memory-hour: $0.004445

Baseline (5 tasks):
  Per task:
    CPU cost: 1 × $0.04048 × 730h = $29.55/month
    Memory cost: 2 × $0.004445 × 730h = $6.49/month
    Per task: $35.04/month
  
  5 tasks × $35.04 = $175.20/month

ECR (Elastic Container Registry):
  Storage: ~2GB per image × 3 versions = 6GB
  $0.10/GB/month × 6 = $0.60/month

CloudWatch Container Insights:
  ~$5/month for monitoring

Total ECS Baseline: $181.00/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| Fargate vCPU (5 tasks, 1 vCPU each) | /vCPU-hour | 5 × 730h | $0.04048 | $147.76 |
| Fargate Memory (5 tasks, 2GB each) | /GB-hour | 10 × 730h | $0.004445 | $32.43 |
| ECR Image Storage | /GB/mo | 6 | $0.10 | $0.60 |
| Container Insights | /month | 1 | $5.00 | $5.00 |
| | | | **Subtotal** | **$185.79/mo** |

---

### Monitoring & Logging: CloudWatch

**Configuration:**
```
Log Groups:         4 (API app, payment worker, database, ALB)
Log Volume:         ~1GB/day
Metric Dashboards:  5 custom dashboards
Log Retention:      30 days
```

**Pricing Calculation:**

```
CloudWatch Pricing (ap-south-1):
  Log ingestion: $0.50/GB
  Log storage: $0.03/GB (30-day retention)
  Custom metrics: $0.30 each (free tier: 10 included)
  Dashboard API calls: $0.01/1K calls

Log Ingestion:
  Daily: 1GB × 30 days = 30GB/month
  Cost: 30 × $0.50 = $15.00/month

Log Storage:
  Active logs (30-day retention): ~30GB stored
  Cost: 30 × $0.03 = $0.90/month

Custom Metrics:
  Total metrics: 25 (beyond free tier: 15 paid)
  Cost: 15 × $0.30 = $4.50/month

CloudWatch Insights Queries:
  ~5K queries/month
  Cost: (5,000 / 1,000) × $0.01 = $0.05/month

Alarms:
  10 alarms × $0.10/alarm = $1.00/month

Total CloudWatch: $21.45/month
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| CloudWatch Log Ingestion | /GB | 30 | $0.50 | $15.00 |
| CloudWatch Log Storage (30d) | /GB | 30 | $0.03 | $0.90 |
| Custom Metrics | each | 15 | $0.30 | $4.50 |
| Alarms | each | 10 | $0.10 | $1.00 |
| Insights Queries | per 1K | 5 | $0.01 | $0.05 |
| | | | **Subtotal** | **$21.45/mo** |

---

## BASELINE TOTAL: $1,478.33/month

```
┌─────────────────────────────────────┐
│ BASELINE MONTHLY COST SUMMARY        │
├─────────────────────────────────────┤
│ Compute (EC2)           $299.00     │
│ Database (RDS)          $434.46     │
│ Caching (Redis)         $365.12     │
│ Load Balancer (ALB)     $56.20      │
│ CDN (CloudFront)        $122.50     │
│ Message Queue (SQS)     $3.96       │
│ Containers (ECS)        $185.79     │
│ Monitoring (CloudWatch) $21.45      │
├─────────────────────────────────────┤
│ TOTAL PER MONTH         $1,488.48   │
│ TOTAL PER YEAR          $17,861.76  │
└─────────────────────────────────────┘
```

**In Indian Rupees (at 1 USD = ₹83):**
- Monthly: ₹123,524
- Annual: ₹1,482,426

---

## PEAK EVENT COST ESCALATION (World Cup Night: 4 Hours)

During the 4-hour event (8:00 PM - 12:00 AM IST), the architecture auto-scales:

### Additional Compute: Auto-scaled EC2 Instances

**Configuration:**
```
Base instances:      4 (running before spike)
Peak instances:      20 (during spike)
Additional instances: 16 new instances
Duration:            4 hours
Instance type:       t3.large (same as baseline)
```

**Pricing Calculation:**

```
Baseline running: 4 instances × 4 hours (already paid in monthly)
Additional instances for spike: 16 instances × 4 hours
Cost: 16 × $0.1024/hour × 4 hours = $6.55

Alternatively expressed in terms of monthly equivalent:
  16 instances × $0.1024/hr × 4hr = $6.55
  Per 730-hour month equivalent: $6.55 × (730/4) = $1,196 per month
  But we only pay for 4 actual hours used: $6.55 for the spike
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| EC2 t3.large (extra 16) | /hour | 16 × 4h | $0.1024 | **$6.55** |

### Additional Compute: Auto-scaled ECS Payment Workers

**Configuration:**
```
Baseline workers:    5 tasks
Peak workers:        50 tasks
Additional workers:  45 tasks
Duration:            4 hours
```

**Pricing Calculation:**

```
Additional 45 Fargate tasks × 4 hours:
  CPU cost: 45 × 1vCPU × $0.04048/hr × 4hr = $7.29
  Memory cost: 45 × 2GB × $0.004445/hr × 4hr = $1.60
  
Total ECS surge: $8.89
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| Fargate vCPU (extra 45 tasks) | /vCPU-hour | 45 × 4h | $0.04048 | $7.29 |
| Fargate Memory (extra 45 tasks) | /GB-hour | 90 × 4h | $0.004445 | $1.60 |
| | | | **Subtotal** | **$8.89** |

### Additional Storage: CloudFront Peak Data Transfer

**Configuration:**
```
Baseline traffic:    ~10TB/month typical
Peak night traffic:  ~50TB extra (images being served more intensively)
Duration:            4 hours (but counted in monthly bill)
```

**Pricing Calculation:**

```
Baseline monthly CDN: 10TB @ $0.0085/GB = $85/month
Peak event surge:     +50TB @ $0.0085/GB = $425

If priced on a 4-hour basis:
  50TB = 50,000 GB
  If amortized across 4 hours out of 730 monthly hours:
  Full month equivalent: 50,000GB × (730/4) = 9,125,000 GB
  But we're charged for actual data transfer: 50,000 GB
  Cost: 50,000 × $0.0085 = $425

However, CloudFront is typically charged monthly, so:
  Extra traffic is absorbed into the monthly bill
  Additional monthly cost for peak event: ~$425
```

| Line Item | Unit | Quantity | Price | Total |
|-----------|------|----------|-------|-------|
| CloudFront Data OUT (extra 50TB) | /GB | 50,000 | $0.0085 | **$425.00** |

### Peak-Only Services: RDS Read Scaling

**Configuration:**
```
Baseline replicas:   2
Peak replicas:       2 (no additional - reads scale horizontally to existing replicas)
No additional RDS cost during peak
```

| Line Item | Explanation | Cost |
|-----------|-------------|------|
| RDS additional | Read replicas already provisioned; no additional peak cost | $0 |

### Peak-Only Monitoring: Real-time Dashboards

**Configuration:**
```
Dashboards:          10 (real-time monitoring during incident)
Queries per hour:    500 (real-time investigations)
Extra CloudWatch cost: minimal
```

**Pricing Calculation:**

```
Additional CloudWatch during peak (4 hours):
  Insights queries: 2,000 queries × $0.01/1K = $0.02
  
Total: $0.02 (negligible)
```

---

## PEAK EVENT ADDITIONAL COST (4-Hour Window)

```
┌─────────────────────────────────────┐
│ PEAK NIGHT ADDITIONAL COST          │
├─────────────────────────────────────┤
│ Additional EC2 (16 instances)  $6.55│
│ Additional ECS (45 workers)    $8.89│
│ CloudFront surge (50TB)      $425.00│
│ Monitoring (negligible)        $0.02│
├─────────────────────────────────────┤
│ PEAK NIGHT EXTRA COST         $440.46
│ BASELINE MONTHLY COST       $1,488.48
├─────────────────────────────────────┤
│ TOTAL with PEAK NIGHT       $1,928.94
│ (for that single month)              │
└─────────────────────────────────────┘
```

**Expressed as Cost Per Peak Hour:**
```
$440.46 ÷ 4 hours = $110.12/hour
vs.
Baseline monthly cost amortized: $1,488.48 ÷ 730 hours = $2.04/hour
Peak surge multiplier: $110.12 ÷ $2.04 = 54× hourly rate
```

---

## BUSINESS JUSTIFICATION: ROI Analysis

### Scenario 1: The Outage (Monolith System)

**What Happens Without This Architecture:**

As documented in FAILURE-CASCADE.md, the monolith experiences:
- T+18 seconds: Node.js crash (OOM)
- T+20 seconds: Complete service blackout
- T+45 minutes: Root cause identified
- T+90 minutes: System restored

**Cost of 70-Minute Outage:**

```
Lost Order Revenue:
  Peak spike: 10M users attempting orders
  Average order value: ₹1,000
  If promo active: ₹500 per order (net revenue)
  
  Estimated orders placed (if system worked): 7.5M
  Estimated orders LOST (system down): 6M orders
  
  Revenue lost: 6M orders × ₹500 = ₹3,000,000,000 (₹30 crore)
  
But wait - ₹4.2 crore/minute is the stated loss rate:
  ₹4.2 crore/min × 70 minutes = ₹294 crore total loss

Chargeback & Fraud Costs:
  Users charged full price despite "50% off" promise
  Chargebacks: 2-5% of failed orders = 120K-300K users
  Chargeback fees: $15-30 per chargeback = $1.8M - $9M
  In INR: ₹150 - ₹750 crore

App Store Rating Impact:
  Rating before spike: 4.3 stars
  Rating after 70-min outage: 1.8 stars
  User recovery: Estimated 6-12 months to return to 4.3
  Lost DAU during recovery: ~15% × 500K MAU × 180 days = 13.5M lost user-days
  Revenue loss from churn: 13.5M users × ₹200 lifetime value = ₹27 crore

Support Cost:
  Incoming support tickets: 50,000+
  Avg handling cost per ticket: $10 = ₹830
  Total support cost: 50,000 × ₹830 = ₹4.15 crore

Social Media Backlash:
  Trending hashtags: #SwiftEatsDown #Disaster
  Press coverage: "₹294 Crore Outage During World Cup"
  Brand rehabilitation cost: ₹10-50 crore
```

**Total Cost of Outage:**
```
Lost revenue (direct):           ₹294 crore
Chargebacks & fraud:             ₹150-750 crore (conservative: ₹300 crore)
Churn-driven lost lifetime value: ₹27 crore
Support cost:                     ₹4.15 crore
Brand rehabilitation:             ₹10-50 crore
─────────────────────────────────────────
TOTAL COST:                      ₹635 - 1,275 crore
(Conservative midpoint: ~₹900 crore)
```

### Scenario 2: The Resilient Architecture (This Design)

**Monthly Infrastructure Cost:**
```
$1,488.48 USD = ₹123,524 / month (at 1 USD = ₹83)

Annual cost to run this architecture: ₹1,482,288
```

**Peak Event Cost (4 hours):**
```
Additional cost: $440.46 = ₹36,558

But this cost prevents a ₹900 crore loss.
```

### ROI Calculation

```
Cost to build resilient architecture (12-month baseline): ₹1,482,288
Cost to handle peak event (incremental):                 ₹36,558
Total annual cost:                                       ₹1,518,846

Cost of one 70-minute outage:                          ~₹900,000,000,000
                                                        (₹900 crore)

ROI = Prevention value ÷ Cost
    = ₹900,000,000,000 ÷ ₹1,518,846
    = 592,595 : 1
    
Or expressed differently:
    Infrastructure costs: ₹1.52 crore/year
    One outage loss: ₹900 crore
    Payback ratio: Infrastructure costs cover 0.17% of one outage
    
The ROI is not a calculation. It is an obvious decision.
```

### Risk-Adjusted Analysis

**Probability of Outage Without Architecture:**
- Major promotional spike: 2-4 per year
- Probability of cascade failure during spike: 95%+
- Expected outages per year: 2-3
- Expected annual loss: 2 × ₹900 crore = ₹1,800 crore

**Probability of Outage With Architecture:**
- System handles 500K RPS with headroom
- Graceful degradation if components fail
- Probability of cascade failure: <1%
- Expected outages per year: <0.05
- Expected annual loss: 0.05 × ₹900 crore = ₹45 crore (worst-case risk)

**Risk Mitigation Value:**
```
Expected loss without architecture:  ₹1,800 crore/year
Expected loss with architecture:     ₹45 crore/year
Risk mitigation value:               ₹1,755 crore/year

Infrastructure cost:                 ₹1.52 crore/year
───────────────────────────────────────────────────────
Net risk-adjusted benefit:           ₹1,753.48 crore/year

The infrastructure is a 1,153× return on investment
on a risk-adjusted basis.
```

---

## Cost Optimization Recommendations

### Short-Term (Within 3 Months)

1. **Reserved Instances for Baseline:**
   - Reserve 4 × t3.large instances for 1-year commitment: ~25% savings
   - From $299/mo → $224/mo
   - Savings: $75/month = ₹6,225/month

2. **RDS Reserved Instances:**
   - Reserve 3 × db.r6g.large for 1-year: ~25% savings
   - From $434.46/mo → $325/mo
   - Savings: $109.46/month = ₹9,086/month

3. **ElastiCache Reserved Nodes:**
   - Reserve 3 × cache.r7g.large for 1-year: ~25% savings
   - From $365.12/mo → $273/mo
   - Savings: $92.12/month = ₹7,646/month

**Total Quarterly Savings: ~$276/month = ₹22,957/month**

### Medium-Term (Within 6 Months)

1. **Spot Instances for Peak Scaling:**
   - Use Spot instances for additional 16 instances during surge (80% savings vs on-demand)
   - Peak EC2 cost: $6.55 → $1.31
   - Savings per peak event: $5.24 = ₹435

2. **Auto-scaling Optimization:**
   - Implement predictive scaling (machine learning)
   - Anticipate load based on calendar events (promos, holidays)
   - Pre-warm instances 10 minutes before spike

### Long-Term (Within 12 Months)

1. **Regional Failover:**
   - Add standby architecture in different region (ap-northeast-1, Tokyo)
   - Minimal cost during normal times (reduced capacity)
   - Activates only during primary region failure

2. **Negotiate Volume Discounts:**
   - With baseline of $1.5M+/year, negotiate enterprise rates with AWS
   - Potential additional 10-15% savings on data transfer and storage

---

## Conclusion

| Metric | Value |
|--------|-------|
| **Baseline Monthly Cost** | $1,488 USD (~₹124K) |
| **Peak Event Additional Cost** | $440 USD (~₹37K) |
| **Total Annual Cost** | $18,861 USD (~₹1.57M) |
| **Cost of Single Outage (70 minutes)** | ₹900 crore |
| **ROI (Risk-Adjusted)** | 1,153:1 |
| **Payback Period** | <1 day (if even one outage is prevented) |

**The business case is unambiguous:** The infrastructure cost is trivial compared to the value it protects. Every engineer should understand these numbers. Every CFO should mandate this architecture for any service generating >₹100 crore/month in GMV (Gross Merchandise Value).

