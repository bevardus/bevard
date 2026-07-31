# The Deploy That Kept Failing While Everything Looked Fine: A Root-Cause Walkthrough

**July 31, 2026**

## TL;DR

A completely unrelated, already-reviewed feature branch quietly broke every deployment to production for most of a working day. Not because its own code was wrong — because of how our platform's config-style metadata (permission sets, layouts) gets deployed as whole files instead of diffs. The kicker: while this was happening, several *other* legitimate deploys succeeded and posted green checkmarks in the same channel, which is exactly what let it hide in plain sight for hours.

## The Symptom

Our pipeline runs a periodic consistency check after every deploy: does the shared integration branch still match what's actually running in production? It's normally invisible — it passes silently, all day, every day.

One afternoon, it started failing. Every run produced some version of the same message:

```
Deployment of 13 component(s) failed, because 4 component(s) failed validation.

Item: Vendor_Intake_Layout | Type: Layout | Reason: Parent entity failed to deploy
Item: Vendor_Reviewer | Type: PermissionSet | Reason: In field: object -
  no CustomObject named Vendor_Intake_Response__c found
Item: Vendor_Reviewer_Manager | Type: PermissionSet | Reason: In field: object -
  no CustomObject named Vendor_Intake_Response__c found
```

Nobody had touched anything related to vendor intake that day. The team's actual work in flight was a small internal-notifications feature — nowhere near this.

## Round Two: The Ones That Kept Succeeding

Here's what made it genuinely confusing to triage: at almost the same timestamps as these failures, other feature branches were deploying to production *successfully*, one after another, all afternoon. Green checkmark, green checkmark, red X, green checkmark.

That pattern reads like a flake. If unrelated work keeps shipping fine, the natural assumption is "something's wrong with the check, not with us." It took someone actually sitting with the failed-run history — not just the day's overall noise — to notice that this wasn't random. It was the *same four components*, failing the *same way*, every single time, for six-plus runs in a row, spanning hours.

The distinction that mattered: individual feature deploys ship directly from their own branch's snapshot of each file. The consistency check compares the shared upstream branch as a whole against production. Two different questions. One of them was quietly, consistently broken; the other kept looking fine because it was answering a narrower question each time.

## The Real Root Cause

The offending branch was for a small, boring, entirely legitimate feature — a notification tweak, nothing controversial, already reviewed and approved. It had been cut correctly, from the team's standard shared integration branch, exactly per process.

The problem was upstream of that PR entirely: the shared integration branch already contained a separate, unrelated feature — an internal vendor-intake workflow — that had been merged into it weeks earlier but was *deliberately* being held back from production while it finished internal review. That's normal; branches carry in-progress work all the time.

What's not normal, and what nobody had really priced in, is that on this platform, permission sets and page layouts don't merge or deploy as line-level diffs — they deploy as complete file replacements. So the moment the notification branch was cut from the shared branch, it silently inherited the *entire current state* of two permission sets that the vendor-intake work had modified, including grants to three custom objects that had never been deployed to production. The notification branch's author never opened, edited, or even knew about those files. They came along for free, as an artifact of branching, not authorship.

## What Was Actually Riding Along

Once we went looking, the actual footprint was bigger than the four failing components suggested:

- Two existing permission sets, modified to grant access to three objects that didn't exist yet in production
- One brand-new permission set that had never shipped anywhere
- A page layout for one of those not-yet-real objects
- A handful of Apex classes that reference those objects as compiled types — meaning they can't even compile, let alone run, without the objects existing first

None of this ever reached an actual user. Deploys are all-or-nothing, so the bad components blocked the whole batch every time rather than partially applying. But that's cold comfort when it means *every* subsequent deploy attempt — for completely unrelated work — was inheriting the same landmine and failing the same way.

## The Part That Actually Made This Hard

The genuinely disorienting part wasn't the root cause once found — it's fairly mechanical in hindsight. It was that "everything's green except this one recurring thing" doesn't look like an incident. It looks like background noise. There was no user-facing outage, no support ticket, no dramatic failure. Just a health check quietly and consistently lying about whether the pipeline was actually deployable, while unrelated proof-of-life kept arriving in the same channel to reassure everyone that things were fine.

The fix for *that* part wasn't technical — it was recognizing that a check failing identically six times in a row is a different animal than a check failing once. The former deserves someone stopping to read the actual diff of what's different between the passing and failing runs, not just re-triggering it and hoping.

## What Actually Fixed It

The fix itself was almost anticlimactic once the cause was clear: a targeted revert that removed exactly the orphaned vendor-intake content — permission set grants, the new permission set, the layout, the field, the Apex classes referencing it all — while leaving the actual shipped notification feature untouched. Roughly 2,300 lines across nine files, cut cleanly, verified against a clean copy of what production already had.

The one deliberate process change: that revert was branched from a known-clean, production-equivalent branch, not from the shared integration branch — since the shared branch was the thing carrying the problem in the first place. Branching from it again would have just re-inherited the same landmine.

Deploys the next day, and every day after, went back to being invisible. Which is exactly what you want a deploy pipeline to be.

## What I'd Do Differently

- Treat whole-file-replacement metadata (permission sets, profiles, layouts, and anything similar) as **shared mutable state**, not per-PR diffs. Any branch that touches one of these files inherits *everything* currently in it, not just the lines that branch's author cares about.
- Don't let intentionally-unshipped work sit merged into a shared upstream branch indefinitely. The longer it sits there, the more innocent branches will silently pick it up as a side effect of just following normal branching process.
- Escalate on **identical repeated failures**, not just failure count. A health check that fails the same way six times in one day should trigger a "go read the diff" step well before failure number six, not after someone happens to notice the pattern by eye.
- When a "just branch from the shared branch" process assumes that branch is always safe to build on, that assumption needs to be actively defended — not just hoped for.

I do a version of this dance regularly in my day job doing Salesforce release engineering, and it's the same lesson every time, just wearing a different hat: the diff you're reviewing is never the full set of state your change actually carries with it. That's not a Salesforce-specific problem — it's true of Kubernetes manifests, Terraform state, any config format where "merge" means "replace the whole object" instead of "apply these specific lines." The tooling will happily show you a clean, reviewable diff for the part you changed, while quietly deploying everything else that happened to be sitting in the same file. Worth remembering next time a change that "shouldn't have touched anything" breaks something it apparently didn't touch.

---

**Author:** James BeVard
[LinkedIn](https://www.linkedin.com/in/jimbevard/) | [Salesforce Profile](https://www.salesforce.com/trailblazer/mrjim)
