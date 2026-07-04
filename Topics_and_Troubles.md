## 1. Inbound and Outbound rules of Security-group

**Inbound rule = traffic coming *into* the instance**

Who/what is allowed to initiate a connection to this instance. 
Source can be:
- The internet (`0.0.0.0/0`)
- Your specific IP (`x.x.x.x/32`)
- Another security group (like `Bastion-SG` allowed into `MySQL-SG`)
- A CIDR range (e.g. `10.0.0.0/16`)

Example: your MySQL-SG inbound rule allows port 3306 **from Bastion-SG** — that's not internet traffic at all, it's internal VPC traffic. Inbound just means "someone is initiating a connection to me."

**Outbound rule = traffic leaving the instance, initiated by the instance**

What this instance is allowed to connect *out* to. 

Example: MySQL instance outbound to `0.0.0.0/0` on port 443 — for OS package updates via NAT.

**The key correction to your framing:**
It's **"into this instance" vs "out of this instance"** — not "internet vs instance." Both inbound and outbound can be internet traffic, internal VPC traffic, or another instance entirely, depending on the source/destination you specify.

**One more important detail — security groups are stateful:**
If inbound allows something in, the **response traffic is automatically allowed out** — you don't need a matching outbound rule for replies. So when your bastion connects in in on port 22, MySQL-SG doesn't need an outbound rule to let the SSH *response* back out — that's automatic. Outbound rules only matter for traffic the instance itself *initiates* (like MySQL reaching out to apt-server or yum-servers).

**Quick way to think about it in your setup:**
- Bastion-SG inbound (22, your IP) = "who can start an SSH session into the bastion"
- MySQL-SG inbound (22/3306, Bastion-SG) = "who can start a session into MySQL" — scoped to the bastion only
- MySQL-SG outbound (443/80, `0.0.0.0/0`) = "what MySQL is allowed to reach out to" — needed for updates via NAT, not for replying to the bastion's SSH

---

## 2. Why is NAT Gateway inside a subnet, but IGW isn't?

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

---

## 3. IGW vs NAT Gateway — core differences

| | Internet Gateway (IGW) | NAT Gateway |
|---|---|---|
| **Direction** | Bidirectional — inbound and outbound | Outbound-only (private instances can't be reached from the internet through it) |
| **NAT type** | 1:1 (each instance gets its own public/Elastic IP) | Many:1 (many private IPs share one NAT public IP) |
| **Where it lives** | Attached to the VPC, not a subnet | Deployed inside a specific public subnet, consumes an IP from it |
| **Requires** | Nothing extra | An Elastic IP, and the public subnet it sits in must itself have a route to an IGW |
| **AZ scope** | VPC-wide, spans all AZs automatically | Single-AZ; needs one per AZ for HA |
| **Cost** | Free | Hourly charge + per-GB data processing charge |
| **Who uses it** | Instances with a public IP (needs both an IGW route *and* a public IP/EIP on the instance) | Instances with no public IP that need outbound-only access |

---

## 4. Other common points of confusion

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

## 5. Troubleshooting Guide

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

