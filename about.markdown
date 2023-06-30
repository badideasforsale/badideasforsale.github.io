---
layout: page
title: About
permalink: /about/
---

# About me

I do cloud security things. And cloud things. And security things. And things.

## Presentations

### 2023
[The Ground Shifts Underneath Us](https://fwdcloudsec.org/speakers.html#ground-shifts-underneath-us)

One of the hardest parts of working in a cloud environment is the unstable ground we build on. While the APIs themselves are usually quite stable, the actual implementation of those APIs in the cloud provider’s systems can— and do— frequently change. What were once safe assumptions and architectures can, and have, been broken by updates to services.

### 2020
[What I Wished Someone Told Me Before Going Multi-Account](https://fwdcloudsec.org/2020/speakers.html#what-i-wished-someone-told-me-before-going-mutli-account) [Video](https://www.youtube.com/watch?v=_JGXdOyVugg&list=PLCPCP1pNWD7OBQvDY7vLCFhxWxok9DITl&index=2)

something worth pursuing as a strategy, a debate which is finally beginning to coalesce around the answer: "Yes"

I have some news for you, and it's the first thing I wish I would have known — your organization already has multiple accounts. You just might not know about them.

There are a variety of reasons this strategy is one to officially support and encourage. In this talk, I will cover each of them. More accounts can make your organization more secure, more resilient, grant better control over data and encryption keys which are subject to multiple and increasing compliance regimes, and more. All of these reasons are good ones from a security practitioners perspective, but none of the previous reasons are necessarily good enough to convince the rest of your organization. I wished someone had told me budgetary controls and growth planning were more convincing, if less exciting, arguments to push for a multi-account strategy.

I have made a number of mistakes in this journey, some of which I will cover in this talk, including the reasons why you need to adopt a multi-account strategy, how to convince others to join you, the benefits you get from having multiple accounts both security and not, and the technical concerns that will need to be addressed along the way to keep everything functioning. All of these things and more I wished someone had told me before embarking on my current multi-account journey. Now, I'm ready to share my experience with others so you don't have to make the same mistakes.

### 2019
[Cloud Forensics: Putting The Bits Back Together](https://static.sched.com/hosted_files/appseccalifornia2019/ab/Putting%20the%20Bits%20Back%20Together%20-%20OWASP%20AppSec%20CA%202019.pdf) [Video](https://www.youtube.com/watch?v=nQHmiCFqrRE&list=PLpr-xdpM8wG-bXotGh7OcWk9Xrc1b4pIJ&index=10)

Cloud computing security response is no different to servers racked in a regular datacenter, except for a key difference: When a server is breached, and the need exists to perform a forensic evaluation of that server, the responder has no idea where, or what, that server is. The very first steps of imaging a disk need to be rethought in an environment where disks are of variable sizes and capabilities, and are only exposed via APIs. Many things which are taken for granted in the physical world are implementation details in the cloud. Recent product launches in AWS, such as the next-generation of EC2 instances which access EBS in a different manner, as well as bare-metal instances, have changed some of these implementation details— which potentially changes what an incident responder may encounter.

