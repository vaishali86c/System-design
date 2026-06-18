# System-design

Hey, welcome to the course. I hope this course provides a great learning experience.

⭐ Star this repo to follow along as more topics are added!

---

## Table of Contents

### 1. [What is System Design?](./01-WHAT-IS-SYSTEM-DESIGN.md)
- Evolution from single machine to distributed architecture
- The 5 core pillars — scalability, reliability, availability, latency, maintainability
- Key trade-offs every system designer must reason through
- Interview questions

### 2. [Client-Server Model & Networking](./02-CLIENT-SERVER-MODEL.md)
- Client-server architecture and how an app server can act as a client
- IP addresses — public, private, IPv4, IPv6
- DNS resolution and why DNS uses UDP not TCP
- OSI model and TCP/IP model
- TCP vs UDP — when to use what
- 3-way handshake
- Application layer protocols
- HTTP versions (1.1, 2, 3) and HTTP methods
- HTTPS and TLS encryption — symmetric vs asymmetric
- Interview questions

### 3. [Real-Time Communication — Polling, WebSockets & SSE](./03-NETWORKING-REALTIME.md)
- The problem with HTTP for real-time updates
- Short polling — how it works, code examples, pros and cons
- Long polling — how it works, code examples, pros and cons
- WebSockets
- WSS — WebSocket Secure
- Server-Sent Events (SSE)
- Comparison and decision guide
- Interview questions

### 4. [Monolith, Microservices, RPC, gRPC & Webhooks](./04-MONOLITH-MICROSERVICES-RPC-GRPC-WEBHOOKS.md)
- Monolith architecture — advantages, disadvantages, when it works
- Microservices architecture — advantages, disadvantages, hidden costs
- Monolith vs Microservices
- Synchronous vs asynchronous communication patterns
- RPC 
- gRPC — Protobuf, HTTP/2, streaming modes, gRPC vs REST
- Webhooks
- Interview questions

### 5. [Distributed Systems, Consistency, Availability & CAP Theorem](./05-CONSISTENCY-AVAILABILITY-CAP.md)
- What is a distributed system and why it matters
- Consistency levels — strong, eventual, read-your-write
- Availability and the nines — what 99.99% really means
- SLA vs SLO
- CAP Theorem 
- How to increase availability
- Functional vs non-functional requirements
- System properties — latency, throughput, bandwidth, fault, failure, resiliency, redundancy
- Stateful vs stateless systems
- Interview questions
---

## Contributing

This repo is my personal learning journey through system design — documenting everything I learn, one topic at a time.

