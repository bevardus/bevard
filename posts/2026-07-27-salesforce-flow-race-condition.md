# Diagnosing a Silent Salesforce Automation Failure: A Root-Cause Walkthrough

**July 27, 2026**

## TL;DR

A Salesforce automation responsible for creating a downstream record silently stopped working for a subset of incoming records — no errors, no failed jobs, nothing in any log. The incident involved a flow that had been deliberately deactivated months earlier in favor of a replacement, and that replacement had an unrelated timing bug that let records slip through without ever raising a fault. This document serves as a future reference guide for similar incidents.

## The symptom

A routine data check turned up records that should have triggered a downstream record and hadn't. Nothing flagged it — not the Flow error email, not Apex debug logs, not the nightly batch job's run history. The automation simply hadn't fired for those records, and nothing in the platform considered that worth reporting.

The first theory was the obvious one: the nightly batch job that catches stragglers must be broken. Its run history said otherwise — clean execution, every night, on schedule, zero errors. That ruled out "the job is broken." Something else was wrong, and the gap kept growing.

## Round two: the count kept climbing

Widening the query window turned up far more gaps than expected — going back several months, not days. That ruled out a recent regression; whatever this was had been quietly happening for a while.

Comparing what was supposed to be running against what was actually active in the org surfaced the real clue: the flow everyone assumed was doing the work had zero active versions. It had been deactivated months earlier in favor of a newer flow — an intentional, documented change, not an accident. The newer flow should have picked up the slack. It wasn't, and it wasn't erroring while it didn't.

## The real root cause

The replacement automation was a record-triggered flow with a scheduled path: on update, if a status field matched a specific value, wait one minute, then continue. That one-minute wait looked harmless in Flow Builder. It was the entire problem.

A scheduled path doesn't just wait and resume — when the wait elapses, Salesforce re-evaluates the flow's entry conditions against the record's *current* state before resuming. If the status field had already moved on by the time that minute was up — because some other automation, or a person, advanced it quickly — the flow silently declined to resume. No error, no fault, nothing written anywhere. It just quietly stood down, and nothing tells you that happened.

The status value being checked for was transient by design — a brief in-between state, not a resting one — which made it exactly the kind of value likely to have already changed a minute later on any record processed reasonably fast. The automation was effectively penalizing itself for working quickly.

**Key takeaway:** a scheduled path's entry criteria are re-checked at resume time, not just at trigger time. Anything gated on a transient value is a candidate for silent, invisible drop-out.

## What had actually been happening

Once the mechanism was clear, the scope came into focus: on the order of a hundred-plus records over several months had silently skipped their downstream record, with nothing in any log or job history to flag it. It surfaced because of a routine reconciliation check — not an alert, not a failed job, not a ticket.

This wasn't corruption or a security event, just quiet, compounding data loss from an automation that looked, on every operational signal available, like it was working fine.

## The part that actually made this hard

The race condition itself wasn't the hard part — once the theory existed, it was easy to confirm by re-running the flow's entry logic against a few affected records by hand. The hard part was that every available signal said the system was healthy: the batch job ran clean nightly, the flow was active with no fault emails on file, and nothing in Setup surfaced a single error anywhere in the chain. A process that fails loudly is a debugging problem. A process that quietly declines to run is a detection problem, and native tooling has very little to say about an automation that correctly "decides" to do nothing.

**Critical lesson:** absence of errors is not evidence of correctness. When an automation's whole job is to create records, the metric that matters is whether the records got created — not whether the automation reported success.

## What actually fixed it, roughly in order

1. Added the durable, non-transient field behind the status change — the actual completion flag, not the transient status label that only passes through it — into both the flow's start filter and its decision logic, so entry no longer depended on catching a fleeting value in time.
2. Reactivated the older flow, not as the primary path but as a permanent nightly reconciliation safety net, gated on that same durable field with a dedup check so it can never create a duplicate.
3. Sized the backlog and confirmed the reconciliation job would self-heal it within a bounded number of nightly runs at its existing batch size, rather than pushing one large one-off batch through production.
4. Flagged a secondary rule-ordering question surfaced during the fix — two decision rules that could both match the same record — to the business owner for a call, rather than silently picking an interpretation.

## What I'd do differently

Treat any automation whose entire job is "create a record" as needing a reconciliation safety net by default, not as something bolted on after the fact. A single flow with a timing dependency is one bad interaction away from silently doing nothing, and there's no native alert for an automation that runs clean while accomplishing nothing. I'd also be more suspicious of scheduled paths gated on transient status values in general — if a value only exists for a moment, it's a bad candidate for anything that resumes later and re-checks it. And deactivating an existing automation in favor of a replacement deserves a short overlap window where both run and their output gets diffed, rather than a clean cutover on trust.

---

*This one happened on a Salesforce org, not the WordPress side project from the last write-up like this — but the pattern held again: don't settle for the first clean-looking explanation, verify a fix actually produces the expected downstream effect instead of assuming it does, and treat "everything reports success" with real suspicion when the actual output says otherwise.*

[Full technical writeup with flow logic and exact configuration — *link placeholder, see note*]

---

**Author:** James BeVard
[LinkedIn](https://www.linkedin.com/in/jimbevard/) | [Salesforce Profile](https://www.salesforce.com/trailblazer/mrjim)
