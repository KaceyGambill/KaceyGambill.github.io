---
layout: post
title:  "CI/CD Observability"
date:   2023-11-22 00:00:00 +0000
categories: SRE, DevOps, CI/CD, Observability
---

There are plenty of posts on the importance of Observability. While it is important to have within the applications running the workloads. It is equally important to have it inside the CI/CD pipelines as well.

Having a deep understanding of your CI/CD pipeline will show

when changes happen, how long does it take for that change to become live.

If an incident happens, how long will it take till the fix is live?

how many changes result in a failure?

what is the deployment frequency?

## Why CI/CD Observability is Important

Normally, when we think about Observability, it is in relation to applications and the various endpoints, functions, database calls associated with it.

I find that one place that lacks observability is often inside of CI/CD Pipelines themselves. This is necessary for us to be able to answer questions like:
- how long do builds usually take?
- do we have flaky tests? If so, which ones are they?
- how many apps are within xx compliance?
- how long do deploys take for a given application?
- did this rise or fall within the last week, and if so, why?
- how many times is code pushed for X repository?

## How Observability Can Help

A lot of these questions are extremely difficult to answer unless we are logging and gathering traces of the CI/CD pipeline throughout its various stages.

But a lot of these questions are really important.

For example, if we know an application had been fully deploying within 5 minutes, and it is trending to 7 minutes we can start to inspect other areas of the system that might be the root cause for this. This will also help visually highlight when this trend started.

In that example we might look at:
- has application startup time increased?
- are we fighting pod disruption budgets due to startup time increase?
- are we having trouble scheduling our workloads? What does node pressure look like?
- do we need to scale up new nodes preemptively?

## Summary

From there, if we are logging and graphing the various metrics we need to make these determinations we can go back and easily see what changed, and when the deployment time increased. This makes it relatively easy to determine the commit it happened on, and start diagnosing the root cause from there. Or diagnose the correct system that is responsible for the new delay.

Having these metrics and this level of observability within the CI/CD pipeline starts to enable the ability to easily roll back changes, notify relevant developers and helps ensure that the applications remain healthy and happy.

Embedding observability into the CI/CD pipeline not only helps understand code deployment, but it also really hightlights deficiencies in our testing frameworks. We can use observability to identify unreliable / flaky tests. Once we have identified the various tests, we can start to see commonalities between failures and then put those metrics into more visual graphs. This will help identify the test and allow us to either fix it or remove it.

Follow up to come on how we are implementing observability inside of GitHub actions!


