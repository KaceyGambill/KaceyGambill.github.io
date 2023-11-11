---
layout: post
title:  "Alert Fatigue, and How to Fix it."
date:   2023-11-10 00:00:00 +0000
categories: SRE, devops, alerts
---


## What is Alert Fatigue?

For somebody working in tech, especially as a Site Reliability Engineer or in a DevOps role, they are very likely facing a barrage of alerts that show numerous problems with plenty of the services they are supporting.

Alert fatigue generally happens when alerts are not actionable, or they are so frequent that eventually you end up tuning out the Slack channel because it would be impossible to actually get work done while triaging each alert.

A good example of alerts that end up causing fatigue are utilization alerts:
- disk space is high
- memory for a service is high
- cpu utilization is high

When a service is near a metric threshold, we often get alerted about it, but if that alert is not actionable, it is not actually helpful.

I find that implementing these utilization alerts with an evaluation period certainly helps reduce the stream of alerts because we are no longer capturing just utilization spikes.For example, if a memory alert is set to alert on `max_memory > 85%` the alert could become really noisy. A better alert might look like: `avg_memory > 85% for 5 minutes`.
This is still going to capture samples that would help indicate if we were to need to increase request/limit's for a service, but this will be much less noisy.


## The Impacts

Alert fatigue can cause us to overlook or miss the alerts that are actually important. Or the engineer might spend their whole day looking into the alerts, not realizing that they are maybe set to be a bit to sensitive. I have seen this happen numerous times. One engineer will ignore all of them because they are used to them and in the past most have not been actionable, and the next engineer who is on call will spend their entire shift looking into each alert, when the likely culprit is just normal load.

The actionable item at this point is adjusting the alert, or cpu/memory requests and limits.

## Combating Alert Fatigue

The first thing engineers can and should do is try to make the alerts as simple as possible. If there is a noisy alert that is plaguing you today. Ask yourself:
- Why is this alerting?
- Is this actionable?
  - how do I make this actionable? **this should become the new alert**
    The solution might be to add a longer evaluation period, increase memory/cpu, or remove the alert.

## The Challenge of Alarms

Often times teams do not get to set up their alerts from the ground up, but even when they do, it is hard to not alert on everything. To get detailed memory and cpu alerts for a new service, we could load test in a production like environment, but not everyone has the time or infrastructure to set that up.
To set up these alerts for a service that has been running, hopefully we have historical metrics that we can look at. As these alerts start to happen, we should be frequently revising these alerts until there is very few of them, or they are actionable when they do happen.

Another interesting thing we should look at is composite alerts.

If a service has > 85% memory utilization, how does this affect the service? Are we noticing latency increases? Has our error rate went up?
These might be factors that would provide really actionable alerts, that are not just barraging the Slack channel.

If we see an increase in: _latency_, _traffic_, _errors_ or _saturation_.
We likely need to know about this, but it's usually not just one thing that that got us to this point. Which is why it is just as important to have a runbook or dashboard for each alert. This helps ensure that if we are alerted on request latency going up, we can quickly verify that the application has enough memory to handle the request, and then start digging into downstream factors such as the latency of the database, or perhaps we hit a ton of cache-misses from our redis instance.

## Solution

Setting up composite alerts can be difficult to get right. I prefer to keep alerts as simple as possible.

Ideally, I'd like an alert on the [four golden signals](https://sre.google/sre-book/monitoring-distributed-systems/).
- *latency* time it takes to service a request
- *traffic* how much demand is being placed on the system (request per second)
- *errors* rate of requests that fail
  - note, requests that succeed, but show the wrong content would be considered errors, they are just much harder to capture
- *saturation* measure of utilization of memory, cpu, space available. How much load can the sytem handle?

With these alerts, they should only be created when we can link to a dashboard or a runbook along with the alert. This is going to save valuable time for the engineer who is looking into these alerts.

For example, if we alerted on receiving a high rate of errors, I would expect to see a dashboard or runbook indicating that we should look at:
- what kind of errors are happening and how many errors are there??
- sum of 500 internal server error
- sum of 501 not implemented
- sum of 502 bad gateway
- sum of 503 service unavailable
- sum of 504 gateway timeout's
If we can break up the alerts, that's great, if not, we can aggregate them under `http_code > 500 for 5 minutes`
From here I would like to be able to see at a quick glance answers to the following questions:
- Are the various services up?
- When was the last deploy pushed, and what was deployed?
  - Is the service being deployed right now?
- How does saturation and latency look?
  - Are resources saturated?
- Is there any downstream system affecting something upstream? In this case, a service map can be really helpful. Especially, if at a glance we can see latency, and saturation of those services.


If anyone has any feedback or suggestions, please let me know!
I would love to have a conversation on how you are handling alerts for your services!


