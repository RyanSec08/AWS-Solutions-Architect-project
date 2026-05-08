# Adding Outbound-Only Internet to a Private Subnet: NAT Gateways and the Asymmetric Connectivity Pattern

**Date:** April 29, 2026
**Topics covered:** NAT Gateways, NAT/PAT, route tables, asymmetric connectivity, attack surface reduction, AWS managed services, 
cost-aware infrastructure

## What this writeup is

This is a hands-on walkthrough of solving a problem I created in [my previous writeup](2026-04-28-ec2-bastion-host.md).
That problem: a private app server that's secure but completely cut off from the internet.
It couldn't even pull OS updates.

The solution is a **NAT Gateway**.
It's an AWS managed service that lives in the public subnet.
It gives the private subnet **outbound-only** internet access.
The internet still can't initiate any connection to the private server.
But the private server can reach out to the internet whenever it needs to.

That asymmetric pattern — "I can call out, but you can't call in" — is one of the most powerful security primitives in cloud architecture.

## Why I built this

After the bastion host writeup, my private app server was bulletproof from a security standpoint.
No public IP.
No inbound route from the internet.
No way for anything outside the VPC to even know it exists.

The problem was that it was *too* isolated.
The first time I tried `sudo yum update` on the app server, the connection just hung.
That makes sense — the private subnet had no route to the internet at all.
But in any real-world scenario, that server needs to pull OS patches, install packages, and call external APIs.

This writeup is about adding one carefully designed door to the private subnet.
Outbound only, never inbound.
And proving it works.

"Outbound only" took me a minute to understand.
I thought it meant traffic only flowed one way, which made no sense for things like OS updates.
Once I realized it's really about who starts the conversation, it clicked.
The fix ended up being a lot simpler than I expected.

## The architecture (after adding the NAT Gateway)

```
                            Internet
                               |
              +----------------+----------------+
              |                                 |
              | INBOUND only                    | OUTBOUND only
              | (your laptop's IP)              | (NAT Gateway's EIP)
              |                                 |
              v                                 v
       +-------------+                  +-------------+
       |   Bastion   |                  | NAT Gateway |
       |  (Public)   |                  |  (Public)   |
       | 10.0.1.56   |                  | 10.0.1.134  |
       | EIP attached|                  | EIP attached|
       +-------------+                  +-------------+
              |                                 ^
              | (SSH chain)                     | (outbound only)
              v                                 |
       +----------------------------------------+
       |             App Server                 |
       |             (Private)                  |
       |             10.0.2.132                 |
       |             No public IP               |
       +----------------------------------------+
```

Two doors into the public subnet.
Both controlled.
Both serving opposite purposes:

- **Bastion host's Elastic IP** handles **inbound** traffic.
The internet → me.
Used for SSH access.
- **NAT Gateway's Elastic IP** handles **outbound** traffic.
Me → the internet.
Used for OS updates, API calls, anything outbound.

The app server itself still has no public IP at all.
It's reachable from the bastion (for me).
It can reach out (for updates).
But it's invisible to the internet in every other direction.

## Step 1 — Create the NAT Gateway

NAT Gateways live in the **public** subnet, not the private one.
This is counterintuitive at first.
The whole point is to serve the private subnet, so it feels like it should belong there.

But the NAT Gateway needs to talk to the **Internet Gateway**.
The IGW only routes traffic for the public subnet.
If you put the NAT Gateway in the private subnet, it can't reach the IGW.
The whole thing would be useless.

So: NAT Gateway lives in public, *acts on behalf of* the private subnet.

I created the NAT Gateway with these settings:

- **Subnet:** `bryan-lab-public-subnet`
- **Connectivity type:** Public
- **Availability mode:** Zonal (single-AZ — fine for a lab; in production you'd run one per AZ for redundancy)
- **Elastic IP:** Allocated a fresh one — `100.50.96.244`

Once it provisioned (~2 minutes), it ended up with two IPs:

- **Primary public IPv4:** `100.50.96.244` — the EIP, the address the internet sees
- **Primary private IPv4:** `10.0.1.134` — the address inside the VPC, used to receive traffic from the private subnet

That dual-IP setup is exactly NAT in action.
One foot in the private network, one foot facing the internet.
It translates between the two.

![NAT Gateway details — Available, with public and private IPs](screenshots-nat-gateway/01-nat-gateway-details.png)

## Step 2 — Update the private subnet's route table

Creating the NAT Gateway alone does nothing.
It exists, it has an IP, but no traffic is being sent to it.
You have to *tell* the private subnet to use it.

That's done by adding a route to the private subnet's route table:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local *(default — VPC-internal traffic)* |
| `0.0.0.0/0` | the NAT Gateway *(new — everything else, including the internet)* |

The first rule is automatic.
It means "anything destined for this VPC stays internal."

The second rule is the new one.
`0.0.0.0/0` is the catch-all default route, meaning "any IP address in the world."

So combined, the logic for any packet leaving the app server is:

- **Destined for `10.0.0.0/16`?** Stay inside the VPC.
- **Destined for anywhere else?** Send to the NAT Gateway.

The NAT Gateway then forwards it to the internet.
The internet replies.
The response comes back through the same NAT Gateway.

I created a dedicated `bryan-lab-private-rt` route table.
I explicitly associated it with the private subnet.

You can technically just edit the VPC's main route table.
But using a named, explicit route table is cleaner.
Every subnet has its own clearly-labeled routing logic.
And there's no risk of a future subnet accidentally inheriting routes.

![Private route table with the NAT Gateway route](screenshots-nat-gateway/02-private-route-table.png)

## Step 3 — Prove it works

Before adding the NAT Gateway, I ran `curl -m 10 https://www.amazon.com` from the app server.
It hung for 10 seconds and timed out.
The private subnet had no route to the internet, so the packets had nowhere to go.

After adding the NAT Gateway and the route, I ran the real-world test:

```bash
sudo yum update -y
```

The output:

```
Amazon Linux 2023 repository                63 MB/s |  59 MB  00:00
Amazon Linux 2023 Kernel Livepatch repo    193 kB/s |  31 kB  00:00
Dependencies resolved.
Nothing to do.
Complete!
```

That `63 MB/s` line is the proof.
The app server — sitting in the private subnet with no public IP — just pulled 59 MB of repository metadata.
Directly from Amazon's package mirrors on the public internet.

"Nothing to do" means the system was already current.
There were no updates to apply.
But the entire round trip happened: query, response, complete.
The connection worked.

![Successful yum update from the private app server](screenshots-nat-gateway/03-yum-update-success.png)

## What's actually happening on the wire

When the app server sends a packet to Amazon's package server, here's the journey:

1. **App server (10.0.2.132)** wants to talk to the **Amazon Linux package repository** on the public internet
2. The packet hits the **private route table**.
Destination doesn't match `10.0.0.0/16`, so it falls through to the `0.0.0.0/0 → NAT Gateway` rule
3. The packet reaches the **NAT Gateway's private IP (10.0.1.134)**
4. The NAT Gateway **rewrites the source IP** of the packet.
From `10.0.2.132` (private, not routable on the internet) to `100.50.96.244` (its public EIP)
5. The packet exits via the **Internet Gateway** to the public internet
6. Amazon's server replies, addressing the response to `100.50.96.244`
7. The Internet Gateway routes the response back to the NAT Gateway
8. The NAT Gateway recognizes "this is a reply to a conversation 10.0.2.132 started."
It tracks active connections in a **state table**
9. It rewrites the destination IP back to `10.0.2.132` and forwards it to the app server

That source-rewriting step is the heart of NAT (Network Address Translation).
And the state table — tracking which internal IP started which conversation — is what makes the connection asymmetric.
Inbound packets that don't match any active conversation get dropped.
There's no entry for them in the state table.

## The Network+ angle: NAT, PAT, and your home router

This is the same fundamental concept that runs your home internet.

Your laptop has a private IP like `192.168.1.50`.
Your phone has another, like `192.168.1.51`.
Both are on your home network, both unreachable from the internet directly.

When you load a webpage, your home router does **NAT**.
It rewrites the source IP from your private address to your home's single public IP.
That's whatever your ISP assigned you.
The webpage's response comes back to that public IP.
The router maps it back to whichever device started the conversation.

More accurately, this is **PAT** (Port Address Translation).
It's a flavor of NAT where multiple devices share one public IP.
They're distinguished by port numbers.
Your laptop's request might use source port 51234.
Your phone's might use 51235.
The router tracks the port-to-device mapping in its state table.

AWS NAT Gateways do exactly the same thing for your VPC.
Many private servers, sharing one public Elastic IP, distinguished by ports.
The pattern you've used your whole life on home internet — just managed by AWS at scale.

## The Security+ angle: asymmetric connectivity as a defense pattern

Most attacks require the attacker to **initiate** a connection to your server.
Port scans.
Brute-force SSH attempts.
Exploit attempts against web servers.
They all require knocking on the server's door first.

If the server can't be knocked on, but can still go out and grab what it needs, you've blocked nearly all attack vectors.
While keeping full functionality.

The NAT Gateway is the architectural piece that creates this asymmetry.
The state table only allows traffic that matches a conversation initiated from inside.
Anything coming from outside that doesn't match an existing conversation is dropped.
There's no "open port" for an attacker to scan, because there's no inbound listener.

This pattern shows up everywhere in production:

- **Application servers** behind a load balancer, where the load balancer is public and the app servers are private with NAT 
for outbound calls (DB updates, webhook calls, third-party APIs)
- **Database servers** that need to pull updates from package mirrors but should never be directly reachable
- **Internal microservices** that call out to external APIs but accept no inbound traffic
- **Build servers** that pull dependencies from npm, PyPI, GitHub but stay invisible to the internet

In every case, the same shape: outbound allowed, inbound blocked, attack surface minimized.

## A note on cost (and an AWS gotcha)

NAT Gateways are not free.

They cost **~$0.045/hour** plus a small per-GB data processing fee.
Run one 24/7 for a month and you're at ~$32/month.
Just for the NAT Gateway itself.
Before you've sent a single byte of data through it.

This is one of the most common AWS surprise-bill traps.
People build a NAT Gateway for a tutorial.
They forget about it.
And they find a $30 line item on their invoice at month-end.

For this lab, I created the NAT Gateway, proved it worked, captured screenshots, and **deleted it immediately afterward**.
Total cost: under $0.10.

In production, NAT Gateways are absolutely worth the cost.
They're a managed, redundant, scalable service that handles billions of connections without you babysitting it.
But for a lab? Build, test, screenshot, tear down.
Same lesson, fraction of the cost.

I also released the Elastic IP that was attached to the NAT Gateway.
Unattached EIPs cost ~$0.005/hour even when nothing is using them.
Another small but real charge that sneaks up on people.

## Why this matters for security engineering

Egress traffic — what private workloads call **out** to — is one of the highest-signal data sources a Cloud Security Engineer monitors.
A compromised server doesn't usually announce itself with inbound noise.
It betrays itself in what it tries to call.
A workload that suddenly starts beaconing to a strange IP, a database server fetching data it's never fetched before, a build server pulling from an unfamiliar package mirror — those are exfiltration and command-and-control patterns.

The NAT Gateway in this lab is the chokepoint where all that egress traffic gets translated and audited.
A few things security teams typically build on top of this foundation:

- **VPC Flow Logs** capture every packet that traverses the NAT Gateway, with source/destination IPs, ports, and bytes transferred.
That's the audit trail that makes egress detection possible.
- **VPC endpoints (PrivateLink)** are the production-grade answer for AWS-internal traffic.
If a private workload only needs to call S3, DynamoDB, or Secrets Manager, you don't need a NAT Gateway at all — VPC endpoints route that traffic across AWS's internal network without ever touching the public internet.
That's both cheaper *and* more secure (smaller egress surface).
- **Egress filtering** at the network layer (AWS Network Firewall, third-party proxies) lets you allowlist exactly which external endpoints workloads can reach.
A database server that only ever needs to talk to Amazon Linux package mirrors should not be able to reach `pastebin.com`.

Building this NAT-only setup first is the right starting point.
But in any production environment I worked on, the goal would be to minimize what flows through the NAT Gateway by pushing AWS-service traffic through VPC endpoints, and to put guardrails on whatever egress remains.

## What I learned

This was the first piece of AWS infrastructure I built that actually costs real money to keep running.
That changed how I worked through it.
I was more deliberate, more focused, and I made sure to delete it as soon as I had what I needed.
The actual concept finally clicked when I started thinking of internet access as "who started the conversation" instead of just on/off.
The home router parallel — that I've been using NAT every day without thinking about it — made it feel a lot less abstract.

## What's next

This writeup completes the basic two-tier pattern.
A public subnet with controlled inbound (bastion) and outbound (NAT Gateway).
A private subnet that's secure but functional.

Next, I'm going to expand this into a true **multi-tier architecture**.
A web tier, an app tier, and a database tier.
Each in its own subnet.
Each with its own security group.
With traffic flowing between them in carefully controlled patterns.
That'll be the writeup that pulls everything from this series together into one production-shaped system.

---

*This is the fourth writeup in my AWS Solutions Architect portfolio. Earlier writeups covered [securing a fresh 
AWS account](2026-04-24-securing-fresh-aws-account.md), [building the VPC and subnets](2026-04-25-mini-segmented-network.md), 
and [the bastion host pattern](2026-04-28-ec2-bastion-host.md).*
