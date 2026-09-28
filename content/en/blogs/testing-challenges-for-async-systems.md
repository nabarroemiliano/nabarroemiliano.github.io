---
title: "Testing asynchronous systems"
date: 2026-09-01
draft: false
tags: ["Testing", "Async", "Distributed Systems"]
description: "Challanges faced while testing event-driven systems and patterns that help."
---

If you've ever used an app that updates in real time (a delivery tracker, a bank transfer, a white-boarding session), you've relied on what's called an **asynchronous system**, or "async" for short. It means different parts of the software don't wait for each other to finish before moving on. Multiple tasks run concurrently, and each completes when it completes, not necessarily in the order you'd expect.

This makes apps fast, responsive, and able to scale to handle many users at once, since the system isn't stuck waiting on one task before starting the next. It also makes the whole system genuinely hard to test.

## The Problem: There's No Fixed Order or Timing

In traditional software, testing is straightforward: you do one thing, wait for the result, and check if it's correct. Simple, predictable, easy to verify.

Async systems don't work that way. A single action, like placing an order, might trigger several independent processes: charging a payment, updating a database, sending a confirmation email, notifying a delivery service. Each of these can take a different amount of time depending on server load, network speed, or other factors outside anyone's control. Sometimes they finish in a normal, expected order. Sometimes they don't.

When they don't, you get bugs that are hard to catch: a confirmation email arriving before a payment has actually cleared, or a status update showing "delivered" while the item is still in transit.

The bigger issue is that these bugs are **inconsistent**. A test can pass 99% of the time and then fail the remaining 1% because two background processes happened to finish in a different order that time. These are called "flaky tests," and they're notoriously frustrating because they don't fail for a clear, repeatable reason. I talk more about reducing test flakiness [here]({{< relref "/blogs/reduce-test-flakiness" >}}).

## Why This Affects You Directly

This is the reason:

- Your bank app sometimes briefly shows an outdated balance after a transfer
- A "like" on social media doesn't always appear instantly
- Order confirmation emails sometimes arrive later than expected

None of this necessarily means the software is poorly built. It usually means multiple processes are running independently, and occasionally one takes slightly longer than the others (something a simple, linear test would never catch).

## The Fix: Wait for the Actual Result, Not a Guessed Amount of Time

The most common (and weakest) way to test async systems is to add a fixed delay: "wait 3 seconds, then check the result." This is easy to write, but unreliable. Three seconds might be too short under heavy load, causing the test to fail even though the system is working correctly. Or it might be far longer than necessary, slowing down testing for no reason.

The stronger approach doesn't guess a time at all. Instead, the test repeatedly checks the actual condition it cares about, for example, "has the order status changed to 'Confirmed'?", checking every fraction of a second until that specific condition is true, then immediately moving forward. This is commonly called **polling** or using **event-based waits**. The test reacts to what's actually happening, rather than assuming how long it should take.

A popular tool for this in Java is Awaitility. Instead of writing manual retry loops, it lets you describe what you're waiting for in plain, readable code:

```java
await()
    .atMost(30, SECONDS)
    .until(() -> orderStatus.get().equals("Confirmed"));
```

That's the whole idea in one line: keep checking, for up to 30 seconds, until the order status actually says "Confirmed." No fixed delay, no guessing, just waiting for the real condition to become true.

Good testing teams go a step further and deliberately test unusual orderings on purpose. Instead of only checking the case where everything finishes on schedule, they intentionally simulate slow responses, out-of-order completions, and dropped connections. This way, problems get caught during testing, not after the software is already in front of users.

## Conclusion

Async systems are efficient and scalable precisely because their different parts don't wait on each other, but that same independence makes them difficult to test with traditional, step-by-step methods. The solution isn't to slow everything down or force predictability; it's to build tests that check for the real, final outcome directly, rather than guessing how long that outcome should take. It's a small shift in approach, but it's the difference between a test that occasionally fails for no good reason and one that reliably catches real problems.
