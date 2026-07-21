# Multi-VPC Deployment with Centralized Internet Egress (Pulumi + AWS Transit Gateway)

## 1. Overview

Same architecture as before, now built with **Pulumi (Python)** instead of console clicks — everything below is code you commit, not screenshots.

| VPC | CIDR | Role |
|---|---|---|
| Egress VPC | `172.16.0.0/16` | Public subnet (IGW, NAT GW, Bastion) + Private subnet (TGW attachment) |
| App VPC | `10.1.0.0/16` | Node.js API server, private subnet, TGW attachment |
| Database VPC | `10.2.0.0/16` | PostgreSQL, private subnet, TGW attachment |

**Key design difference from the console version:** this Pulumi code uses `default_route_table_association="disable"` / `default_route_table_propagation="disable"` on the TGW — meaning **you** create and wire 3 separate TGW route tables by hand instead of relying on TGW's single default table. This is the production-grade pattern (per-attachment routing control) and is what you'll build step by step.

**Traffic flow, App → internet:** 
<img width="1146" height="1342" alt="image" src="https://github.com/user-attachments/assets/6748f5bb-6153-4966-886d-81c4e27b9c13" />

---

## 2. Prerequisites

```bash
# Pulumi CLI
curl -fsSL https://get.pulumi.com | sh

# uv (fast Python package/venv manager, replaces pip + venv)
curl -LsSf https://astral.sh/uv/install.sh | sh

# AWS credentials must already be configured (aws configure / env vars)
# An EC2 key pair named tgw-lab-key must already exist in ap-southeast-1
```

```bash
mkdir tgw-lab-egress && cd tgw-lab-egress
pulumi new aws-python   # scaffolds the project, then replace __main__.py content phase by phase below

# uv creates and manages the venv for you — no separate `python -m venv` step
uv venv
source .venv/bin/activate
uv pip install pulumi pulumi_aws
```

From here on, run `uv pip install <pkg>` instead of `pip install <pkg>` for anything else you add later.

---

## 3. Phase 1 — Config and VPCs

```python
import pulumi
import pulumi_aws as aws

config = pulumi.Config()
project_name = "tgw-lab-lab"
environment = "dev"
key_name = "tgw-lab-key"
region = "ap-southeast-1"
availability_zone = "ap-southeast-1a"
ubuntu_ami = "ami-01811d4912b4ccb26"  # Ubuntu 22.04 LTS in ap-southeast-1

common_tags = {"Project": project_name, "Environment": environment, "ManagedBy": "Pulumi"}

egress_vpc = aws.ec2.Vpc(f"{project_name}-egress-vpc",
    cidr_block="172.16.0.0/16", enable_dns_support=True, enable_dns_hostnames=True,
    tags={**common_tags, "Name": f"{project_name}-egress-vpc"})

app_vpc = aws.ec2.Vpc(f"{project_name}-app-vpc",
    cidr_block="10.1.0.0/16", enable_dns_support=True, enable_dns_hostnames=True,
    tags={**common_tags, "Name": f"{project_name}-app-vpc"})

database_vpc = aws.ec2.Vpc(f"{project_name}-database-vpc",
    cidr_block="10.2.0.0/16", enable_dns_support=True, enable_dns_hostnames=True,
    tags={**common_tags, "Name": f"{project_name}-database-vpc"})
```

`pulumi up` here creates just the 3 VPCs — run it now to catch credential/region issues early before the stack gets bigger.

---

## 4. Phase 2 — Internet Gateway + Subnets

```python
internet_gateway = aws.ec2.InternetGateway(f"{project_name}-igw",
    vpc_id=egress_vpc.id, tags={**common_tags, "Name": f"{project_name}-egress-igw"})

# Egress VPC subnets
egress_public_subnet = aws.ec2.Subnet(f"{project_name}-egress-public-subnet",
    vpc_id=egress_vpc.id, cidr_block="172.16.1.0/24", availability_zone=availability_zone,
    map_public_ip_on_launch=True,
    tags={**common_tags, "Name": f"{project_name}-egress-public-subnet", "Type": "Public"})

egress_private_subnet = aws.ec2.Subnet(f"{project_name}-egress-private-subnet",
    vpc_id=egress_vpc.id, cidr_block="172.16.2.0/24", availability_zone=availability_zone,
    map_public_ip_on_launch=False,
    tags={**common_tags, "Name": f"{project_name}-egress-private-subnet", "Type": "Private"})

# App VPC subnet — holds both the app instance AND the TGW attachment
app_subnet = aws.ec2.Subnet(f"{project_name}-app-subnet",
    vpc_id=app_vpc.id, cidr_block="10.1.1.0/24", availability_zone=availability_zone,
    map_public_ip_on_launch=False,
    tags={**common_tags, "Name": f"{project_name}-app-subnet", "Type": "Private"})

# Database VPC subnet
database_subnet = aws.ec2.Subnet(f"{project_name}-database-subnet",
    vpc_id=database_vpc.id, cidr_block="10.2.1.0/24", availability_zone=availability_zone,
    map_public_ip_on_launch=False,
    tags={**common_tags, "Name": f"{project_name}-database-subnet", "Type": "Private"})
```

Note this lab uses **one subnet per App/DB VPC** (single-AZ) — simpler than the two-AZ console version. `map_public_ip_on_launch=True` on the egress public subnet is what auto-assigns the bastion's public IP later; you don't set it on the instance itself.

---

## 5. Phase 3 — NAT Gateway (Egress VPC only)

```python
nat_eip = aws.ec2.Eip(f"{project_name}-nat-eip", domain="vpc",
    tags={**common_tags, "Name": f"{project_name}-nat-eip"})

nat_gateway = aws.ec2.NatGateway(f"{project_name}-nat-gateway",
    allocation_id=nat_eip.id, subnet_id=egress_public_subnet.id,
    tags={**common_tags, "Name": f"{project_name}-nat-gateway"})
```

`subnet_id` must be the **public** subnet — a NAT Gateway needs its own path to the IGW. App VPC and Database VPC get no NAT Gateway of their own.

---

## 6. Phase 4 — Transit Gateway (custom route table mode)

```python
transit_gateway = aws.ec2transitgateway.TransitGateway(f"{project_name}-tgw",
    amazon_side_asn=64512,
    auto_accept_shared_attachments="enable",
    default_route_table_association="disable",   # we wire route tables manually
    default_route_table_propagation="disable",
    description="Transit Gateway for Docker Lab",
    tags={**common_tags, "Name": f"{project_name}-tgw"})
```

Disabling the default association/propagation means the TGW starts with **no** route tables auto-attached — every attachment must be explicitly associated with a route table you create (Phase 6), and every propagation must be explicit (Phase 7). This is more typing but gives per-VPC routing isolation instead of one shared table.

---

## 7. Phase 5 — TGW attachments (one ENI-set per VPC)

```python
egress_tgw_attachment = aws.ec2transitgateway.VpcAttachment(f"{project_name}-egress-tgw-attachment",
    subnet_ids=[egress_private_subnet.id], transit_gateway_id=transit_gateway.id, vpc_id=egress_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-egress-tgw-attachment"})

app_tgw_attachment = aws.ec2transitgateway.VpcAttachment(f"{project_name}-app-tgw-attachment",
    subnet_ids=[app_subnet.id], transit_gateway_id=transit_gateway.id, vpc_id=app_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-app-tgw-attachment"})

database_tgw_attachment = aws.ec2transitgateway.VpcAttachment(f"{project_name}-database-tgw-attachment",
    subnet_ids=[database_subnet.id], transit_gateway_id=transit_gateway.id, vpc_id=database_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-database-tgw-attachment"})
```

`subnet_ids` for the egress attachment is the **private** subnet — never expose the TGW ENI in the public one.

---

## 8. Phase 6 — TGW route tables + associations

Three separate tables, one per attachment (this is the part the console version skips by leaving the default table enabled):

```python
egress_route_table = aws.ec2transitgateway.RouteTable(f"{project_name}-egress-route-table",
    transit_gateway_id=transit_gateway.id, tags={**common_tags, "Name": f"{project_name}-egress-route-table"})

app_route_table = aws.ec2transitgateway.RouteTable(f"{project_name}-app-route-table",
    transit_gateway_id=transit_gateway.id, tags={**common_tags, "Name": f"{project_name}-app-route-table"})

database_route_table = aws.ec2transitgateway.RouteTable(f"{project_name}-database-route-table",
    transit_gateway_id=transit_gateway.id, tags={**common_tags, "Name": f"{project_name}-database-route-table"})

aws.ec2transitgateway.RouteTableAssociation(f"{project_name}-egress-route-association",
    transit_gateway_attachment_id=egress_tgw_attachment.id, transit_gateway_route_table_id=egress_route_table.id)

aws.ec2transitgateway.RouteTableAssociation(f"{project_name}-app-route-association",
    transit_gateway_attachment_id=app_tgw_attachment.id, transit_gateway_route_table_id=app_route_table.id)

aws.ec2transitgateway.RouteTableAssociation(f"{project_name}-database-route-association",
    transit_gateway_attachment_id=database_tgw_attachment.id, transit_gateway_route_table_id=database_route_table.id)
```

Each attachment is associated with **its own** route table — this table controls what that attachment can send *into* the TGW.

---

## 9. Phase 7 — TGW routes + propagations (the actual egress logic)

This is the core of the pattern — read it carefully:

```python
# App -> Database, direct (both are "spoke" VPCs, can talk to each other)
aws.ec2transitgateway.Route(f"{project_name}-app-to-database-route",
    destination_cidr_block="10.2.0.0/16",
    transit_gateway_attachment_id=database_tgw_attachment.id,
    transit_gateway_route_table_id=app_route_table.id)

# App -> everywhere else (i.e. internet) goes to the Egress attachment
aws.ec2transitgateway.Route(f"{project_name}-app-internet-route",
    destination_cidr_block="0.0.0.0/0",
    transit_gateway_attachment_id=egress_tgw_attachment.id,
    transit_gateway_route_table_id=app_route_table.id)

# Database -> App, direct
aws.ec2transitgateway.Route(f"{project_name}-database-to-app-route",
    destination_cidr_block="10.1.0.0/16",
    transit_gateway_attachment_id=app_tgw_attachment.id,
    transit_gateway_route_table_id=database_route_table.id)

# TEMPORARY: Database -> internet via Egress (needed once, to docker pull postgres image)
database_internet_route = aws.ec2transitgateway.Route(f"{project_name}-database-internet-route",
    destination_cidr_block="0.0.0.0/0",
    transit_gateway_attachment_id=egress_tgw_attachment.id,
    transit_gateway_route_table_id=database_route_table.id)

# So the Egress side knows how to reach App/DB VPCs for return traffic
aws.ec2transitgateway.RouteTablePropagation(f"{project_name}-app-propagation",
    transit_gateway_attachment_id=app_tgw_attachment.id, transit_gateway_route_table_id=egress_route_table.id)

aws.ec2transitgateway.RouteTablePropagation(f"{project_name}-database-propagation",
    transit_gateway_attachment_id=database_tgw_attachment.id, transit_gateway_route_table_id=egress_route_table.id)
```

**Read this twice — it's the whole lab in 4 blocks:**
- `app_route_table` and `database_route_table` each get a static `0.0.0.0/0 → egress_tgw_attachment` route — that's what makes internet-bound traffic from either spoke VPC land in the Egress VPC.
- `egress_route_table` gets no static routes at all here — it only gets **propagations** from App and Database, meaning it *learns* `10.1.0.0/16` and `10.2.0.0/16` automatically from those attachments, so return traffic can find its way back.
- The database's `0.0.0.0/0` route is flagged `TEMPORARY` in the code because it exists only so the DB instance's user-data script can `docker pull postgres:13-alpine` during first boot — see the security note in Key Takeaways for what to do with it afterward.

---

## 10. Phase 8 — VPC route tables

```python
egress_public_route_table = aws.ec2.RouteTable(f"{project_name}-egress-public-rt", vpc_id=egress_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-egress-public-rt"})
egress_private_route_table = aws.ec2.RouteTable(f"{project_name}-egress-private-rt", vpc_id=egress_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-egress-private-rt"})
app_route_table_vpc = aws.ec2.RouteTable(f"{project_name}-app-rt", vpc_id=app_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-app-rt"})
database_route_table_vpc = aws.ec2.RouteTable(f"{project_name}-database-rt", vpc_id=database_vpc.id,
    tags={**common_tags, "Name": f"{project_name}-database-rt"})

# Egress public subnet: internet directly, plus TGW for reaching App/DB (bastion needs this)
aws.ec2.Route(f"{project_name}-egress-public-internet-route", route_table_id=egress_public_route_table.id,
    destination_cidr_block="0.0.0.0/0", gateway_id=internet_gateway.id)
aws.ec2.Route(f"{project_name}-egress-public-to-app-route", route_table_id=egress_public_route_table.id,
    destination_cidr_block="10.1.0.0/16", transit_gateway_id=transit_gateway.id)
aws.ec2.Route(f"{project_name}-egress-public-to-database-route", route_table_id=egress_public_route_table.id,
    destination_cidr_block="10.2.0.0/16", transit_gateway_id=transit_gateway.id)

# Egress private subnet (TGW attachment lives here): only needs NAT for outbound
aws.ec2.Route(f"{project_name}-egress-private-nat-route", route_table_id=egress_private_route_table.id,
    destination_cidr_block="0.0.0.0/0", nat_gateway_id=nat_gateway.id)

# App VPC: single 0.0.0.0/0 -> TGW covers BOTH internet and DB VPC traffic (DB CIDR isn't "local", so it matches this too)
aws.ec2.Route(f"{project_name}-app-internet-route", route_table_id=app_route_table_vpc.id,
    destination_cidr_block="0.0.0.0/0", transit_gateway_id=transit_gateway.id)

# Database VPC: same pattern
aws.ec2.Route(f"{project_name}-database-internet-route", route_table_id=database_route_table_vpc.id,
    destination_cidr_block="0.0.0.0/0", transit_gateway_id=transit_gateway.id)

aws.ec2.RouteTableAssociation(f"{project_name}-egress-public-rt-association",
    subnet_id=egress_public_subnet.id, route_table_id=egress_public_route_table.id)
aws.ec2.RouteTableAssociation(f"{project_name}-egress-private-rt-association",
    subnet_id=egress_private_subnet.id, route_table_id=egress_private_route_table.id)
aws.ec2.RouteTableAssociation(f"{project_name}-app-rt-association",
    subnet_id=app_subnet.id, route_table_id=app_route_table_vpc.id)
aws.ec2.RouteTableAssociation(f"{project_name}-database-rt-association",
    subnet_id=database_subnet.id, route_table_id=database_route_table_vpc.id)
```

Notice the public subnet also gets direct TGW routes to App/DB CIDRs — that's specifically so the **bastion** (sitting in that public subnet) can SSH straight to App/DB instances without hairpinning through the private subnet's route table.

---

## 11. Phase 9 — Security groups

```python
bastion_security_group = aws.ec2.SecurityGroup(f"{project_name}-bastion-sg", vpc_id=egress_vpc.id,
    description="Security group for bastion host",
    ingress=[aws.ec2.SecurityGroupIngressArgs(protocol="tcp", from_port=22, to_port=22,
        cidr_blocks=["0.0.0.0/0"], description="SSH access from internet")],
    egress=[aws.ec2.SecurityGroupEgressArgs(protocol="-1", from_port=0, to_port=0,
        cidr_blocks=["0.0.0.0/0"], description="All outbound traffic")],
    tags={**common_tags, "Name": f"{project_name}-bastion-sg"})

app_security_group = aws.ec2.SecurityGroup(f"{project_name}-app-sg", vpc_id=app_vpc.id,
    description="Security group for application server",
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(protocol="tcp", from_port=22, to_port=22,
            cidr_blocks=["172.16.0.0/16"], description="SSH from bastion host"),
        aws.ec2.SecurityGroupIngressArgs(protocol="tcp", from_port=3000, to_port=3000,
            cidr_blocks=["172.16.0.0/16"], description="API access from bastion host")],
    egress=[aws.ec2.SecurityGroupEgressArgs(protocol="-1", from_port=0, to_port=0,
        cidr_blocks=["0.0.0.0/0"], description="All outbound traffic")],
    tags={**common_tags, "Name": f"{project_name}-app-sg"})

database_security_group = aws.ec2.SecurityGroup(f"{project_name}-database-sg", vpc_id=database_vpc.id,
    description="Security group for database server",
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(protocol="tcp", from_port=22, to_port=22,
            cidr_blocks=["172.16.0.0/16"], description="SSH from bastion host"),
        aws.ec2.SecurityGroupIngressArgs(protocol="tcp", from_port=5432, to_port=5432,
            cidr_blocks=["10.1.0.0/16"], description="PostgreSQL from app server")],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(protocol="tcp", from_port=0, to_port=65535,
            cidr_blocks=["10.0.0.0/8"], description="Internal network only"),
        aws.ec2.SecurityGroupEgressArgs(protocol="tcp", from_port=80, to_port=80,
            cidr_blocks=["0.0.0.0/0"], description="HTTP for package updates (temporary)"),
        aws.ec2.SecurityGroupEgressArgs(protocol="tcp", from_port=443, to_port=443,
            cidr_blocks=["0.0.0.0/0"], description="HTTPS for package updates (temporary)")],
    tags={**common_tags, "Name": f"{project_name}-database-sg"})
```

The DB security group is the odd one out — instead of default allow-all egress, it explicitly whitelists ports 80/443 out (for the one-time `docker pull`) plus internal traffic on `10.0.0.0/8`. This SG-level restriction is your **real** security boundary; the TGW `0.0.0.0/0` route in Phase 7 only controls where traffic *can* go, this SG controls what's actually *allowed* out.

---

## 12. Phase 10 — EC2 instances

### 12.1 Bastion — unchanged

```python
bastion_user_data = """#!/bin/bash
apt-get update
apt-get install -y htop curl awscli jq
"""
```

### 12.2 Database instance — Postgres container + seeded `users` table

The container runs Postgres itself; a mounted init script creates and seeds the table on first startup (Postgres only ever runs `/docker-entrypoint-initdb.d/*.sql` once, against an empty data directory — that's the built-in "self-provisioning" hook, same principle as the `CREATE TABLE IF NOT EXISTS` pattern from the earlier Node/MySQL lab, just done DB-side here instead of app-side).

```python
database_user_data = """#!/bin/bash
apt-get update
apt-get install -y docker.io curl
systemctl start docker
systemctl enable docker
usermod -aG docker ubuntu

mkdir -p /opt/postgres-init
cat > /opt/postgres-init/init.sql << 'PGINIT'
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO users (name, email) VALUES
    ('Alice Rahman', 'alice@example.com'),
    ('Bilal Hasan', 'bilal@example.com'),
    ('Chandni Islam', 'chandni@example.com')
ON CONFLICT (email) DO NOTHING;
PGINIT

docker run -d \\
  --name postgres-db \\
  --restart unless-stopped \\
  -e POSTGRES_DB=appdb \\
  -e POSTGRES_USER=appuser \\
  -e POSTGRES_PASSWORD=ChangeMe123! \\
  -v /opt/postgres-init/init.sql:/docker-entrypoint-initdb.d/init.sql \\
  -p 5432:5432 \\
  postgres:13-alpine

touch /tmp/docker-setup-complete
"""
```

`--restart unless-stopped` gives you the EC2-level equivalent of `systemctl enable` — the container comes back up automatically after an instance reboot without you having to re-run `docker run`. Hardcoding `POSTGRES_PASSWORD` here is lab-only; for anything beyond this exercise, move it into a `pulumi config set --secret db_password` value and read it with `config.require_secret("db_password")` instead.

### 12.3 App instance — Node.js API, `/health` and `/users`, running continuously via systemd

```python
app_user_data = """#!/bin/bash
apt-get update
apt-get install -y curl gnupg

# Node.js 20.x LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

mkdir -p /opt/app
cat > /opt/app/package.json << 'PKGJSON'
{
  "name": "tgw-lab-app",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.19.2",
    "pg": "^8.11.5"
  }
}
PKGJSON

cat > /opt/app/server.js << 'SERVERJS'
const express = require('express');
const { Pool } = require('pg');

const app = express();
const port = process.env.PORT || 3000;

const pool = new Pool({
  host: process.env.DB_HOST || '10.2.1.100',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'appdb',
  user: process.env.DB_USER || 'appuser',
  password: process.env.DB_PASSWORD || 'ChangeMe123!',
});

// Health check — confirms the process is up AND the DB is reachable over the TGW path
app.get('/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.status(200).json({ status: 'ok', db: 'connected' });
  } catch (err) {
    res.status(503).json({ status: 'error', db: 'unreachable', error: err.message });
  }
});

// Get all users
app.get('/users', async (req, res) => {
  try {
    const result = await pool.query('SELECT id, name, email, created_at FROM users ORDER BY id');
    res.status(200).json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App listening on port ${port}`);
});
SERVERJS

cd /opt/app && npm install --production

cat > /etc/systemd/system/tgw-lab-app.service << 'SVCFILE'
[Unit]
Description=TGW Lab Node.js API
After=network.target

[Service]
WorkingDirectory=/opt/app
ExecStart=/usr/bin/node /opt/app/server.js
Restart=always
User=ubuntu
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
SVCFILE

systemctl daemon-reload
systemctl enable tgw-lab-app
systemctl start tgw-lab-app
"""
```

`Restart=always` + `systemctl enable` is what makes this "continuously listen" — the API comes back after both process crashes and instance reboots, without needing a manual SSH session to restart it. `DB_HOST` defaults to the DB instance's static private IP (`10.1.1.100` → `10.2.1.100` path, i.e. reachable purely because App VPC's `0.0.0.0/0 → TGW` route matches the DB VPC CIDR — see Phase 8).

### 12.4 Instance resources — unchanged

```python
bastion_host = aws.ec2.Instance(f"{project_name}-bastion-host",
    ami=ubuntu_ami, instance_type="t2.micro", key_name=key_name,
    subnet_id=egress_public_subnet.id, vpc_security_group_ids=[bastion_security_group.id],
    user_data=bastion_user_data, tags={**common_tags, "Name": f"{project_name}-bastion-host"})

app_server = aws.ec2.Instance(f"{project_name}-app-server",
    ami=ubuntu_ami, instance_type="t2.micro", key_name=key_name,
    subnet_id=app_subnet.id, private_ip="10.1.1.100",
    vpc_security_group_ids=[app_security_group.id], user_data=app_user_data,
    tags={**common_tags, "Name": f"{project_name}-app-server"})

database_server = aws.ec2.Instance(f"{project_name}-database-server",
    ami=ubuntu_ami, instance_type="t2.micro", key_name=key_name,
    subnet_id=database_subnet.id, private_ip="10.2.1.100",
    vpc_security_group_ids=[database_security_group.id], user_data=database_user_data,
    tags={**common_tags, "Name": f"{project_name}-database-server"})
```

---

## 13. Phase 11 — Outputs

```python
pulumi.export("bastion_public_ip", bastion_host.public_ip)
pulumi.export("bastion_private_ip", bastion_host.private_ip)
pulumi.export("app_private_ip", app_server.private_ip)
pulumi.export("database_private_ip", database_server.private_ip)
pulumi.export("transit_gateway_id", transit_gateway.id)
pulumi.export("nat_gateway_id", nat_gateway.id)
pulumi.export("database_route_table_id", database_route_table.id)
pulumi.export("egress_vpc_id", egress_vpc.id)
pulumi.export("app_vpc_id", app_vpc.id)
pulumi.export("database_vpc_id", database_vpc.id)
pulumi.export("ssh_command", pulumi.Output.format("ssh -i ~/.ssh/{0}.pem ubuntu@{1}", key_name, bastion_host.public_ip))
pulumi.export("tunnel_command", pulumi.Output.format("ssh -i ~/.ssh/{0}.pem -L 8080:10.1.1.100:3000 ubuntu@{1}", key_name, bastion_host.public_ip))
```

---

## 14. Deploy and verify

```bash
pulumi up          # review the plan, type "yes"
pulumi stack output ssh_command
pulumi stack output tunnel_command
```

SSH to the bastion, then hop to the app server via its private IP (agent forwarding, same as the console lab):
```bash
ssh-add ~/.ssh/tgw-lab-key.pem
ssh -A ubuntu@$(pulumi stack output bastion_public_ip)
ssh ubuntu@10.1.1.100
```

Verify centralized egress from inside the App instance:
```bash
curl -m 5 https://checkip.amazonaws.com
```
Should return the `nat_eip` address, not an App-VPC-local IP.

On the Database instance, confirm Postgres is up and the table was seeded:
```bash
ls /tmp/docker-setup-complete
docker ps                                                    # postgres-db should be Up
docker exec -it postgres-db psql -U appuser -d appdb -c "SELECT * FROM users;"
```

On the App instance, confirm the API is running and can reach the DB over the TGW path:
```bash
systemctl status tgw-lab-app --no-pager
curl -s http://localhost:3000/health
curl -s http://localhost:3000/users
```

From the bastion (no need to hop further — `app-sg` allows port 3000 from the whole Egress VPC CIDR):
```bash
curl -s http://10.1.1.100:3000/health
curl -s http://10.1.1.100:3000/users
```

Or from your local machine using the exported tunnel, then hit `localhost:8080` instead:
```bash
$(pulumi stack output tunnel_command) &
curl -s http://localhost:8080/health
curl -s http://localhost:8080/users
```

---

## 15. Validation checklist

- [ ] `pulumi up` completes with no errors, all resources green
- [ ] 3 VPCs, non-overlapping CIDRs (`172.16.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16`)
- [ ] 3 TGW route tables exist, each associated with exactly one attachment
- [ ] `app_route_table` and `database_route_table` each have `0.0.0.0/0 → egress attachment`
- [ ] `egress_route_table` shows propagated `10.1.0.0/16` and `10.2.0.0/16` routes (Console: TGW → Route Tables → egress table → Propagations tab)
- [ ] Bastion → App instance SSH hop succeeds (private IP, agent forwarding)
- [ ] `curl checkip.amazonaws.com` from App instance returns the NAT EIP
- [ ] `/tmp/docker-setup-complete` exists on DB instance and `postgres-db` container shows `Up` in `docker ps`
- [ ] `SELECT * FROM users;` on the DB instance returns the 3 seeded rows
- [ ] `systemctl status tgw-lab-app` on the App instance shows `active (running)`
- [ ] `GET /health` returns `{"status":"ok","db":"connected"}`
- [ ] `GET /users` returns the same 3 users as JSON, proving the App VPC → TGW → DB VPC path works end-to-end for real application traffic (not just SSH/ping)

---

## 16. Quick reference

| Pulumi resource | Type | Purpose |
|---|---|---|
| `egress_vpc`, `app_vpc`, `database_vpc` | `aws.ec2.Vpc` | 3 isolated VPCs |
| `transit_gateway` | `aws.ec2transitgateway.TransitGateway` | Hub, custom route table mode |
| `*_tgw_attachment` (x3) | `aws.ec2transitgateway.VpcAttachment` | One per VPC |
| `*_route_table` (x3, TGW-side) | `aws.ec2transitgateway.RouteTable` | Per-attachment routing control |
| `*_route_table_vpc` (x4, VPC-side) | `aws.ec2.RouteTable` | Standard VPC subnet routing |
| `nat_gateway` | `aws.ec2.NatGateway` | Sole internet egress point |
| `bastion_host`, `app_server`, `database_server` | `aws.ec2.Instance` | The 3 EC2 instances |

---

## 17. Key takeaways

- Disabling TGW default association/propagation trades convenience for control: 3 explicit route tables instead of 1 shared one, but each spoke VPC's blast radius through the TGW is now scoped precisely by what you propagate/route, not by a shared default.
- The `0.0.0.0/0 → TGW` route in App/DB VPC route tables does double duty: it's simultaneously "how do I reach the internet" and "how do I reach my sibling VPC," because the sibling's CIDR isn't `local` so it also falls under the `0.0.0.0/0` catch-all — the TGW route table (Phase 7) is what actually splits that traffic apart afterward.
- The DB instance's `0.0.0.0/0` TGW route and its 80/443 SG egress rules are explicitly commented `TEMPORARY` in the source — they exist only to let `docker pull` succeed on first boot. Production follow-up: remove `database-internet-route` (Phase 7) and the 80/443 SG egress rules (Phase 9) once the image is baked into an AMI, or replace the pull with a VPC endpoint / private ECR mirror so the DB VPC never needs a path to the public internet at all.
- SG egress rules are the real enforcement layer here — the DB SG's explicit port whitelist (vs. the App/Bastion SGs' default allow-all) is what actually stops the DB instance from reaching arbitrary internet hosts, independent of what the route tables technically permit.
- Static private IPs (`private_ip="10.1.1.100"` / `"10.2.1.100"`) are set on the App/DB instances specifically so the `tunnel_command` output and SSH hop instructions don't need to be looked up after every `pulumi up`.
- `POSTGRES_PASSWORD` / `DB_PASSWORD` are plain-text in this guide for lab speed only — the values live in `user_data`, which is visible to anyone with `ec2:DescribeInstanceAttribute` on the instance. Before this touches anything real, move it to a Pulumi secret and inject it as an environment variable instead.
- `uv` replaces both `python -m venv` and `pip` — `uv venv` creates the environment, `uv pip install` resolves and installs into it, same commands elsewhere in this repo should switch the same way.
