# Nginx Layer-4 Load Balancing on Node.js + MySQL with AWS VPC

## Overview

This guide deploys a Node.js application behind an **Nginx Layer 4 (TCP) load balancer**, backed by a shared MySQL instance, inside a two-tier VPC in `ap-southeast-1`.

**Architecture:**
<img width="985" height="500" alt="image" src="https://github.com/user-attachments/assets/16815a17-991d-4a3f-bef5-5edda0f33e7b" />

| Component | Subnet | CIDR | Role |
|---|---|---|---|
| VPC | — | `10.0.0.0/16` | Regional container |
| public_subnet | AZ-a | `10.0.1.0/24` | Internet-facing tier |
| private_subnet | AZ-a | `10.0.2.0/24` | App + data tier |

- **NAT Gateway** (public_subnet) — outbound internet for private instances (package installs, updates)
- **Nginx EC2** `10.0.1.14` (public_subnet) — L4 TCP load balancer, single public entry point
- **Node1 EC2** `10.0.2.10`, **Node2 EC2** `10.0.2.11` (private_subnet) — identical Node.js app instances
- **MySQL EC2** `10.0.2.12` (private_subnet) — shared database, not directly reachable from the internet

**Traffic flow:**

```
Client → IGW → Nginx (10.0.1.14:80) → [stream module: round robin]
                                          ├─→ Node1 (10.0.2.10:3000) → MySQL (10.0.2.12:3306)
                                          └─→ Node2 (10.0.2.11:3000) → MySQL (10.0.2.12:3306)
```

**Why Layer 4, not Layer 7:** Nginx's `stream` module operates at the TCP level — it forwards raw byte streams between client and upstream without parsing HTTP headers, cookies, or paths. This means:
- No host/path-based routing, no header rewriting, no HTTP-aware health checks
- Lower CPU/memory overhead per connection — nginx isn't buffering or inspecting payloads
- Works for any TCP protocol, not just HTTP (this lab happens to proxy HTTP traffic, but the LB itself is protocol-agnostic)

This is the right tradeoff when you need simple round-robin distribution across identical app instances and don't need L7 features. Node1 and Node2 are stateless and interchangeable — session state, if any, must live in MySQL or a shared cache, not in-process.

---

## 1. VPC and Subnets

| Resource | CIDR | AZ | Auto-assign public IP |
|---|---|---|---|
| VPC | `10.0.0.0/16` | ap-southeast-1 | — |
| public_subnet | `10.0.1.0/24` | ap-southeast-1a | Yes |
| private_subnet | `10.0.2.0/24` | ap-southeast-1a | No |

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24 --availability-zone ap-southeast-1a
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.2.0/24 --availability-zone ap-southeast-1a
aws ec2 modify-subnet-attribute --subnet-id <public-subnet-id> --map-public-ip-on-launch
```

## 2. Internet Gateway, NAT Gateway, Routing

```bash
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id <vpc-id> --internet-gateway-id <igw-id>

aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id <public-subnet-id> --allocation-id <eip-alloc-id>
```

| Route Table | Destination | Target |
|---|---|---|
| Public RT | `10.0.0.0/16` | local |
| Public RT | `0.0.0.0/0` | igw |
| Private RT | `10.0.0.0/16` | local |
| Private RT | `0.0.0.0/0` | nat-gateway |

Associate public RT with public_subnet, private RT with private_subnet. NAT Gateway must show `Available` before private instances can reach package repos — this is the same dependency that caused blackholed routes in the Docker Compose lab if the NAT was deleted before instances finished pulling images.

## 3. Security Groups

**nginx-sg** (attached to Nginx EC2):

| Direction | Protocol | Port | Source/Destination |
|---|---|---|---|
| Inbound | TCP | 22 | Your admin IP |
| Inbound | TCP | 80 | `0.0.0.0/0` |
| Outbound | TCP | 3000 | node-sg |
| Outbound | TCP | 22 | node-sg (SSH passthrough to private nodes) |

**node-sg** (attached to Node1, Node2):

| Direction | Protocol | Port | Source/Destination |
|---|---|---|---|
| Inbound | TCP | 3000 | nginx-sg |
| Inbound | TCP | 22 | nginx-sg |
| Outbound | TCP | 3306 | mysql-sg |
| Outbound | TCP | 443/80 | `0.0.0.0/0` (via NAT, for npm installs) |

**mysql-sg** (attached to MySQL EC2):

| Direction | Protocol | Port | Source/Destination |
|---|---|---|---|
| Inbound | TCP | 3306 | node-sg |
| Inbound | TCP | 22 | nginx-sg |
| Outbound | TCP | 443/80 | `0.0.0.0/0` (via NAT, for apt updates) |

Chaining by SG ID (not CIDR) keeps this correct even if instances are replaced with new private IPs.

## 4. Launch Instances

| Instance | Subnet | Private IP | AMI | Type | Security Group |
|---|---|---|---|---|---|
| Nginx | public_subnet | 10.0.1.14 | Ubuntu 22.04 | t3.micro | nginx-sg |
| Node1 | private_subnet | 10.0.2.10 | Ubuntu 22.04 | t3.micro | node-sg |
| Node2 | private_subnet | 10.0.2.11 | Ubuntu 22.04 | t3.micro | node-sg |
| MySQL | private_subnet | 10.0.2.12 | Ubuntu 22.04 | t3.small | mysql-sg |

Launch each with `--private-ip-address` pinned to the values above so the nginx upstream config and app connection strings are known ahead of time.

## 5. SSH Access to Private Instances

Nginx EC2 doubles as the jump host (no separate bastion in this topology). Use agent forwarding so private keys never touch the Nginx instance:

```bash
chmod 400 key.pem
eval $(ssh-agent)
ssh-add key.pem

ssh -A ubuntu@<nginx-public-ip>

# from inside Nginx EC2:
ssh ubuntu@10.0.2.10   # Node1
ssh ubuntu@10.0.2.11   # Node2
ssh ubuntu@10.0.2.12   # MySQL
```

Agent forwarding is a convenience that keeps the key off the jump host — the actual connection to the private subnet is permitted by the implicit local VPC route plus the `node-sg`/`mysql-sg` rules allowing SSH from `nginx-sg`, not by NAT (NAT is outbound-to-internet only).

## 6. MySQL Setup (10.0.2.12)

Connect to mysql instance install Docker and run the following command to run mysql database container:
```bash
docker run \
  --name my_mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=my_db \
  -e MYSQL_USER=my_user \
  -e MYSQL_PASSWORD=my_password \
  -p 3306:3306 \
  -v mysql_data:/var/lib/mysql \
  -d mysql:latest
```

This command is creating a mySQL container with:

```bash
--name my_mysql                  # Container name
-e MYSQL_ROOT_PASSWORD=root      # Root password
-e MYSQL_DATABASE=my_db          # Create a database named my_db
-e MYSQL_USER=my_user            # Create a new user
-e MYSQL_PASSWORD=my_password    # Password for the new user
-p 3306:3306                     # Map host port 3306 to container port 3306
-v mysql_data:/var/lib/mysql     # Persist MySQL data in a Docker volume
-d                               # Run in detached mode
mysql:latest                     # MySQL image
```

This command creates and runs a MySQL database container with specified database, user, and password and volume mounting for persistent storage, accessible externally on port 3306.

We need to set inbound security-group to give access on port 3306.

## 7. Node.js App Setup (10.0.2.10 and 10.0.2.11 — identical steps on both)

```bash
sudo apt update && sudo apt install -y nodejs npm
mkdir ~/app && cd ~/app
npm init -y
npm install express mysql2
```

`server.js`:

```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const app = express();

const pool = mysql.createPool({
  host: '10.0.2.12',
  user: 'appuser',
  password: '<strong-password>',
  database: 'appdb'
});

app.get('/', async (req, res) => {
  const [rows] = await pool.query('SELECT NOW() AS time');
  res.send(`Served by ${require('os').hostname()} — DB time: ${rows[0].time}`);
});

app.listen(3000, () => console.log('Listening on 3000'));
```

Systemd unit `/etc/systemd/system/nodeapp.service`:

```ini
[Unit]
Description=Node app
After=network.target

[Service]
ExecStart=/usr/bin/node /home/ubuntu/app/server.js
Restart=always
User=ubuntu

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable nodeapp
sudo systemctl start nodeapp
```

Repeat identically on Node2 — the only difference between the two instances is the hostname in the response, which is how you'll visually confirm load balancing in Section 9.

## 8. Nginx L4 Load Balancer Setup (10.0.1.14)

The `stream` module ships separately from core nginx on Ubuntu:

```bash
sudo apt update && sudo apt install -y nginx libnginx-mod-stream
```

Add a `stream` block at the top level of `/etc/nginx/nginx.conf` (outside and alongside the existing `http {}` block — `stream` cannot be nested inside `http`):

```nginx
stream {
    upstream node_backend {
        server 10.0.2.10:3000;
        server 10.0.2.11:3000;
    }

    server {
        listen 80;
        proxy_pass node_backend;
        proxy_connect_timeout 2s;
    }
}
```

```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx
```

Note: since `listen 80` is now claimed by the `stream` block, remove or move any default `http` server block also listening on 80 to avoid a bind conflict.

## 9. Verification
 
**1. Basic load balancer check**
 
Visit `http://<nginx-public-ip>` in a browser. You should get a response from one of the Node apps (`Served by <hostname> — DB time: ...`).
 
**2. `/users` endpoints**
 
- `GET http://<nginx-public-ip>/users` — fetch all users
- `POST http://<nginx-public-ip>/users` — add a new user
POST body (JSON):
 
```json
{
  "name": "John Doe",
  "email": "john@example.com"
}
```
 
Using Postman for the `POST`: set method to `POST`, URL to `http://<nginx-public-ip>/users`, Body → raw → JSON, paste the payload above, and send. You should get back a `201` with the new user's `id` and the hostname of whichever node handled the request.
 
Then confirm the write landed by immediately issuing a `GET` to `http://<nginx-public-ip>/users` — the new user should appear in the returned list regardless of which node serves the GET, since both nodes read from the same MySQL instance (10.0.2.12).
 
**3. Confirm load balancing across requests**:


Refresh the browser or make multiple requests to observe the load balancing in action. You should see responses alternating between Node-app-1 and Node-app-2 running on different ports:
<img width="1038" height="257" alt="image" src="https://github.com/user-attachments/assets/ede16040-a6ee-4232-a903-ddb70b0fc76d" />
<img width="933" height="214" alt="image" src="https://github.com/user-attachments/assets/cca212ff-3290-42d8-95b5-d1652bd66891" />

You should see the hostname alternate between Node1 and Node2 responses.

**Validation checklist:**

- [ ] NAT Gateway state = `Available`
- [ ] Private RT default route points to NAT Gateway, public RT to IGW
- [ ] `nginx -t` passes with no syntax errors
- [ ] `systemctl is-enabled nginx/nodeapp/mysql` returns `enabled` on all four instances
- [ ] Repeated curls to Nginx public IP alternate between Node1 and Node2 hostnames
- [ ] Node1/Node2 can independently reach MySQL: `mysql -h 10.0.2.12 -u appuser -p appdb`
- [ ] MySQL instance has no public IP and is unreachable from the internet
- [ ] Killing `nodeapp` on one node (`sudo systemctl stop nodeapp`) causes nginx to route all traffic to the surviving node without dropping the LB itself

## 10. Quick Reference

| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 |
| Private subnet | 10.0.2.0/24 |
| Nginx | 10.0.1.14 : 80 (public), listens via stream module |
| Node1 / Node2 | 10.0.2.10 / 10.0.2.11 : 3000 |
| MySQL | 10.0.2.12 : 3306 |
| LB config file | /etc/nginx/nginx.conf (stream block) |
| LB method | round robin (default; no directive needed) |

## Key Takeaways

- `stream` is nginx's L4 module — it proxies TCP/UDP connections without HTTP awareness, unlike `proxy_pass` inside an `http` block (L7)
- The `stream` block sits at the same nesting level as `http` in `nginx.conf`, not inside it
- Round robin is the nginx upstream default; no load-balancing method directive is required unless you want `least_conn` or `ip_hash`
- Node1/Node2 must be stateless for round robin to work correctly — any session state belongs in MySQL, not in the Node process
- Same SG-chaining and NAT/route-table dependencies from earlier labs apply: nginx-sg → node-sg → mysql-sg by SG reference, private subnet reaches the internet only via NAT
- `systemctl enable` on all three services (mysql, nodeapp, nginx) is required for reboot persistence
