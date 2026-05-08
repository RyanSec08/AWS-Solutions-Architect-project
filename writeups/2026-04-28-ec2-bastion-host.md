# Building a Bastion Host in AWS: SSH, Defense in Depth, and the Jump Box Pattern

**Date:** April 28, 2026
**Topics covered:** EC2 instances, bastion host pattern, security groups, SSH key authentication, agent forwarding, Elastic IPs, defense in depth, principle of least privilege

## What this writeup is

This is a hands-on walkthrough of launching two EC2 instances inside the custom VPC I built in [my previous writeup](2026-04-25-mini-segmented-network.md), and connecting them using a real-world security pattern called a **bastion host** (also known as a **jump box**).

The setup: one server sits in the public subnet and is the only thing reachable from the internet.
The other server sits in the private subnet and has no public IP at all.
To get to the private server, you have to SSH through the public one first.

That single design choice — "you can't talk to the sensitive server directly" — is the foundation of how real production environments protect critical workloads.

## Why I built this

In Writeup #2, I built the network — VPC, subnets, route tables, Internet Gateway.
But a network with no servers in it is just plumbing.
This lab was about putting actual workloads inside that network and proving the segmentation does what it's supposed to do.

I also wanted to physically experience the bastion host pattern, not just read about it.
The Security+ exam talks about "jump boxes" and "defense in depth" in the abstract.
SSHing through one server to reach another, and then watching `curl` time out from the private server because it has no internet route, makes those concepts stop being abstract.

It was so nice to see that everything worked.
There's a sense of safety knowing there is no way to reach the app server other than through the bastion host.

## The architecture

```
                    Internet
                        |
                        |  (Elastic IP: 98.87.60.175)
                        v
        +-----------------------------------+
        |        Bastion Host (Public)      |
        |        bryan-bastion-host         |
        |        Public IP: 98.87.60.175    |
        |        Private IP: 10.0.1.56      |
        |        SG: SSH from my IP only    |
        +-----------------------------------+
                        |
                        |  (SSH over private network only)
                        v
        +-----------------------------------+
        |       App Server (Private)        |
        |        bryan-app-server           |
        |        Private IP: 10.0.2.132     |
        |        No public IP, no IGW route |
        |        SG: SSH from bastion only  |
        +-----------------------------------+
```

The bastion host is the **only** door into this network from the internet.
The app server has no door to the internet at all — it can only be reached from inside the VPC, and only from the bastion host's security group specifically.

## Step 1 — Launching the bastion host in the public subnet

I launched a t3.micro Amazon Linux instance into `bryan-lab-public-subnet` and named it `bryan-bastion-host`.

Two key configuration choices at launch:

- **Auto-assign public IP: Enabled.** This is what makes the instance reachable from outside AWS.
- **Generate a new key pair (`bryan-lab-key.pem`).** This is the private key I'll use to SSH in. AWS keeps the public half on the instance.

The security group attached to it (`bastion-sg`) only allows inbound SSH (port 22) from my home IP — not from the world.

![Both instances running in EC2 console](screenshots-bastion-host/01-instances-running.png)

![Bastion host details — public IP and public subnet](screenshots-bastion-host/02-bastion-details.png)

## Step 2 — Launching the app server in the private subnet

The app server is `bryan-app-server`, also a `t3.micro`, but launched into, but launched into `bryan-lab-private-subnet`.

The critical differences from the bastion:

- **Auto-assign public IP: Disabled.** This server has no public IP at all.
- **Same key pair (`bryan-lab-key.pem`).** Same key works for both — that's important for the SSH chain to work, which I'll explain below.
- **Different security group (`app-server-sg`).** This one only allows SSH from the bastion's security group, not from any IP.

When you look at this server in the console, the "Public IPv4 address" field is just blank.
There is no internet-side address. It does not exist on the public internet.

![App server details — no public IP, private subnet only](screenshots-bastion-host/03-app-server-details.png)

## Step 3 — Security groups: the actual defense in depth

This is where the design pays off.
Most people think of a firewall as a single thing.
In AWS, security groups are stateful firewalls attached to each instance, and you can chain them so that one server's access is gated by another server's identity.

**Bastion's security group (`bastion-sg`):**

- Inbound: SSH (port 22) from my home IP only (`/32` — exactly one IP)
- Outbound: All traffic allowed (default)

![Bastion security group inbound rules](screenshots-bastion-host/04-bastion-sg.png)

**App server's security group (`app-server-sg`):**

- Inbound: SSH (port 22) from `bastion-sg` only — not an IP, the security group itself
- Outbound: All traffic allowed (default)

![App server security group — source is bastion-sg, not an IP](screenshots-bastion-host/05-app-server-sg.png)

That second rule is the trick.
Instead of saying "SSH from `10.0.1.56`," I'm saying "SSH from anything attached to `bastion-sg`."
If I ever add a second bastion or replace the first one, I don't have to update the app server's rules — anything wearing the bastion-sg badge can talk to the app server.
Anything else, including any other resource inside this VPC, cannot.

This is **defense in depth**:

- Layer 1: Subnet routing. The private subnet has no route to the Internet Gateway, so even if something got onto that subnet, it can't talk to the internet.
- Layer 2: No public IP on the app server. There's no internet-facing address to scan or attack.
- Layer 3: The app server's security group only accepts SSH from the bastion's security group.
- Layer 4: The bastion's security group only accepts SSH from my home IP.

Any one of those failing wouldn't be enough on its own to expose the app server.
That's the whole point of layering — no single failure equals breach.

## Step 4 — The SSH chain in action

Here's what actually happens when I want to reach the app server from my laptop.

From my laptop, I SSH into the bastion using its Elastic IP and my private key:

```bash
ssh -i "C:\Users\<username>\.ssh\bryan-lab-key.pem" -A ec2-user@98.87.60.175
```

That `-A` flag is **agent forwarding**.
It tells my laptop to keep the SSH key available for the next hop, so the bastion can use it to authenticate to the app server without me having to copy the private key onto the bastion itself.
Copying the private key onto the bastion would be bad practice — if the bastion ever got compromised, the attacker would have the key to everything.
Agent forwarding keeps the private key on my laptop where it belongs.

**Production note on agent forwarding:**
Agent forwarding is convenient for learning, but it has a known security trade-off — if the bastion is ever compromised, an attacker on the bastion can use your forwarded SSH agent to authenticate to other servers as you for as long as your local agent is running.
Modern bastion patterns prefer SSH's **ProxyJump** feature (`-J`) instead, which tunnels the connection through the bastion without ever exposing your agent to it.
The equivalent ProxyJump command would be:

```bash
ssh -J ec2-user@98.87.60.175 -i "C:\Users\<username>\.ssh\bryan-lab-key.pem" ec2-user@10.0.2.132
```

The bastion sees encrypted traffic but never gets a chance to use your key.
For this lab I stuck with agent forwarding because the lesson it teaches about key management is foundational — but in any production environment I worked on, ProxyJump would be the default.

Once I'm on the bastion (using the agent-forwarding approach), I SSH to the app server using its private IP:

```bash
ssh ec2-user@10.0.2.132
```

No password.
No second key file.
The forwarded SSH agent handles the authentication transparently.

![Full SSH chain — laptop to bastion to app server](screenshots-bastion-host/06-ssh-chain.png)

*Note: the path in this screenshot (`OneDrive\Desktop\bryan-lab-key.pem`) reflects my original setup.
I later caught that this meant my private key had been silently syncing to Microsoft's cloud — see the "What I learned" section below for the full story and the fix.*

## Step 5 — Proving the private server is actually isolated

Once I was on the app server, I ran:

```bash
curl https://www.google.com
```

It hung.
Eventually it timed out.

That's the proof.
The private subnet has no route to the Internet Gateway, so packets going outbound to the public internet have nowhere to go.
The server is alive, the OS is fine, the network stack works — there is just no path to the outside world.

In a real production setup, if this private server needed to reach the internet for things like OS updates, you'd add a **NAT Gateway** to the public subnet and add a route from the private subnet to the NAT.
That gives outbound-only internet access without ever giving the server an inbound public IP.
I haven't built that yet — that's a future writeup.

## The Network+ angle: what's actually happening over the wire

SSH (Secure Shell) is a **TCP-based protocol on port 22**.
TCP guarantees ordered, reliable delivery — every packet is acknowledged, anything missing is retransmitted.
That's the right tradeoff for a remote shell session, where losing or reordering a keystroke would be a real problem.

Authentication uses **asymmetric (public/private) key cryptography**:

- The **public key** sits on the server, in `~/.ssh/authorized_keys`. AWS put it there automatically when I launched the instance with that key pair.
- The **private key** sits on my laptop, in the `.pem` file. It never leaves.
- When I connect, the server sends a challenge encrypted with the public key. Only the matching private key can decrypt it and prove who I am.

This is the same mathematical principle that protects HTTPS, signed software updates, and cryptocurrency wallets.
The server never sees my private key.
There's no password to phish.

The reason agent forwarding (`-A`) matters in this lab specifically is that the second hop (bastion → app server) needs to authenticate too — and without agent forwarding, my laptop's private key isn't reachable from the bastion.

## The Security+ angle: why this pattern exists

Three Security+ concepts show up directly in this design.

**Defense in depth.** Multiple independent layers of protection, each one capable of stopping or detecting an attack on its own. I covered the four layers earlier — subnet, no public IP, security group chaining, IP whitelisting on the bastion.

**Principle of least privilege.** Every component has the minimum access required to do its job and nothing more.
The app server's security group doesn't allow SSH from anywhere; it allows SSH from the bastion's security group only.
My home IP isn't allowed to talk to the app server directly.
The app server can't browse the internet.
None of these restrictions break the lab's functionality — they just close doors that don't need to be open.

**Attack surface reduction.** The internet-facing surface of this whole environment is one server, one port (22), restricted to one IP.
That's a tiny target.
Compare that to a setup where every server has a public IP and SSH open to the world — that's a much wider attack surface and a much easier compromise.

## Elastic IP: why "permanent" matters

When I first launched the bastion, AWS gave it a temporary public IP (`34.227.53.218`).
When I stop and start an EC2 instance, that public IP changes.
Which means every script, SSH config, firewall rule, or DNS entry pointing to that IP would need to be updated every time.

An **Elastic IP** is AWS's name for a static public IPv4 address you reserve to your account and attach to an instance.
I attached `98.87.60.175` to the bastion, and now that IP is stable across stops, starts, and reboots.

In a real environment, this matters because:

- DNS records pointing to your bastion don't break
- Documentation and runbooks don't go stale
- Monitoring tools don't lose the host
- Most importantly, your team's `~/.ssh/config` doesn't need to be updated every Monday

**A note on cost:**
As of February 1, 2024, AWS charges **$0.005/hour (~$3.60/month)** for *all* public IPv4 addresses, including Elastic IPs that are attached to running instances.
The same charge applies to detached EIPs and ones associated with stopped instances.
The free tier includes 750 hours/month of public IPv4 address usage for the first 12 months of a new AWS account, but past that, every public IPv4 address costs money regardless of how it's being used.

The pricing change was AWS's response to global IPv4 address exhaustion — IPv4 addresses are a genuinely scarce public resource, and AWS is encouraging more efficient use (and pushing customers toward IPv6 where possible).
For a lab with one bastion and no other public IPs, the cost is small enough not to matter.
For a production environment with dozens of EIPs, it's the kind of thing that adds up to a real line item.

## What I learned

The first time the SSH chain worked, it felt powerful — like I was doing real cloud work instead of just reading about it.
The thing that actually clicked for me was how SSH and key authentication work in practice.
I'd seen public/private keys explained for Security+, but using them across two hops with agent forwarding made the concept finally feel real.
The one snag was my home IP changing partway through and having to update the bastion's security group — small thing, but it made me appreciate why production environments don't rely on IP allowlists for human access.

**A real lesson learned during the writeup review:**
Long after publishing this lab, I caught a subtle but real security issue with my own setup.
When I'd originally generated `bryan-lab-key.pem`, I saved it to my Windows Desktop without realizing that Microsoft's **OneDrive Folder Backup** feature was silently syncing my Desktop to the cloud.
My private SSH key had been sitting on Microsoft's servers for weeks.
I never explicitly chose to put it there — OneDrive's default backup behavior captured it the moment I saved the file.

The fix was straightforward once I noticed:

1. Move the key out of any OneDrive-synced folder. The conventional location on Windows is `C:\Users\<username>\.ssh\` — a folder OneDrive doesn't backup by default.
2. Delete every cloud copy: from the OneDrive web Recycle bin (which keeps deleted files for 30 days otherwise), and from the local Windows Recycle Bin.
3. Disable OneDrive Folder Backup for any folders I don't actively want auto-synced (Settings → Sync and backup → Manage backup → uncheck Desktop, Documents, Pictures).

The lesson is broader than just this key: **on a Windows machine with OneDrive Folder Backup enabled, the Desktop is the cloud.**
Anything saved there gets uploaded automatically, with no notification.
That's fine for documents and photos.
It is not fine for SSH keys, API tokens, or any other security-sensitive file.
Going forward, the rule I follow is: credentials only live in `.ssh\`, `.aws\`, or another folder explicitly outside of any sync engine's scope.

## Why this matters for security engineering

Bastion hosts are an auditable choke point for human access — a single place to log who connected, when, and what they did.
Cloud Security Engineers spend a lot of time hardening and monitoring exactly this layer:

- SSH session logging and recording
- Just-in-time bastion access (no permanent SSH keys sitting on developer laptops)
- Tying access to identity, not IPs (since IPs change and IP allowlists don't scale)
- Where private SSH keys actually live on disk — and what's silently syncing them

The lab in this writeup is the *foundational* version of the pattern.
The production-grade modern alternative on AWS is **Systems Manager Session Manager** — it provides bastion-equivalent shell access to instances *without* exposing SSH to the internet at all.
No bastion host with a public IP, no SSH keys to manage, no port 22 open anywhere.
Authentication is via IAM, every session is logged in CloudTrail, and access can be granted or revoked through IAM policies in real time.

Building this SSH-based bastion first is the right way to learn the pattern — it makes the underlying concepts (key authentication, security group chaining, defense in depth) tangible.
But in any account I worked on professionally, my goal would be to migrate human access from SSH to Session Manager whenever possible.
Same architectural intent, dramatically smaller attack surface.

## What's next

Next on the list is **NAT Gateways** — giving the private server outbound-only internet access (so it can pull OS updates and packages) without ever exposing it to inbound traffic from the internet.
After that, I'm planning to build out a multi-tier architecture: web tier, app tier, and database tier, each in its own subnet with its own security group.
That'll be the writeup that pulls all the previous ones together into one production-shaped system.

---

*This is the third writeup in my AWS Solutions Architect portfolio. The first covered [securing a fresh AWS account](2026-04-24-securing-fresh-aws-account.md), and the second covered [building the VPC and subnets this lab runs inside](2026-04-25-mini-segmented-network.md).*
