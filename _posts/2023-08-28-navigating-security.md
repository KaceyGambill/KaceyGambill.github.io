---
layout: post
title:  "Navigating Security and Compliance"
date:   2023-08-28 00:00:00 +0000
categories: security
---

## The Challenge of Silos in Team Collaboration

Teams within companies often operate in specialized areas, focusing on their unique responsibilities. Development teams work on features and maintenance, security teams emphasize compliance, and platform teams build tools for developers. While each team has its part in achieving overarching business objectives, collaboration is key to ensuring that efforts align with the company's goals.

Even in smaller companies, where teams may share resources like a Jira board, silos can form. It's the responsibility of managers and individual contributors to foster communication and prevent barriers that hinder collaboration.

Understanding the bigger picture is essential. Developers may not need to know specific IP address ranges, but they should be aware of how things run across various environments. The platform team should understand development efforts that might impact server load.

Isolation can lead to a lack of holistic understanding, diminishing the value of individual contributions to business goals. Collaboration bridges these gaps.

## Bridging the Gap Between Security and Development

Sometimes it feels like there is a war. Security vs the rest of the engineering teams. But this is because these teams are not talking on a daily, or at least weekly basis. There is no list or shared set of priorities. Security is focused on keeping the company within the various compliances and making sure that they do not end up on the news. Developers are focused maintaining the various systems, or are sprinting to create new features. The platform team is usually trying to juggle priorities between the various teams they are supporting. 

But there will come a time that security will create high priority tickets and cite some obscure, but very valid, compliance article that mentions having 30, 60, or 90 days to remmediate these tickets or fall out of compliance. Then their tickets will jump to the front of the queue. This then pauses developer's and platform team members work. This is where the frustration comes in. Now the various engineering managers have to make a choice, do we keep all this work in progress and focus on paying down security debt? Or do we finish the work in progress, if it is possible within the time frame, and then tackle the security tickets, hoping they do not take too long, so that we do not fall out of compliance. 

This is why it is important that the Security team, the development team, and the platform team meet to discuss priorities, observe each others work, and determine how to achieve business goals together.

## Aligning Security with Business Goals

Sometimes I have felt that Security teams feel like their only goal is to keep the company within compliance. It is so much more than that. They should be involved in setting business goals and taking a large role in vendor selection. They should be one of the determining forces when a team is trying to decide to build something in house, or to outsource it. Having security involved earlier, is always better. 

## Practical Security and Working as a Team

I've ran into it so many times, that the security team are unaware of what domains the developers are responsible for, or even what the platform team members are responsible for.
This is unacceptable, like it would be unacceptable for a platform team member to not know what the network topology consists of. 

Involving security from the start helps them be aware of when an effort is related to marketing, or if a specific domain needs to be PCI compliant. Too many times I have seen a security team run their automated scans with little insight to how the business is actually ran. They come back with a bunch of tickets, which are likely solved by firewall rules, or network topology. What peeves me the most is when these scans have full access to the system, bypassing many of those security controls. This results in tickets created, that are already mitigated, but from securities prospective, are not. 

This is hard to balance though, and I am not sure the right answer. I agree with security at every layer, but the company still has to run, and we still have to meet business objectives.

A good security team has established secure development practices for the developers to reduce the amount of vulnerabilities introduced in the system. They will understand the business objectives and be in vendor talks. They will understand the network topology and various controls that are in place to prevent intrusion. And most of all, they will not create a burden for other engineering teams, but try to limit scope of security work to maintain compliance. 

When this happens, other teams work more with the security team and are more happy to invlove them in product discussions sooner.


### Bonus Checklist

Most people in the engineering department, especially in a smaller company should have a rough idea of:

- What is the general business objectives we are working towards achieving?
- How does what I am doing help in achieving those goals?
- At a high level, what systems do we use or support?

By embracing collaboration and breaking down silos, companies can foster a more unified and effective approach to achieving their goals.

I would love to hear anyone's thoughts on this subject! Please leave a comment and let me know how communication barriers are down at your company!



