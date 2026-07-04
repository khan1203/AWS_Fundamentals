# Deploying MongoDB on EC2 Using Systemd

## Overview

This guide deploys MongoDB on a private EC2 instance, accessed only through a public bastion host, inside a custom VPC. MongoDB is managed via `systemd` (`mongod.service`) so it auto-starts on boot and restarts on failure.

**Architecture**

```
Internet
   |
  IGW
   |
  VPC 10.0.0.0/16 (ap-southeast-1)
   |
   +-- Public Subnet 10.0.1.0/24 -- Public RT (0.0.0.0/0 -> IGW)
   |       -> Bastion Host (public IP) + NAT Gateway
   |
   +-- Private Subnet 10.0.2.0/24 -- Private RT (0.0.0.0/0 -> NAT)
           -> MongoDB Instance (no public IP)
```

Here's how traffic moves through this setup:

**Internet-bound traffic** (public subnet only)
Internet → IGW → Public Route Table (`0.0.0.0/0 → IGW`) → public subnet. This is the only path in or out of the VPC. Both the bastion host and the NAT gateway live in the public subnet and rely on this path — the bastion for inbound SSH, the NAT gateway for relaying outbound traffic from the private subnet.

**Private instance's outbound traffic**
MongoDB instance → Private Route Table (`0.0.0.0/0 → NAT gateway`) → NAT gateway → IGW → Internet. The private subnet has no direct IGW route, so this is its *only* way out (e.g. for OS/package updates). Nothing initiated from the internet can reach in this way — NAT is outbound-only by nature.

**Bastion-to-MongoDB traffic (the SSH hop)**
Bastion → MongoDB instance uses the **local route** (`10.0.0.0/16 → local`), which every route table in the VPC has automatically. Since both subnets sit inside the same VPC CIDR, traffic between them never leaves the VPC and never touches the IGW or NAT gateway at all. This is why:
- SSH from bastion to the private instance keeps working even if the NAT gateway is deleted or blackholed.
- Access control here comes entirely from **security groups** (MongoDB-SG allowing SSH/27017 from Bastion-SG), not from routing.

**The key distinction:** IGW and NAT handle traffic crossing the VPC boundary (to/from the internet). The local route handles traffic staying inside the VPC boundary (subnet-to-subnet). They're independent mechanisms — one can fail without affecting the other.

---

## 1. VPC & Networking

### 1.1 Create VPC
- CIDR: `10.0.0.0/16`
- Region: `ap-southeast-1`

### 1.2 Create Subnets
| Subnet | CIDR | AZ |
|---|---|---|
| Public | 10.0.1.0/24 | ap-southeast-1a |
| Private | 10.0.2.0/24 | ap-southeast-1a |

### 1.3 Internet Gateway
- Create IGW, attach to VPC.

### 1.4 NAT Gateway
- Allocate Elastic IP.
- Create NAT Gateway **in the public subnet**, associate the EIP.

### 1.5 Route Tables
**Public RT**
- `10.0.0.0/16` -> local
- `0.0.0.0/0` -> IGW
- Associate with public subnet.

**Private RT**
- `10.0.0.0/16` -> local
- `0.0.0.0/0` -> NAT Gateway
- Associate with private subnet.

---

## 2. Security Groups

**Bastion-SG** (public subnet)
- Inbound: SSH (22) from your IP
- Outbound: all

**MongoDB-SG** (private subnet)
- Inbound: SSH (22) from Bastion-SG
- Inbound: MongoDB (27017) from Bastion-SG (or app-tier SG)
- Outbound: all

---

## 3. Launch Instances

### 3.1 Bastion Host
- Subnet: public
- Auto-assign public IP: enabled
- SG: Bastion-SG
- Key pair: `bastion-key.pem`

### 3.2 MongoDB Instance
- Subnet: private
- Auto-assign public IP: disabled
- SG: MongoDB-SG
- Key pair: same or different — access via agent forwarding, no `.pem` copied to bastion

---

## 4. SSH Access (Agent Forwarding)

On local machine:
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>
```

From inside bastion, jump to private instance (key never stored on bastion):
```bash
ssh ec2-user@10.0.2.x
```

---

## 5. Install MongoDB

MongoDB isn't in the default OS repos — you need to add MongoDB's own package repository first.

**Ubuntu:**
```bash
sudo apt update -y
sudo apt install -y gnupg curl

curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt update -y
sudo apt install -y mongodb-org
```

**Amazon Linux 2023:**
```bash
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo <<'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
EOF

sudo yum install -y mongodb-org
```

---

## 6. Configure Systemd Service

The `mongodb-org` package ships a ready `mongod.service` unit. Verify and manage it directly:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mongod      # start on boot
sudo systemctl start mongod
sudo systemctl status mongod
```

If building a custom unit file (e.g. non-package/manual binary install):
```bash
sudo nano /etc/systemd/system/mongod.service
```

```ini
# /etc/systemd/system/mongod.service
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongod --config /etc/mongod.conf
PIDFile=/var/run/mongodb/mongod.pid
Type=simple
Restart=on-failure
TimeoutSec=300
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mongod
```

**Important config change** — by default `/etc/mongod.conf` binds only to localhost:

```yaml
# /etc/mongod.conf
net:
  port: 27017
  bindIp: 127.0.0.1,10.0.2.x     # add the instance's private IP so the bastion can reach it
```

Restart after editing:
```bash
sudo systemctl restart mongod
```

---

## 7. Secure & Verify MongoDB

Create an admin user (do this *before* enabling auth, while still on localhost):
```bash
mongosh
```
```javascript
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase"]
})
exit
```

Enable authentication:
```yaml
# /etc/mongod.conf
security:
  authorization: enabled
```

```bash
sudo systemctl restart mongod
mongosh -u admin -p --authenticationDatabase admin
```

Confirm auto-start:
```bash
sudo reboot

# after reboot, from bastion:
ssh ec2-user@10.0.2.x "systemctl is-active mongod"
```

---

## 8. Validation Checklist

- [ ] Bastion reachable via SSH from local machine (public RT + IGW)
- [ ] Private instance reachable from bastion only (local route, MongoDB-SG allows Bastion-SG)
- [ ] Private instance has outbound internet via NAT (test: `curl -I https://amazon.com`)
- [ ] `systemctl status mongod` shows `active (running)` and `enabled`
- [ ] `bindIp` in `mongod.conf` includes the instance's private IP, not just `127.0.0.1`
- [ ] Authentication enabled and admin user can connect with credentials
- [ ] MongoDB survives reboot without manual restart

---

## 9. Concept Clarification

### 9.1 Why is NAT Gateway inside a subnet, but IGW isn't?

These two objects sit at fundamentally different layers of the network:

**IGW is a VPC-level attachment, not a networked device.**
- It's attached once, directly to the VPC (`aws ec2 attach-internet-gateway`) — it doesn't live in any subnet and has no private IP, no ENI, no compute footprint.
- It performs **1:1 NAT**: it maps each instance's Elastic/public IP directly to its private IP at the edge of the VPC. There's no shared resource to route traffic *through* — it's a translation rule referenced by route tables, not a hop.
- Because it isn't "in" a subnet, it doesn't consume subnet IP space and isn't AZ-scoped — it's automatically available across all AZs in the VPC.

**NAT Gateway is a managed, subnet-resident resource.**
- It's provisioned like a mini-instance: it gets an **ENI with a private IP from the subnet's CIDR**, and needs an Elastic IP attached to reach the internet itself.
- It performs **many-to-one NAT (PAT)**: many private instances share the NAT Gateway's single public IP. To do that translation, traffic has to physically arrive *at* the NAT Gateway's ENI — which means it must live in a real subnet with a real route to the internet (the public subnet's `0.0.0.0/0 → IGW` route).
- It's **AZ-scoped** — a NAT Gateway in `ap-southeast-1a` only serves traffic routed to it from route tables in that AZ (or wherever your private subnets route to it). For multi-AZ resilience, you need one NAT Gateway per AZ.

**In short:** IGW is a stateless translation rule at the VPC boundary; NAT Gateway is a stateful, addressable device that must sit inside a subnet to receive and forward traffic like any other host.

### 9.2 IGW vs NAT Gateway — core differences

| | Internet Gateway (IGW) | NAT Gateway |
|---|---|---|
| **Direction** | Bidirectional — inbound and outbound | Outbound-only (private instances can't be reached from the internet through it) |
| **NAT type** | 1:1 (each instance gets its own public/Elastic IP) | Many:1 (many private IPs share one NAT public IP) |
| **Where it lives** | Attached to the VPC, not a subnet | Deployed inside a specific public subnet, consumes an IP from it |
| **Requires** | Nothing extra | An Elastic IP, and the public subnet it sits in must itself have a route to an IGW |
| **AZ scope** | VPC-wide, spans all AZs automatically | Single-AZ; needs one per AZ for HA |
| **Cost** | Free | Hourly charge + per-GB data processing charge |
| **Who uses it** | Instances with a public IP (needs both an IGW route *and* a public IP/EIP on the instance) | Instances with no public IP that need outbound-only access |

### 9.3 Other common points of confusion

**"My instance has a public IP but still can't reach the internet."**
A public IP alone isn't enough — the subnet's route table also needs `0.0.0.0/0 → IGW`, and the instance's security group/NACL must allow the traffic. All three (public IP + IGW route + SG/NACL) are required together.

**"Why does the private subnet still need a route table if it has no direct internet route?"**
Every subnet must be associated with a route table — if you don't explicitly associate one, AWS uses the VPC's default (main) route table. The private route table always contains the `local` route (mandatory, added automatically) plus whatever you add for internet-bound traffic (the NAT route here).

**NAT Gateway vs NAT Instance**
NAT Gateway (used in this lab) is AWS-managed, scales automatically, no patching. A NAT Instance is a regular EC2 instance running NAT software — cheaper at low volume but you manage patching, scaling, and it's a single point of failure unless you build HA yourself. Prefer NAT Gateway unless you have a specific cost/customization reason not to.

**Security Groups vs Network ACLs**
Security groups are stateful (return traffic auto-allowed) and attached to ENIs/instances — this is what governs bastion → MongoDB access in this lab. NACLs are stateless, attached to subnets, and evaluate rules in numeric order. If everything looks correctly configured in security groups but traffic still fails, check for a restrictive NACL on either subnet.

**"Local route" isn't optional or editable in the way other routes are**
The `10.0.0.0/16 → local` route exists in every route table in the VPC by default and can't be deleted. It's what allows any two resources inside the VPC to reach each other directly, regardless of public/private subnet placement — this is the mechanism behind bastion → MongoDB connectivity.

**Why MongoDB refuses remote connections by default even with correct routing/SG**
MongoDB's own `bindIp` setting (in `mongod.conf`), not networking, restricts which interfaces `mongod` listens on. Out of the box it binds to `127.0.0.1` only — so even with perfect routes and security groups, nothing outside the instance itself can connect until `bindIp` explicitly includes the private IP (or `0.0.0.0` for all interfaces, not recommended without auth).

---

## 10. Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---|---|---|
| Can't SSH into bastion at all | Missing `0.0.0.0/0 → IGW` in public RT, SG doesn't allow port 22 from your IP, or instance has no public IP | Verify public RT, SG inbound rule, and that "Auto-assign public IP" was enabled at launch |
| SSH to bastion works, but SSH to private instance from bastion times out | MongoDB-SG doesn't allow port 22 from Bastion-SG (or from bastion's private IP); wrong private IP used | Check MongoDB-SG inbound rule references Bastion-SG, not a CIDR; confirm private instance's actual private IP |
| `Permission denied (publickey)` when jumping to private instance | Private key not loaded in agent, or `-A` flag omitted when SSHing into bastion | Run `ssh-add key.pem` locally, then `ssh -A ec2-user@bastion-ip`; never copy `.pem` onto the bastion itself |
| `Permissions 0644 for 'key.pem' are too open` | Wrong file permissions on the key | `chmod 400 key.pem` |
| Auth works to bastion but fails oddly on private instance (looks like wrong password/key) | Using the wrong key pair for that specific instance | Confirm which key pair was assigned to the private instance at launch — keys aren't interchangeable across instances |
| Private instance has no internet access (can't fetch MongoDB repo) | NAT Gateway missing, deleted, or in the wrong subnet; private RT missing `0.0.0.0/0 → NAT`; NAT Gateway's EIP was released | Confirm NAT Gateway status is "Available", check private RT has the NAT route, verify NAT sits in the *public* subnet |
| Route shows `Blackhole` status | Target resource (NAT Gateway, IGW, peering connection) was deleted but the route wasn't cleaned up | Delete the stale route and recreate pointing to a valid target; refresh console — this can be a stale-state display issue too |
| `mongosh` from bastion gets "connection refused" even though SG/route look fine | `bindIp` in `mongod.conf` still set to `127.0.0.1` only | Add the private IP to `bindIp`, then `sudo systemctl restart mongod` |
| `mongod` fails to start after enabling `authorization` | No admin user was created before auth was turned on — now locked out | Temporarily disable auth, start `mongod`, create the user, re-enable auth |
| `systemctl start mongod` fails immediately | Data directory permissions wrong (`/var/lib/mongo` or `/var/lib/mongodb` not owned by `mongodb` user), or corrupt lock file | Check `sudo journalctl -xeu mongod`; verify ownership: `sudo chown -R mongodb:mongodb /var/lib/mongo` |
| MongoDB doesn't restart after reboot | Service not enabled (only started manually) | `sudo systemctl enable mongod` (not just `start`) |
| Can connect via `mongosh` locally but not from bastion | `bindIp` restricts to localhost, or MongoDB-SG blocks 27017 from Bastion-SG | Check `bindIp` in `mongod.conf`, confirm SG inbound rule for port 27017 |
| "No route with destination X in route table Y" error when deleting | Console showing stale state; route already removed server-side | Refresh the page, re-open the route table, retry against current state |

---

## 11. Quick Reference

**Network layout**
| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 (ap-southeast-1a) |
| Private subnet | 10.0.2.0/24 (ap-southeast-1a) |
| Public RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| Private RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → NAT Gateway` |

**EC2 instance config**
| Item | Bastion Host | MongoDB Instance |
|---|---|---|
| Subnet | Public (10.0.1.0/24) | Private (10.0.2.0/24) |
| Auto-assign public IP | Enabled | Disabled |
| Security group | Bastion-SG | MongoDB-SG |
| Key pair | `bastion-key.pem` | Same or separate key (never copied to bastion) |
| AMI | Amazon Linux 2023 / Ubuntu 22.04 | Amazon Linux 2023 / Ubuntu 22.04 |
| Instance type | t3.micro (sufficient for a bastion) | t3.small or larger (MongoDB needs more headroom) |
| Storage | 8 GB gp3 (default) | 20+ GB gp3 (data directory growth) |
| IAM role | None required | None required (unless using SSM/CloudWatch) |

**Security groups**
| SG | Direction | Type | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|---|---|
| Bastion-SG | Inbound | SSH | TCP | 22 | Your IP (`x.x.x.x/32`) | Admin SSH access from your machine |
| Bastion-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Allows SSH onward to MongoDB-SG + general outbound (default allow-all) |
| MongoDB-SG | Inbound | SSH | TCP | 22 | Bastion-SG (by SG ID, not CIDR) | Allows bastion to jump into the instance |
| MongoDB-SG | Inbound | Custom TCP | TCP | 27017 | Bastion-SG (or App-tier-SG if you add one later) | DB access, scoped to trusted sources only |
| MongoDB-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Needed for repo/package updates via NAT; can be tightened to just HTTPS (443) if you want least-privilege |

**SSH (agent forwarding)**
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>
ssh ec2-user@10.0.2.x                      # from inside bastion, no key needed
```

**MongoDB install + systemd**
```bash
sudo apt install -y mongodb-org             # after adding the repo
sudo systemctl enable --now mongod
sudo systemctl status mongod
mongosh
```

**Fast diagnostics**
```bash
systemctl is-active mongod               # is it running
journalctl -xeu mongod                   # why it failed
curl -I https://amazon.com               # private instance internet check (via NAT)
ip a                                     # confirm instance's actual private IP
mongosh --eval "db.adminCommand('ping')" # confirm mongod is responsive
```

**Golden rules**
- Internet in/out → IGW. Private subnet's internet out → NAT. Subnet-to-subnet → local route. Three separate mechanisms.
- No IGW route = no internet, regardless of public IP.
- `systemctl enable` = survives reboot. `systemctl start` alone does not.
- Correct routing + SG isn't enough for MongoDB — `bindIp` in `mongod.conf` must also include the private IP.
- Create the admin user *before* enabling `authorization` in `mongod.conf`, or you'll lock yourself out.

---

## Key Takeaways

- NAT Gateway = outbound internet for private subnet only; it does not enable bastion-to-instance SSH.
- Bastion-to-private SSH works via the VPC local route + security group rules.
- Agent forwarding (`-A`) avoids storing private keys on the bastion — a security practice, not a connectivity requirement.
- `systemctl enable` ensures boot-time start; `Restart=on-failure` in the unit file handles crash recovery.
- MongoDB's `bindIp` is an application-layer restriction independent of AWS networking — both must be configured correctly for remote access to work.
