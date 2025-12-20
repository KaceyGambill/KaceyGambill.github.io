---
layout: post
title:  "Sleep Tight, Cluster Right: Stop Burning Cash at 3 AM"
date:   2023-07-30 00:00:00 +0000
categories: kubernetes, autoscaling, efficiency, right-sizing
---

Most Kubernetes clusters are wide awake 24/7, even if you users aren't! 

CPU-based HPAs try to help, but quickly fall apart. Especially when we add the VPA to the mix. These two can work in tandem, but only if we implement some smart scaling with KEDA, otherwise it is just a battle between the two, and it makes our clusters tired!

Queues, request rates, latency, and time of day can tell us far more about whether or not our workloads should exist at all.

This is where KEDA shines!

Instead of guessing how busy something *could* be, we scale based on events!
- Prometheus metrics when work exists.
- Cron schedules when traffic in the cluster drops.

During the day, we rely on metric triggers, which help drive rapid scaling decisions.
At night, cron or empty signals help pull our workloads to a minimum amount, sometimes down to zero!

Our VPA is no longer fighting with our HPA, and we are not running idle pods.
The best part, we are no longer paying our cluster to work, while we are asleep.

### Scheduled Scaling: Night Shift

Traffic Patterns are predictable, but lets not guess, we can schedule this with a trigger.
- Cron Scalers set explicit windows to scale down a workload during off hours
- Prometheus Scalers scale our workloads based on metrics

We can use these both in tandem to create a workload that only runs when there is work to do. As soon as we start to get requests, we quickly scale out, and start to serve the traffic, then with a longer scale-down period, we can handle any remaining traffic while the cluster gets ready to scale back down. 

Workload scaling is only half of the story. We aren't getting these crazy savings if we are still paying for nodes on standby! 

The cluster autoscaler can do a decent job, but here is where Karpenter really shines.

As soon as our workloads are gone, Karpenter is consolidating and terminating nodes that are empty, allowing us to use beefy nodes when we need to, but also scales way down in the event that our workloads are ready for bed!

### Day Shift

We use Cron triggers to ensure our workloads are warmed and ready to go in the morning. Pairing this with Karpenter, we end up with zero toil and a fully awake cluster ready to go before the first few developers start signing on!

In the event that there are some early birds, during the night, we still do fallback to metric scaling after our cron triggers. This allows us to ensure the pods are always available if needed. 


### Final Thoughts

Autoscaling is not just about surviving peak traffic, it's about efficiency. By combining KEDA's event based triggers, and Karpenter's node management, we no longer burn cash on empty compute. 

Stop paying for idle time. Make your infrastructure work for you, not the other way around.
