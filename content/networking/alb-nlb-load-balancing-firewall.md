---
title: "Load Balancing at Scale: ALB vs NLB and Firewall Rules Under Load"
date: 2026-06-23
draft: false
tags: ["AWS", "Networking", "Load Balancing", "Security", "ALB", "NLB"]
series: ["Cloud Security"]
description: "How Application Load Balancers and Network Load Balancers distribute traffic differently, and how security group and firewall rules behave when an application scales under real load."
---

So you're probably wondering, **why does it matter which load balancer I pick, they both just spread traffic across instances right?** I would say this is a fair assumption to make, and it is also wrong in a way that only shows up once real traffic hits your application.

I've run into this exact situation more than once. A security group that worked fine in every test, a load balancer that passed every smoke test, and then the moment actual concurrent users showed up, things that looked identical in a console screenshot started behaving completely differently under the hood.

# Gameplan

To understand why this matters, we need to break this down into two pieces: how ALB and NLB actually move traffic differently at the protocol level, and how the firewall layer around them needs to be designed with that difference in mind. I'm going to walk through both, then get into what actually breaks under load and how to catch it before it becomes an outage.

## ALB vs NLB, the Real Difference

The naming makes this confusing. Both are called "load balancers" but they operate at completely different layers.

**ALB** operates at layer 7. It actually reads the HTTP request, which means it can route based on the URL path or the hostname. It terminates the client's connection and opens a brand new one to your backend target.

**NLB** operates at layer 4. It has no idea what's inside the packet, it just forwards the connection as fast as possible. It preserves the original client IP all the way through to your backend, which ALB does not do by default.

That one detail, **source IP preservation**, is the thing that trips people up the most, and I'll get into exactly why below.

## ALB Under Load

Because ALB terminates and re-opens the connection, it absorbs the connection churn itself during a traffic spike, not your application servers directly. That part is good news.

The part that isn't good news is health check tuning. Let's say your unhealthy threshold is set aggressively low:

```bash
aws elbv2 modify-target-group \
    --target-group-arn <target-group-arn> \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 2
```

With `unhealthy-threshold-count` set to 2, a brief CPU spike from completely legitimate load gets misread by the ALB as a failed instance after just two missed checks. The ALB pulls it from rotation, the remaining instances absorb that traffic on top of what they already had, and now you've created a feedback loop. A normal traffic spike just turned into a cascading outage, and it was the health check configuration that caused it, not the traffic itself.

I'd recommend giving yourself more breathing room:

```bash
aws elbv2 modify-target-group \
    --target-group-arn <target-group-arn> \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 5
```

## NLB Under Load

Remember that source IP preservation detail from earlier? Here's where it actually matters.

Since NLB passes the client's real IP straight through, your security group on the backend needs to be written for actual client traffic, not load balancer traffic:

```bash
aws ec2 authorize-security-group-ingress \
    --group-id <backend-sg-id> \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0
```

I've seen teams try to lock this down to just the load balancer's IP range, the same way you'd do it behind an ALB. With NLB that breaks everything, because the load balancer's IP is never actually the source IP your security group sees. The traffic gets blocked and nobody can figure out why, because in the console everything looks like it should be working.

# Firewall Rule Design for Scale

## The Connection Table Problem

Here's something that doesn't show up until you have real volume. Every stateful firewall, Security Groups included, tracks active connections in a table behind the scenes. Short-lived HTTP requests open and close that table entry fast, and under genuine load that table can fill up faster than entries get cleared out.

The symptoms of this look identical to a DDoS attack from the outside. Timeouts. Dropped connections. Random 5xx errors that seem to come and go. Except the traffic causing it might be 100% legitimate users.

## Pushing Rate Limiting Upstream

The actual fix is keeping that connection table from ever getting close to full in the first place, by stopping abusive traffic before it reaches the load balancer at all. AWS WAF sitting in front of your ALB handles this at the HTTP layer, blocking a single IP once it crosses a request threshold, well before that traffic ever touches your target group's connection state.

## NACLs as a Blunt Instrument

Security Groups make the smart, logical allow or deny decision. NACLs sitting at the subnet boundary can do something different, they can act as a circuit breaker during an actual volumetric event.

Because NACLs are stateless and get evaluated before Security Groups in the traffic path, a deny rule placed there stops the traffic before it ever touches your connection table resources. It's a blunter tool than a Security Group, but sometimes blunt is exactly what you need when something is actively trying to flood you.

# Watching for Trouble Before It's an Outage

The earliest tell that your backend targets are starting to struggle, before a single one of them actually fails a health check, is the gap between average and maximum response time widening under steady request volume. I watch that gap in CloudWatch specifically because it shows up well before anything alarms or fails outright.

# Conclusion

The distinction between ALB and NLB isn't academic trivia for a certification exam. Picking NLB for something that actually needed path-based routing, or picking ALB for something that needed the real client IP preserved for security logging, are mistakes that only reveal themselves once production traffic patterns show up, not during a quick test with a handful of requests.

Firewall rules behave the same way. A Security Group that works perfectly at 10 requests a second can fall apart completely at 10,000 a second, and it's not because the rule itself is wrong, it's because the connection tracking behavior underneath it changes character entirely once you're under real volume.

# FAQ

**Can I put an NLB in front of an ALB?**
Yes, and it's a pretty common pattern. You get NLB's static IP and source preservation for firewall allowlisting or compliance reasons, while still keeping ALB's layer 7 routing for the actual application traffic sitting behind it.

**Does WAF add latency under load?**
Not really. WAF evaluation happens at the edge before the request ever reaches your ALB, and it scales on AWS managed infrastructure independent of your own capacity.

**How do I tell if my Security Group is the bottleneck versus my actual application code?**
Compare your ALB's target response time against your application's own internal processing time from your logs. If the gap between the two grows specifically during a traffic spike, the problem is in the network layer, not your code.
