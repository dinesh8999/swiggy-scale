# Swiggy is Down: Scale Simulation & Incident Architecture

## 🚨 The Scenario

**Event:** India vs Pakistan World Cup Final, 8:00 PM IST  
**Action:** SwiftEats (Swiggy-like) sends "50% off all orders tonight!" promo to 180 million users  
**Result:** 10 million users click in 60 seconds → **500,000 RPS** inbound  
**Problem:** The backend is a **single Node.js server + single PostgreSQL database with 100 max connections**  
**Outcome:** Complete system collapse in **18 seconds** → **₹294 crore in lost orders**

This repository documents the full failure cascade analysis, redesigned architecture, AWS cost estimation, and 5-step incident runbook that could save a company **₹900 crore** in a single night.

---

## 📊 Key Findings

| Finding | Number | Impact |
|---------|--------|--------|
| **Peak RPS in spike** | 500,000 | 41× the monolith's 12K RPS capacity |
| **DB pool exhaustion threshold** | 394 RPS | Synchronous payments (800ms hold) amplify connection usage by 40× |
| **Time to total collapse** | 18 seconds | From first user opening app to complete blackout |
| **Revenue loss per minute (outage)** | ₹4.2 crore | ~₹294 crore for a 70-minute outage |
| **MTTD without structured logging** | 45 minutes | Engineers cannot find root cause without correlation IDs |
| **MTTD with distributed tracing** | 2 minutes | Root cause visible immediately in dashboards |
| **Baseline monthly infrastructure cost** | $1,488 USD (~₹124K) | Handles 100K DAU comfortably |
| **Peak event incremental cost** | $440 USD (~₹37K) | 4-hour surge for 10M concurrent users |
| **ROI on resilient architecture** | 1,153:1 | Infrastructure cost = 0.17% of one outage loss |

---

## 📚 Documents in This Repository

### 1. **FAILURE-CASCADE.md** — The Physics of Collapse

**What you'll learn:**
- Exact capacity numbers for every component (PostgreSQL max_connections=100, Node.js RPS limit=12K, Redis throughput=100K ops/sec)
- Math showing why the DB pool exhausts at **just 394 RPS** (not 5,000) due to payment call amplification
- Complete timeline from notification send (T+0s) to full recovery (T+90m)
- 6 cascading failures, each one a consequence of the previous

**Key sections:**
- Traffic simulation math: 10M users × 2.3 API calls ÷ 60 seconds = 383K RPS
- Component capacity numbers with real limits
- Failure cascade with RPS trigger points
- 45-minute incident timeline with second-by-second breakdown

---

### 2. **ARCHITECTURE.md** — The Redesigned System

**What you'll learn:**
- Full multi-tier architecture diagram designed specifically to prevent each failure
- Component justification table: Every component linked to the specific failure it prevents
- How 4 Node.js instances scale to 20 during peak
- How PgBouncer multiplies effective database capacity from 100 to 5,000
- How async payment queue (SQS) eliminates 800ms connection hold time
- How Redis caches 80% of reads before they hit the database
- How CDN prevents 60TB/minute of image traffic from saturating the NIC

**Why each component exists:**
- CloudFront CDN → Prevents Failure 5 (NIC saturation)
- SQS + Payment Worker → Prevents Failure 3 (Sync payment amplification)
- Redis Cache → Reduces DB load by 99%
- PgBouncer → Prevents Failure 1 (Connection pool exhaustion)
- Multiple API nodes + ALB → Prevents Failure 2 (Compute saturation)
- Read replicas → Separates read/write contention

---

### 3. **COST-ESTIMATE.md** — AWS Pricing Breakdown

**What you'll learn:**
- Real AWS instance types and hourly rates for ap-south-1 (Mumbai)
- Baseline monthly cost: **$1,488/month** (or ₹123,524/month)
- Peak event cost for 4-hour surge: **+$440 additional** (or +₹36,558)
- Cost breakdown by component (EC2, RDS, ElastiCache, ALB, CloudFront, SQS, ECS)
- Business justification: Cost of one outage (**₹900 crore**) vs annual infrastructure (**₹1.5 crore**)
- ROI calculation: 1,153:1 (infrastructure is a trivial investment compared to outage loss)

**Pricing details:**
```
EC2 t3.large (baseline 4):        $299/month
RDS Primary + 2 Replicas:         $434/month
ElastiCache Redis Cluster (3):    $365/month
ALB + CloudFront:                 $179/month
SQS + ECS Workers:                $192/month
Monitoring:                       $21/month
─────────────────────────────────────────────
TOTAL BASELINE:                   $1,488/month

Peak 4-hour event:                +$440 extra

This prevents:                    ₹900 crore loss
```

---

### 4. **RUNBOOK.md** — 5-Step Incident Response

**What you'll learn:**
- STEP 1: CloudWatch alerts that must fire (10 specific alerts with thresholds)
- STEP 2: Triage decision tree (30-second root cause identification)
- STEP 3: Component-specific fixes (5 failure branches with exact CLI commands)
- STEP 4: Rollback criteria and procedure
- STEP 5: Postmortem template (fillable form for post-incident review)

**Designed for:**
- A junior engineer who has never seen the system before
- 2 AM incident response (no documentation required, just follow steps)
- Reduces MTTD from 45 minutes (blind guessing) to 2 minutes (structured triage)

---

## 🏗️ Architecture at a Glance

```
10M Users → CloudFront CDN (images cached, 0 origin requests)
            ↓
    ALB (load balance, health check, rate limit)
            ↓
    4-20 Node.js instances (t3.large, auto-scaled)
            ↓
    ┌──────────────────────────────────────────┐
    │  Redis Cache (ElastiCache, 3 nodes)      │
    │  99.5% hit rate on menu requests         │
    └──────────────────────────────────────────┘
            ↓ (cache misses only)
    ┌──────────────────────────────────────────┐
    │  PgBouncer (connection pooler)           │
    │  100 physical → 5,000 effective slots    │
    └──────────────────────────────────────────┘
            ↓
    PostgreSQL Primary (writes)  ← Read Replicas (reads)
            ↓
    ┌──────────────────────────────────────────┐
    │  SQS Payment Queue                       │
    │  (async, no DB connection held)          │
    └──────────────────────────────────────────┘
            ↓
    Payment Workers (ECS, 5-50 auto-scaled)
    (Process payments asynchronously)
```

---

## 💡 Why This Matters

### Before (Monolith)
- ✗ Single Node.js instance
- ✗ Single database instance
- ✗ No cache layer
- ✗ Synchronous payment calls hold connections
- ✗ No CDN, images saturate NIC
- ✗ Complete failure at 394 RPS (monolith handles 12K but payments amplify)
- ✗ 45-minute incident recovery (no structured logging)
- ✗ ₹294 crore loss in one outage

### After (This Architecture)
- ✅ 4-20 auto-scaled Node.js instances
- ✅ Single primary + 2 read replicas
- ✅ Redis caching layer (99% hit rate)
- ✅ Async payment processing via SQS
- ✅ CDN serves all static assets (0 bytes to origin)
- ✅ Handles 500K RPS without breaking
- ✅ 2-minute incident recovery (structured traces)
- ✅ Prevents ₹294 crore loss per outage

---

## 🎯 What You Can Do After Reading This

| Skill | Application |
|-------|-------------|
| **Failure cascade analysis** | Identify which component fails first at what RPS using math (not guessing) |
| **Capacity planning** | Know real numbers: PostgreSQL 100 connections, Node.js 12K RPS, Redis 100K ops/sec |
| **Architecture design** | Every component must prevent a specific failure - no "because it's industry standard" |
| **AWS cost estimation** | Quote real monthly costs using actual instance types and pricing |
| **Incident response** | 30-second triage to root cause, exact CLI commands to fix, measurable success criteria |
| **SRE mindset** | Understand cascading failures, exponential vs linear scaling, failure modes by component |

---

## 📖 How to Use This Repository

1. **For Learning:** Start with FAILURE-CASCADE.md (understand the problem space)
2. **For Incident Response:** Go to RUNBOOK.md (use during outages)
3. **For Architecture Decisions:** Read ARCHITECTURE.md (justify each component)
4. **For Business Cases:** Reference COST-ESTIMATE.md (ROI justification for investments)

---

## 🔍 Real-World Applicability

This analysis is **directly applicable** to:
- Swiggy, Zomato, Uber Eats, any food delivery platform
- Ticketing platforms (BookMyShow during film releases)
- E-commerce (Amazon during sales)
- Ride-sharing (Uber, Ola during peak hours)
- Social media (Twitter during major events)

The principles are **universal:**
- Connection pool exhaustion happens at the same thresholds
- Synchronous operations amplify load by fixed ratios
- Caching reduces database load predictably
- Load distribution scales linearly
- Monitoring enables 22× faster incident response

---

## 📊 Table of Contents

| Document | Sections | Key Output |
|----------|----------|-----------|
| FAILURE-CASCADE.md | 5 sections + timeline | RPS trigger points, cascade order, MTTD numbers |
| ARCHITECTURE.md | 5 sections + diagram | Redesigned system, component justification table |
| COST-ESTIMATE.md | 7 cost categories | Monthly: $1,488, Peak: +$440, ROI: 1,153:1 |
| RUNBOOK.md | 5 steps + appendix | 10 alerts, triage tree, 5 fix procedures, postmortem template |

---

## 💬 Quick Facts

- **Monolith RPS capacity:** 12,000 RPS
- **Spike RPS:** 500,000 RPS (41× capacity)
- **DB pool exhaustion:** 394 RPS (due to payment amplification)
- **System collapse time:** 18 seconds
- **Revenue loss/minute:** ₹4.2 crore
- **Loss for 70-minute outage:** ₹294 crore
- **Infrastructure cost to prevent:** $1,488/month ($440 extra during spike)
- **ROI:** 1,153:1
- **Incident detection with traces:** 2 minutes vs 45 minutes without

---

## 🚀 The Bottom Line

Systems fail predictably. Cascades follow physics.

The engineer who understands these numbers is the engineer who gets paged at 2 AM and fixes it in 10 minutes instead of 45. That's who you are now.

---

**Last Updated:** May 2024  
**Status:** Complete incident analysis and architecture redesign for ₹1,000+ crore GMV platforms  
**Next Steps:** Implement ARCHITECTURE.md, test with RUNBOOK.md