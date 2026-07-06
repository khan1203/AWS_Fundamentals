# Deploying MySQL in a Private Subnet Using Docker Compose

## Overview

This guide deploys MySQL on a private EC2 instance, accessed only through a public bastion host, inside a custom VPC. MySQL runs as a Docker container managed by Docker Compose, which handles restart policy and data persistence in place of `systemd`.

**Architecture**

<img width="1412" height="718" alt="image" src="https://github.com/user-attachments/assets/adae7f18-f30e-4993-ae14-1143a30a18bd" />


Here's how traffic moves through this setup:

**Internet-bound traffic** (public subnet only)
Internet → IGW → Public Route Table (`0.0.0.0/0 → IGW`) → public subnet. This is the only path in or out of the VPC. Both the bastion host and the NAT gateway live in the public subnet and rely on this path — the bastion for inbound SSH, the NAT gateway for relaying outbound traffic from the private subnet.

**Private instance's outbound traffic**
MySQL instance → Private Route Table (`0.0.0.0/0 → NAT gateway`) → NAT gateway → IGW → Internet. The private subnet has no direct IGW route, so this is its *only* way out — and here it matters for a new reason beyond OS updates: pulling the `mysql:8.0` image from Docker Hub requires this same path.

**Bastion-to-MySQL traffic (the SSH hop)**
Bastion → MySQL instance uses the **local route** (`10.0.0.0/16 → local`), which every route table in the VPC has automatically. Since both subnets sit inside the same VPC CIDR, traffic between them never leaves the VPC and never touches the IGW or NAT gateway at all. This is why:
- SSH from bastion to the private instance keeps working even if the NAT gateway is deleted or blackholed.
- Access control here comes entirely from **security groups** (MySQL-SG allowing SSH/3306 from Bastion-SG), not from routing.

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
- Instance type: t3.small (Docker adds overhead vs. a bare systemd install)

---

## 4. SSH Access (Agent Forwarding)

On local machine:
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ubuntu@<bastion-public-ip>
```

From inside bastion, jump to private instance (key never stored on bastion):
```bash
ssh ubuntu@10.0.2.x
```

---

## 5. Install Docker & Docker Compose

```bash
sudo apt update -y
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
# log out/in for group membership to apply
```

---

## 6. Configure the Compose Service

Instead of a systemd unit file, the container's lifecycle is defined in `docker-compose.yml`. `restart: unless-stopped` is the Compose equivalent of `systemctl enable` — it survives reboots and daemon restarts, but won't fight you if you deliberately stop it.

```bash
mkdir ~/mysql-docker && cd ~/mysql-docker
nano docker-compose.yml
```

```yaml
# ~/mysql-docker/docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: noornote
      MYSQL_USER: noornote_app
      MYSQL_PASSWORD: ${MYSQL_APP_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

Create a `.env` file alongside it (not committed to git) for the two password variables, then start the stack:

```bash
nano .env
# MYSQL_ROOT_PASSWORD=...
# MYSQL_APP_PASSWORD=...

docker compose up -d
docker compose ps
docker compose logs mysql
```

**Important**: `ports: "3306:3306"` binds the container to all interfaces on the host (`0.0.0.0:3306`), unlike MongoDB's `bindIp` default of localhost-only. That means Docker's own port mapping — not a config file — is what exposes the DB to the private subnet. Security is enforced entirely by MySQL-SG at the AWS layer.

---

## 7. Secure & Verify MySQL

Connect and confirm the app user/database exist (created automatically from the compose env vars):
```bash
docker exec -it mysql-db mysql -u root -p -e "SHOW DATABASES;"
docker exec -it mysql-db mysql -u noornote_app -p noornote -e "SELECT 1;"
```

Confirm auto-start behavior:
```bash
sudo reboot

# after reboot, from bastion:
ssh ubuntu@10.0.2.x "docker compose -f ~/mysql-docker/docker-compose.yml ps"
```

Confirm the container itself survives being killed, not just the host rebooting:
```bash
docker kill mysql-db
docker compose ps   # should show it back up within a few seconds
```

---

## 8. Validation Checklist

- [ ] Bastion reachable via SSH from local machine (public RT + IGW)
- [ ] Private instance reachable from bastion only (local route, MySQL-SG allows Bastion-SG)
- [ ] Private instance has outbound internet via NAT (test: `curl -I https://amazon.com`)
- [ ] `docker compose ps` shows `mysql-db` as `Up` and `healthy`
- [ ] Data persists across `docker compose down` (without `-v`) and `docker compose up -d`
- [ ] Container restarts automatically after `docker kill` and after host reboot
- [ ] App user can authenticate against the `noornote` database

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
| AMI | Ubuntu 22.04 | Ubuntu 22.04 |
| Instance type | t3.micro (sufficient for a bastion) | t3.small (Docker overhead) |
| Storage | 8 GB gp3 (default) | 20 GB gp3 |
| IAM role | None required | None required (unless using SSM/CloudWatch) |

**Security groups**
| SG | Direction | Type | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|---|---|
| Bastion-SG | Inbound | SSH | TCP | 22 | Your IP (`x.x.x.x/32`) | Admin SSH access from your machine |
| Bastion-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Allows SSH onward to MySQL-SG + general outbound (default allow-all) |
| MySQL-SG | Inbound | SSH | TCP | 22 | Bastion-SG (by SG ID, not CIDR) | Allows bastion to jump into the instance |
| MySQL-SG | Inbound | Custom TCP | TCP | 3306 | Bastion-SG (or App-tier-SG if you add one later) | DB access, scoped to trusted sources only |
| MySQL-SG | Outbound | All traffic | All | All | `0.0.0.0/0` | Needed for Docker Hub image pulls + package updates via NAT; can be tightened to just HTTPS (443) for least-privilege |

**SSH (agent forwarding)**
```bash
chmod 400 bastion-key.pem
eval "$(ssh-agent -s)"
ssh-add bastion-key.pem
ssh -A ubuntu@<bastion-public-ip>
ssh ubuntu@10.0.2.x                        # from inside bastion, no key needed
```

**Docker install + Compose**
```bash
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
docker compose up -d
docker compose ps
```

**Fast diagnostics**
```bash
docker compose ps                        # is it up and healthy
docker compose logs -f mysql             # why it failed
curl -I https://amazon.com               # private instance internet check (via NAT)
ip a                                     # confirm instance's actual private IP
docker exec -it mysql-db mysqladmin ping # confirm mysqld is responsive
```

**Golden rules**
- Internet in/out → IGW. Private subnet's internet out → NAT. Subnet-to-subnet → local route. Three separate mechanisms.
- No IGW route = no internet, regardless of public IP — and no image pull, either.
- `restart: unless-stopped` = survives reboot and crashes. A bare `docker run` without a restart policy does not.
- A named volume (`mysql_data:`) is what makes data survive `docker compose down`. Never use `docker compose down -v` in production — it deletes the volume.
- Docker's `ports:` mapping, not an app config file, is what exposes MySQL to the network — security is enforced by the AWS security group, not by MySQL itself.

---

## Key Takeaways

- NAT Gateway = outbound internet for private subnet only; it does not enable bastion-to-instance SSH, and it's also what makes `docker pull` work from a private subnet.
- Bastion-to-private SSH works via the VPC local route + security group rules — identical to the systemd lab, since this is a networking property, not an application one.
- Agent forwarding (`-A`) avoids storing private keys on the bastion — a security practice, not a connectivity requirement.
- `restart: unless-stopped` in Compose is the direct analog of `systemctl enable` — both ensure the process comes back without manual intervention.
- Named volumes are the container equivalent of a persistent data directory; forgetting one (or force-deleting it with `-v`) is the container-world version of losing `/var/lib/mysql`.
