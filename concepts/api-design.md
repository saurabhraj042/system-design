# API Design

## The Core Idea

An API is the contract between a client and a server. In a system design interview, API design typically takes about 5 minutes — enough to show you can identify the right resources, choose the right HTTP verbs, and handle common patterns like pagination and authentication. Most interviewers care more about your architectural decisions than your exact endpoint syntax.

The overarching principle: APIs should be predictable. A developer who has used your API once should be able to guess how any other endpoint works.

## Mental Model / Analogy

Designing a REST API is like designing a restaurant's ordering system. The menu (resources) lists what the kitchen can produce — "pizzas," "drinks," "desserts" — not what actions the kitchen does ("cook pizza," "pour drink"). When you order, you use standard phrases (HTTP verbs): "I'd like one" (GET to fetch), "add this to my order" (POST to create), "swap this item" (PUT to replace), "remove this" (DELETE). The bill (response) tells you whether it worked and what you're getting. The waiter (API gateway) routes between you and the kitchen without you needing to know which cook handles what.

---

## Choosing the Right API Type

Three main protocols come up in interviews:

### REST (Default Choice)

REST maps HTTP verbs to CRUD operations on resources (nouns). For web and mobile applications accessing a backend, REST is the standard — it's well-understood, has great tooling, and works for 90% of use cases.

**Pick REST unless you have a specific reason not to.** If you're unsure, say "I'll use REST" and move on.

### GraphQL (When Over/Under-Fetching Is a Problem)

GraphQL uses a single endpoint with a query language that lets clients specify exactly what data they need.

**When to suggest GraphQL:**
- Different clients (mobile vs web) need different shapes of the same data
- The interviewer uses phrases like "the mobile app shouldn't download the full object" or "avoiding over-fetching"
- Frontend teams need to iterate quickly without backend changes

**The trade-off:** GraphQL adds complexity — query parsing, schema validation, and the N+1 problem (fetching 100 events with their venues spawns 101 queries: 1 for events + 100 for each venue). Requires DataLoader/batching patterns. Most interviewers prefer you solve core architectural challenges with simpler tools first.

**Decision rule:** If the interviewer specifically mentions over-fetching, under-fetching, or different client data needs → mention GraphQL. Otherwise, don't reach for it.

### gRPC / RPC (For Internal Service Communication)

RPC frameworks like gRPC let services call each other as if they were local function calls. gRPC uses Protocol Buffers (binary) over HTTP/2, making it faster than JSON REST for service-to-service traffic.

**When to suggest gRPC:**
- Internal microservice-to-service communication
- Performance-critical paths between services you control
- Multi-language service mesh (generated client stubs for any language from a single `.proto` file)

**Decision rule:** Client-facing API → REST or GraphQL. Internal service-to-service → gRPC. In interviews, you typically only need to define the client-facing APIs; you can mention "services communicate internally via gRPC" during the high-level design.

---

## Designing REST APIs

### 1. Resource Modeling

Resources are **things** (nouns), not **actions** (verbs). Resources match your core entities.

For Ticketmaster: events, venues, tickets, bookings.

```
GET  /events              # list events
GET  /events/{id}         # get one event
GET  /venues/{id}         # get one venue
GET  /events/{id}/tickets # tickets for a specific event
POST /events/{id}/bookings # create a booking for an event
GET  /bookings/{id}       # get a specific booking
```

**Resources should be plural nouns:** `/events`, not `/event`. Not all interviewers enforce this, but it's easy to get right.

**Nested vs. flat resources:**
- **Nested** (`/events/{id}/tickets`): use when the relationship is required — tickets always belong to an event, the event ID is mandatory
- **Flat with query params** (`/tickets?event_id=123`): use when the relationship is optional — you might want all tickets, or tickets for a specific event, or tickets filtered by section

### 2. HTTP Methods

| Method | Idempotent? | What it does | Example |
|--------|-------------|--------------|---------|
| **GET** | Yes | Retrieve data, no side effects | `GET /events/123` |
| **POST** | No | Create a new resource | `POST /events/123/bookings` |
| **PUT** | Yes | Replace an entire resource | `PUT /users/456` (full user object) |
| **PATCH** | Not guaranteed | Update part of a resource | `PATCH /users/456` (just the email field) |
| **DELETE** | Yes | Remove a resource | `DELETE /bookings/789` |

**Idempotency matters for retries:** If a network failure causes the client to retry, idempotent requests are safe to re-send. POST is not idempotent — a retry creates a duplicate booking. For non-idempotent operations at risk of retry, add an **idempotency key** (client-generated unique ID in a header; server uses it to detect and ignore duplicate submissions).

### 3. Passing Data

Three locations for input data:

**Path parameters** — identify which specific resource you're operating on. Required to make the request meaningful.
```
GET /events/123         # 123 is required; without it, the request doesn't make sense
```

**Query parameters** — optional filters, sorting, pagination. The resource can be retrieved without them.
```
GET /events?city=NYC&date=2026-03-15&page=2&limit=20
```

**Request body** — structured data for creates and updates; anything complex or sensitive.
```
POST /events/123/bookings
{
  "tickets": [{"section": "VIP", "quantity": 2}],
  "payment_method": "credit_card"
}
```

**Decision rule:** path param when required to identify the resource; query param when optional filter; body when you're sending complex structured data.

### 4. Response Codes

Stick to the common ones. Interviewers care more that you know the difference between 4xx (client error) and 5xx (server error) than that you memorize every code.

| Code | Meaning | When to use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input from client |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Something broke server-side |

---

## Common API Patterns

### Pagination

Never return unbounded result sets. A single API that returns "all events" could return millions of records.

**Offset-based pagination** (simple, good for most interviews):
```
GET /events?offset=20&limit=10    # get records 21-30
```
Easy to implement, easy to understand. Breaks at large offsets (offset 10,000 requires loading and discarding 10,000 records) and has a race condition: records added between page 1 and page 2 requests can cause duplicates or skipped items.

**Cursor-based pagination** (stable, better for real-time data):
```
# First request
GET /events?limit=10
→ { "events": [...], "next_cursor": "eyJpZCI6IjUwIn0" }

# Second request  
GET /events?cursor=eyJpZCI6IjUwIn0&limit=10
```
The cursor encodes a pointer to the last item seen (typically the last item's ID or timestamp). Stable regardless of insertions/deletions between pages. Harder to implement "jump to page 5."

**Interview default:** Offset-based is fine unless the interviewer mentions real-time data, large datasets, or asks specifically about consistency. Most interviewers care that you remembered to add pagination at all.

### Versioning

APIs evolve; clients can't all update simultaneously. Two approaches:

**URL versioning** (recommended for interviews):
```
GET /v1/events          # old version
GET /v2/events          # new version with different response shape
```
Explicit and obvious. Clients know exactly which version they're using. Easy to deprecate (sunset the /v1 routes).

**Header versioning:**
```
GET /events
Accept-Version: v2
```
Cleaner URLs but less discoverable and harder to test in a browser.

Use URL versioning in interviews unless asked otherwise.

---

## Security Considerations

### Authentication vs Authorization

**Authentication:** Who are you? (Verify identity)
**Authorization:** What can you do? (Verify permissions)

Example: Authentication confirms you are Alice. Authorization confirms Alice is allowed to cancel booking #789 (which belongs to Alice, not Bob).

### API Keys vs JWT Tokens

**API Keys:**
- Long random strings, stored server-side with associated permissions
- Generated per client application (not per user)
- Best for server-to-server communication (your booking service calling your payment service) and third-party developer access
- Not suitable for user-facing apps: users shouldn't manage API keys, and they don't carry user context

```
GET /events
Authorization: Bearer sk_live_abc123def456...
```

**JWT (JSON Web Tokens):**
- Self-contained token: encodes user_id, roles, expiration, signed with a secret key
- No database lookup on each request — the server verifies the signature and reads the payload
- Best for user sessions in web and mobile apps
- Works well in distributed systems: any service with the secret key can verify the token independently

```json
{
  "user_id": "123",
  "email": "alice@example.com",
  "role": "customer",
  "exp": 1762992000
}
```

**Decision rule:** Internal service calls → API keys. User-facing sessions → JWT tokens.

### Role-Based Access Control (RBAC)

Assign permissions to roles, assign roles to users. In Ticketmaster:
- `customer`: book tickets, view own bookings
- `venue_manager`: create events, view venue sales
- `admin`: access everything

For an interview, it's enough to say "endpoints that modify data require authentication via JWT, and sensitive operations (like deleting events) require the `venue_manager` role."

### Rate Limiting

Prevents abuse and protects your system from accidental overload:
- Per-user: 1,000 requests/hour per authenticated user
- Per-IP: 100 requests/hour for unauthenticated requests  
- Per-endpoint: 10 booking attempts/minute (prevent ticket scalpers)

Return `429 Too Many Requests` when limits are exceeded.

In interviews: "We'll implement rate limiting at the API gateway level" is sufficient. You don't need to design the token bucket algorithm unless asked.

---

## GraphQL Deep Dive (When Needed)

If you choose GraphQL, design a schema instead of resource endpoints:

```graphql
type Event {
  id: ID!
  name: String!
  date: DateTime!
  venue: Venue!
  tickets: [Ticket!]!
}

type Query {
  event(id: ID!): Event
  events(city: String, after: String, limit: Int): [Event!]!
}

type Mutation {
  createBooking(eventId: ID!, tickets: [TicketInput!]!): Booking
}
```

The client specifies exactly which fields it needs; the server returns exactly that structure.

**The N+1 problem:** Querying 100 events with their venues naively runs 1 query for events + 100 queries for each venue = 101 queries. The fix is **DataLoader** (batching pattern): collect all venue IDs needed, run a single `SELECT WHERE id IN (...)`, and distribute results back to each event. Adds complexity you don't have with REST.

---

## Key Trade-offs

| | REST | GraphQL | gRPC |
|--|------|---------|------|
| Client flexibility | Fixed responses | Client specifies shape | Fixed contracts (proto) |
| Performance | Good | Good (with DataLoader) | Best (binary, HTTP/2) |
| Caching | Standard HTTP caching | Complex (single endpoint) | Less standard |
| Learning curve | Low | Medium | Medium |
| Browser support | Native | Native | Requires proxy |
| Best for | Most things | Multiple client types | Internal services |

## When to Use

- **REST:** Default. Pick for any standard web/mobile API
- **GraphQL:** Multiple clients with different data needs; frontend team needs rapid iteration without backend changes
- **gRPC:** Internal microservice communication; performance-critical service-to-service calls; polyglot environments

## Common Gotchas

- **Verbs in REST URLs:** `/getEvent` or `/createBooking` are not REST — the verb is the HTTP method, the URL is the noun
- **Skipping pagination:** Returning all records without pagination will eventually OOM your server or kill client performance
- **POST for idempotent operations:** Use PUT or include idempotency keys for operations that can safely retry
- **Forgetting rate limiting:** Easy to add ("we'll implement at the gateway") and shows production thinking; forgetting it is a miss

## Practice Questions

1. Design the REST API for a social media platform (Twitter-like). Cover: creating posts, following users, fetching a user's timeline. Include HTTP methods, path structure, request/response shapes, and pagination.
2. Your mobile app and web app both query the same user endpoint, but mobile needs only name and avatar while web needs the full profile including activity history. Discuss REST vs GraphQL for this scenario.
3. You're building a public API for third-party developers to access event data, and an internal API for your own booking service to process payments. What authentication mechanism do you choose for each and why?
4. Explain why pagination is important and walk through the difference between offset-based and cursor-based pagination. Give a concrete example where offset-based breaks down.
5. Your payment API is getting duplicate charges because clients retry timed-out requests. Walk through how you'd design an idempotency key system to prevent this.

## One-Line Summary

Default to REST with noun-based resources, appropriate HTTP verbs, and standard patterns for pagination/auth — reach for GraphQL when clients have varying data needs, and gRPC for internal service-to-service communication.
