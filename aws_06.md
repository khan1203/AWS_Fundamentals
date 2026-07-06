# Deploying MongoDB Using Amazon DocumentDB (Industry Standard Approach)

## Overview

This guide deploys a MongoDB-compatible database as a **managed Amazon DocumentDB cluster** instead of self-hosting MongoDB on EC2 with systemd. DocumentDB implements the MongoDB 4.0/5.0 wire protocol, so existing drivers and `mongosh` work against it with minimal changes — but it is not MongoDB itself; it's a purpose-built AWS storage engine that speaks the MongoDB API. Some MongoDB features (certain aggregation stages, multi-document transactions semantics, specific index types) differ or aren't supported — check the compatibility notes before migrating an existing MongoDB workload wholesale.

**Why this is the industry-standard default (not the EC2+systemd lab from before):**
Self-managing MongoDB on EC2 means you own OS patching, replica set configuration, backup scripting, and failover logic. DocumentDB removes all of that — storage is automatically replicated six ways across three Availability Zones underneath a single cluster, and failover to a replica instance is automatic and typically under 30 seconds. That's the entire value proposition, same as RDS for MySQL.

### Architecture

<img width="1024" height="668" alt="image" src="https://github.com/user-attachments/assets/46602122-3bbf-457d-89ce-b0eefb8ef200" />

A DocumentDB deployment is a **cluster**, not a single instance: one cluster has one primary instance (read/write) and zero or more replica instances (read-only), each of which can live in a different AZ for high availability and read scaling. The underlying storage volume is shared and automatically replicated across three AZs regardless of how many compute instances you add — this is different from RDS's Multi-AZ toggle, which provisions a separate standby.

**Key structural similarity to RDS:** like RDS, DocumentDB has no OS for you to manage, so private subnets hosting it don't need a NAT Gateway unless other private resources in the same subnets need outbound internet.

**Why DocumentDB needs a subnet group spanning multiple AZs:**
Same reason as RDS — the cluster's replica instances and the storage layer's cross-AZ replication both need subnets available in at least two (practically, three) AZs. The subnet group is a hard prerequisite you create before the cluster.

## 1. VPC & Networking

### 1.1 Create VPC
- CIDR: `10.0.0.0/16`
- Region: `ap-southeast-1`

### 1.2 Create Subnets
| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| Public | 10.0.1.0/24 | ap-southeast-1a | Bastion / app tier |
| Private A | 10.0.2.0/24 | ap-southeast-1a | DocumentDB subnet group |
| Private B | 10.0.3.0/24 | ap-southeast-1b | DocumentDB subnet group |
| Private C | 10.0.4.0/24 | ap-southeast-1c | DocumentDB subnet group |

Three private subnets, not two — DocumentDB's storage layer replicates across three AZs by design, so giving it a third AZ to place instances in (rather than just satisfying a minimum) matches how the service actually distributes data.

### 1.3 Internet Gateway
- Create IGW, attach to VPC.

### 1.4 Route Tables
**Public RT**
- `10.0.0.0/16` -> local
- `0.0.0.0/0` -> IGW
- Associate with public subnet.

**Private RT**
- `10.0.0.0/16` -> local
- No `0.0.0.0/0` route needed — DocumentDB doesn't require outbound internet.
- Associate with all three private subnets.

> No NAT Gateway required unless other private EC2 resources in these subnets need outbound access.

### 1.5 DB Subnet Group

Same concept as the RDS subnet group: a named collection of subnets that tells DocumentDB where it's allowed to place cluster instances and storage. Console path differs slightly from RDS since it lives under the DocumentDB service, not the RDS service, even though the underlying object type is identical.

**Step 1 — Confirm your subnets exist and are correctly routed**
Verify Private A, B, and C (from section 1.2) are created and each has only the local route from section 1.4 — no IGW route. Placing the cluster in a subnet with a public route doesn't expose it directly (DocumentDB clusters have no public-access toggle at all — see section 3.2), but it breaks the private-tier design intent.

**Step 2 — Open the subnet group creation screen**
Console: **DocumentDB** → **Subnet groups** (left sidebar) → **Create**.

Note this is a separate console section from RDS's subnet groups, even though both ultimately create the same underlying resource type — a subnet group created in one service does not appear in the other's picker.

**Step 3 — Fill in the group details**
| Field | Value |
|---|---|
| Name | `docdb-subnet-group` |
| Description | e.g. "Private subnets for NoorNote DocumentDB cluster, spans AZ a/b/c" |
| VPC | Your custom VPC from section 1.1 |

**Step 4 — Add subnets**
Select Private A, Private B, and Private C. Unlike the RDS wizard, DocumentDB's subnet picker doesn't require selecting the AZ first — it lists all subnets in the chosen VPC directly. Double check you're picking one subnet per AZ, not two from the same AZ, or you lose the cross-AZ guarantee without any error being raised.

**Step 5 — Create and verify**
```bash
aws docdb describe-db-subnet-groups \
  --db-subnet-group-name docdb-subnet-group \
  --query "DBSubnetGroups[0].Subnets[*].[SubnetIdentifier,SubnetAvailabilityZone.Name]" \
  --output table
```
Confirm three subnets across three distinct AZs. This group is referenced by name in section 3.2.

---

## 2. Security Groups

**Bastion-SG** (public subnet)
- Inbound: SSH (22) from your IP
- Outbound: all

**DocDB-SG** (attached to the cluster, not a subnet)
- Inbound: Custom TCP 27017 from Bastion-SG (or App-tier-SG)
- Outbound: all (default; rarely needs tightening)

---

## 3. Launch Resources

### 3.1 Bastion Host / App Server
- Subnet: public
- Auto-assign public IP: enabled
- SG: Bastion-SG
- Key pair: `bastion-key.pem`

### 3.2 DocumentDB Cluster

This walks through the "Create cluster" wizard end to end. DocumentDB's wizard is structured around the **cluster** as the top-level object, with instances added underneath it — different mental model from RDS's single-instance-first flow.

**Step 1 — Open the wizard**
Console: **DocumentDB** → **Clusters** → **Create**.

**Step 2 — Choose deployment type**
Select **Instance-based cluster** (the standard option; "Elastic clusters" is a separate sharded variant for very large workloads — not needed here).

**Step 3 — Engine options**
| Field | Value |
|---|---|
| Engine version | Latest supported (e.g. 5.0.x) |

**Step 4 — Templates**
Choose **Production** or **Development and testing** — this only pre-fills instance class and instance count defaults below; everything remains editable.

**Step 5 — Settings**
| Field | Value |
|---|---|
| Cluster identifier | `app-docdb-prod` |
| Master username | e.g. `dbadmin` |
| Credentials management | Select **Manage master credentials in AWS Secrets Manager** (recommended) over self-managed, same reasoning as RDS |

**Step 6 — Instance configuration**
| Field | Value |
|---|---|
| Instance class | `db.t3.medium` for dev/test (smallest supported class), `db.r5.large`+ for production workloads |
| Number of instances | 1 for dev (primary only), 2-3 for production (1 primary + 1-2 replicas across the other AZs) |

**Step 7 — Connectivity (network isolation)**
| Field | Value |
|---|---|
| Virtual private cloud (VPC) | Your custom VPC from section 1.1 |
| Subnet group | `docdb-subnet-group` created in section 1.5 |
| VPC security group | Choose existing → `DocDB-SG` (remove any default SG pre-selected) |

Note: there is no "Public access" field to configure here at all — unlike RDS, DocumentDB clusters can never be assigned a public endpoint. Network isolation is enforced structurally, not by a togglable setting you could get wrong.

**Step 8 — Backup**
| Field | Value |
|---|---|
| Backup retention period | 7 days minimum for production (1-35 day range) |
| Backup window | Default, or a known low-traffic window |

**Step 9 — Encryption**
Enable **encryption at rest**, same rule as RDS — must be set at creation time, not retrofittable without a snapshot/restore cycle.

**Step 10 — Maintenance**
Leave **auto minor version upgrade** enabled unless you have a specific compatibility reason to pin the exact minor version.

**Step 11 — Create and wait**
Click **Create cluster**. Status moves `creating` → `available`, typically 10-15 minutes given the multi-instance, multi-AZ storage setup. Wait for `available` before attempting to connect.

**Step 12 — Verify from the CLI**
```bash
aws docdb describe-db-clusters \
  --db-cluster-identifier app-docdb-prod \
  --query "DBClusters[0].[Status,DBSubnetGroup,StorageEncrypted]" \
  --output table
```
Confirm `Status` is `available`, `DBSubnetGroup` matches `docdb-subnet-group`, and `StorageEncrypted` is `true` if you enabled it in step 9 — this last one can't be fixed after the fact, so it's worth catching here rather than in production later.

---

## 4. SSH Access to Bastion (Agent Forwarding)

On local machine:
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>
```

No jump to a private *instance* is needed for the database itself — DocumentDB is accessed directly via its cluster endpoint from the bastion (or app server) over the network, not via SSH.

---

## 5. Connect to DocumentDB

DocumentDB requires TLS by default, which the standard `mongosh` client needs a CA bundle to validate. From the bastion host (or app server) inside the VPC:

```bash
# install mongosh
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb.gpg --dearmor
sudo apt install -y mongodb-mongosh

# download the AWS CA bundle required for TLS validation
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

mongosh --tls --tlsCAFile global-bundle.pem \
  --host <cluster-endpoint>:27017 \
  --username dbadmin --password
```

Get the cluster endpoint from the console (DocumentDB → Clusters → your cluster → Connectivity & security) or via CLI:
```bash
aws docdb describe-db-clusters \
  --db-cluster-identifier app-docdb-prod \
  --query "DBClusters[0].Endpoint" \
  --output text
```

**Always connect via the cluster endpoint, not an instance endpoint**, unless you specifically want to pin reads to one replica — the cluster endpoint always routes writes to the current primary even after a failover, while an individual instance endpoint doesn't follow the primary if it changes.

---

## 6. Configuration Management (Parameter Groups)

Like RDS, DocumentDB replaces direct config file edits with **Cluster Parameter Groups**.

```bash
# Create a custom parameter group (default groups are not editable)
aws docdb create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name custom-docdb5 \
  --db-parameter-group-family docdb5.0 \
  --description "Custom params for app-docdb-prod"

# Modify a parameter
aws docdb modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name custom-docdb5 \
  --parameters "ParameterName=tls,ParameterValue=enabled,ApplyMethod=pending-reboot"

# Attach it to the cluster (some parameters require a reboot to apply)
aws docdb modify-db-cluster \
  --db-cluster-identifier app-docdb-prod \
  --db-cluster-parameter-group-name custom-docdb5 \
  --apply-immediately
```

Parameter groups apply at the **cluster** level in DocumentDB (not per-instance), which is simpler than RDS in this one respect — no risk of instances in a cluster drifting to different parameter values.

---

## 7. Secure & Verify

```bash
mongosh --tls --tlsCAFile global-bundle.pem --host <cluster-endpoint>:27017 \
  --username dbadmin --password --eval "db.adminCommand('ping')"
```

Recommended hardening (handled differently than self-hosted MongoDB):
- Rotate the master password via **Secrets Manager**, not manually.
- Create least-privilege application users with `db.createUser()` — don't use the master user for app connections.
- TLS is enabled by default and cannot be disabled without a parameter group change — leave it on; disabling it removes encryption in transit for no operational benefit.
- Enable **encryption at rest** (must be set at creation time — no retrofit without snapshot/restore).
- Restrict DocDB-SG inbound to only the specific SGs that need access (Bastion-SG, App-tier-SG) — never `0.0.0.0/0`.

Confirm failover works (production clusters with ≥2 instances only):
```bash
aws docdb failover-db-cluster \
  --db-cluster-identifier app-docdb-prod
```

---

## 8. Validation Checklist

- [ ] Bastion reachable via SSH from local machine (public RT + IGW)
- [ ] DocumentDB subnet group spans three AZs
- [ ] DocDB-SG allows 27017 only from Bastion-SG/App-tier-SG, not `0.0.0.0/0`
- [ ] Can connect from bastion using the **cluster endpoint**, with `--tls --tlsCAFile global-bundle.pem`
- [ ] Automated backups enabled with a sensible retention window
- [ ] At least one replica instance provisioned (if production) and failover tested
- [ ] Master credentials stored in Secrets Manager, not hardcoded
- [ ] `StorageEncrypted` confirmed `true` via CLI

---

## 9. Quick Reference

**Network layout**
| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 (ap-southeast-1a) |
| Private subnet A | 10.0.2.0/24 (ap-southeast-1a) |
| Private subnet B | 10.0.3.0/24 (ap-southeast-1b) |
| Private subnet C | 10.0.4.0/24 (ap-southeast-1c) |
| Public RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| Private RT | `10.0.0.0/16 → local` only — no NAT route needed |
| DB Subnet Group | Private A + B + C |

**EC2 / DocumentDB config**
| Item | Bastion Host | DocumentDB Cluster |
|---|---|---|
| Subnet | Public (10.0.1.0/24) | Subnet group (Private A + B + C) |
| Public endpoint | N/A (has public IP) | None available — structurally private |
| Security group | Bastion-SG | DocDB-SG |
| Instance class | t3.micro | db.t3.medium (dev) / db.r5.large+ (prod) |
| Instance count | 1 | 1 (dev) / 2-3 (prod, primary + replicas) |
| Storage | 8 GB gp3 | Auto-scales, no pre-allocation needed |
| Backups | N/A (not applicable) | Automated, 7+ day retention |

**Security groups**
| SG | Direction | Type | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|---|---|
| Bastion-SG | Inbound | SSH | TCP | 22 | Your IP (`x.x.x.x/32`) | Admin SSH access |
| Bastion-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | General outbound + reaching DocDB-SG |
| DocDB-SG | Inbound | Custom TCP | TCP | 27017 | Bastion-SG (or App-tier-SG) | DB access, scoped to trusted sources only |
| DocDB-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Default; rarely needs tightening |

**SSH + connect**
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>

wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
mongosh --tls --tlsCAFile global-bundle.pem --host <cluster-endpoint>:27017 --username dbadmin --password
```

**Fast diagnostics**
```bash
aws docdb describe-db-clusters --db-cluster-identifier app-docdb-prod   # full cluster status
aws docdb describe-db-clusters --db-cluster-identifier app-docdb-prod \
  --query "DBClusters[0].Endpoint" --output text                        # get cluster endpoint
mongosh --tls --tlsCAFile global-bundle.pem --host <endpoint>:27017 \
  --username dbadmin --password --eval "db.adminCommand('ping')"        # connectivity check
```

**Golden rules**
- Always connect via the **cluster endpoint**, not an instance endpoint, unless you deliberately want to pin to a specific replica.
- No NAT Gateway needed for DocumentDB-only private subnets — no OS to patch.
- Parameter groups apply at the cluster level — every instance in the cluster shares the same config, unlike RDS read replicas.
- There is no public-access toggle to misconfigure — DocumentDB clusters are never publicly reachable, by design, not by setting.
- TLS is on by default; connecting without `--tls --tlsCAFile` fails, it doesn't silently downgrade to plaintext.

---

## Key Takeaways

- DocumentDB speaks the MongoDB wire protocol but is a distinct engine underneath — verify feature compatibility (aggregation stages, transaction semantics) before migrating an existing MongoDB workload.
- Storage replicates six ways across three AZs automatically; adding replica instances is about compute/failover, not data durability, which is already handled at the storage layer.
- A three-AZ subnet group lines up with how DocumentDB actually distributes storage, unlike RDS where two AZs is the bare minimum for Multi-AZ.
- Parameter groups are cluster-scoped, removing a class of per-instance config drift that RDS read replicas can have.
- This is the default professional approach for a MongoDB-compatible workload on AWS; self-hosting MongoDB on EC2 (systemd or Docker) is the exception, chosen only when DocumentDB's compatibility gaps rule it out.
