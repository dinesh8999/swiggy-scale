# Failure Cascade Analysis: SwiftEats Monolith Under 10M User Spike

## Executive Summary
During the India vs Pakistan World Cup Final at 8 PM IST, SwiftEats sends a 50% off promo to 180 million users. 10 million click in 60 seconds. The current monolith (single Node.js server + single PostgreSQL instance with 100 max connections) will experience total system collapse within 18 seconds. **No single component saves the system - the cascade is inevitable.** This document traces every failure with precise capacity numbers.

---

## Section 1: Traffic Simulation Math

### The Arrival Pattern

```
Total push recipients:           180,000,000 users
Industry CTR on major promos:     ~8%
Users who open the app:           14,400,000
Typical click spike concentration: 60 seconds (first minute)
Reserved for technical users:     ~70% of CTR
Expected active users:            ~10,000,000
```

### API Calls Per User (First 60 seconds)

Each user opening the app makes:
- **GET /restaurants** → fetch active restaurants (cached or not) - 10ms
- **GET /restaurant/:id** → view menu details with promo applied - 15ms  
- **POST /orders** → place order with payment validation - 500-2000ms (includes payment gateway call)

Conservative estimate:
- 70% of users browse without ordering: 2 API calls
- 30% of users place orders: 3 API calls

```
Average calls per user: (0.70 × 2) + (0.30 × 3) = 1.4 + 0.9 = 2.3 calls/user
```

### Peak RPS Calculation

```
Active users in 60-second spike:    10,000,000
Average API calls per user:         2.3
Total API calls in 60 seconds:      23,000,000

Peak RPS = 23,000,000 ÷ 60 = 383,333 RPS
```

**Reality: ~500,000 RPS accounting for retry storms and connection reuse patterns.**

### Context: What This Means

| System | Peak RPS Capacity | Our Scenario |
|--------|-----------------|-------------|
| SwiftEats Monolith | ~12,000 RPS | Incoming: 500,000 RPS |
| Scale Factor | 1x | **41× capacity** |
| Twitter (2018) | ~300,000 RPS | 1.67× our scenario |
| Netflix Global | ~800,000 RPS | 1.6× our scenario |
| Google Search | ~8,500,000 RPS | 17× our scenario |

---

## Section 2: Component Capacity Numbers - The Hard Limits

### PostgreSQL Connection Pool: max_connections = 100

This single number is the domino that triggers all subsequent failures.

**Configured Limit:**
```
max_connections = 100 (default PostgreSQL configuration)
Connection establishment time: 50-100ms
Connection close time: 10-20ms
```

**Query Hold Times by Type:**
```
SELECT /restaurants (cached case):         5ms  (rare, not in pool calc)
SELECT /restaurant/:id (no cache):        20ms  (constant)
POST /orders + payment validation:       500ms  (payment RPC holds connection)
Payment gateway timeout:                2000ms  (worst case - still holds connection)
```

### Node.js Event Loop Saturation Point

A single t3.medium (2 vCPU, 4GB RAM) Node.js Express process saturates at:

```
Single-threaded event loop RPS limit:    12,000-15,000 RPS
At saturation:
  - Request queue depth:                 15,000 pending requests
  - Response time (p99):                 5,000-10,000ms
  - Heap usage:                          3.5-4GB
  - GC pause frequency:                  every 200-500ms
```

Once the event loop queue exceeds 15,000 pending requests, new requests encounter backoff delay before processing even begins.

### Synchronous Payment Call Amplification

**The Critical Detail:**
Every POST /orders makes a synchronous HTTPS request to Razorpay or PayU:
```
Payment gateway URL: https://api.razorpay.com/v1/payments/create
HTTP method:         POST (synchronous - no callback, waits for response)
Timeout:             2 seconds (hard timeout if gateway is slow)
Response handling:   includes retry logic (can extend to 3s in network jitter)
Connection held:     ENTIRE DURATION
```

**Impact on DB Pool:**
- Regular query (20ms hold): 1 query per connection
- Payment query (800ms hold average): 1 query per connection, but **40× longer**

```
At 1,000 RPS incoming:
  - Non-payment queries (70%): 700 RPS × 0.020s = 14 connections held
  - Payment queries (30%):     300 RPS × 0.800s = 240 connections held
  - TOTAL:                     254 connections needed
  - POOL SIZE:                 100 connections
  - STATUS:                    EXHAUSTED, FAILED
```

This is the math: **At just 394 RPS, the pool is 100% exhausted.**

```
Pool exhaustion threshold = 100 ÷ (0.70 × 0.020 + 0.30 × 0.800)
                           = 100 ÷ 0.254
                           = ~394 RPS
```

And we are sending **500,000 RPS**.

### Node.js Heap Memory Model

```
Node.js process on t3.medium (4GB RAM):
  - Heap limit:                     2.8-3.5GB (Node.js default)
  - Average request size:           ~200KB (with response buffering)
  - At 15,000 pending requests:     15,000 × 200KB = 3GB
  - Add application overhead:       +500MB for globals, cache, etc.
  - Result:                         HEAP LIMIT EXCEEDED → OOM Kill
```

### NIC Saturation (No CDN)

```
Restaurant images served by Node.js:
  - Users fetching menus:           10,000,000
  - Average images per menu:        20 thumbnails + 10 full res
  - Average image size:             200KB per image
  - Data transfer in first 60s:     10M × 30 images × 200KB = 60TB
  
Typical AWS t3 instance NIC:        ~5Gbps max throughput
5Gbps = 625 MB/s maximum
60TB ÷ 625MB/s = 96,000 seconds ≈ 26.7 hours to transfer
```

The NIC will saturate in the first 10 seconds of the spike, at which point no new connections can even establish to the server.

---

## Section 3: The Cascade - Each Failure in Order

### Failure 1: PostgreSQL Connection Pool Exhaustion

**Severity:** CRITICAL  
**Trigger RPS:** 394 RPS (within 3 seconds of spike start)  
**Why:** See calculation above. 30% of requests (payment calls) hold connections for 800ms.

**What It Looks Like to Users:**
```
Response on smartphone at T+3s:
  - Status code: 500 Internal Server Error
  - Message: "Connection refused"  OR timeout after 30 seconds
  - Application: Completely unresponsive
  - In logs: "Error: connect ECONNREFUSED - no more connections in pool"
```

**What Fails Next:**
- New connection requests queue in Node.js event loop
- Event loop queue builds up as requests wait for DB connections that will never come available
- Node.js memory usage climbs as request objects remain buffered
- Triggers Failure 2 and Failure 3 in parallel

---

### Failure 2: Node.js Event Loop Saturation

**Severity:** CRITICAL  
**Trigger RPS:** 12,000 RPS (within 5 seconds of spike start)  
**Why:** Once the DB pool is exhausted, all requests queue in the event loop. Queue depth grows exponentially.

**Queue Depth Over Time:**
```
T+3s:   DB pool exhausted, request queue depth: 0 (but growing)
T+4s:   Requests queued: ~5,000 (500K RPS × 1s ÷ 100 concurrent handlers)
T+5s:   Requests queued: ~15,000 → Event loop overload threshold hit
T+6s:   Requests queued: ~50,000 → Heap memory at 3.5GB
```

**What It Looks Like to Users:**
```
T+5s onward:
  - All requests timeout after 30-60 seconds
  - Response times spike from 50ms → 5,000ms → timeout
  - User sees spinning loader for 60 seconds, then "Connection Timeout"
  - App becomes completely unresponsive
```

**What Fails Next:**
- Request objects accumulate in memory
- Node.js garbage collector runs continuously (GC pause: 500-1000ms per cycle)
- Each GC pause temporarily halts request processing entirely
- Heap pressure increases
- Triggers Failure 5 (OOM crash) within 5-10 more seconds

---

### Failure 3: Synchronous Payment Call Amplification

**Severity:** CRITICAL  
**Trigger RPS:** Concurrent with Failure 1 (from T+3s onward)  
**Why:** Payment calls are synchronous and hold DB connections. They are the reason the pool exhausts at just 394 RPS instead of 5,000 RPS.

**The Payment Call Flow:**
```
1. POST /orders arrives
2. Validate order: SELECT from orders table (20ms)
3. Validate promo: SELECT from promos table (15ms)
4. Deduct promo budget: UPDATE promos SET remaining = remaining - 1 (20ms)
5. Process payment: HTTPS POST to Razorpay (800-2000ms) ← CONNECTION HELD
6. On payment success: INSERT into orders table (25ms)
7. Return 200 OK to user

Total time holding connection: 50-2050ms
```

**The Amplification:**
At 12,000 RPS with 30% payment calls:
```
Payment calls per second: 3,600
Each holding connection for 1 second (average): 3,600 connections
Pool size: 100
Concurrent requests waiting for a connection: 3,500 requests
```

**What It Looks Like to Users:**
```
Users placing orders:
  - See spinner for 30+ seconds
  - Either: "Payment gateway timeout" OR "Order placed successfully" (race condition)
  - Cannot tell if payment was charged (funds debited but order not confirmed)
```

**What Fails Next:**
- Users retry orders, causing duplicate charges
- Support team gets flooded with "Did my order go through?" complaints
- Triggers Failure 4: The race condition becomes more severe with retries

---

### Failure 4: Promo Code Race Condition (TOCTOU)

**Severity:** HIGH (Business-facing)  
**Trigger Time:** T+10 seconds  
**Why:** Promo validation and deduction happen in two non-atomic queries.

**The Race:**
```
Query 1: SELECT remaining FROM promos WHERE code='WORLD50' → returns 1,000,000
Query 2: UPDATE promos SET remaining = remaining - 1 WHERE code='WORLD50'

With 10M concurrent users:
  - At T+10s, 10M users have all executed Query 1 simultaneously
  - All read: remaining = 1,000,000
  - All execute Query 2, decrementing by 1
  - Query 1 is re-executed by a retry storm
  - Remaining = 0 after ~100,000 confirmations
  - BUT: 10,000,000 users received "promo applied" confirmation
```

**Cost of This Failure:**
```
Promo budget: ₹50,00,000 (~10% of total orders at 50% off)
Orders incorrectly given promo: 9,900,000
Actual promo cost: ₹50,00,000
Unaccounted promo cost: 9,900,000 × (₹1,000 avg order value × 0.5) = ₹49,50,00,00,000 (~₹495 crore)
```

**What It Looks Like to Users:**
```
9.9M users: Confirmation page says "Promo applied! You save 50%"
Later: Payment charged FULL price (₹1,000 instead of ₹500)
User sees: "Wait, was I charged full price?"
App logs: "Promo budget exhausted" - but user already got confirmation
```

**What Fails Next:**
- Massive support ticket volume
- Chargeback fraud investigation
- Revenue reconciliation nightmare

---

### Failure 5: Static Asset NIC Saturation (No CDN)

**Severity:** CRITICAL  
**Trigger Time:** T+12 seconds  
**Why:** Restaurant images (200KB each) are served by the same Node.js process. No CDN.

**The Math:**
```
Active users in first 60s: 10,000,000
Restaurant images per user: 30 (thumbnails + full images)
Average image size: 200KB

Total data: 10M × 30 × 200KB = 60TB

Server NIC capacity (AWS t3 instance): 5 Gbps = 625 MB/s
Time to transfer 60TB: 60TB ÷ 625MB/s = 96,000 seconds

In reality: The NIC saturates within 10 seconds.
After that: NO NEW CONNECTIONS CAN ESTABLISH.
The server becomes unreachable entirely.
```

**What It Looks Like to Users:**
```
T+12s onward:
  - Requests to api.swifteats.com: "Connection refused" or timeout
  - Users see blank screens
  - No response at all - even health checks fail
  - ALB marks instance as unhealthy → removes from load balancer
```

**What Fails Next:**
- ALB health checks fail (HTTP requests to /health get no response)
- Instance marked "UNHEALTHY"
- If this is the only instance (it is), all traffic is dropped
- Triggers Failure 6: Complete blackout

---

### Failure 6: Monitoring Blindness (The Invisible Killer)

**Severity:** CRITICAL  
**Trigger Time:** T+15 seconds  
**Why:** No distributed tracing, no correlation IDs, no structured logs.

**What the On-Call Engineer Sees:**
```
Alert: "5xx error rate > 50%"
Dashboard shows:
  - EC2 CPU: 95%
  - DB connections: Can't query (DB unreachable)
  - Node.js heap: Unknown (process unresponsive)
  - Request latency: >30s timeout
  - Stack trace: Generic "socket hang up"

What the engineer CAN'T see:
  - Which query is slow (no slow query log)
  - Which user's request is queued (no trace ID)
  - Whether DB pool is exhausted (no pool metrics)
  - Whether payment gateway is down (no external dependency monitoring)
  - Whether it's a cascade from DB → Node → NIC or some other path
```

**MTTD (Mean Time To Detection):** 20-45 minutes  
**MTTR (Mean Time To Resolution):** 90-180 minutes (manual DB restart required)

---

## Section 4: The Incident Timeline

```
T+0s:     Push notification sent to 180M users
          → "50% off all orders tonight! Open now."

T+0-3s:   Notification arrives on phones
          → Users start opening the app
          → Initial requests (GET /restaurants, GET /restaurant/:id) begin
          → Database handles requests fine (still ~50 RPS)

T+3s:     PostgreSQL connection pool reaches capacity
          → First "connection refused" error
          → New DB connection requests start queuing in Node.js event loop
          → FAILURE 1 TRIGGERED

T+4s:     Payment calls continue to arrive (30% of requests)
          → Each holds connection for 500-2000ms
          → Connection pool remains at 100% utilization
          → No connections freed fast enough to handle new requests
          → Queue depth in Node.js: ~5,000 requests waiting

T+5s:     Node.js event loop saturation threshold reached
          → Request queue: 12,000-15,000 pending requests
          → Response times: 50ms → 5,000ms (1000× increase)
          → Garbage collector running continuously
          → User experience: Apps frozen, spinners, no response
          → FAILURE 2 TRIGGERED

T+8s:     All new database connection attempts rejected
          → Error rate jumps from 5% → 80%
          → Retry storms begin (users closing and re-opening app)
          → Payment calls timing out after 2 seconds
          → FAILURE 3 AMPLIFICATION

T+10s:    Promo code validation becomes unreliable
          → Race conditions in TOCTOU pattern
          → Some users get promo applied, others don't
          → Confirmation messages show promo but payment charged full price
          → Support begins receiving complaints
          → FAILURE 4 TRIGGERED

T+12s:    Node.js process attempting to send restaurant images
          → NIC bandwidth saturated with image transfers
          → No new HTTP connections can establish
          → Health check requests timeout
          → FAILURE 5 TRIGGERED

T+14s:    Node.js event loop heap memory approaching limit
          → Garbage collection pauses: 1-2 seconds each
          → No progress on request processing
          → Memory pressure mounting toward OOM

T+18s:    Node.js process crashes with Out-Of-Memory (OOM)
          → Process exits with code 137 (SIGKILL)
          → ALB health checks fail (no process to respond)
          → Instance marked UNHEALTHY
          → Traffic stops flowing to this instance

T+20s:    If there are multiple instances (there aren't - single server)
          → They would take over traffic
          → But in the monolith: THIS IS THE ONLY SERVER
          → All traffic is now dropped
          → Users see: "Connection refused" or "Service Unavailable"
          → COMPLETE BLACKOUT

T+20-45m: On-call engineer wakes up (paged at T+20s)
          → Sees "5xx rate 100%"
          → Begins triage: Check logs, check DB, check metrics
          → Logs are unstructured; hard to find root cause
          → Database is not responding (overload)
          → Cannot query slow_query_log
          → Cannot see stack traces of hanging requests
          → FAILURE 6 TRIGGERED: Detection takes 20-45 minutes

T+45m:    Root cause narrowed down to Node.js OOM + DB pool exhaustion
          → Manual decision: Restart the server
          → Stop the Node.js process
          → Flush idle connections from PostgreSQL: 
            SELECT pg_terminate_backend(pid) FROM pg_stat_activity 
            WHERE state = 'idle' AND query_start < NOW() - INTERVAL '10 seconds';
          → Restart Node.js

T+47m:    Node.js restarted, initial requests succeed
          → DB connections normalize
          → System begins recovery

T+50m:    System partially operational
          → New requests processing normally
          → But: Promo budget exhausted due to race condition
          → Promo disabled for remaining users
          → Revenue already lost

T+90m:    System fully recovered
          → All services operational
          → Users can now place orders
          → Support team still fielding complaints about promo/payment issues

T+24h:    Post-incident review
          → Lost revenue: ₹4.2 crore/min × 70 minutes = ₹294 crore
          → User complaints: ~50,000 support tickets
          → App Store rating: 4.3 → 1.8 (45-minute outage is visible)
```

---

## Section 5: Why Each Number Matters

| Number | Component | Why It Matters |
|--------|-----------|----------------|
| 100 | PostgreSQL max_connections | This single number determines when the entire system fails. Exhausts at 394 RPS. |
| 12,000 | Node.js RPS limit | Beyond this, event loop queues back up. We're asking for 500,000 RPS (41×). |
| 394 | Pool exhaustion threshold | With 30% payment calls at 800ms hold, the pool exhausts at just 394 RPS. |
| 800ms | Payment gateway latency | Synchronous payment calls hold DB connections. This is the amplification factor. |
| 2GB | Node.js heap limit | At 15,000 queued requests × 200KB, heap exhausted → OOM crash. |
| 60TB | Image transfer (first 60s) | Without CDN, 60TB/minute exceeds server NIC capacity by 96×. |
| 5Gbps | Server NIC throughput | Max sustainable bandwidth. Saturated in 10 seconds by image transfers. |
| 45 min | MTTD (no structured logs) | Without correlation IDs and structured logging, root cause analysis takes 45 minutes. |

---

## Conclusion

The cascade is not a failure of one component - it is the inevitable physics of overload:

1. **Payment calls** (synchronous, 800ms) amplify DB connection hold times
2. **DB pool exhaustion** (100 connections) causes all requests to queue
3. **Event loop saturation** (15,000 requests) consumes heap memory
4. **NIC saturation** (60TB images, no CDN) makes the server unreachable
5. **Process crash** (OOM) removes the last instance
6. **Monitoring blindness** (no traces, no structured logs) delays recovery by 45 minutes

**Every failure is a consequence of the previous one.** Fixing one component without fixing all of them does not help. The entire architecture must be redesigned to survive this load.

