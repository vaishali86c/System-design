## 01. What is System Design?

Before we start this course, let's talk about what even is system design.

System design is the process of defining the architecture, interfaces, and data for a system that satisfies specific requirements. System design meets the needs of your business or organization through coherent and efficient systems. It requires a systematic approach to building and engineering systems. A good system design requires us to think about everything, from infrastructure all the way down to the data and how it's stored.

### Why is System Design so important?
System design helps us define a solution that meets the business requirements. It is one of the earliest decisions we can make when building a system. Often it is essential to think from a high level as these decisions are very difficult to correct later. It also makes it easier to reason about and manage architectural changes as the system evolves.

### Single Machine → Distributed System

Every app starts simple — one server does everything. As users grow, you split responsibilities:

``` mermaid
flowchart LR

A["Single Server<br/>Web Server<br/>App Logic<br/>Database"]

A -->|"User Growth"| B["Load Balancer"]

B --> S1["Server 1"]
B --> S2["Server 2"]
B --> S3["Server 3"]

S1 --> DB["Database"]
S2 --> DB
S3 --> DB

S1 --> C["Cache"]
S2 --> C
S3 --> C
```

### The 5 Core Pillars

| Pillar | What it means |
|--------|--------------|
| **Scalability** | Handle more users over time |
| **Reliability** | Keep working when things fail |
| **Availability** | System is accessible 24/7 |
| **Latency** | How fast a single request is served |
| **Maintainability** | Easy for engineers to change & deploy |

``` mermaid
mindmap
  root((System Design))
    Scalability
      More Users
      More Traffic
      Horizontal Scaling

    Reliability
      Fault Tolerance
      Recovery
      Redundancy

    Availability
      24x7 Access
      Minimal Downtime

    Latency
      Fast Response
      Better User Experience

    Maintainability
      Easy Changes
      Easy Deployment
      Clean Architecture
```
### System Design - Decision flow

```mermaid
flowchart TD

A[Business Requirements] --> B[Estimate Scale]

B --> C[Choose Architecture]

C --> D[Scalability]
C --> E[Reliability]
C --> F[Availability]
C --> G[Latency]
C --> H[Maintainability]

D --> I[Final Design]
E --> I
F --> I
G --> I
H --> I
```

### Key Trade-offs

> Every design decision is a trade-off. There is no perfect system.

```mermaid
graph TD

A[System Design Trade-Offs]

A --> B[Higher Availability]
B --> C[More Servers]
C --> D[Higher Cost]

A --> E[Lower Latency]
E --> F[Cache Data]
F --> G[Possible Stale Data]

A --> H[Strong Consistency]
H --> I[Node Synchronization]
I --> J[Higher Latency]

A --> K[More Microservices]
K --> L[Independent Scaling]
L --> M[Operational Complexity]
```

**Example:** Instagram shows your feed with a slight delay — they chose *eventual consistency* over *strong consistency* because 1 second delay is invisible to users and saves huge infrastructure cost.

### ❓Questions

1. What is System Design and why does it matter?
2. What are the 5 core pillars of a well-designed system?
3. Explain the trade-off between consistency and availability with an example.
4. How does a system evolve from a single machine to a distributed architecture?
5. What would you consider first when designing a system for 1M vs 1B users?

---
