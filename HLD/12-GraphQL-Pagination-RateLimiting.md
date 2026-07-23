# 12. API Design — REST, GraphQL, Pagination, Filtering & Rate Limiting

> Every product you build eventually exposes an API. How you design it determines whether clients can use it comfortably, whether it survives traffic spikes, and whether it scales as your data grows. This topic covers the full picture — from REST fundamentals, through GraphQL as an alternative query model, to the subtle differences between rate limiting algorithms that come up in senior interviews.

---

## Table of Contents

1. [REST API](#1-rest-api)
2. [Request Components](#2-request-components)
3. [CRUD Operations & HTTP Methods](#3-crud-operations--http-methods)
4. [HTTP Status Codes](#4-http-status-codes)
5. [Response Components & Headers](#5-response-components--headers)
6. [API Versioning](#6-api-versioning)
7. [Caching with ETag](#7-caching-with-etag)
8. [GraphQL](#8-graphql)
9. [Pagination](#9-pagination)
10. [Filtering](#10-filtering)
11. [Rate Limiting](#11-rate-limiting)
12. [Rate Limiting Algorithms](#12-rate-limiting-algorithms)
13. [Interview Questions](#-interview-questions)

---

## 1. REST API

REST (Representational State Transfer) is an architectural style for building APIs over HTTP. It is not a protocol or a standard — it is a set of constraints that, when followed, produce APIs that are predictable, scalable, and easy to understand.

The core idea: everything is a **resource** (a user, an order, a product), and you interact with resources using standard HTTP methods. The server is stateless — each request contains everything the server needs. There is no session memory between requests.

```mermaid
graph LR
    Client -->|HTTP Request\nMethod + URL + Headers + Body| Server
    Server -->|HTTP Response\nStatus Code + Headers + Body| Client
    Note[No session memory\nEach request is self-contained]
```

**Why stateless matters:** Because any server can handle any request. The load balancer can route your request to Server 1 today and Server 4 tomorrow — and both will respond identically because the request contains everything they need. This is what makes REST APIs horizontally scalable.

---

## 2. Request Components

Every HTTP request has three parts. Understanding them is fundamental to API design.

**Body** — The actual data you are sending. Used in POST, PUT, PATCH requests. Usually JSON.

```json
POST /users
Content-Type: application/json

{
  "name": "Vaishali",
  "email": "v@example.com",
  "role": "engineer"
}
```

**Headers** — Metadata about the request. Not visible in the URL. Used for authentication, content type, caching instructions.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json
Accept: application/json
X-Request-ID: abc-123
```

**Query Parameters** — Appended to the URL after `?`. Used for filtering, sorting, pagination — anything that modifies what data you get back, not what you are sending.

```
GET /products?category=electronics&sort=price&order=asc&page=2&limit=20
```

---

## 3. CRUD Operations & HTTP Methods

Each HTTP method has a specific meaning. Misusing them — like using GET to delete something — breaks the contract that makes REST predictable.

| Method | Operation | Example | Idempotent? |
|--------|-----------|---------|-------------|
| **GET** | Read | `GET /users/101` — fetch user | ✅ Yes |
| **POST** | Create | `POST /users` — create new user | ❌ No |
| **PUT** | Replace | `PUT /users/101` — replace entire user | ✅ Yes |
| **PATCH** | Partial update | `PATCH /users/101` — update only email | ✅ Yes |
| **DELETE** | Remove | `DELETE /users/101` — delete user | ✅ Yes |

**Idempotent** means calling it multiple times gives the same result. GET the same resource 10 times — same response each time. DELETE the same resource twice — second call returns 404, but the state is the same (resource is gone). POST creates a new resource each time — not idempotent.

```mermaid
graph LR
    subgraph REST Resource: /users
        G[GET /users — list all] 
        P[POST /users — create one]
    end
    subgraph REST Resource: /users/:id
        GI[GET /users/101 — get one]
        PU[PUT /users/101 — replace]
        PA[PATCH /users/101 — partial update]
        D[DELETE /users/101 — remove]
    end
```

---

## 4. HTTP Status Codes

Status codes tell the client what happened. Always use the right one — returning 200 for an error is one of the worst API design mistakes.

| Code | Meaning | When to use |
|------|---------|-------------|
| **200 OK** | Success | Successful GET, PUT, PATCH |
| **201 Created** | Resource created | Successful POST |
| **204 No Content** | Success, nothing to return | Successful DELETE |
| **400 Bad Request** | Client sent invalid data | Missing required field, wrong format |
| **401 Unauthorized** | Not authenticated | No token, expired token |
| **403 Forbidden** | Authenticated but not allowed | User cannot access this resource |
| **404 Not Found** | Resource does not exist | Wrong ID, deleted resource |
| **409 Conflict** | State conflict | Duplicate email on registration |
| **422 Unprocessable** | Valid format, invalid data | Age = -5, future date in wrong field |
| **429 Too Many Requests** | Rate limit exceeded | Client sent too many requests |
| **500 Internal Server Error** | Server crashed | Unhandled exception |
| **503 Service Unavailable** | Server overloaded or down | Maintenance, overload |

**The difference between 401 and 403 trips everyone up:**
- **401** — I do not know who you are. Send credentials.
- **403** — I know who you are. You are just not allowed to do this.

---

## 5. Response Components & Headers

Every response has a status code, headers, and usually a body.

**Standard response body (success):**

```json
{
  "success": true,
  "data": {
    "id": 101,
    "name": "Vaishali",
    "email": "v@example.com"
  }
}
```

**Standard response body (error):**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is already registered",
    "field": "email"
  }
}
```

**Important response headers:**

| Header | Purpose |
|--------|---------|
| `Content-Type` | Format of response body — `application/json` |
| `Location` | URL of newly created resource — `Location: /users/101` |
| `Cache-Control` | Caching instructions — `Cache-Control: max-age=3600` |
| `ETag` | Resource version identifier for cache validation |
| `X-RateLimit-Limit` | Max requests allowed per window |
| `X-RateLimit-Remaining` | Requests left in current window |
| `Retry-After` | Seconds until rate limit resets |

---

## 6. API Versioning

Your API will change. New fields, removed endpoints, changed behaviour. Clients using your API cannot update instantly. API versioning lets old clients keep working on the old version while new clients use the new one.

**URL versioning — most common:**
```
GET /v1/users/101
GET /v2/users/101
```

Simple. Clear. Easy to see which version you are calling. The downside is it changes the URL.

**Header versioning:**
```
GET /users/101
Accept: application/vnd.myapi.v2+json
```

Cleaner URLs but less visible. Clients must know to set the header.

**Query parameter versioning:**
```
GET /users/101?version=2
```

Easy to test in a browser. Not recommended for production — query params are for data, not routing.

```mermaid
graph LR
    OldClient[Old Client] -->|/v1/users| V1[API v1\nOld behaviour]
    NewClient[New Client] -->|/v2/users| V2[API v2\nNew behaviour]
    V1 & V2 --> Server[Same Backend]
```

**The rule:** Never break existing clients. When you need to change behaviour, create a new version. Maintain old versions for a deprecation period, communicate the timeline, then sunset them.

---

## 7. Caching with ETag

Every time a client requests a resource, it downloads the full response — even if nothing changed since last time. ETags solve this by letting the client say "I have version X of this resource — has it changed?"

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: GET /products/55
    S-->>C: 200 OK\nETag: "v3-abc123"\n{ product data }

    Note over C: Client caches the response + ETag

    C->>S: GET /products/55\nIf-None-Match: "v3-abc123"
    Note over S: Check if ETag matches current version
    alt ETag matches — nothing changed
        S-->>C: 304 Not Modified\n(no body sent)
        Note over C: Use cached version ✅ No data transfer
    else ETag changed — resource updated
        S-->>C: 200 OK\nETag: "v4-xyz789"\n{ updated product data }
        Note over C: Update cache with new data
    end
```

**Why it matters:** If a product page has not changed, the server sends back just a `304 Not Modified` with no body. Zero data transfer. At scale — millions of users requesting the same product pages — this saves enormous bandwidth and reduces server load significantly.

---

## 8. GraphQL

REST exposes many fixed endpoints, each returning a fixed shape of data. GraphQL exposes a **single endpoint**, and the client describes exactly what data it wants in the query itself. The server returns exactly that — nothing more, nothing less.

```mermaid
graph LR
    subgraph REST
        C1[Client] -->|GET /users/101| S1[Server]
        C1 -->|GET /users/101/posts| S1
        C1 -->|GET /users/101/followers| S1
    end
    subgraph GraphQL
        C2[Client] -->|POST /graphql\nquery: user + posts + followers| S2[Server]
    end
```

### The problem GraphQL solves

**Over-fetching** — a REST endpoint like `GET /users/101` returns the full user object even if the client only needs the name. A mobile app pays for bandwidth it does not use.

**Under-fetching** — the client needs a user, their posts, and their followers. That is three separate REST calls (or one bloated endpoint built just for this screen). GraphQL gets it all in a single request.

```graphql
query {
  user(id: 101) {
    name
    posts {
      title
    }
    followers {
      name
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "Vaishali",
      "posts": [{ "title": "Scaling Rate Limiters" }],
      "followers": [{ "name": "Aman" }]
    }
  }
}
```

The response shape mirrors the query shape exactly. No unused fields, no missing nested data, no second round trip.

### Schema, Queries, Mutations, Resolvers

A GraphQL API is defined by a **schema** — a strongly typed contract of what data is available.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
}

type Query {
  user(id: ID!): User
}

type Mutation {
  createPost(title: String!, authorId: ID!): Post!
}
```

- **Query** — read data. Equivalent to REST's GET.
- **Mutation** — write data (create, update, delete). Equivalent to REST's POST/PUT/PATCH/DELETE.
- **Resolver** — the function on the server that actually fetches the data for a given field. Every field in the schema has one resolver behind it.

```mermaid
sequenceDiagram
    participant C as Client
    participant G as GraphQL Server
    participant DB as Database

    C->>G: POST /graphql\n{ user(id:101){ name posts{title} } }
    G->>DB: resolver: getUser(101)
    DB-->>G: user row
    G->>DB: resolver: getPostsByUser(101)
    DB-->>G: post rows
    G-->>C: { data: { user: { name, posts } } }
```

### The N+1 Query Problem

The most common GraphQL performance bug. Fetching a list of users, then fetching each user's posts one at a time triggers 1 query for the users plus N queries for their posts.

```
Query: { users { name posts { title } } }

1 query  → SELECT * FROM users
N queries → SELECT * FROM posts WHERE user_id = ? (once per user)
```

**Fix:** Batch and cache resolver calls within a single request using a **DataLoader** — it collects all the individual `getPostsByUser(id)` calls made during one tick, and issues a single `SELECT * FROM posts WHERE user_id IN (...)` instead.

### REST vs GraphQL

| | REST | GraphQL |
|---|------|---------|
| **Endpoints** | Many, one per resource | One, `/graphql` |
| **Data shape** | Fixed by the server | Chosen by the client |
| **Over/under-fetching** | Common problem | Solved by design |
| **HTTP verbs** | GET/POST/PUT/PATCH/DELETE | Almost always POST |
| **Caching** | Native — HTTP caching, ETag, CDN caching works out of the box | Harder — single endpoint, single method; needs client-side cache (Apollo, Relay) or persisted queries |
| **File uploads** | Native (multipart form data) | Needs extensions/spec add-ons |
| **Status codes** | Meaningful (404, 403, 429...) | Usually always 200; errors live inside the response body |
| **Versioning** | New URL versions (`/v1`, `/v2`) | Usually version-free — add fields, deprecate old ones in place |
| **Learning curve** | Low | Higher — schema, resolvers, N+1 awareness |
| **Best for** | Simple CRUD, public APIs, caching-heavy systems | Complex/nested data, many client types (web, mobile) with different data needs |

**Rule of thumb:** REST wins when your data is simple, resources are cacheable, and you want the predictability of HTTP semantics. GraphQL wins when clients have very different data needs from the same backend — e.g. a mobile app that wants less data and a web dashboard that wants everything — and when the data is deeply nested and relational.

---

## 9. Pagination

Your API returns 50,000 products. Sending them all in one response is catastrophic — slow to generate, slow to transfer, the client can only show 20 on screen anyway. Pagination breaks data into pages.

### Offset-Based Pagination

```
GET /products?page=2&limit=10
```

Internally: `SELECT * FROM products LIMIT 10 OFFSET 10`

```json
{
  "data": [ ...10 products... ],
  "pagination": {
    "page": 2,
    "limit": 10,
    "total": 50000,
    "total_pages": 5000
  }
}
```

Simple and familiar. But at `OFFSET 100000`, the database scans and discards 100,000 rows before returning results. Performance degrades linearly as you go deeper. Also, if new data is inserted between page loads, items shift — users see duplicates.

### Cursor-Based Pagination

Instead of a page number, the server returns a cursor pointing to a position in the dataset.

```
GET /products?limit=10
→ { data: [...], next_cursor: "eyJpZCI6MTAxMH0=" }

GET /products?limit=10&cursor=eyJpZCI6MTAxMH0=
→ { data: [...], next_cursor: "eyJpZCI6MTAwMH0=" }
```

Internally: `SELECT * FROM products WHERE id < 1010 LIMIT 10` — uses an index, always fast.

No duplicates if data changes. No performance degradation. But you cannot jump to page 500.

```mermaid
graph LR
    Offset["Offset Pagination\npage=500 → scan 5000 rows 🐢\nDuplicates if data changes ❌"]
    Cursor["Cursor Pagination\nWHERE id < cursor → index hit 🚀\nNo duplicates ✅\nNo jumping to page 500 ❌"]
```

**Rule:** Use offset for admin dashboards where jumping to any page matters. Use cursor for feeds, infinite scroll, and high-volume APIs.

---

## 10. Filtering

Pagination gives you smaller chunks. Filtering gives you the right chunks.

```
GET /products?category=electronics&price_max=10000&in_stock=true&sort=price&order=asc
```

```mermaid
graph LR
    Client -->|Filter parameters in URL| Server
    Server -->|WHERE category='electronics'\nAND price < 10000\nAND in_stock = true\nORDER BY price ASC| DB[(Database)]
    DB --> Server
    Server -->|Only matching products| Client
```

**Common filter patterns:**

```
Exact match:   GET /orders?status=shipped
Range:         GET /products?price_min=500&price_max=5000
Search:        GET /articles?q=system+design
Date range:    GET /orders?from=2024-01-01&to=2024-06-30
Sort:          GET /products?sort=rating&order=desc
```

**Performance note:** Every filter column must have a database index. A filter on `category` without an index causes a full table scan on every request. At scale, that destroys your database.

---

## 11. Rate Limiting

Your API is well-designed and fast. Nothing stops someone from writing a script that calls it 100,000 times per minute. Your servers go down. Real users cannot access the service. Rate limiting prevents this.

```mermaid
sequenceDiagram
    participant C as Client
    participant RL as Rate Limiter
    participant S as Server

    C->>RL: Request 1 ✅
    RL->>S: Forward
    C->>RL: Request 2 ✅
    RL->>S: Forward
    C->>RL: Request 1000 — limit hit ❌
    RL-->>C: 429 Too Many Requests\nRetry-After: 30
```

**Why it matters:**
- Prevents brute force attacks — someone trying thousands of passwords per second
- Protects downstream services — your payment provider charges per call
- Ensures fair usage — one client cannot monopolize all server capacity
- System stability — uncontrolled spikes crash databases

**Rate limit response headers:**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716825600
Retry-After: 30
```

---

## 12. Rate Limiting Algorithms

### Fixed Window

Time is divided into fixed windows (every 60 seconds). Each client gets N requests per window. Counter resets at the window boundary.

```
Window: 12:00:00 - 12:01:00 → 100 requests allowed
Window: 12:01:00 - 12:02:00 → resets to 100 again
```

**The boundary burst problem:**

```
12:00:59 → client sends 100 requests (end of Window 1) ✅
12:01:01 → client sends 100 requests (start of Window 2) ✅
= 200 requests in 2 seconds. Rate limiting defeated.
```

Simple to implement. But exploitable at window boundaries.

---

### Sliding Window

The window slides with time. Always counts requests from the last N seconds, not from a fixed boundary.

At 12:01:30 — counts requests from 12:00:30 to 12:01:30. At 12:01:45 — counts from 12:00:45 to 12:01:45. No hard reset. No boundary exploits.

More accurate than fixed window. Requires storing timestamps of recent requests — slightly more memory.

---

### Token Bucket

Each client has a bucket that fills with tokens at a fixed rate. Each request costs one token. Empty bucket = request rejected.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Client idle for 5 seconds → 50 tokens accumulated
Client sends 50 requests instantly → all allowed (burst) ✅
Client sends 51st request → rejected, wait for refill ❌
After 1 second → 10 new tokens → 10 more requests allowed
```

```mermaid
graph TD
    Fill["Refill: 10 tokens/second"] --> Bucket["Token Bucket\nMax: 100 tokens"]
    Request["Incoming Request"] --> Check{Tokens left?}
    Check -->|Yes - take 1 token| Allow["Serve request ✅"]
    Check -->|No| Reject["429 Too Many Requests ❌"]
```

**Key insight:** Unused tokens accumulate. A client that was quiet for 10 seconds can burst 100 requests at once. This is often correct behaviour — reward quiet clients, punish abusive ones.

Used by GitHub, Stripe, Twitter API.

---

### Leaky Bucket

Requests fill a queue. The queue drains at a fixed rate regardless of how fast requests arrive. If the queue is full, requests are dropped.

```mermaid
graph TD
    Requests["Requests arriving at any rate"] --> Bucket["Leaky Bucket Queue"]
    Bucket -->|Fixed: 10 req/sec| Server
    Requests -->|Queue full| Drop["Drop ❌"]
```

Output to your server is perfectly smooth — always exactly N requests per second. But burst traffic is queued or dropped rather than served immediately.

**Best for:** Protecting a downstream system that needs perfectly smooth input — like a database or an external API with strict limits.

---

### Algorithm Comparison

| Algorithm | Burst allowed | Memory | Smoothness | Best for |
|-----------|--------------|--------|------------|----------|
| **Fixed Window** | ❌ Boundary burst | Low | Low | Simple cases |
| **Sliding Window** | ✅ No boundary | Medium | High | General APIs |
| **Token Bucket** | ✅ Natural burst | Low | Medium | Most production APIs |
| **Leaky Bucket** | ❌ Queued/dropped | Low | Very high | Smooth downstream protection |

---

## Interview Questions

**REST API**
1. What is REST? What does stateless mean and why does it matter for scaling?
2. What is the difference between PUT and PATCH?
3. What does idempotent mean? Which HTTP methods are idempotent?
4. What is the difference between 401 and 403?
5. When do you return 204 vs 200?

**Versioning & Headers**
1. What is API versioning? What are three ways to implement it?
2. What is an ETag? How does it reduce bandwidth?
3. What is the difference between `Cache-Control` and `ETag`?
4. What does `Location` header contain and when is it used?

**Pagination**
1. What is pagination and why is it needed?
2. What is the difference between offset-based and cursor-based pagination?
3. Why does `OFFSET 100000` hurt database performance?
4. What is the data drift problem in offset pagination?
5. When would you choose cursor-based over offset-based pagination?

**Filtering**
1. How do you design filtering for a product API with 20 filter options?
2. Why must columns used for filtering have database indexes?

**Rate Limiting**
1. What is rate limiting and what problems does it solve?
2. What HTTP status code is returned when rate limited? What headers should you include?
3. Explain the token bucket algorithm. How does it handle bursts?
4. What is the boundary burst problem with fixed window rate limiting?
5. Why do you need Redis to implement rate limiting in a distributed system with multiple servers?
6. You are building a login endpoint — which rate limiting algorithm and what limits would you set?

**GraphQL**
1. What problem does GraphQL solve that REST does not? Explain over-fetching and under-fetching.
2. What is the difference between a Query, a Mutation, and a resolver?
3. What is the N+1 query problem in GraphQL and how does a DataLoader fix it?
4. Why is caching harder in GraphQL than in REST?
5. Why does GraphQL mostly return HTTP 200 even for errors — where do errors show up instead?
6. When would you choose REST over GraphQL for a new API?

---