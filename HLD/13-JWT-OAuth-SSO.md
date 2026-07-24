# 13. Authentication & Authorization — JWT, OAuth 2.0 & SSO

> Every system that stores user data needs to answer two questions before doing anything: who are you, and what are you allowed to do? These sound like the same question but they are fundamentally different — and confusing them leads to security vulnerabilities. This topic covers the full auth stack from first principles.

---

## Table of Contents

1. [Authentication vs Authorization](#1-authentication-vs-authorization)
2. [JWT — JSON Web Tokens](#2-jwt--json-web-tokens)
3. [OAuth 2.0](#3-oauth-20)
4. [SSO — Single Sign-On](#4-sso--single-sign-on)
5. [Access Control — RBAC & ACL](#5-access-control--rbac--acl)
6. [Interview Questions](#-interview-questions)

---

## 1. Authentication vs Authorization

These two words are used interchangeably in conversation but they are completely different operations happening at different points in every request.

**Authentication** — verifying identity. Who are you? Prove it.

**Authorization** — verifying permission. Okay, I know who you are. What are you allowed to do?

```mermaid
sequenceDiagram
    participant U as User
    participant S as System

    U->>S: Login with email + password
    S->>S: Authentication: verify credentials ✅
    Note over S: Identity confirmed — you are Vaishali

    U->>S: Delete another user's post
    S->>S: Authorization: can Vaishali delete this? ❌
    S-->>U: 403 Forbidden
    Note over S: Identity is fine. Permission is not.
```

**Key differences:**

| | Authentication | Authorization |
|--|---------------|--------------|
| Question | Who are you? | What can you do? |
| Happens | First | Second — after authentication |
| Methods | Password, OTP, biometrics, tokens | ACL, RBAC, policies |
| Outcome | Identity confirmed | Access granted or denied |
| HTTP code on failure | 401 Unauthorized | 403 Forbidden |

---

## 2. JWT — JSON Web Tokens

JWT is the most common mechanism for authentication in modern APIs. After you log in, the server gives you a JWT. You send this token with every subsequent request to prove who you are.

The key property of JWT: **the server does not need to store anything**. The token itself contains all the information the server needs — and the server can verify it has not been tampered with using a cryptographic signature.

### Structure of a JWT

A JWT has three parts separated by dots: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEwMSwicm9sZSI6ImFkbWluIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header** — base64 encoded JSON. Contains the token type and signing algorithm.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload** — base64 encoded JSON. Contains claims — facts about the user.

```json
{
  "userId": 101,
  "email": "v@example.com",
  "role": "admin",
  "iat": 1716825600,
  "exp": 1716912000
}
```

**Signature** — cryptographic proof that the token was issued by your server and has not been modified.

```
HMACSHA256(
  base64(header) + "." + base64(payload),
  SECRET_KEY
)
```

If anyone changes even one character in the payload, the signature no longer matches. The server rejects it.

```mermaid
graph LR
    subgraph JWT Structure
        H["Header\nalg: HS256"] --- P["Payload\nuserId, role, exp"]
        P --- S["Signature\nHMAC of header+payload\nwith server secret"]
    end
```

### The JWT Flow

```mermaid
sequenceDiagram
    participant U as User
    participant S as Server
    participant DB as Database

    U->>S: POST /login {email, password}
    S->>DB: Verify credentials
    DB-->>S: Valid ✅
    S->>S: Generate JWT with userId, role, expiry
    S-->>U: { token: "eyJ..." }

    Note over U: Stores JWT in localStorage or cookie

    U->>S: GET /dashboard\nAuthorization: Bearer eyJ...
    S->>S: Verify signature ✅ Decode payload
    Note over S: No DB lookup needed — token is self-contained
    S-->>U: Dashboard data ✅
```

**Why this is powerful for scaling:** The server does not store session state. Any server can verify any JWT using the same secret key. Stateless authentication — exactly what REST requires.

### JWT Security Considerations

**Always set an expiry** — `exp` claim. A JWT with no expiry is valid forever. If it leaks, attackers can use it indefinitely. Short-lived tokens (15 minutes to 1 hour) reduce risk.

**Refresh tokens** — When the access token expires, instead of logging in again, the client sends a long-lived refresh token to get a new short-lived access token.

```mermaid
graph LR
    Login -->|Short-lived access token - 15min| Client
    Login -->|Long-lived refresh token - 7 days| Client
    Client -->|API requests| AccessToken["Access Token\n15 min validity"]
    AccessToken -->|Expired| RefreshEndpoint["POST /refresh"]
    RefreshEndpoint -->|New access token| Client
```

**Do not store sensitive data in payload** — the payload is base64 encoded, not encrypted. Anyone can decode it. Never put passwords or private keys in JWT payload.

**Use HTTPS always** — JWTs in transit over plain HTTP can be intercepted.

---

## 3. OAuth 2.0

You have seen this a hundred times: "Continue with Google" or "Login with GitHub." You click it, get redirected to Google, approve access, and land back on the original site — logged in, without giving that site your Google password.

That is OAuth 2.0.

OAuth 2.0 is an authorization framework that lets a user grant a third-party application limited access to their account on another service — without sharing their password.

### The Key Players

**Resource Owner** — The user. You.

**Client** — The third-party app wanting access. A todo app that wants to read your Google Calendar.

**Authorization Server** — The server that authenticates the user and issues tokens. Google's auth server.

**Resource Server** — The API holding the user's data. Google Calendar API.

### The OAuth 2.0 Flow

```mermaid
sequenceDiagram
    participant U as User
    participant App as TodoApp (Client)
    participant Auth as Google Auth Server
    participant API as Google Calendar API

    U->>App: Click "Connect Google Calendar"
    App->>Auth: Redirect user to Google\nclient_id, scope: read_calendar, redirect_uri
    Auth->>U: Show permission consent screen\n"TodoApp wants to read your calendar"
    U->>Auth: User clicks Allow ✅
    Auth->>App: Redirect back with authorization code
    App->>Auth: Exchange code for tokens\n(client_id + client_secret + code)
    Auth-->>App: Access token + Refresh token
    App->>API: GET /calendar/events\nAuthorization: Bearer {access_token}
    API-->>App: Calendar events ✅
    App-->>U: Shows your calendar in TodoApp
```

### Why Authorization Code, Not Token Directly?

The code is short-lived and single-use. The final token exchange happens server-to-server (not in the browser), so the token never passes through the browser's URL where it could be intercepted or leaked in browser history.

### OAuth 2.0 Scopes

Scopes define exactly what access the app is requesting. Users see this on the consent screen.

```
scope=read:calendar         (read calendar events only)
scope=read:email            (read email address only)
scope=read:calendar write:calendar    (read + write calendar)
```

Apps should request only the minimum scopes they need — this is the principle of least privilege.

---

## 4. SSO — Single Sign-On

You work at a company. Every morning you log into one portal and immediately have access to your email, Slack, JIRA, GitHub, and Figma — without logging into each one separately. That is SSO.

SSO lets users authenticate once with an Identity Provider (IdP) and access multiple applications (Service Providers) without re-entering credentials.

```mermaid
sequenceDiagram
    participant U as Employee
    participant IdP as Identity Provider\n(Okta / Azure AD)
    participant App1 as Slack
    participant App2 as JIRA
    participant App3 as GitHub

    U->>IdP: Login once — email + password + MFA
    IdP-->>U: SSO Session + Token

    U->>App1: Open Slack
    App1->>IdP: Validate token
    IdP-->>App1: Valid ✅
    App1-->>U: Logged in ✅

    U->>App2: Open JIRA
    App2->>IdP: Validate token
    IdP-->>App2: Valid ✅
    App2-->>U: Logged in ✅ No re-login needed
```

### How SSO Works

1. User tries to access an application (say, Slack).
2. Slack redirects to the company's Identity Provider (Okta).
3. If the user is not logged in — they authenticate at the IdP (once).
4. IdP generates a token and redirects back to Slack.
5. Slack validates the token with the IdP and grants access.
6. User opens JIRA next — already has an IdP session, gets access immediately.

### Common SSO Protocols

**SAML (Security Assertion Markup Language)** — Enterprise standard. XML-based. Used for browser-based SSO between a company's apps.

**OpenID Connect (OIDC)** — Built on top of OAuth 2.0. JSON-based. The modern standard. Used by Google, Apple, Microsoft login buttons.

### Why SSO Matters Beyond Convenience

**Security:** Users maintain fewer passwords. Fewer passwords means fewer compromised credentials. MFA can be enforced at the IdP level once — applies everywhere.

**Centralized access control:** When an employee leaves the company, IT revokes access at the IdP. All applications lose access simultaneously. No hunting through 20 apps to revoke permissions individually.

---

## 5. Access Control — RBAC & ACL

Authentication answers "who are you." Authorization answers "what can you do." The mechanism that enforces authorization is access control.

### RBAC — Role-Based Access Control

Users are assigned roles. Roles have permissions. Users inherit the permissions of their roles.

```mermaid
graph LR
    User1[Vaishali] --> Role1[Role: Engineer]
    User2[Rahul] --> Role2[Role: Admin]
    User3[Priya] --> Role1

    Role1 --> P1[Read code repos]
    Role1 --> P2[Create pull requests]
    Role2 --> P1
    Role2 --> P2
    Role2 --> P3[Delete users]
    Role2 --> P4[Manage billing]
```

Instead of assigning permissions to each user individually — which becomes unmanageable at scale — you assign roles. When permissions need to change, you update the role once, and all users with that role are updated automatically.

**Real examples:** GitHub's organization roles (Owner, Member, Billing Manager). AWS IAM roles. Database user roles (read-only, read-write, admin).

### ACL — Access Control List

An ACL is a list of permissions attached to a specific resource. It defines exactly which users or roles can do what with that specific thing.

```
File: report-q4.pdf
  - Vaishali: READ, WRITE
  - Rahul: READ
  - Everyone else: NONE

API endpoint: DELETE /users/:id
  - Role: Admin → ALLOWED
  - Role: Engineer → DENIED
  - Role: Viewer → DENIED
```

ACLs give fine-grained control at the resource level. RBAC gives broad role-level control. Production systems often use both together.

---

## Interview Questions

**Authentication vs Authorization**
1. What is the difference between authentication and authorization?
2. A user is logged in but tries to access another user's data. Which HTTP code do you return and why?
3. In what order do authentication and authorization happen?

**JWT**
1. What is a JWT? What are its three parts?
2. Why is JWT good for stateless, horizontally scalable systems?
3. Can anyone read the JWT payload? Is it encrypted?
4. What is the `exp` claim? Why must you always set it?
5. What is the difference between an access token and a refresh token?
6. If a JWT is stolen, how do you invalidate it? (This is intentionally hard — JWT has no built-in revocation.)
7. Where should you store a JWT in the browser — localStorage or httpOnly cookie? What are the trade-offs?

**OAuth 2.0**
1. What is OAuth 2.0? What problem does it solve?
2. Walk through the OAuth 2.0 authorization code flow step by step.
3. Why does OAuth use an authorization code first, instead of sending the token directly?
4. What are OAuth scopes? Give an example.
5. What is the difference between OAuth 2.0 and OpenID Connect?

**SSO**
1. What is SSO? How does it work?
2. What is an Identity Provider?
3. What happens to all app access when an employee leaves a company that uses SSO?
4. What is the difference between SAML and OpenID Connect?

**Access Control**
1. What is RBAC? How is it different from assigning permissions directly to users?
2. What is an ACL? When would you use it over RBAC?
3. Design the access control system for a GitHub-like platform — what roles would you create and what permissions would each have?

---