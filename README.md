# SecureWebLab — Segmented Web Platform on AWS

This project is a complete AWS network segmentation build, put together end-to-end from zero prior AWS experience: a public status website served from S3, and a private backend server reachable by nothing on the internet — not even by the administrator — except through an identity-verified, fully logged session. No SSH key ever exists anywhere in this build.

It demonstrates the discipline network segmentation actually requires: deciding deliberately what should be public and what shouldn't be, enforcing that boundary at the network layer instead of trusting a firewall rule to catch it, and proving — not assuming — that the private side is actually unreachable.

- **Region:** `us-east-1` (N. Virginia) — held constant for the entire build
- **Account:** personal AWS account, AWS Free Tier + signup credit
- **Tools used:** AWS Console only (no CLI, no Infrastructure-as-Code) — deliberately, to understand each piece by clicking through it before automating any of it later

Every control described below was verified with a real test — a command that returned real output, a screenshot of a real console state, a log entry that actually appeared — not configured and assumed to work.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Network Foundation — VPC & Subnets](#network-foundation)
3. [The Public Path — Internet Gateway & Routing](#public-path)
4. [Reducing Attack Surface — Security Group](#security-group)
5. [The Private Path — NAT Gateway & Private Routing](#private-path)
6. [The Private Server — EC2](#private-server)
7. [Identity-Based Access — IAM Role for Session Manager](#iam-role)
8. [Proof of Access — Session Manager](#session-manager)
9. [The Public Half — S3 Static Website](#s3-website)
10. [Proof of Logging — CloudTrail](#cloudtrail)
11. [Conclusion & Next Steps](#conclusion)

---

## Architecture Overview

![SecureWebLab architecture diagram](diagrams/architecture.svg)
*The complete build: a single VPC split into a public and a private subnet, a public S3 website sitting entirely outside the VPC, and an IAM-authenticated path into the private server that never touches an open port.*

### Skills covered

| Area | Where it's demonstrated |
|---|---|
| VPC design & CIDR planning | Network Foundation |
| Public/private subnet segmentation | Network Foundation |
| Internet Gateway & route table design | Public Path |
| Least-privilege security group design | Security Group |
| NAT Gateway outbound-only routing | Private Path |
| Cost-aware resource lifecycle management | Private Path |
| EC2 provisioning without SSH exposure | Private Server |
| IAM roles for resource identity (not user identity) | IAM Role |
| IAM-authenticated remote access (AWS Systems Manager) | Session Manager |
| S3 static website hosting & bucket policy design | S3 Website |
| Access logging & audit trail verification | CloudTrail |

---

## Network Foundation

**A private, isolated network, deliberately divided into a public half and a private half before anything else was built.**

Everything in this project sits inside one VPC, `SecureWebLab-VPC` (`10.0.0.0/16`). Before any server, any route, or any access rule existed, the network itself was split into two subnets:

| Subnet | CIDR | Purpose |
|---|---|---|
| `SecureWebLab-Public-Subnet` | `10.0.1.0/24` | Will hold anything that legitimately needs a path to the internet (the NAT Gateway) |
| `SecureWebLab-Private-Subnet` | `10.0.2.0/24` | Will hold the actual server — no internet-facing door at all |

### Why this comes first, before anything else

Almost every real cloud security finding traces back to how the network was shaped on day one. A single, undivided VPC is the network equivalent of a building with no internal walls — if anything inside gets compromised, it's immediately standing next to everything else. Splitting public and private *before* placing a single resource means the boundary is structural, not a rule someone has to remember to configure correctly later.

### Key decisions and why

- **`/24` for each subnet, not smaller.** 256 addresses per subnet is far more than this project needs, but it leaves room to add more resources later without re-architecting the network.
- **`.1.x` for public, `.2.x` for private.** Not a technical requirement — a naming convention, so that any IP address in this environment is self-describing at a glance. This is a real pattern companies use across dozens of VPCs for the same reason.
- **Single Availability Zone.** A production system would spread across multiple AZs for resilience; this project intentionally stayed in one (`us-east-1a`) to keep the focus on the segmentation pattern itself rather than multi-AZ complexity.

### Evidence

![VPC created and available](Screenshots/VPC%20proof%201.png)
*VPC Dashboard → Your VPCs, showing `SecureWebLab-VPC` (`10.0.0.0/16`) in state Available.*

![Both subnets created inside the VPC](Screenshots/subnets%20segmented%20proof%202.png)
*Subnets list showing `SecureWebLab-Public-Subnet` and `SecureWebLab-Private-Subnet`, both under `SecureWebLab-VPC`, both Available.*

---

## Public Path

**An Internet Gateway attached to the VPC, and a route table that gives that door to the public subnet — and only the public subnet.**

A VPC has no path to the internet at all until an Internet Gateway (IGW) is attached to it. `SecureWebLab-IGW` was created and attached, then a dedicated route table, `SecureWebLab-Public-RT`, was built with a single route: `0.0.0.0/0 → SecureWebLab-IGW` — "anything not otherwise known, send out through the internet door." That route table was then associated with the public subnet only.

### Why the route table is its own object, not a default

AWS creates a "main" route table automatically with every VPC. This project deliberately created a **separate, explicitly-named** route table instead of editing the default one, specifically so it's unambiguous later which route table does what — a real habit that matters once an account has dozens of VPCs and route tables in it.

### The mistake this was built to avoid

The single most common beginner error at this stage is associating the *same* public route table with the private subnet too, "just to be safe." Doing that silently defeats the entire point of having a private subnet — it becomes reachable from the internet the moment anything inside it gets a public IP by accident. This build deliberately kept the public route table scoped to the public subnet alone, and gave the private subnet its own separate route table later (see [Private Path](#private-path)).

### Evidence

![Internet Gateway attached to the VPC](Screenshots/internet%20gateway%20proof%203.png)
*Internet gateways → `SecureWebLab-IGW`, State: Attached, tied to `SecureWebLab-VPC`.*

![Both route tables, correctly scoped](Screenshots/route%20table%20proof%203.png)
*Route tables list showing `SecureWebLab-Public-RT` and `SecureWebLab-Private-RT` as two distinct, explicitly-named tables — not the default main route table.*

---

## Security Group

**The firewall that sits directly on the future EC2 instance — configured with zero inbound rules, on purpose.**

`SecureWebLab-SG` was created with an empty inbound rule set. No SSH (port 22), no HTTP, nothing. Outbound traffic was left at its default (allowed), since the server still needs to reach out for software updates and to check in with AWS.

### Why an empty inbound rule set is the actual point of this project

Nearly every beginner AWS tutorial says to open port 22 to your own IP address "so you can connect later." This project deliberately skips that, because the entire access model here is built around [Session Manager](#session-manager) instead — which needs precisely zero inbound rules to function. A security group with nothing open at all is, on paper, as close to fully locked down as a running resource can be.

This single decision is also the most common real-world audit finding this project was built to demonstrate *not* making: security groups with unnecessary open ports — especially SSH open to `0.0.0.0/0`, the entire internet — are one of the most frequent root causes behind real cloud breaches.

### Evidence

![Security group with zero inbound rules](Screenshots/security%20group%20proof%204.png)
*`SecureWebLab-SG`, Description: "no inbound SSH", attached to `SecureWebLab-VPC`.*

---

## Private Path

**A NAT Gateway that lets the private subnet reach *out* to the internet, without ever allowing anything from the internet to reach *in* — plus the one deliberate cost-management discipline this whole project revolves around.**

A NAT Gateway is the only resource in this build that bills continuously — roughly $0.05/hour combined (the gateway itself plus its attached public IPv4 address) whether it's doing anything or not. Everything else in this project — VPC, subnets, security groups, IAM, S3 — is free indefinitely. This asymmetry shaped how the whole build was sequenced: **Phases 5 through 8 were done together, in one sitting, and the NAT Gateway was deleted the moment the private server was proven reachable — not left running between sessions.**

`SecureWebLab-NATGW` was created inside the *public* subnet (even though it exists to serve the private one — this placement is deliberate, since the NAT Gateway itself needs its own path to the internet through the IGW). A second route table, `SecureWebLab-Private-RT`, was created with the route `0.0.0.0/0 → SecureWebLab-NATGW`, and associated with the private subnet only.

### A real issue hit and resolved: Zonal vs. Regional NAT Gateway

AWS shipped a new NAT Gateway feature — **Regional availability mode** — in November 2025, after most tutorials (including the plan this project started from) were written. The console defaulted to offering "Regional" mode, which automatically spans every Availability Zone in the VPC and doesn't ask for a specific subnet at all. That's a legitimate feature for production environments spread across multiple AZs, but it would have hidden the exact concept this phase exists to teach — *why* a NAT Gateway specifically lives in the public subnet while serving the private one. **Zonal** mode was selected instead, explicitly placing the NAT Gateway in `SecureWebLab-Public-Subnet`, matching the architecture this project set out to understand and keeping the cost model predictable.

### Key decisions and why

- **A separate, dedicated route table for the private subnet.** Same reasoning as the public route table — explicit, named, and unambiguous about what it does.
- **NAT Gateway deleted immediately after Phase 8 was verified working, not at end-of-day.** The billing clock starts the moment the resource exists; stopping it the instant it's no longer needed, rather than "later today," is the actual discipline — not a symbolic gesture.
- **Elastic IP released along with the NAT Gateway.** An allocated-but-unattached Elastic IP quietly bills on its own — releasing it is part of the same cleanup step, not a separate afterthought.

### Evidence

![NAT Gateway available, Zonal mode, in the public subnet](Screenshots/NAT%20gateway%20proof%205.png)
*NAT gateways → `SecureWebLab-NATGW`, Connectivity type: Public, State: Available, Availability: Zonal.*

---

## Private Server

**The actual EC2 instance — the thing every prior phase exists to protect the path to.**

`SecureWebLab-Server` (`t3.micro`, Amazon Linux 2023) was launched directly into `SecureWebLab-Private-Subnet`, with **Auto-assign public IP explicitly disabled** and `SecureWebLab-SG` attached. No key pair was created — "Proceed without a key pair" was selected deliberately, since this build never intends to use SSH at all.

### The moment that looks broken but isn't

Immediately after launch, the instance is running but completely unreachable — no SSH, no public IP, nothing. That blank "Public IPv4 address" column isn't a bug; it's the proof the instance is genuinely private. For a real backend server, database, or internal service, "exists, is running, and cannot be reached from outside" is frequently the *intended* end state, not a problem to fix by opening access. The next two phases are specifically about proving the legitimate administrator can still reach it when needed, without ever compromising that isolation for anyone else.

### Evidence

![EC2 instance running, in the private subnet, no public IP](Screenshots/EC2%20running%20proof%206.png)
*Instances list showing `SecureWebLab-Server`, State: Running, Instance type: t3.micro, 3/3 status checks passed, Public IPv4 column empty.*

---

## IAM Role

**"Permission papers" attached to the instance itself — not a person — granting exactly one capability: the ability to be reached through Session Manager.**

`SecureWebLab-SSM-Role` was created as an **IAM Role** (not an IAM User — a role is an identity a *resource* wears, not a person who logs in), with the AWS-managed policy `AmazonSSMManagedInstanceCore` attached, then attached to `SecureWebLab-Server`.

### Why this specific policy, and nothing broader

`AmazonSSMManagedInstanceCore` grants only what Session Manager needs to function — checking in with AWS and accepting an authenticated session. It is not `AdministratorAccess`, not broad EC2 permissions, nothing beyond this one narrow capability. A common real-world audit finding is EC2 instances carrying IAM roles with far more permission than the instance actually uses — sometimes literally full admin access, out of convenience. This role was scoped to the minimum from the moment it was created, not tightened later after a finding.

### Evidence

![IAM role attached to the running instance](Screenshots/IAM%20role%20Proof%208.png)
*Instance Details tab showing `IAM role: SecureWebLab-SSM-Role`, confirming the role is actually attached, not just created.*

---

## Session Manager

**The actual proof this entire architecture works: reaching a server with no SSH, no password, no open port, and no key file — using nothing but a verified AWS identity and one narrow permission.**

### A real issue hit and resolved: "Not connected" on first attempt

The first connection attempt failed with `SSM Agent unable to acquire credentials` and `Session Manager default role not enabled`. This is a timing issue, not a misconfiguration — the SSM Agent on a freshly role-attached instance needs a window of time to notice the change and successfully complete its first credential handshake with AWS, and it retries on its own roughly every few minutes rather than instantly. The instance's own status (3/3 checks passed, role correctly shown as attached) confirmed the instance and IAM side were both already correct; a manual refresh followed by a short wait resolved the connection on the next attempt, with no configuration changed.

### The proof itself

Once connected, the session was used to verify two things concretely:

1. **Outbound connectivity through the NAT Gateway:** `curl -s ifconfig.me` returned the NAT Gateway's Elastic IP — direct proof the private instance's outbound traffic is routing through the NAT path, not directly.
2. **A real service, running on a genuinely private instance:** `httpd` was installed, started, and enabled, a test page was written to it, and `curl localhost` returned that page's real HTML — proof a working web server exists on an instance reachable by nothing on the public internet.

```
curl -s ifconfig.me
sudo dnf install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<h1>Private backend server is alive</h1>" | sudo tee /var/www/html/index.html
curl localhost
```

What this chain actually demonstrates: identity-based access replacing network-based access entirely — no shared secret, no network-level "you're in, you're trusted" assumption, just an AWS identity being verified and a specific, narrow permission granted deliberately ahead of time.

### Evidence

![Full Session Manager session: package install, service start, curl proof](Screenshots/Session%20Manager%20proof%208.png)
*The complete terminal session showing `httpd` installing successfully (itself proof the NAT path works), the service starting, and `curl localhost` returning the real test page.*

---

## S3 Website

**The deliberate public half of this project — a status page anyone on the internet can visit, built with the same intentionality as the private half, just in the opposite direction.**

An S3 bucket was created with "Block all public access" explicitly unchecked (with the confirming warning acknowledged), an `index.html` uploaded, static website hosting enabled, and a bucket policy attached granting `s3:GetObject` — read-only — to everyone, scoped to this bucket alone.

### Why this isn't a contradiction of the "make everything private" theme

A properly segmented architecture doesn't make *everything* private — it makes the *right* things private, and is equally deliberate about what's intentionally public. This bucket is public because it's supposed to be, with a policy limited to read-only access on exactly this bucket — not an accidentally-public bucket holding something sensitive, which is the actual pattern behind most real S3 breach headlines. Being able to explain specifically *why* something is public, rather than "it happened to end up that way," is the real distinction being demonstrated here.

### Evidence

![Status site live at the S3 website endpoint](Screenshots/s3%20bucket%20live%20proof%209.png)
*The public status page loading successfully at the bucket's static website hosting endpoint.*

---

## CloudTrail

**Proof that every access to the private server is automatically, permanently, and tamper-resistantly logged — not just claimed.**

CloudTrail retains the last 90 days of account activity automatically, with zero setup. Filtering Event history by Event name = `StartSession` surfaced every Session Manager connection made during this build, each with a timestamp, the calling identity, and the event source (`ssm.amazonaws.com`).

### Why this matters more than it looks like it should

Traditional SSH access to a shared key frequently leaves **no record at all** of who actually connected and when, by default. This architecture provides that record automatically, for every single connection, at no extra cost and no extra configuration — which is precisely the kind of answer a real security team wants to "how would you know if someone accessed this server," and precisely the kind of answer traditional SSH access usually can't give without additional tooling bolted on afterward.

### Evidence

![CloudTrail Event history showing logged StartSession events](Screenshots/proof%2010%20cloudtrail%20screenshot.png)
*Event history filtered to `StartSession`, showing three logged connection events with timestamp, identity, and event source — direct proof access to this server is auditable.*

---

## Conclusion

This build is one continuous decision about where trust should and shouldn't be assumed, not a checklist of AWS services clicked through in sequence. Network Foundation established the boundary before anything crossed it. Public Path and Security Group made the public side's exposure deliberate and the private side's exposure exactly zero. Private Path connected the private subnet to the internet in one direction only, with a real cost-management discipline built around the one resource that actually bills. Private Server proved that "exists, running, unreachable" is a valid and often correct end state. IAM Role and Session Manager replaced network-level trust with identity-level trust, end to end, with zero SSH anywhere in the build. S3 Website proved the same deliberateness applies to what's *meant* to be open. CloudTrail closed the loop, proving every access to the private side is logged, not just theoretically loggable.

Two real issues were hit and resolved during the build — a newly-released NAT Gateway console option that would have hidden the subnet-placement concept this project exists to teach, and a timing-related SSM Agent credential handshake failure on first connection — and both are documented above rather than smoothed over, because that troubleshooting is as much a part of understanding this architecture as the parts that worked cleanly the first time.

### CV bullet

> Designed and deployed a segmented AWS network architecture consisting of a public static website (S3, publicly accessible) and a private backend server isolated in a dedicated private subnet with zero exposed inbound ports. Implemented outbound-only internet access via a NAT Gateway, and replaced traditional SSH key-based access with IAM role-authenticated AWS Systems Manager Session Manager. Documented the full architecture, including deliberate cost-management practices around ephemeral NAT Gateway usage.

### Suggested next steps

- Enable MFA on the AWS root account and move day-to-day work to a dedicated IAM admin user, rather than operating as root.
- Add a CloudWatch billing alarm alongside the existing AWS Budgets alert, as a second independent cost tripwire.
- Run IAM Access Analyzer against `SecureWebLab-SSM-Role` to formally confirm its permissions are as narrowly scoped as designed.
- Use this project's networking and IAM primitives — VPC segmentation, least-privilege roles, identity-based access — as the foundation for a larger, production-style build: Application Load Balancer, Auto Scaling, a Multi-AZ managed database, and automated monitoring, solving a specific, real operational problem rather than demonstrating the pattern in isolation.