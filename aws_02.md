# Deploying MySQL on EC2 Using Systemd

## Overview

This guide deploys MySQL on a private EC2 instance, accessed only through a public bastion host, inside a custom VPC. MySQL is managed via `systemd` so it auto-starts on boot and restarts on failure.

**Architecture**
![AWS VPC Diagram](assets/arch_2.2.png)

Here's the fuller picture of how traffic moves through this setup:

**Internet-bound traffic** (public subnet only)
Internet → IGW → Public Route Table (`0.0.0.0/0 → IGW`) → public subnet. This is the only path in or out of the VPC. Both the bastion host and the NAT gateway live in the public subnet and rely on this path — the bastion for inbound SSH, the NAT gateway for relaying outbound traffic from the private subnet.

**Private instance's outbound traffic**
MySQL instance → Private Route Table (`0.0.0.0/0 → NAT gateway`) → NAT gateway → IGW → Internet. The private subnet has no direct IGW route, so this is its *only* way out (e.g. for OS updates). Nothing initiated from the internet can reach in this way — NAT is outbound-only by nature.

**Bastion-to-MySQL traffic (the SSH hop)**
Bastion → MySQL instance uses the **local route** (`10.0.0.0/16 → local`), which every route table in the VPC has automatically. Since both subnets sit inside the same VPC CIDR, traffic between them never leaves the VPC and never touches the IGW or NAT gateway at all. This is why:
- SSH from bastion to the private instance keeps working even if the NAT gateway is deleted or blackholed.
- Access control here comes entirely from **security groups** (MySQL-SG allowing SSH/3306 from Bastion-SG), not from routing.

**The key distinction:** IGW and NAT handle traffic crossing the VPC boundary (to/from the internet). The local route handles traffic staying inside the VPC boundary (subnet-to-subnet). They're independent mechanisms — one can fail without affecting the other.

<img width="1440" height="1080" alt="image" src="https://github.com/user-attachments/assets/ff6daa79-c932-41b2-8e31-2f9b11326e10" />

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

**MySQL-SG** (private subnet)
- Inbound: SSH (22) from Bastion-SG
- Inbound: MySQL (3306) from Bastion-SG (or app-tier SG)
- Outbound: all

---

## 3. Launch Instances

### 3.1 Bastion Host
- Subnet: public
- Auto-assign public IP: enabled
- SG: Bastion-SG
- Key pair: `bastion-key.pem`

### 3.2 MySQL Instance
- Subnet: private
- Auto-assign public IP: disabled
- SG: MySQL-SG
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

## 5. Install MySQL

On the private MySQL instance:

```bash
# Ubuntu
sudo apt update -y
sudo apt install -y mysql-server

# Amazon Linux 2/2023
sudo yum update -y
sudo yum install -y mysql-server                              
```
---

## 6. Configure Systemd Service

Amazon Linux/Ubuntu MySQL packages ship a unit file already (`mysqld.service` or `mysql.service`). Verify and manage it directly:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mysqld      # start on boot
sudo systemctl start mysqld
sudo systemctl status mysqld
```

If building a custom unit file (e.g. non-package install):
```bash
sudo nano /etc/systemd/system/mysqld.service
```

```ini
# /etc/systemd/system/mysqld.service
[Unit]
Description=MySQL Server
After=syslog.target
After=network.target

[Service]
Type=simple
PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /var/run/mysqld
ExecStartPre=/bin/chown mysql:mysql -R /var/run/mysqld
ExecStart=/usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib/mysql/plugin --log-error=/var/log/mysql/error.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/var/run/mysqld/mysqld.sock --port=3306
TimeoutSec=300
PrivateTmp=true
User=mysql
Group=mysql
WorkingDirectory=/usr

[Install]
WantedBy=multi-user.target

```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysqld
```

---

## 7. Secure & Verify MySQL

```bash
sudo mysql_secure_installation
mysql -u root -p -e "STATUS;"
```
Follow the prompts to secure your MySQL installation (e.g., set root password, remove anonymous users, disallow root login remotely, remove test database, and reload privilege tables).

Confirm auto-start:
```bash
sudo reboot

# after reboot, from bastion:
ssh ec2-user@10.0.2.x "systemctl is-active mysqld"
```

---

## 8. Validation Checklist

- [ ] Bastion reachable via SSH from local machine (public RT + IGW)
- [ ] Private instance reachable from bastion only (local route, MySQL-SG allows Bastion-SG)
- [ ] Private instance has outbound internet via NAT (test: `curl -I https://amazon.com`)
- [ ] `systemctl status mysqld` shows `active (running)` and `enabled`
- [ ] MySQL survives reboot without manual restart

---

## 9. Quick Reference
 
**Network layout**
| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 (ap-southeast-1a) |
| Private subnet | 10.0.2.0/24 (ap-southeast-1a) |
| Public RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| Private RT | `10.0.0.0/16 → local`, `0.0.0.0/0 → NAT Gateway` |

**EC2 instance config**
| Item | Bastion Host | MySQL Instance |
|---|---|---|
| Subnet | Public (10.0.1.0/24) | Private (10.0.2.0/24) |
| Auto-assign public IP | Enabled | Disabled |
| Security group | Bastion-SG | MySQL-SG |
| Key pair | `bastion-key.pem` | Same or separate key (never copied to bastion) |
| AMI | Amazon Linux 2023 / Ubuntu 22.04 | Amazon Linux 2023 / Ubuntu 22.04 |
| Instance type | t3.micro (sufficient for a bastion) | t3.small or larger (headroom for DB workload) |
| Storage | 8 GB gp3 (default) | 20+ GB gp3 (data directory growth) |
| IAM role | None required | None required (unless using SSM/CloudWatch) |

**Security groups**
| SG | Direction | Type | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|---|---|
| Bastion-SG | Inbound | SSH | TCP | 22 | Your IP (`x.x.x.x/32`) | Admin SSH access from your machine |
| Bastion-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Allows SSH onward to MySQL-SG + general outbound (default allow-all) |
| MySQL-SG | Inbound | SSH | TCP | 22 | Bastion-SG (by SG ID, not CIDR) | Allows bastion to jump into the instance |
| MySQL-SG | Inbound | MySQL/Aurora | TCP | 3306 | Bastion-SG (or App-tier-SG if you add one later) | DB access, scoped to trusted sources only |
| MySQL-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Needed for OS updates via NAT; can be tightened to just HTTPS (443) to package mirrors if you want least-privilege |
 
**SSH (agent forwarding)**
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ec2-user@<bastion-public-ip>
ssh ec2-user@10.0.2.x                      # from inside bastion, no key needed
```
 
**MySQL install + systemd**
```bash
sudo apt install -y mysql-server
sudo systemctl enable --now mysqld
sudo systemctl status mysqld
sudo mysql_secure_installation
```
 
**Fast diagnostics**
```bash
systemctl is-active mysqld              # is it running
journalctl -xeu mysqld                  # why it failed
curl -I https://amazon.com              # private instance internet check (via NAT)
ip a                                    # confirm instance's actual private IP
```
 
**Golden rules**
- Internet in/out → IGW. Private subnet's internet out → NAT. Subnet-to-subnet → local route. Three separate mechanisms.
- No IGW route = no internet, regardless of public IP.
- `systemctl enable` = survives reboot. `systemctl start` alone does not.
- Blackhole route / stale console error → refresh, re-check target still exists.

## Key Takeaways

- NAT Gateway = outbound internet for private subnet only; it does not enable bastion-to-instance SSH.
- Bastion-to-private SSH works via the VPC local route + security group rules.
- Agent forwarding (`-A`) avoids storing private keys on the bastion — a security practice, not a connectivity requirement.
- `systemctl enable` ensures boot-time start; `Restart=on-failure` in the unit file handles crash recovery.
