# Building a Three-Tier Network VPC from Scratch in AWS

## Overview

This guide builds a VPC with three isolated network tiers — presentation (web), application, and data — each in its own set of subnets with its own routing and security posture. Unlike the single-service labs so far (MySQL, MongoDB, DocumentDB), this is a **network architecture** guide: the goal isn't one database, it's the reusable VPC skeleton that every tier of NoorNote (NGINX → FastAPI → Postgres/Mongo) will eventually sit inside.

**Why three tiers, not one flat private subnet:**
Putting everything in one private subnet works, but it means a compromised web server has network-level reach to your database. Splitting into tiers means a security group misconfiguration or a compromised instance in one tier still can't reach the tier two steps away — the web tier can talk to the app tier, but never directly to the data tier. This is defense in depth applied to network layout, not just to individual security group rules.

**Why each tier spans two Availability Zones:**
A three-tier design exists to scale and stay available under load — that requirement doesn't hold if any single tier is pinned to one AZ. The ALB in section 3.2 refuses to attach to a single-AZ target group in the first place; it requires subnets in at least two AZs by design.

### Architecture

<img width="1687" height="1100" alt="image" src="https://github.com/user-attachments/assets/98776c2c-2f0f-4b06-8a13-b5b97c535314" />
.

Traffic flows strictly downward through the tiers, never sideways or skipping a tier:

**Inbound request path**
Internet → IGW → Public subnets → Application Load Balancer → Web tier (private) → App tier (private) → Data tier (private). Each hop is enforced by security groups, not by routing — the local route makes every subnet in the VPC reachable from every other subnet at the network layer, so the entire tiering model depends on getting the security group chain right in section 2.

**Web tier's outbound traffic**
Web tier instances → Private Route Table (`0.0.0.0/0 → NAT Gateway`) → NAT Gateway (public subnet) → IGW → Internet. Needed for package updates and pulling dependencies, same reasoning as the private EC2 labs before this one.

**App tier's outbound traffic**
Same pattern as the web tier — routes through NAT for outbound package/dependency access, but is only *reachable* from the web tier's security group, never from the public subnet or the internet directly.

**Data tier's traffic**
No outbound internet route at all. The data tier's route table has only the local route — if you're running RDS/DocumentDB here (recommended, per the earlier guides), no NAT route is needed since managed services have no OS to patch. If you're running a self-hosted database on EC2 in this tier instead, you'd need to add a NAT route the same way the web/app tiers have one.

---

## 1. VPC & Networking

### 1.1 Create VPC
- CIDR: `10.0.0.0/16`
- Region: `ap-southeast-1`

### 1.2 Create Subnets

Four subnet groups, each spanning two AZs — twelve /24s carved out with room to grow (only eight are used here).

| Tier | Subnet | CIDR | AZ |
|---|---|---|---|
| Public (ALB, NAT, bastion) | Public A | 10.0.1.0/24 | ap-southeast-1a |
| Public (ALB, NAT, bastion) | Public B | 10.0.2.0/24 | ap-southeast-1b |
| Web (presentation) | Web A | 10.0.11.0/24 | ap-southeast-1a |
| Web (presentation) | Web B | 10.0.12.0/24 | ap-southeast-1b |
| App (application) | App A | 10.0.21.0/24 | ap-southeast-1a |
| App (application) | App B | 10.0.22.0/24 | ap-southeast-1b |
| Data (back-end) | Data A | 10.0.31.0/24 | ap-southeast-1a |
| Data (back-end) | Data B | 10.0.32.0/24 | ap-southeast-1b |

### 1.3 Internet Gateway
- Create IGW, attach to VPC.

### 1.4 NAT Gateways

Production practice is **one NAT Gateway per AZ**, not one shared NAT Gateway — a single NAT Gateway is both a cost saving and a single point of failure that defeats the two-AZ design everywhere else in this VPC.

- Allocate two Elastic IPs.
- Create NAT Gateway 1 in Public A, associate EIP 1.
- Create NAT Gateway 2 in Public B, associate EIP 2.

For a dev/learning environment where cost matters more than AZ-level redundancy, one NAT Gateway in Public A is an acceptable trade-off — just note that if Public A's AZ has an issue, App/Web instances in AZ b lose outbound internet even though they're otherwise healthy.

### 1.5 Route Tables

Three tiers, three distinct route tables. This is the step where "three-tier" actually becomes real — everything before this was just subnets sitting in a VPC with no traffic policy.

**Step 1 — Create the public route table**
- Console: VPC → Route tables → Create route table. Name: `public-rt`, VPC: this VPC.
- Add route: `0.0.0.0/0` → target: the Internet Gateway from 1.3.
- Associate subnets: Public A, Public B.

**Step 2 — Create the web/app route table**
- Create route table. Name: `private-app-rt`, VPC: this VPC.
- Add route: `0.0.0.0/0` → target: NAT Gateway 1 (or NAT Gateway 2 — pick per-subnet if using two, see step 2b).
- Associate subnets: Web A, Web B, App A, App B — a single route table can be associated with multiple subnets, and sharing one here is fine since all four subnets have identical outbound needs.

**Step 2b — If using per-AZ NAT Gateways (recommended for production)**
Instead of one shared `private-app-rt`, create two: `private-app-rt-a` (routes `0.0.0.0/0` → NAT Gateway 1, associated with Web A + App A) and `private-app-rt-b` (routes `0.0.0.0/0` → NAT Gateway 2, associated with Web B + App B). This keeps each AZ's outbound traffic inside that same AZ instead of crossing AZs to reach a NAT Gateway in the other one — avoiding both the cross-AZ latency and the inter-AZ data transfer charge.

**Step 3 — Create the data route table**
- Create route table. Name: `private-data-rt`, VPC: this VPC.
- Add no default route — leave only the automatic local route (`10.0.0.0/16 → local`) that every route table gets.
- Associate subnets: Data A, Data B.

**Step 4 — Verify associations**
```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<your-vpc-id>" \
  --query "RouteTables[*].[RouteTableId,Associations[*].SubnetId,Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId]]" \
  --output table
```
Confirm `public-rt` shows a route to an `igw-*` target, the private-app table(s) show a route to a `nat-*` target, and `private-data-rt` shows only the local route with no `nat-*` or `igw-*` entry. A stray `0.0.0.0/0` route on the data table is the single most common mistake that quietly undoes the entire tiering model — the data tier would gain outbound internet access it was never meant to have.

---

## 2. Security Groups

The security groups form a chain — each tier only accepts traffic from the SG immediately above it, never from a CIDR range and never skipping a tier.

**Bastion-SG** (public subnet)
- Inbound: SSH (22) from your IP
- Outbound: all

**ALB-SG** (public subnet, attached to the load balancer)
- Inbound: HTTP (80) / HTTPS (443) from `0.0.0.0/0`
- Outbound: HTTP (80) (or your app's port) to Web-SG only

**Web-SG** (web tier)
- Inbound: HTTP (80) (or app port) from ALB-SG only
- Inbound: SSH (22) from Bastion-SG
- Outbound: app port (e.g. 8000) to App-SG only, plus 443/80 to `0.0.0.0/0` for NAT-routed package updates

**App-SG** (app tier)
- Inbound: app port (e.g. 8000) from Web-SG only
- Inbound: SSH (22) from Bastion-SG
- Outbound: DB port (e.g. 5432/3306/27017) to Data-SG only, plus 443/80 to `0.0.0.0/0` for NAT-routed package updates

**Data-SG** (data tier)
- Inbound: DB port from App-SG only
- Outbound: none required beyond default (data tier has no internet route to use anyway, per 1.5)

Note the pattern: every SG rule references another **security group ID** as its source, never a CIDR block, except for the two genuinely public entry points (Bastion-SG's SSH rule and ALB-SG's HTTP/HTTPS rule). This is what makes the chain enforceable — SG-to-SG references don't care which subnet or AZ an instance is in, so the rule stays correct even as instances scale in and out via Auto Scaling Groups in section 3.

---

## 3. Launch Resources

### 3.1 Bastion Host
- Subnet: Public A
- Auto-assign public IP: enabled
- SG: Bastion-SG
- Key pair: `bastion-key.pem`

### 3.2 Application Load Balancer

The ALB is the single public entry point for the whole VPC — nothing else in this design accepts inbound traffic from the internet.

**Step 1 — Create the target group first**
The wizard for creating an ALB asks for a target group before it lets you finish, so create it first to avoid the wizard state resetting.
- Console: EC2 → Target groups → Create target group.
- Target type: **Instances** (or **Instance** if using an Auto Scaling Group, which is the plan in 3.3).
- Protocol/port: HTTP / 8080 (or whatever port your web tier listens on).
- VPC: this VPC.
- Health check path: `/health` (or your app's actual health endpoint) — an ALB with a health check pointed at a path that doesn't exist will mark every target unhealthy and silently 502 all traffic, so confirm this path is real before moving on.
- Name: `web-tier-tg`.

**Step 2 — Create the load balancer**
- Console: EC2 → Load Balancers → Create load balancer → **Application Load Balancer**.
- Name: `app-alb`.
- Scheme: **Internet-facing**.
- IP address type: IPv4.
- VPC: this VPC.
- Mappings: select Public A and Public B — the wizard blocks you from proceeding with only one subnet, which is the ALB enforcing the two-AZ requirement mentioned in the Overview.

**Step 3 — Configure listeners**
- Listener 1: HTTP:80 → forward to `web-tier-tg`.
- If you have a certificate, add Listener 2: HTTPS:443 → forward to `web-tier-tg`, and optionally change Listener 1 to redirect HTTP → HTTPS instead of forwarding directly.

**Step 4 — Security group**
- Attach ALB-SG (from section 2) — remove the default SG the wizard pre-selects.

**Step 5 — Create and verify**
- Click **Create load balancer**. Status moves `Provisioning` → `Active`, usually within a couple of minutes.
- Verify from the CLI:
```bash
aws elbv2 describe-load-balancers \
  --names app-alb \
  --query "LoadBalancers[0].[State.Code,VpcId,AvailabilityZones[*].SubnetId]" \
  --output table
```
Confirm `State.Code` is `active` and both Public A and Public B subnet IDs appear.

**Step 6 — Confirm target health once instances exist**
This step depends on 3.3 being done first — come back to it after launching web tier instances:
```bash
aws elbv2 describe-target-health \
  --target-group-arn <web-tier-tg-arn> \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" \
  --output table
```
`healthy` means the ALB's health check is succeeding against that instance's health check path; `unhealthy` almost always traces back to either the wrong port in the target group, Web-SG not allowing inbound from ALB-SG, or the health check path returning a non-2xx response.

### 3.3 Web Tier (Presentation)

- Launch template: AMI with your web server (NGINX/reverse proxy) pre-baked or bootstrapped via user data.
- Subnet placement: Web A, Web B (via an Auto Scaling Group, not launched directly — so instances spread across both AZs automatically).
- Auto Scaling Group: min 2, desired 2, max 4 (adjust to load) — target group: `web-tier-tg` from 3.2, so the ASG registers/deregisters instances with the ALB automatically as it scales.
- SG: Web-SG.
- Auto-assign public IP: disabled — this tier is private; all inbound traffic arrives via the ALB, never directly.

### 3.4 App Tier (Application)

- Launch template: AMI with your app/API server (e.g. FastAPI) pre-baked or bootstrapped via user data.
- Subnet placement: App A, App B via Auto Scaling Group.
- Auto Scaling Group: min 2, desired 2, max 4.
- SG: App-SG.
- Auto-assign public IP: disabled.
- Reachable only from Web-SG, per the security group chain in section 2 — there's no ALB in front of this tier in the minimal version of this design; the web tier calls app tier instances directly (e.g. via an internal ALB or service discovery, which is a good next lab once this VPC skeleton is working).

### 3.5 Data Tier (Back-End)

- Use a managed service here rather than self-hosted EC2 — RDS (MySQL/Postgres) or DocumentDB (MongoDB-compatible), per the earlier guides in this series.
- DB Subnet Group: Data A, Data B.
- SG: Data-SG.
- Public access: No.
- No NAT route needed on `private-data-rt` for a managed service, per section 1.5.

---

## 4. SSH Access (Agent Forwarding)

On local machine:
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ubuntu@<bastion-public-ip>
```

From inside the bastion, jump to any instance in the web or app tier for debugging (their SGs allow SSH from Bastion-SG):
```bash
ssh ubuntu@10.0.11.x   # a web tier instance
ssh ubuntu@10.0.21.x   # an app tier instance
```

There's no SSH path to the data tier if it's a managed RDS/DocumentDB service — access it over the network from the app tier instead, same as the earlier database guides.

---

## 5. End-to-End Verification

```bash
# From your local machine — hits the public ALB
curl -I http://<alb-dns-name>

# From a web tier instance (via bastion) — confirms it can reach the app tier
ssh ubuntu@10.0.11.x "curl -I http://10.0.21.x:8000/health"

# From an app tier instance (via bastion, then hop again) — confirms it can reach the data tier
ssh ubuntu@10.0.21.x "curl -I http://<data-tier-endpoint>:5432"   # or your DB's actual port/protocol
```

Each hop should succeed only through its intended path. As a negative test, confirm a web tier instance **cannot** reach the data tier directly:
```bash
ssh ubuntu@10.0.11.x "timeout 3 curl -I http://<data-tier-endpoint>:5432"
```
This should time out — Data-SG only allows inbound from App-SG, not Web-SG, even though the local route makes the two subnets reachable at the network layer.

---

## 6. Validation Checklist

- [ ] All eight subnets created, two per tier, across two AZs
- [ ] `public-rt` routes `0.0.0.0/0` to the IGW; associated with both public subnets
- [ ] Private-app route table(s) route `0.0.0.0/0` to a NAT Gateway; associated with both web and both app subnets
- [ ] `private-data-rt` has no default route — local only
- [ ] Security groups reference each other by SG ID, not CIDR, except at the two public entry points
- [ ] ALB is `active`, spans both public subnets, and target group health checks pass
- [ ] Web tier instances have no public IP and are only reachable via the ALB
- [ ] App tier instances have no public IP and are only reachable from Web-SG
- [ ] Data tier is a managed service (or, if self-hosted, has no direct route from the web tier)
- [ ] A direct web-tier-to-data-tier connection attempt fails/times out

---

## 7. Quick Reference

**Network layout**
| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnets | 10.0.1.0/24 (AZ a), 10.0.2.0/24 (AZ b) |
| Web subnets | 10.0.11.0/24 (AZ a), 10.0.12.0/24 (AZ b) |
| App subnets | 10.0.21.0/24 (AZ a), 10.0.22.0/24 (AZ b) |
| Data subnets | 10.0.31.0/24 (AZ a), 10.0.32.0/24 (AZ b) |
| Public RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| Private-app RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → NAT Gateway` |
| Private-data RT | `10.0.0.0/16 → local` only |

**Security groups (chain)**
| SG | Inbound from | Port | Outbound to | Port |
|---|---|---|---|---|
| Bastion-SG | Your IP | 22 | all | all |
| ALB-SG | `0.0.0.0/0` | 80/443 | Web-SG | 80/8080 |
| Web-SG | ALB-SG, Bastion-SG | 80/8080, 22 | App-SG, `0.0.0.0/0` | 8000, 443/80 |
| App-SG | Web-SG, Bastion-SG | 8000, 22 | Data-SG, `0.0.0.0/0` | DB port, 443/80 |
| Data-SG | App-SG | DB port | — | — |

**ALB + tier resources**
| Resource | Placement | Notes |
|---|---|---|
| ALB | Public A + B | Internet-facing, only public entry point |
| Web tier ASG | Web A + B | Behind ALB, no public IP |
| App tier ASG | App A + B | Behind web tier, no public IP |
| Data tier | Data A + B | Managed service, no SSH/public access |

**Fast diagnostics**
```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"     # confirm route associations
aws elbv2 describe-target-health --target-group-arn <tg-arn>              # ALB → web tier health
curl -I http://<alb-dns-name>                                             # public entry point check
```

**Golden rules**
- Traffic flows down the tiers (web → app → data), enforced by security groups, not by routing — the local route makes every subnet reachable from every other subnet regardless of tier.
- Security groups reference other security groups, never CIDR blocks, except at the two genuinely public boundaries (internet → ALB, admin IP → bastion).
- A default route on the data tier's route table silently breaks the entire isolation model — check this first if something feels wrong.
- The ALB requires ≥2 AZs by design; so should every tier behind it, or you've built a highly-available front door onto a single point of failure.
- Managed data-tier services (RDS, DocumentDB) need no NAT route at all — a `0.0.0.0/0` route appearing on `private-data-rt` is a signal something was misconfigured, not a feature.

---

## Key Takeaways

- Tiering is enforced by the security group chain, not by subnet placement alone — subnets provide the *option* for isolation, security groups make it *actual*.
- Splitting web and app into separate subnet tiers (rather than one combined "private" tier) means a compromised web-facing instance still can't reach the database directly, even with full network-layer connectivity to that subnet.
- Every tier that needs to scale or stay available needs two AZs, not just the load balancer sitting in front of it.
- One NAT Gateway per AZ avoids both a cost trade-off (cross-AZ data transfer) and a redundancy trade-off (single point of failure) — a shared NAT Gateway is a reasonable dev-environment shortcut, not a production pattern.
- This VPC skeleton is the target for NoorNote's own three tiers: NGINX in the web tier, FastAPI in the app tier, and Postgres/Mongo (via RDS/DocumentDB) in the data tier.
