# Deploying MySQL on EC2 Using Systemd

## Overview

This guide deploys MySQL on a private EC2 instance, accessed only through a public bastion host, inside a custom VPC. MySQL is managed via `systemd` so it auto-starts on boot and restarts on failure.

**Architecture**

```
Internet
   |
  IGW
   |
  VPC 10.0.0.0/16 (ap-southeast-1)
   |
   +-- Public Subnet 10.0.0.0/24 -- Public RT (0.0.0.0/0 -> IGW)
   |       -> Bastion Host (public IP) + NAT Gateway
   |
   +-- Private Subnet 10.0.1.0/24 -- Private RT (0.0.0.0/0 -> NAT)
           -> MySQL Instance (no public IP)
```

<img width="1440" height="1080" alt="image" src="https://github.com/user-attachments/assets/ff6daa79-c932-41b2-8e31-2f9b11326e10" />


Bastion reaches MySQL instance via the VPC **local route** (10.0.0.0/16), not through the NAT/IGW — those only handle internet-bound traffic.

---

## 1. VPC & Networking

### 1.1 Create VPC
- CIDR: `10.0.0.0/16`
- Region: `ap-southeast-1`

### 1.2 Create Subnets
| Subnet | CIDR | AZ |
|---|---|---|
| Public | 10.0.0.0/24 | ap-southeast-1a |
| Private | 10.0.1.0/24 | ap-southeast-1a |

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
ssh ec2-user@10.0.1.x
```

---

## 5. Install MySQL

On the private MySQL instance:

```bash
sudo yum update -y
sudo yum install -y mysql-server   # Amazon Linux 2/2023
# or: sudo apt update && sudo apt install -y mysql-server   # Ubuntu
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

```ini
# /etc/systemd/system/mysqld.service
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=simple
User=mysql
Group=mysql
ExecStart=/usr/sbin/mysqld
Restart=on-failure

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

Confirm auto-start:
```bash
sudo reboot
# after reboot, from bastion:
ssh ec2-user@10.0.1.x "systemctl is-active mysqld"
```

---

## 8. Validation Checklist

- [ ] Bastion reachable via SSH from local machine (public RT + IGW)
- [ ] Private instance reachable from bastion only (local route, MySQL-SG allows Bastion-SG)
- [ ] Private instance has outbound internet via NAT (test: `curl -I https://amazon.com`)
- [ ] `systemctl status mysqld` shows `active (running)` and `enabled`
- [ ] MySQL survives reboot without manual restart

---

## Key Takeaways

- NAT Gateway = outbound internet for private subnet only; it does not enable bastion-to-instance SSH.
- Bastion-to-private SSH works via the VPC local route + security group rules.
- Agent forwarding (`-A`) avoids storing private keys on the bastion — a security practice, not a connectivity requirement.
- `systemctl enable` ensures boot-time start; `Restart=on-failure` in the unit file handles crash recovery.
