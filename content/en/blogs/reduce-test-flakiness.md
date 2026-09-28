---
title: "Reduce test flakiness"
date: 2026-07-20
draft: false
tags: ["Flakiness", "CI/CD", "Test Automation"]
description: "Common causes of flaky tests in CI and concrete tactics to drive flakiness down without slowing delivery."
---

# Why your tests are flaky, and four ways to fix it

A flaky test passes on your machine, fails in CI, then passes again on retry with no code change in between. Flakiness is not a minor annoyance: it trains your team to ignore red builds, which is the same as having no tests at all. Most flaky tests trace back to four causes, and each one has a concrete fix.

## Cause 1: tests that cover too much

A test that logs in, creates an order, applies a discount, checks the invoice, and verifies the email all in one run is testing five things with one assertion path. If any step is slow, order-dependent, or touches a shared resource, the whole test is fragile, and when it fails you cannot tell which step broke without re-reading the whole thing.

The fix is to write atomic tests. Each test should exercise one behavior and have one reason to fail. Split the flow above into: a test for login, a test for order creation, a test for discount calculation, a test for invoice generation. Keep one or two true end-to-end tests to prove the pieces connect, and push everything else down to smaller, faster, independent tests.

```java
// Before: one test, five failure points
@Test
void checkoutFlowWorksEndToEnd() { ... }

// After: one behavior per test
@Test
void login_withValidCredentials_returnsSessionToken() { ... }

@Test
void discount_appliesTenPercentForLoyaltyMembers() { ... }
```

Smaller tests fail with a clear name and a clear cause. That alone removes a large share of "why did this fail" debugging time.

## Cause 2: tests that share data

If test A creates a user named `test@example.com` and test B deletes every user matching `test@`, whoever runs second depends on run order. Order dependence is a classic source of flakiness: the suite passes in one CI runner and fails in another because parallel execution scheduled the tests differently.

The fix is isolation: each test creates the data it needs and cleans up after itself, without assuming what other tests left behind.

```java
@BeforeEach
void setUp() {
    testUser = userFactory.createUnique(); // unique email/id per run
}

@AfterEach
void tearDown() {
    userRepository.delete(testUser);
}
```

A few practical patterns that keep isolation cheap:

- Generate unique identifiers per test run (a UUID suffix on emails, usernames, order IDs) instead of hardcoded strings.
- Prefer a fresh transaction rolled back after each test over deleting rows by hand.
- If tests share a database, run them in separate schemas or containers rather than trusting cleanup code to catch everything.

## Cause 3: tests that assume things happen instantly

Async work is the other big source of flakiness: a message gets published, a background job processes it, a cache gets invalidated. If your test asserts the result immediately after triggering the action, it is racing the system under test, and it will lose that race under load.

`Thread.sleep(2000)` is the usual workaround, and it is a bad one. Sleep too little and the test still flakes; sleep too much and your suite gets slow on every run, pass or fail.

The fix is a graceful retry that polls until the condition is true or a timeout is reached. Awaitility does this well in the JVM world:

```java
await()
    .atMost(5, SECONDS)
    .pollInterval(100, MILLISECONDS)
    .until(() -> orderRepository.findById(orderId).getStatus() == SHIPPED);
```

This checks every 100ms instead of waiting a fixed amount, so the test finishes as soon as the condition is met and only fails if it genuinely never happens within the timeout. The failure message also tells you what condition was never met, which is far more useful than a stack trace pointing at a sleep call.

## Cause 4: tests that depend on the outside world

If your test hits a real third-party API, its result now depends on that provider's uptime, rate limits, and network latency, none of which your team controls. A payment gateway's staging environment having a slow afternoon should not turn your build red.

The fix is to mock or stub every external HTTP call in your test suite. Your test should verify how your code behaves given a response, not whether the third party is currently reachable.

```java
stubFor(post(urlEqualTo("/v1/charges"))
    .willReturn(aResponse()
        .withStatus(200)
        .withBody("{\"status\": \"succeeded\"}")));
```

Tools like WireMock, MockServer, or your language's HTTP mocking library let you simulate both the happy path and the failure modes (timeouts, 500s, malformed responses) on demand, which is something a real sandbox environment rarely lets you do reliably. As a side effect, your tests also get faster, since they no longer wait on real network round trips.

## Putting it together

None of these fixes are exotic. Atomic tests give you one failure point per test. Isolated data removes order dependence. Polling with a timeout replaces fixed sleeps with a check that resolves as soon as it can. Mocking external calls removes the internet as a variable in your build.

Applied together, they turn a flaky suite into one your team trusts again, which is the actual goal: a red build that means something broke, not a coin flip you rerun until it goes green.
