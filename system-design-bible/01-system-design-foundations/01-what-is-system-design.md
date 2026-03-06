# What Is System Design?

## Definition

System design is the process of defining the **architecture, components, modules, interfaces, and data flows** of a system to satisfy specified requirements. In an engineering context, it bridges the gap between a problem statement and a working production system.

At Senior/Staff level, system design is less about knowing specific technologies and more about:

- Understanding **trade-offs** between competing concerns
- Reasoning about **scale, failure, and evolution**
- Making **principled decisions** with incomplete information
- Communicating **clearly** to stakeholders with different backgrounds

---

## The System Design Process

A rigorous system design follows a structured path:

```
1. Requirements Gathering
      |
      v
2. Constraint Identification (scale, SLAs, budget)
      |
      v
3. API Design (contract with callers)
      |
      v
4. High-Level Architecture (major components)
      |
      v
5. Deep Dives (bottlenecks, critical paths)
      |
      v
6. Failure Analysis (what breaks, how to handle it)
      |
      v
7. Operational Concerns (deployment, monitoring, migration)
```

Each step informs the next. Skipping steps leads to designs that look elegant but collapse under production conditions.

---

## Functional vs Non-Functional Requirements

### Functional Requirements

What the system **does** — its behaviors and features.

Examples:
- Users can post tweets of up to 280 characters
- Users can follow other users
- The home timeline shows tweets from followed users in reverse-chronological order

### Non-Functional Requirements

How the system **performs** — its quality attributes.

| Attribute | Description | Example Metric |
|-----------|-------------|----------------|
| Availability | System is operational | 99.99% uptime = 52 min/year downtime |
| Latency | Time to respond | p99 < 200ms |
| Throughput | Requests handled per unit time | 100K req/sec |
| Durability | Data is not lost | 11 nines (S3) |
| Consistency | All nodes agree on state | Strong / eventual |
| Scalability | Handles growth | 10x traffic without redesign |
| Security | Protects data and access | PCI-DSS compliant |
| Maintainability | Ease of operating/evolving | MTTR < 30 min |

### The Tension

Non-functional requirements almost always conflict:

- High availability + strong consistency is extremely hard (CAP theorem)
- Low latency + high durability requires careful trade-offs
- High security + low latency adds overhead
- High scalability + low cost requires clever architecture

A Senior+ engineer's job is to **identify which attributes matter most** for this specific system and make principled trade-offs.

---

## Dimensions of a System

When designing any system, consider:

```
                    ┌─────────────────────────────────┐
                    │           SYSTEM                │
                    │                                 │
  ┌─────────┐       │  ┌──────┐  ┌──────┐  ┌──────┐  │       ┌──────────┐
  │ Clients │──────>│  │ API  │  │Logic │  │ Data │  │──────>│ Storage  │
  └─────────┘       │  └──────┘  └──────┘  └──────┘  │       └──────────┘
                    │                                 │
                    │  ┌────────────────────────────┐ │
                    │  │   Cross-cutting concerns   │ │
                    │  │ Auth | Observability | Rate │ │
                    │  │ Limiting | Caching | Circuit│ │
                    │  │ Breakers | Retries          │ │
                    │  └────────────────────────────┘ │
                    └─────────────────────────────────┘
```

**Five dimensions every design must address:**

1. **Data model** — What data do we store? How is it structured? What are the access patterns?
2. **API contract** — What does the system expose? What are the guarantees?
3. **Processing logic** — How does data flow through the system? Where is computation done?
4. **Storage layer** — Which databases? How are they scaled? How is data partitioned?
5. **Cross-cutting concerns** — Auth, caching, rate limiting, observability, failure handling

---

## The Iceberg Model

In interviews, the visible system is the tip of the iceberg:

```
                    API  ←── What they ask about
                  /     \
          Services        Storage
        /                           \
   Caching   Queues   CDN   Search    Replication
  /                                           \
Circuit Breakers  Retries  Backpressure  Failover
 \                                           /
    Monitoring   Alerting   Capacity Planning
        \                           /
             Deployment Strategy
```

Staff and Principal engineers are expected to discuss **below the waterline** without being asked.

---

## What Good System Design Looks Like

| Dimension | Junior | Senior | Staff/Principal |
|-----------|--------|--------|-----------------|
| Requirements | Takes them as given | Clarifies and challenges | Identifies unstated requirements |
| Scale | Ignores or guesses | Estimates and uses to drive decisions | Works backward from business goals |
| Architecture | Uses familiar tech | Chooses appropriate tech for trade-offs | Evaluates build vs buy, migration cost |
| Failure | Ignores edge cases | Handles obvious failures | Designs for unknown failure modes |
| Trade-offs | Single solution | Presents alternatives | Quantifies trade-offs with data |
| Evolution | Point-in-time design | Considers v2 | Designs for evolutionary change |
| Operations | "It works on my machine" | Considers deployment | Considers on-call, runbooks, cost |

---

## System Design vs Software Design

| Aspect | Software Design | System Design |
|--------|-----------------|---------------|
| Scope | Single service or module | Multiple services and infrastructure |
| Concern | Code structure, patterns | Component interaction, data flow |
| Failure unit | Function, class | Network partition, datacenter |
| Scaling unit | Thread, process | Service, database, region |
| Discipline | CS fundamentals | Distributed systems theory |

They overlap — a well-designed system requires well-designed software inside it.

---

## Interview Discussion Points

- "What are the core features? Let me focus on the critical path."
- "What scale are we targeting? Let me do a quick estimation."
- "Before I go deep, let me validate the high-level design with you."
- "Here's a trade-off: option A gives us X but costs Y. Option B gives us Z. Given our requirements, I'd choose A because..."
- "One thing I haven't addressed is [failure mode]. Let me think about how to handle that."
- "In production, I'd also want to add monitoring for [metric] and alert on [threshold]."
