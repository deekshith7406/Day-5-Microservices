# Day 5 — Service-to-Service Communication (standalone exercise)

## Why this is a separate project from `one-enterprise-platform`

Days 2–4 all said "continue using your existing services." Day 5 doesn't —
it walks through building the User Service → Order Service call **from
scratch**, using a different HTTP client (Spring's newer `RestClient`
instead of `RestTemplate`), plain endpoint paths (`/users/{id}`, not
`/api/users/{id}`), and record-based DTOs. That's a deliberate fresh
teaching exercise, not another increment to the bigger platform — so it
lives in its own folder rather than being bolted onto Days 1–4's project.

If you're looking for the accumulated platform (gateway, resilience
patterns, payment service, etc.), that's the separate
`one-enterprise-platform` project from Days 1–4.

```
day5-service-communication/
├── user-service/    → provider API, port 8081
└── order-service/    → consumer, calls user-service via RestClient, port 8082
```

## Prerequisites

- Java 17+
- Maven 3.8+

## Running it

**Terminal 1 — User Service (port 8081)**
```bash
cd user-service
mvn spring-boot:run
```

**Terminal 2 — Order Service (port 8082)**
```bash
cd order-service
mvn spring-boot:run
```

## Step 3/4 — the successful path (Hands-on Challenge 1)

```bash
curl http://localhost:8081/users/1001
# {"id":1001,"name":"John","email":"john@example.com"}

curl http://localhost:8082/orders/5001
# {"orderId":5001,"userId":1001,"userName":"John"}
```

The second call only succeeds because Order Service made a real HTTP
request to User Service and used the response — check User Service's
console log for the incoming request if you want to see it happen.

## Step 5 / Hands-on Challenge 2 — break it on purpose

```bash
# Stop User Service (Ctrl+C in Terminal 1), then:
curl -i http://localhost:8082/orders/5001
```

You'll get:
```json
{
  "error": "USER_SERVICE_UNAVAILABLE",
  "message": "User service is unavailable. Please try again later.",
  "status": 503,
  "timestamp": "..."
}
```

That response comes from `GlobalExceptionHandler`, not a try/catch in
`OrderController` — the handbook explicitly prefers centralized handling
over scattering try/catch blocks through controllers, so that's what this
does.

Restart User Service and confirm the normal response comes back:
```bash
curl http://localhost:8082/orders/5001
# back to {"orderId":5001,"userId":1001,"userName":"John"}
```

## Automated tests

`OrderControllerTest` covers both paths from the "Mini Project" checklist
without needing two live processes during a build — `UserClient` is
mocked so each test controls exactly what "User Service" does:

```bash
cd order-service
mvn test
```

- `returnsOrderWithUserDetails_whenUserServiceRespondsSuccessfully` — the
  successful end-to-end path
- `returns503_whenUserServiceIsUnavailable` — User Service unreachable

(These are the practical equivalent of the handbook's "a successful
end-to-end test" and "a test for User Service being unavailable" —
a true multi-process end-to-end test is the manual curl walkthrough above.)

## Section 12 — configuration mini-exercise

Try this yourself to confirm the URL really is external to the Java code:

1. Stop User Service.
2. Change its port: edit `user-service/src/main/resources/application.properties`,
   set `server.port=9091`.
3. Update **only** Order Service's config — edit
   `order-service/src/main/resources/application.properties`,
   set `user.service.base-url=http://localhost:9091`.
4. Restart both services.
5. Call `curl http://localhost:8082/orders/5001` again — it should work,
   and you never touched `UserClient.java` or `RestClientConfig.java`.
6. Put both files back to `8081` / `http://localhost:8081` afterward.

## Connecting today to Day 4

The handbook asks you to map this call onto yesterday's resilience
concepts. This project intentionally does **not** implement timeout,
retry, or a circuit breaker — Day 5's focus is the communication mechanics
themselves — but here's where each would slot in if it did:

| Day 4 pattern | Where it would go here |
|---|---|
| **Timeout** | On the `RestClient` — Spring's `RestClient` delegates to an underlying `ClientHttpRequestFactory`; you'd configure its connect/read timeout, same idea as `order-service`'s `paymentRestTemplate` in the Day 1–4 platform project. |
| **Retry** | Around `UserClient.getUser(...)`, limited to a few attempts, only for exceptions that look transient (connection refused, timeout) — not for a legitimate 4xx. |
| **Circuit breaker** | Also around `UserClient.getUser(...)`, so repeated failures stop hitting User Service for a cooldown period instead of retrying forever. |
| **Fallback** | Already partially here in spirit — `GlobalExceptionHandler` turns any `RestClientException` into a clear 503, never a raw stack trace. A "real" fallback would live at the same layer, potentially returning cached/default data instead of just an error. |

If you want to see all four of these actually implemented and wired up
with Resilience4j against a deliberately-breakable dependency, that's
exactly what Day 4 built in the `one-enterprise-platform` project
(`order-service`'s `PaymentServiceClient` + `RESILIENCE.md`).

## What changed between Section 5 and Section 11 in this repo

- `RestClientConfig` in this project already reflects **Section 11's**
  fix (externalized `user.service.base-url`), not Section 5's hard-coded
  version — so you're looking at the "after" state. If you want to
  experience the "before" state the handbook describes, try hard-coding
  `.baseUrl("http://localhost:8081")` directly in `RestClientConfig`
  first, get it working, and then refactor to the externalized version
  yourself — that refactor *is* the Section 11 exercise.
