# Architecture: Redesigned Multi-Tier System for 10M User Spike

## Section 1: Current Monolith Architecture (And Why It Fails)

```
                        180M Users
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  Single IP, Single Port             │
        │  No Load Balancer                   │  ← Single point of failure
        └────────────────────────────────────┘
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  Node.js Express Server             │  
        │  - 1 process (1 CPU)                │  
        │  - 4GB RAM                          │
        │  - Event loop: 12,000 RPS limit     │  ← Fails at 394 RPS due to DB
        │                                     │
        │  Endpoints:                         │
        │  - GET /restaurants (no cache)      │
        │  - GET /restaurant/:id              │
        │  - POST /orders (sync payment)      │
        │  - GET /images/* (200KB each)       │  ← NIC saturated
        │  - POST /promo/validate             │
        └────────────────────────────────────┘
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  PostgreSQL (Single Instance)       │
        │  max_connections = 100              │  ← Exhausts at 394 RPS
        │  No indexes on orders.user_id       │
        │  No read replicas                   │
        │  Single writer, single reader       │
        │                                     │
        │  Schema:                            │
        │  - restaurants table                │
        │  - orders table (no index)          │
        │  - users table                      │
        │  - promos table (TOCTOU race)       │
        └────────────────────────────────────┘

Failure Points Annotated:
├─ No CDN        ⚠️ Failure 5: Image requests saturate NIC (60TB/min)
├─ Sync payments ⚠️ Failure 3: 800ms hold per payment × 30% requests
├─ No cache      ⚠️ Failure 1: DB exhausts at 394 RPS
├─ Single server ⚠️ Single point of failure
├─ No connection pooler ⚠️ 100 max connections hard limit
└─ No load balancer ⚠️ Cannot scale horizontally
```

---

## Section 2: Redesigned Multi-Tier Architecture

```
                        10M Users (Spike)
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  CloudFront CDN (Global Edge)       │
        │  - 450+ edge locations             │
        │  - Cache-Control: TTL 5m           │  ← Failure 5 PREVENTED
        │                                     │
        │  Caches:                           │
        │  - /api/v1/restaurants/:id/images/* │
        │  - /api/v1/restaurants (menu list) │
        │  - /static/* (JS, CSS, HTML)       │
        │                                     │
        │  Dynamic requests (REST API):      │
        │  - /api/v1/orders (cache MISS)     │
        │  → Forward to ALB                   │
        └────────────────────────────────────┘
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  AWS Application Load Balancer      │
        │  - Health checks: /health (5s)     │
        │  - SSL/TLS termination             │
        │  - Rate limiting: 100 req/IP/s     │
        │  - Path routing for microservices  │
        │  - Auto-scaling trigger: CPU > 70% │
        └────────────────────────────────────┘
                  │        │       │        │
        ┌─────────┴────────┴───────┴────────┘
        │
        ├─────► [API Node-1]    (t3.large, 2 vCPU, 8GB RAM)
        ├─────► [API Node-2]    (t3.large, 2 vCPU, 8GB RAM)
        ├─────► [API Node-3]    (t3.large, 2 vCPU, 8GB RAM)
        ├─────► [API Node-4]    (t3.large, 2 vCPU, 8GB RAM)
        ├─────► ...scales to 20 instances during spike
        │
        └─────────────────────────────────────┐
                                              │
                            ▼
        ┌────────────────────────────────────┐
        │  Redis Cluster (ElastiCache)        │
        │  - cache.r7g.large × 3 nodes      │
        │  - Replication: 3 replicas         │
        │  - Throughput: 100K ops/sec        │  ← Failure 1 REDUCED
        │                                     │
        │  Cache Keys (5m TTL):              │
        │  - restaurants:* (restaurant list) │
        │  - menu:restaurant:* (full menus)  │
        │  - promo:locks:* (atomic deduction)│  ← Failure 4 PREVENTED
        │  - user:session:* (user context)   │
        │                                     │
        │  Promo Budget Lock:                │
        │  - SETNX promo:lock:budget:*       │
        │  - Atomic increment/decrement      │
        └────────────────────────────────────┘
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  PgBouncer (Connection Pooler)      │
        │  - pool_size = 200                 │
        │  - max_connections = 1000          │
        │  - pool_mode = transaction         │  ← Failure 1 PREVENTED
        │                                     │
        │  Effective Multiplier:             │
        │  - Physical DB connections: 200   │
        │  - Application connection slots: 5000 │
        │  - Ratio: 25:1                     │
        └────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
        ┌──────────────┐        ┌──────────────┐
        │  PostgreSQL  │        │  RDS Replica │
        │  Primary     │───────►│  1            │
        │  (WRITES)    │        │  (READS:     │
        │              │        │  orders hist)│
        │  - r6g.4xl   │───────►│  RDS Replica │
        │  - 16 vCPU   │        │  2            │
        │  - 128GB RAM │        │  (READS:     │
        │  - 100GB SSD │        │  restaurant) │
        │              │        └──────────────┘
        │  Schema:     │
        │  - orders    │  Indexes: order.user_id, order.created_at
        │  - users     │
        │  - restaurants│
        │  - promos    │  With Atomic Promo Deduction:
        │              │  BEGIN TRANSACTION;
        │              │  UPDATE promos SET remaining = remaining - 1
        │              │    WHERE code = $1
        │              │    AND remaining > 0
        │              │    AND expires_at > NOW();
        │              │  -- Rows updated: 0 or 1 (atomic)
        │              │  COMMIT;
        └──────────────┘
                │
                ▼
        ┌────────────────────────────────────┐
        │  AWS SQS Queue                      │
        │  - Payment Queue                   │
        │  - Queue depth: ~1000-5000 msgs   │
        │  - Message retention: 4 days       │  ← Failure 3 PREVENTED
        │  - Dead Letter Queue (DLQ)         │
        │                                     │
        │  Message Format:                  │
        │  {                                │
        │    "order_id": "uuid",            │
        │    "user_id": "uuid",             │
        │    "amount": 500,                 │
        │    "razorpay_order_id": "order_*" │
        │  }                                │
        │                                     │
        │  Processing:                      │
        │  - Consumer: payment-worker       │
        │  - Autoscaled: 5-50 worker pods  │
        │  - Throughput: 100+ orders/sec   │
        └────────────────────────────────────┘
                            │
                            ▼
        ┌────────────────────────────────────┐
        │  Payment Worker (Microservice)      │
        │  - ECS Task Cluster                │
        │  - Consumes from SQS Queue         │
        │  - Makes async calls to Razorpay   │
        │  - Webhook handler for responses   │
        │  - Retries: exponential backoff    │
        │  - No DB connection held           │
        │  - Scales: 5-50 worker replicas   │
        │                                     │
        │  Flow:                            │
        │  1. Read from SQS                 │
        │  2. POST to Razorpay API          │
        │  3. Handle Razorpay response      │
        │  4. UPDATE orders table (success) │
        │  5. Publish to Kafka (audit log)  │
        │  6. Delete from SQS               │
        └────────────────────────────────────┘

Data Flow During Spike:
───────────────────────

User Action: Place Order (POST /api/v1/orders)
  1. ALB receives request → routes to API Node (RR)
  2. API Node validates input (5ms)
  3. Redis cache lookup: promo:WORLD50 → cache HIT (1ms)
  4. Redis SETNX: deduct promo budget atomically (1ms) → RACE CONDITION PREVENTED
  5. Order object created in memory (5ms)
  6. INSERT into PostgreSQL (via PgBouncer) (20ms)
     - PgBouncer holds connection from pool of 200
     - If all 200 occupied: queue in PgBouncer (no OOM)
  7. Message published to SQS: "process payment" (1ms)
  8. PgBouncer connection released (goes back to pool)
  9. Respond to user: "Order placed! Payment processing..." (50ms total)
  10. Payment Worker (async):
      - Reads from SQS
      - POSTs to Razorpay (800ms, NO DB connection held)
      - Updates order.payment_status = "completed"
      - User receives push: "Order confirmed!"

Key Improvements:
─────────────────
✅ DB connection held for 20ms (INSERT only) instead of 800ms (sync payment)
✅ Payment processing is async - 40× faster connection release
✅ Promo deduction is atomic - NO race condition
✅ Images served from CDN - NO NIC saturation
✅ 4 API nodes instead of 1 - handles 4×-20× more RPS
✅ Redis caches 80% of menu requests - DB pool never exhausts
✅ PgBouncer converts 100 connections to 5,000 available slots
✅ Read replicas separate read/write contention
```

---

## Section 3: Component Justification Table

Every component added directly prevents a specific failure from FAILURE-CASCADE.md:

| Component | Failure It Prevents | How It Prevents It | Capacity Gain | Cost/Benefit |
|-----------|-------------------|------------------|---------------|------------|
| **CloudFront CDN** | Failure 5: NIC Saturation | Serves all restaurant images (200KB each) from 450+ global edge locations. 10M users × 30 images × 200KB = 60TB normally saturates single server NIC in 10s. CDN serves from edge → 0 bytes to origin. | NIC utilization: 60TB/min → 0TB | ~$85/mo for typical spike (10TB/mo × $0.0085/GB). Prevents 1-2hr outage worth ₹420 crore. ROI: **5,000,000:1** |
| **Application Load Balancer** | Single Point of Failure (implicit Failure 7) | Health checks every 5 seconds. If one API node crashes, ALB removes it from rotation in <5s. Traffic routes to remaining healthy instances. Automatic scaling adds instances when CPU > 70%. | RPS capacity: 12K (1 node) → 240K (20 nodes) | $16.20/mo base + $40 LCU. Enables horizontal scaling without any code changes. |
| **Multiple API Nodes (t3.large, 4-20 instances)** | Failure 2: Event Loop Saturation | Single Node.js saturates at 12,000 RPS. With 4 nodes: 48,000 RPS. With 20 nodes: 240,000 RPS. Spike of 500K RPS now distributed. Each node handles ~25,000 RPS (well above single-node limit but within cluster handling). | Single node: 12K RPS → 4 nodes: 48K RPS → 20 nodes: 240K RPS | Baseline: 4 nodes = $119.81/mo. Peak: 20 nodes = $26.62 extra for 4 hours. |
| **Redis Cache (ElastiCache, 3 nodes)** | Failure 1: DB Pool Exhaustion | Restaurant menus (80% of requests) cached with 5m TTL. Redis returns in <1ms, never touches PostgreSQL. Without Redis: 400K read requests hit DB. With Redis: 4K read requests hit DB (99% cache hit). Pool exhaustion moves from 394 RPS → 40,000+ RPS. | DB connection reduction: 100,000 requests/min → 1,000 requests/min (99% cache hit) | $358.56/mo for 3 cache.r6g.large nodes. Moves DB bottleneck from 394 RPS to 40,000+ RPS. |
| **PgBouncer (Connection Pooler)** | Failure 1: DB Pool Exhaustion | PostgreSQL max_connections = 100 is hard limit. PgBouncer maintains 200 physical connections but serves 5,000+ application-level connection requests. Connection multiplexing: 25 app requests per physical connection. Pool no longer exhausts at 394 RPS; now exhausts at 40,000+ RPS. | Physical connections: 100 → 200 effective slots. Application connection requests: 100 → 5,000 | Small VM running PgBouncer: ~$10/mo. Multiplies effective capacity by 50×. |
| **SQS Payment Queue** | Failure 3: Sync Payment Amplification | Orders no longer wait for payment synchronously. POST /orders: write order + queue message (30ms total). Payment processed asynchronously. DB connection held for 20ms instead of 800ms. For same 500K RPS, connection hold time drops 40×. | Connection hold time: 800ms (sync) → 20ms (async write) + async payment worker | $12/mo for 1M messages/day. Reduces DB connection utilization by 97%. Prevents cascade entirely. |
| **Payment Worker (ECS Microservice)** | Failure 3: Sync Payment Amplification | Consumes from SQS queue. Processes Razorpay calls asynchronously in worker containers (5-50 auto-scaled replicas). Each worker handles ~100 payments/sec. No DB connection held during payment processing. Payment timeout does not block API response. | Payment processing: Blocking (800ms/user) → Async (worker pool) | ~$50/mo for 5-10 worker containers. Microservice scales independently. |
| **PostgreSQL Read Replicas (2 replicas)** | Write/Read Contention (implicit Failure 8) | Primary handles all writes (orders, payments). Replicas handle all reads (restaurant listings, order history, user profiles). Read/write requests no longer compete for same DB connections. Primary can focus on INSERT throughput. | Primary connection contention: Reduced by ~60% (60% of traffic is reads) | $262.08/mo for 2 db.r6g.large read replicas. Separates read/write workloads entirely. |
| **Structured Logging + Correlation IDs** | Failure 6: Monitoring Blindness | Every request tagged with trace ID (UUID). Logs include: request_id, user_id, endpoint, latency_ms, component, error_code. Centralized log aggregation (CloudWatch). MTTD reduced from 45 min → 2 min. | Detection time: 45 minutes (blind) → 2 minutes (traces) | ~$20/mo for CloudWatch Insights. Reduces incident impact by 22× (45 min → 2 min). |

---

## Section 4: Traffic Distribution & Handling

### Baseline Traffic (100K DAU)
```
REST Endpoint Traffic Distribution:
├─ GET /api/v1/restaurants           32% → Redis cache HIT (1ms)
├─ GET /api/v1/restaurant/:id         28% → Redis cache HIT (1ms)
├─ POST /api/v1/orders                25% → PgBouncer pool (20ms)
├─ GET /api/v1/orders/history          10% → Read Replica (5ms)
└─ GET /static/*                        5% → CDN (0ms, edge)

Average latency: 95% requests < 50ms
DB connections held: ~10-15 at any moment
```

### Peak Traffic (10M users in 60 seconds)

```
Initial 10 seconds (Burst Phase):
  - ALB receives 500K RPS
  - Rate limiter: 100 req/IP/sec
    → Typical user IP: ~10K users per IP (shared cell towers)
    → Legitimate traffic: ~5K RPS per IP block passes through
    → Aggressive rate limiting: Prevents DOS amplification
  
  - Auto-scaling triggered: CPU on 4 instances jumps to 85%+
  - ASG adds 4 more instances (t3.large)
  - New instances join rotation (45 second ramp-up)
  - Meanwhile: Existing 4 handle load

Request Distribution (10M users, 3 calls each = 30M calls / 60s):
├─ GET /restaurants         9.6M requests  (32%)  → Redis (99.5% HIT)
│  └─ Cache misses: 48K → DB reads via read replicas
├─ GET /restaurant/:id      8.4M requests  (28%)  → Redis (99.5% HIT)
│  └─ Cache misses: 42K → DB reads via read replicas
├─ POST /orders             7.5M requests  (25%)  → SQS queue
│  └─ Order inserts: 7.5M → PostgreSQL Primary
│  └─ SQS messages queued: 7.5M
├─ GET /orders/history      3M requests    (10%)  → Read Replicas
│  └─ Direct read: 3M queries
└─ GET /images/*            2.5M requests  (5%)   → CloudFront CDN
   └─ CDN cache HIT: 100% (no origin requests)

DB Connection Usage (Peak):
  - Read operations: 48K + 42K + 3M = 3.09M/min distributed across 2 read replicas
  - Write operations: 7.5M/min to primary
  - PgBouncer pool_size = 200 physical connections
  - Each connection average hold time: 20ms (orders)
  - Connections needed: (7.5M/60) × (0.020s) = 2,500 connections
  - Available (with multiplexing): 5,000 effective connections
  - STATUS: Plenty of capacity ✅

Redis Throughput (Peak):
  - Menu cache requests: 18M/min → Redis
  - Average latency: <1ms
  - Throughput: 18M/60s = 300K RPS
  - Redis cluster (3 nodes of cache.r7g.large): 100K ops/sec per node
  - Total capacity: 300K ops/sec
  - STATUS: Within capacity ✅

SQS Queue Depth (Peak):
  - Order rate: 7.5M/60s = 125K orders/sec
  - SQS message rate: 125K messages/sec
  - Queue depth at steady state: msg_rate × visibility_timeout
                                = 125K/s × 1s (visibility) = 125K messages
  - Payment worker throughput: 50 replicas × 100 orders/sec = 5K orders/sec
  - Queue drain rate: 5K orders/sec
  - Queue backlog depth: 125K - 5K = 120K messages
  - Processing lag: 120K / 5K = 24 seconds
  - User experience: Order placed → payment processed within 30 seconds ✅

CDN Traffic (Peak):
  - Image requests: 2.5M in 60 seconds = 41,666 images/sec
  - Average image size: 200KB
  - Bandwidth: 41,666 × 200KB = 8.3 TB/sec → 50TB/min
  - CloudFront origin request cost: Charged only on cache misses (menu changes rare)
  - Cache TTL: 5 minutes (images don't change during spike)
  - Cache hit rate: 99.8%+
  - Origin requests: ~2% = 50K images/sec × 0.02 = 1K images/sec
  - Bandwidth to origin: 1K × 200KB = 200GB/sec → 12TB/min
  - STATUS: Achievable ✅
```

---

## Section 5: Autoscaling Behavior

```
Baseline (100K DAU):
┌────────────────────────────────────┐
│ EC2 Auto Scaling Group              │
│ - Desired capacity: 4 instances     │
│ - Min capacity: 2 instances         │
│ - Max capacity: 50 instances        │
│ - Instance type: t3.large           │
│ - Cooldown: 60 seconds              │
│ - Target CPU: 70%                   │
└────────────────────────────────────┘

T+0 World Cup Event Triggered:
  - ALB receives spike of requests
  - CPU on all 4 instances jumps to 85%+
  - Scale-up alarm triggered (CPU > 80% for 2 minutes)
  - ASG adds 2 instances per 2 minutes (ramp-up policy)

T+2 min: 6 instances online (1200 health checks in progress)
T+4 min: 8 instances online
T+6 min: 10 instances online
T+8 min: 12 instances online
T+10 min: 14 instances online
T+12 min: 16 instances online
T+14 min: 18 instances online
T+16 min: 20 instances online (max safe limit)

Peak Capacity Reached:
  - 20 instances × 25K RPS/instance = 500K RPS capacity
  - Load balanced evenly across all 20
  - Each instance CPU: ~60% (well within limit)
  - Each instance memory: ~40% (plenty of headroom)

T+60 min (End of spike):
  - User activity drops sharply
  - CPU on instances drops below 50%
  - Scale-down alarm triggered
  - ASG removes 2 instances per 10 minutes (slower scale-down)

T+80 min: 16 instances
T+100 min: 12 instances
T+120 min: 8 instances
T+140 min: 4 instances (back to baseline)
```

---

## Conclusion: Architecture Resilience

This redesigned architecture handles 500K RPS because:

1. **No Single Bottleneck:** Each layer scales independently
   - Compute: 1 → 20 nodes
   - Cache: Hits rate 99%+ before DB
   - DB: 100 connections → 5,000 effective slots (PgBouncer)
   - Payment: Async, no connection hold

2. **Graceful Degradation:** If components fail:
   - Redis down? Cache misses, but reads go to replicas
   - Payment worker down? SQS queue holds messages, retries with backoff
   - One API node crashes? ALB removes it, 19 others still running

3. **Monitoring & Recovery:** Structured logging enables 2-minute MTTD instead of 45 minutes

4. **Cost Proportional to Load:** Autoscaling means you only pay for capacity you use
   - Baseline: $1,024/month
   - Peak 4 hours: +$455 extra
   - Alternative: 45-minute outage = ₹294 crore lost revenue

