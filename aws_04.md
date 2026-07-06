# Deploying MySQL Using Amazon RDS (Industry Standard Approach)

## Overview

This guide deploys MySQL as a **managed Amazon RDS instance** instead of self-hosting it on EC2 with systemd. This is the approach production teams reach for by default — AWS handles patching, backups, failover, and storage scaling, so there's no `mysqld.service` to babysit. EC2 is still used, but only for the **application/bastion tier** that connects to the database.

**Why this is the industry-standard default (not the EC2+systemd lab from before):**
Self-managing MySQL on EC2 (as in the earlier guide) is valuable for *learning* how MySQL and Linux service management work under the hood. In production, that same setup means you own patching, backup scripts, failover logic, and disk monitoring yourself. RDS removes all of that operational burden — which is exactly why it's the default choice unless you have a specific reason not to use it (licensing, exotic config, extreme cost optimization at massive scale).

### Architecture

<img width="516" height="518" alt="image" src="https://github.com/user-attachments/assets/ff0b1ad9-d985-4fab-88c5-10f63e6c5a06" />

.

**Key structural difference from the EC2+systemd setup:** RDS doesn't need outbound internet access — AWS manages the instance itself, so there's no OS to patch or packages to fetch. That means the private subnets hosting RDS **don't require a NAT Gateway** at all, unless other private resources in those subnets need outbound access. This alone removes a cost line and a failure point compared to self-hosting.

**Why RDS needs *two* private subnets, not one:**
Amazon RDS requires a **DB Subnet Group** spanning at least two Availability Zones — even for a Single-AZ deployment, the subnet group itself must be multi-AZ so RDS can fail over into a standby AZ if you later enable Multi-AZ, or if AWS needs to replace underlying hardware.

## 1. VPC & Networking

### 1.1 Create VPC
- CIDR: `10.0.0.0/16`
- Region: `ap-southeast-1`

### 1.2 Create Subnets
| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| Public | 10.0.1.0/24 | ap-southeast-1a | Bastion / app tier |
| Private A | 10.0.2.0/24 | ap-southeast-1a | RDS subnet group (primary) |
| Private B | 10.0.3.0/24 | ap-southeast-1b | RDS subnet group (standby AZ) |

### 1.3 Internet Gateway
- Create IGW, attach to VPC.

### 1.4 Route Tables
**Public RT**
- `10.0.0.0/16` -> local
- `0.0.0.0/0` -> IGW
- Associate with public subnet.

**Private RT**
- `10.0.0.0/16` -> local
- No `0.0.0.0/0` route needed — RDS doesn't require outbound internet.
- Associate with both private subnets.

> No NAT Gateway is required for this architecture unless you're also running other private EC2 resources that need outbound access.

### 1.5 DB Subnet Group

A DB Subnet Group is a named collection of subnets (spanning ≥2 AZs) that tells RDS *where in your VPC* it's allowed to place the database and its standby. You create this before launching the instance — RDS won't let you skip it, since it has no way to pick subnets on your own.

**Step 1 — Confirm your subnets exist and are tagged correctly**
Before opening the RDS console, verify Private A (10.0.2.0/24, AZ `a`) and Private B (10.0.3.0/24, AZ `b`) are already created from section 1.2, and that neither has a route to an IGW (they should only have the local route, per section 1.4). RDS doesn't enforce this, but placing the DB in a subnet with a public route defeats the purpose of `Publicly accessible: No` in section 3.2.

**Step 2 — Open the DB Subnet Group creation screen**
Console: RDS → **Subnet groups** (left sidebar, under "Databases") → **Create DB subnet group**.

**Step 3 — Fill in the group details**
| Field | Value |
|---|---|
| Name | `app-db-subnet-group` |
| Description | e.g. "Private subnets for app-db-prod, spans AZ a and b" |
| VPC | Your custom VPC from section 1.1 |

**Step 4 — Add Availability Zones and subnets**
- Under "Availability Zones", select both `ap-southeast-1a` and `ap-southeast-1b`. Selecting the AZ first is what makes its subnets selectable in the next dropdown.
- Under "Subnets", select Private A (`10.0.2.0/24`) and Private B (`10.0.3.0/24`). If a subnet doesn't appear, it usually means its AZ wasn't selected above, or it belongs to a different VPC.

**Step 5 — Create and verify**
- Click **Create**. The group should show `Status: Complete` within a few seconds.
- Verify via CLI:
```bash
aws rds describe-db-subnet-groups \
  --db-subnet-group-name app-db-subnet-group \
  --query "DBSubnetGroups[0].Subnets[*].[SubnetIdentifier,SubnetAvailabilityZone.Name]" \
  --output table
```
You should see exactly two subnets listed, one per AZ. If only one appears, the group won't satisfy RDS's multi-AZ requirement and instance creation in section 3.2 will fail with a subnet group error.

This group is referenced by name in section 3.2 (DB Subnet Group field) — nothing further to do with it until then.

---

## 2. Security Groups

**Bastion-SG** (public subnet)
- Inbound: SSH (22) from your IP
- Outbound: all

**RDS-SG** (attached to the RDS instance, not a subnet)
- Inbound: MySQL/Aurora (3306) from Bastion-SG (or App-tier-SG)
- Outbound: all (default; RDS rarely needs outbound rules tightened)

---

## 3. Launch Resources

### 3.1 Bastion Host / App Server
- Subnet: public
- Auto-assign public IP: enabled
- SG: Bastion-SG
- Key pair: `bastion-key.pem`

### 3.2 RDS MySQL Instance

This walks through the RDS console's "Create database" wizard end to end. Every field below maps to a specific decision — skipping one silently defaults to something you may not want (e.g. public access defaults to "Yes" if you're not careful).

**Step 1 — Open the wizard**
Console: RDS → **Databases** → **Create database**.

**Step 2 — Choose a creation method**
Select **Standard create** (not "Easy create") — this exposes the subnet group, SG, and public-access fields the "Easy create" path hides behind defaults.

**Step 3 — Engine options**
| Field | Value |
|---|---|
| Engine type | MySQL |
| Engine version | Latest supported minor version (e.g. 8.0.x) — avoid pinning to an old minor unless a driver/compatibility issue requires it |

**Step 4 — Templates**
Choose **Production** for Multi-AZ + provisioned IOPS defaults, or **Dev/Test** for a cheaper Single-AZ baseline. This only pre-fills later fields — everything it sets can still be overridden manually below.

**Step 5 — Settings**
| Field | Value |
|---|---|
| DB instance identifier | `app-db-prod` |
| Master username | e.g. `admin` |
| Credentials management | Select **Manage master credentials in AWS Secrets Manager** (recommended) instead of self-managed — this avoids a plaintext password ever existing in the console or in your notes |

**Step 6 — Instance configuration**
| Field | Value |
|---|---|
| DB instance class | `db.t3.micro` for dev/test, `db.t3.small` or larger for light production |

**Step 7 — Storage**
| Field | Value |
|---|---|
| Storage type | gp3 |
| Allocated storage | Start at 20 GB |
| Storage autoscaling | Enable, set a sensible max (e.g. 100 GB) so a runaway table doesn't silently balloon your bill |

**Step 8 — Connectivity (the section that determines network isolation)**
| Field | Value |
|---|---|
| Compute resource | Don't connect to an EC2 compute resource (connect via bastion/app tier instead) |
| VPC | Your custom VPC from section 1.1 |
| DB subnet group | `app-db-subnet-group` created in section 1.5 |
| Public access | **No** — this is the entire point of the private-subnet design; double-check this field specifically, since it's the most common misconfiguration in this wizard |
| VPC security group | Choose existing → `RDS-SG` (remove the default SG AWS pre-selects) |
| Availability Zone | No preference (let RDS choose, unless you have a reason to pin it) |
| Database port | 3306 (default) |

**Step 9 — Multi-AZ**
Under "Availability & durability", enable **Multi-AZ DB instance** for production. This provisions a synchronous standby in the second subnet from your DB Subnet Group and handles automatic failover — this is why section 1.5's two-subnet requirement exists in the first place.

**Step 10 — Backup**
| Field | Value |
|---|---|
| Automated backups | Enable |
| Backup retention period | 7 days minimum for production |
| Backup window | Default (or a low-traffic window if you know one) |

**Step 11 — Encryption**
Enable **encryption at rest**. This can only be set at creation time — enabling it later requires taking a snapshot, copying it with encryption enabled, and restoring from that copy, so decide now.

**Step 12 — Additional configuration**
- Initial database name (optional, creates an empty schema on first boot)
- Enable **Enhanced monitoring** if you want OS-level metrics beyond the default CloudWatch basics
- Leave "Deletion protection" enabled for anything beyond a throwaway dev instance

**Step 13 — Create and wait**
Click **Create database**. Status moves from `Creating` → `Backing up` → `Available`, typically 5-10 minutes for Single-AZ, longer for Multi-AZ. Don't attempt to connect until status is `Available`.

**Step 14 — Verify from the CLI**
```bash
aws rds describe-db-instances \
  --db-instance-identifier app-db-prod \
  --query "DBInstances[0].[DBInstanceStatus,PubliclyAccessible,MultiAZ,DBSubnetGroup.DBSubnetGroupName]" \
  --output table
```
Confirm `PubliclyAccessible` reads `False` and `DBSubnetGroupName` matches `app-db-subnet-group` — these are the two fields most worth double-checking after creation, since fixing them retroactively (especially public access) can require a reboot or a new instance.

---

## 4. SSH Access to Bastion (Agent Forwarding)

On local machine:
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>
```

No jump to a private *instance* is needed for the database itself — RDS is accessed directly via its endpoint from the bastion (or app server) over the network, not via SSH.

---

## 5. Connect to RDS

There's no install step — RDS ships MySQL already running and managed. From the bastion host (or app server) inside the VPC:

```bash
# install just the client, not the server
sudo yum install -y mysql        # Amazon Linux
# or
sudo apt install -y mysql-client # Ubuntu

mysql -h <rds-endpoint>.rds.amazonaws.com -u admin -p
```

Get the endpoint from the RDS console (Databases → your instance → Connectivity & security → Endpoint) or via CLI:
```bash
aws rds describe-db-instances \
  --db-instance-identifier app-db-prod \
  --query "DBInstances[0].Endpoint.Address" \
  --output text
```

**Always connect via the DNS endpoint, never a resolved IP** — RDS can change the underlying IP during failover, patching, or Multi-AZ switchover; the endpoint DNS name stays constant.

---

## 6. Configuration Management (Parameter & Option Groups)

RDS replaces `my.cnf` with **DB Parameter Groups** — you can't edit config files directly on a managed instance.

```bash
# Create a custom parameter group (default groups are not editable)
aws rds create-db-parameter-group \
  --db-parameter-group-name custom-mysql8 \
  --db-parameter-group-family mysql8.0 \
  --description "Custom params for app-db-prod"

# Modify a parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name custom-mysql8 \
  --parameters "ParameterName=max_connections,ParameterValue=200,ApplyMethod=pending-reboot"

# Attach it to the instance (requires a reboot for static parameters)
aws rds modify-db-instance \
  --db-instance-identifier app-db-prod \
  --db-parameter-group-name custom-mysql8 \
  --apply-immediately
```

Some parameters apply immediately; others (`pending-reboot`) require rebooting the instance to take effect — check each parameter's `ApplyType` before assuming a change is live.

---

## 7. Secure & Verify

```bash
mysql -h <rds-endpoint> -u admin -p -e "STATUS;"
```

Recommended hardening (handled differently than self-hosted MySQL):
- Rotate the master password via **Secrets Manager**, not manually.
- Create least-privilege application users — don't use the master user for app connections.
- Enable **IAM database authentication** if you want to avoid long-lived DB passwords entirely.
- Enable **encryption at rest** (must be set at creation time — can't be added retroactively without a snapshot/restore).
- Restrict RDS-SG inbound to only the specific SGs that need access (Bastion-SG, App-tier-SG) — never `0.0.0.0/0`.

Confirm Multi-AZ failover works (production instances only):
```bash
aws rds reboot-db-instance \
  --db-instance-identifier app-db-prod \
  --force-failover
```

---

## 8. Validation Checklist

- [ ] Bastion reachable via SSH from local machine (public RT + IGW)
- [ ] RDS instance shows `Publicly accessible: No`
- [ ] DB Subnet Group spans two AZs
- [ ] RDS-SG allows 3306 only from Bastion-SG/App-tier-SG, not `0.0.0.0/0`
- [ ] Can connect from bastion using the RDS **endpoint DNS name**
- [ ] Automated backups enabled with a sensible retention window
- [ ] Multi-AZ enabled (if production) and failover tested
- [ ] Master credentials stored in Secrets Manager, not hardcoded

---

## 9. Quick Reference

**Network layout**
| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 (ap-southeast-1a) |
| Private subnet A | 10.0.2.0/24 (ap-southeast-1a) |
| Private subnet B | 10.0.3.0/24 (ap-southeast-1b) |
| Public RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| Private RT | `10.0.0.0/16 → local` only — no NAT route needed |
| DB Subnet Group | Private A + Private B |

**EC2 / RDS instance config**
| Item | Bastion Host | RDS MySQL Instance |
|---|---|---|
| Subnet | Public (10.0.1.0/24) | DB Subnet Group (Private A + B) |
| Publicly accessible | N/A (has public IP) | No |
| Security group | Bastion-SG | RDS-SG |
| Instance class | t3.micro | db.t3.micro (dev) / db.t3.small+ (prod) |
| Storage | 8 GB gp3 | 20+ GB gp3, autoscaling enabled |
| Multi-AZ | N/A | Enabled for production |
| Backups | N/A (not applicable) | Automated, 7+ day retention |

**Security groups**
| SG | Direction | Type | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|---|---|
| Bastion-SG | Inbound | SSH | TCP | 22 | Your IP (`x.x.x.x/32`) | Admin SSH access |
| Bastion-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | General outbound + reaching RDS-SG |
| RDS-SG | Inbound | MySQL/Aurora | TCP | 3306 | Bastion-SG (or App-tier-SG) | DB access, scoped to trusted sources only |
| RDS-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Default; rarely needs tightening |

**SSH + connect**
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>

mysql -h <rds-endpoint>.rds.amazonaws.com -u admin -p
```

**Fast diagnostics**
```bash
aws rds describe-db-instances --db-instance-identifier app-db-prod   # full instance status
aws rds describe-db-instances --db-instance-identifier app-db-prod \
  --query "DBInstances[0].Endpoint.Address" --output text            # get endpoint
mysql -h <endpoint> -u admin -p -e "STATUS;"                          # connectivity check
```

**Golden rules**
- Always connect via the RDS **DNS endpoint**, never a cached IP — it can change on failover.
- No NAT Gateway needed for RDS-only private subnets — RDS has no OS to patch.
- Static parameter changes need a reboot; dynamic ones apply immediately — check before assuming.
- Multi-AZ ≠ backups. You need both: one for availability, one for recoverability.
- `Publicly accessible: No` + correct SG + inside-VPC connection = the only way in. No SSH access to the DB host exists.

---

## Key Takeaways

- RDS removes OS/engine patching, backup scripting, and failover engineering from your responsibilities — that's the entire value proposition.
- A DB Subnet Group spanning ≥2 AZs is mandatory, even for Single-AZ deployments.
- Config changes go through Parameter Groups, not `my.cnf` — and some require a reboot to apply.
- RDS instances have no SSH access and no visible OS — logs and metrics come through the console/CloudWatch instead.
- This is the default professional approach; self-hosting MySQL on EC2 (systemd or Docker) is the exception, chosen only for specific constraints RDS can't satisfy.
