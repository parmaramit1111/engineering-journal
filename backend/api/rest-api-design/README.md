---
Article Type: Engineering Guide
Category: Backend / API
Technology: REST API
Difficulty: Intermediate
Tags: REST API, API Design, Backend Engineering, Software Architecture, API Development
---

# REST API Design: Practical Decisions That Matter

**Estimated Reading Time:** 10 Minutes

## Overview

A REST API can be easy to build.

Designing one that remains predictable, maintainable, and easy to evolve is a different problem.

As an application grows, small API decisions start becoming important:

- How should resources be represented?
- Which HTTP methods should be used?
- How should APIs be versioned?
- How should pagination work?
- How should filtering and sorting be handled?
- What should an error response look like?
- How should duplicate requests be handled?
- How do we change an API without breaking existing clients?

There is no single API design that works for every application.

The goal is to create an API that is **consistent, understandable, and safe to evolve**.

This article covers the practical decisions I consider when designing REST APIs.

---

# Start With the Resource

A good API usually starts with a clear representation of the resource.

For example:

```text
GET /api/orders
GET /api/orders/{id}
POST /api/orders
PUT /api/orders/{id}
DELETE /api/orders/{id}
```

The resource is the important part:

```text
/orders
```

rather than creating endpoints that describe implementation details.

For example, this:

```text
GET /api/getAllOrders
```

is less useful than:

```text
GET /api/orders
```

The second one describes the resource rather than the operation.

That makes the API easier to understand and keeps the URL structure consistent.

---

# Use HTTP Methods Consistently

HTTP methods already communicate intent.

A common pattern is:

| Method | Purpose |
|---|---|
| GET | Read |
| POST | Create |
| PUT | Replace |
| PATCH | Partially update |
| DELETE | Delete |

For example:

```text
GET    /api/orders/123
POST   /api/orders
PUT    /api/orders/123
PATCH  /api/orders/123
DELETE /api/orders/123
```

The important thing is consistency.

If one API uses `POST` for updates while another uses `PUT`, clients have to learn different rules for different endpoints.

Predictability is more valuable than cleverness.

---

# Return Meaningful HTTP Status Codes

HTTP status codes are part of the API contract.

For example:

```text
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
500 Internal Server Error
```

The exact set depends on the application, but the principle is simple:

> **The status code should help the client understand what happened.**

For example:

A successful creation:

```http
201 Created
```

A resource that does not exist:

```http
404 Not Found
```

A request that conflicts with the current state:

```http
409 Conflict
```

Returning `200 OK` for every situation makes the API harder for clients to reason about.

---

# Keep Error Responses Consistent

Error handling is often overlooked until different clients start consuming the API.

A consistent error structure makes life much easier for frontend applications, integrations, and other services.

For example:

```json
{
  "code": "ORDER_NOT_FOUND",
  "message": "The requested order was not found.",
  "details": null,
  "traceId": "7f1c9b2a..."
}
```

The exact structure can vary.

What matters is consistency.

Clients should not have to parse five different error formats depending on which endpoint failed.

A useful error response should generally provide:

- A stable error code
- A readable message
- Optional validation details
- A correlation or trace identifier

The trace identifier is particularly useful when investigating production problems.

---

# Validate at the API Boundary

Validation should happen as early as practical.

If an API expects:

```json
{
  "pageSize": 50
}
```

and the client sends:

```json
{
  "pageSize": -10
}
```

the API should reject the request before unnecessary application or database work happens.

Validation can include:

- Required fields
- Data types
- Length limits
- Numeric ranges
- Allowed values
- Format validation
- Business-independent request rules

But validation should not turn the controller into a large business-logic layer.

A useful separation is:

```text
API
 ↓
Request Validation
 ↓
Application Logic
 ↓
Business Rules
 ↓
Data Access
```

Each layer has a clear responsibility.

---

# Design Pagination Early

Returning every record from an endpoint may work with a small dataset.

It becomes a problem when the data grows.

Instead of:

```text
GET /api/orders
```

returning thousands or millions of records, define a predictable pagination model.

For example:

```text
GET /api/orders?page=1&pageSize=50
```

The response could contain:

```json
{
  "items": [],
  "page": 1,
  "pageSize": 50,
  "totalCount": 1240
}
```

The exact response model can vary.

The important part is to define pagination consistently.

For very large datasets, cursor-based pagination can also be considered.

The right approach depends on the workload.

---

# Filtering and Sorting

As APIs become more useful, clients usually need more than a simple list.

For example:

```text
GET /api/orders?status=completed
```

or:

```text
GET /api/orders?status=completed&sort=createdAt
```

The important consideration is to avoid creating a new endpoint for every combination of filters.

Instead:

```text
GET /api/orders?status=completed&customerId=123
```

is generally easier to maintain than:

```text
GET /api/completed-orders
GET /api/customer-orders
GET /api/customer-completed-orders
```

Filtering should also have clear limits.

Do not allow clients to construct arbitrary database queries through an API.

The API should expose supported filters deliberately.

---

# API Versioning

APIs eventually change.

The question is not whether an API will change.

It is how safely it can change.

One common approach is URL-based versioning:

```text
/api/v1/orders
/api/v2/orders
```

Other approaches include headers or media types.

There is no single correct strategy.

The important part is having a versioning strategy before breaking changes become necessary.

For example, changing:

```json
{
  "customerName": "Amit"
}
```

to:

```json
{
  "customer": {
    "name": "Amit"
  }
}
```

may be a breaking change for existing clients.

Versioning provides a boundary between the old contract and the new one.

---

# Backward Compatibility

Versioning alone does not solve every compatibility problem.

Small changes can also break clients.

For example:

- Renaming a response field
- Removing a property
- Changing a data type
- Changing an error structure
- Changing pagination behavior
- Changing the meaning of an existing status

When designing an API, I try to think about:

> **Who is consuming this API, and how difficult will it be for them to adapt to change?**

This becomes particularly important for APIs used by external partners or long-lived applications.

---

# Idempotency Matters

Some operations can be repeated safely.

Others cannot.

For example:

```text
GET /api/orders/123
```

can normally be repeated without changing the resource.

A payment or order creation request is different.

A client may send the same request twice because of:

- Network timeout
- Client retry
- Connection failure
- User action
- Message redelivery

Without protection, the same operation could happen twice.

For operations where duplicate processing is dangerous, an idempotency key can help.

For example:

```http
Idempotency-Key: 9f7a2c1e-...
```

The server can use that key to recognize a repeated request and avoid performing the same operation twice.

The implementation depends on the business workflow, but the design question should be considered early.

---

# Don't Put Business Logic in Controllers

A controller should not become the place where everything happens.

Avoid turning this:

```text
Controller
    ↓
Validation
    ↓
Business Rules
    ↓
Database Queries
    ↓
External API Calls
    ↓
Response Mapping
```

into one large method.

A cleaner flow is:

```text
HTTP Request
      ↓
Controller
      ↓
Application Service / Command
      ↓
Business Logic
      ↓
Repository / Provider
      ↓
Response
```

This makes the API layer easier to test and easier to maintain.

It also keeps the API contract separate from internal implementation details.

---

# Authentication and Authorization Are Different

Authentication answers:

> **Who are you?**

Authorization answers:

> **What are you allowed to do?**

An API should not treat them as the same problem.

For example:

```text
Authenticated User
       ↓
Has Access to API
       ↓
Authorization Check
       ↓
Can Access This Resource?
```

This becomes especially important when an API exposes data belonging to multiple users, customers, organizations, or tenants.

A valid token does not automatically mean the caller should be allowed to access every resource.

---

# Timeouts and External Dependencies

APIs often depend on other services.

For example:

```text
API
 ↓
Payment Service
 ↓
External Provider
```

If the external provider becomes slow, the API can also become slow.

Every external dependency should therefore have appropriate controls such as:

- Timeouts
- Retry policies where appropriate
- Circuit breaking where appropriate
- Clear failure handling
- Correlation IDs
- Logging

Retries should not be added blindly.

A retry can make an outage worse if hundreds of requests repeatedly call an already struggling dependency.

The retry strategy should consider whether the operation is safe to repeat.

---

# Logging and Correlation IDs

When an API request fails in production, the response alone is rarely enough to understand what happened.

A correlation or trace ID provides a way to connect:

```text
API Request
   ↓
Application Logs
   ↓
Database Logs
   ↓
External Service Logs
```

For example:

```text
TraceId: 7f1c9b2a...
```

The same identifier can help engineers follow a request across multiple components.

This becomes increasingly valuable as systems become distributed.

---

# Don't Return Internal Details

API responses should provide useful information without exposing internal implementation details.

Avoid returning:

```text
SQL exception
Stack trace
Database connection information
Internal file paths
```

to clients.

Instead, return a stable error response and log the technical details internally.

For example:

```json
{
  "code": "INTERNAL_ERROR",
  "message": "Something went wrong while processing the request.",
  "traceId": "7f1c9b2a..."
}
```

The client gets enough information to understand that the operation failed.

The engineering team gets the trace ID needed to investigate.

---

# Keep the API Contract Stable

An API is a contract.

Once other systems depend on it, changing it becomes more expensive.

That means API design should consider:

```text
Clients
   ↓
API Contract
   ↓
Application
   ↓
Database / Services
```

The database schema can change internally without necessarily changing the API contract.

That separation is valuable.

It allows implementation details to evolve while keeping the external interface stable.

---

# Don't Over-Design the API

There is also a risk in the other direction.

It is possible to spend too much time designing an API for problems that do not exist yet.

Not every API needs:

- Five versions
- Complex filtering
- Cursor pagination
- GraphQL
- Event-driven architecture
- Multiple gateways
- Distributed tracing
- Advanced rate limiting

The right level of design depends on the actual system.

A small internal API may need very little infrastructure.

A public API used by thousands of clients will need much more attention.

The goal is not to build the most sophisticated API.

The goal is to build the **simplest API that can safely support the requirements**.

---

# A Practical API Design Checklist

Before releasing an API, I like to ask:

```text
Resource Design
    ↓
Are the resources clear?

HTTP Contract
    ↓
Are methods and status codes consistent?

Validation
    ↓
Are invalid requests rejected early?

Pagination
    ↓
Can the endpoint handle growing data?

Filtering
    ↓
Are supported filters clear and controlled?

Errors
    ↓
Are error responses consistent?

Versioning
    ↓
How will breaking changes be handled?

Security
    ↓
Are authentication and authorization separated?

Reliability
    ↓
Are timeouts and retries appropriate?

Observability
    ↓
Can a failed request be traced?

Compatibility
    ↓
Can existing clients continue working?
```

This checklist is simple, but it catches many problems before they reach production.

---

# What I Learned

Good API design is less about memorizing REST rules and more about making good boundaries.

The API should make it clear:

- What resources exist
- What operations are supported
- What the client can expect
- What happens when something goes wrong
- How the API can evolve

The most useful APIs I have worked with tend to have something in common:

> **They are predictable.**

A developer should not need to guess how pagination works on one endpoint and how it works on another.

They should not need to learn five different error formats.

They should not need to understand the database structure to use the API.

And they should be able to upgrade to a new API version without unexpected surprises.

---

# Final Thoughts

REST API design does not need to be complicated.

The difficult part is making small decisions consistently.

A good API provides a clear contract between the client and the system.

It hides internal implementation details while exposing the operations that clients actually need.

It handles errors predictably.

It considers performance and reliability.

And most importantly, it can evolve without constantly breaking its consumers.

The goal is not to follow every REST rule perfectly.

The goal is to build an API that another engineer can understand quickly and use confidently.

> **A good API is predictable today and safe to change tomorrow.**

---

# Key Takeaways

* Design APIs around clear resources rather than implementation details.
* Use HTTP methods and status codes consistently.
* Keep error responses predictable.
* Validate requests at the API boundary.
* Design pagination before datasets become large.
* Expose filtering and sorting deliberately.
* Have a clear versioning strategy.
* Think about backward compatibility before making breaking changes.
* Use idempotency for operations where duplicate processing is a risk.
* Keep business logic outside controllers.
* Separate authentication from authorization.
* Use timeouts and carefully designed retry policies for external dependencies.
* Use correlation IDs to make production troubleshooting easier.
* Never expose internal implementation details through API errors.
* Keep the API contract stable even when internal implementation changes.
* Avoid over-engineering APIs before the requirements justify it.
* **A good API is predictable today and safe to change tomorrow.**

---

# Related Engineering Topics

- REST API
- API Design
- Backend Engineering
- HTTP
- API Versioning
- Pagination
- API Security
- Authentication
- Authorization
- Idempotency
- API Reliability
- Software Architecture
- Distributed Systems
- Production Troubleshooting

---

**Article Version:** 1.0

**First Published:** August 2026

**Last Reviewed:** August 2026
