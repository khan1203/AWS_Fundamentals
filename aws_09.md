# Nginx Layer-7 Load Balancing on Node.js + MySQL with AWS VPC

## Overview

This document outlines the process of setting up a **Layer 7 (HTTP) load balancer** application environment using Nginx, consisting of two identical Node.js applications, backed by a shared MySQL instance, inside a two-tier VPC in `ap-southeast-1`.

**Architecture:**

<img width="985" height="500" alt="image" src="https://github.com/user-attachments/assets/bb39fb37-aac7-4641-b6e1-2e4612b0695b" />


| Component | Subnet | CIDR | Role |
|---|---|---|---|
| VPC | — | `10.0.0.0/16` | Regional container |
| public_subnet | AZ-a | `10.0.1.0/24` | Internet-facing tier |
| private_subnet | AZ-a | `10.0.2.0/24` | App + Data tier |

- **NAT Gateway** (public_subnet) — outbound internet for private instances (package installs, updates)
- **Nginx EC2** `10.0.1.14` (public_subnet) — L7 HTTP reverse proxy / load balancer, single public entry point
- **Node1 EC2** `10.0.2.10`, **Node2 EC2** `10.0.2.11` (private_subnet) — identical Node.js app instances
- **MySQL EC2** `10.0.2.12` (private_subnet) — shared database, not directly reachable from the internet

**Traffic flow:**

```
Client → IGW → Nginx (10.0.1.14:80) → [http module: proxy_pass, round robin]
                                          ├─→ Node1 (10.0.2.10:5001) → MySQL (10.0.2.12:3306)
                                          └─→ Node2 (10.0.2.11:5002) → MySQL (10.0.2.12:3306)
```

This is the same physical topology as the Layer-4 guide — same VPC, same subnets, same three private instances. What changes is entirely inside the Nginx box: it now terminates and parses HTTP instead of blindly forwarding TCP bytes.

---

## 💯 Layer 4 vs Layer 7: The Actual Difference

This is the conceptual core of this guide. The two previous guides share infrastructure but solve the load-balancing problem at different layers of the OSI model, and that choice has real consequences.

### Industry requirement — when each is chosen

| | Layer 4 (`stream`) | Layer 7 (`http`) |
|---|---|---|
| Chosen when | Raw throughput matters more than routing intelligence; protocol may not even be HTTP (databases, message brokers, custom TCP services) | Traffic is HTTP/HTTPS and you need to make decisions based on the request itself |
| Typical production use | Load balancing MySQL read replicas, SMTP relays, generic TCP microservices, or as a cheap front door before an L7 tier | API gateways, microservice routing, blue/green and canary deploys, WAF/rate-limiting layers |
| Team ownership | Usually infra/network team — treated like a network appliance | Usually app/platform team — treated like part of the request pipeline |

### Configuration — what actually changes in nginx.conf

| | Layer 4 | Layer 7 |
|---|---|---|
| Top-level block | `stream {}` | `http {}` |
| Directive | `proxy_pass node_backend;` (bare, no scheme) | `proxy_pass http://node_backend;` (inside a `location` block, scheme required) |
| Unit of routing | A TCP connection | An individual HTTP request |
| Can route by path/host/header? | No — nginx never looks past the TCP handshake | Yes — `location /api`, `location /users`, `map $http_host ...`, etc. |
| Can set/modify headers? | No — payload is opaque bytes | Yes — `proxy_set_header X-Forwarded-For ...` |

### Practical implementation — what you gain and lose

- **Client IP visibility:** In L4, the backend still sees Nginx's IP as the connection source unless you explicitly enable `proxy_protocol` on both nginx and the upstream app. In L7, nginx can simply attach the real client IP as an `X-Forwarded-For` header on every request — no extra protocol needed on either side. This guide uses that header, which is the most common reason teams move from L4 to L7 in the first place.
- **Health checks:** L4 can only detect a dead upstream via a failed TCP handshake (`max_fails`/`fail_timeout`). L7 can hit an actual endpoint (e.g. `/health`) and fail an upstream out based on a non-200 response or malformed body — a TCP port can be open while the application behind it is broken, and only L7 catches that.
- **SSL/TLS termination:** L7 nginx terminates HTTPS once at the edge and speaks plain HTTP to the backends, so certificate management lives in one place. L4 either passes the encrypted stream through untouched (backends each need their own cert) or you don't get TLS visibility at all.
- **Session persistence:** L7 can do cookie-based or `ip_hash` stickiness at the request level. L4's `ip_hash` only works at the connection level, which is coarser and breaks down with connection reuse/pooling.
- **Overhead:** L4 is cheaper per connection — no request parsing, buffering, or header rewriting. This is the one place L4 still wins outright, which is why it's the right choice when you don't need any of the above.

**Bottom line for this pair of guides:** the L4 guide proved you can distribute raw TCP connections across two stateless Node instances. This guide proves the same distribution can be made "aware" of the HTTP request — visible client IPs, inspectable requests, and a foundation for path-based routing and app-level health checks later.

---

## AWS Setup

The VPC, subnets, route tables, gateways, and security groups are identical to the Layer-4 guide — this guide reuses that same network, and only replaces the Nginx configuration and the load-balancing directive. If you already built the L4 guide's VPC, you can skip straight to Section 8.

The `resource-map` of our VPC should be as follows:

<img width="1572" height="394" alt="image" src="https://github.com/user-attachments/assets/1267e4c6-b1bd-4855-b47a-2aabdbd0e2bc" />

## 1. VPC and Subnets

| Resource | CIDR | AZ | Auto-assign public IP |
|---|---|---|---|
| VPC | `10.0.0.0/16` | ap-southeast-1 | — |
| public_subnet | `10.0.1.0/24` | ap-southeast-1a | Yes |
| private_subnet | `10.0.2.0/24` | ap-southeast-1a | No |

## 2. Internet Gateway, NAT Gateway, Routing

| Route Table | Destination | Target |
|---|---|---|
| Public RT | `10.0.0.0/16` | local |
| Public RT | `0.0.0.0/0` | igw |
| Private RT | `10.0.0.0/16` | local |
| Private RT | `0.0.0.0/0` | nat-gateway |

Associate public RT with public_subnet, private RT with private_subnet. NAT Gateway must show `Available` before private instances can reach package repos.

## 3. Security Groups

**nginx-sg** (attached to Nginx EC2):

| Direction | Type | Port | Source/Destination |
|---|---|---|---|
| Inbound | SSH | 22 | Your admin IP |
| Inbound | HTTP | 80 | `0.0.0.0/0` |
| Inbound | HTTPS | 443 | `0.0.0.0/0` |
| Outbound | Custom TCP | 5001, 5002 | node-sg |
| Outbound | SSH | 22 | node-sg (SSH passthrough to private nodes) |
| Outbound | SSH | 22 | mysql-sg (SSH passthrough to private nodes) |

To get `admin ip` run in your terminal:
```
curl ifconfig.me
```

**node-sg** (attached to Node1, Node2):

| Direction | Type | Port | Source/Destination |
|---|---|---|---|
| Inbound | Custom TCP | 5001, 5002 | nginx-sg |
| Inbound | SSH | 22 | nginx-sg |
| Outbound | MySQL/Aurora | 3306 | mysql-sg |
| Outbound | HTTP | 80 | `0.0.0.0/0` (via NAT, for npm installs) |
| Outbound | HTTPS | 443 | `0.0.0.0/0` (via NAT, for npm installs) |

**mysql-sg** (attached to MySQL EC2):

| Direction | Type | Port | Source/Destination |
|---|---|---|---|
| Inbound | MySQL/Aurora | 3306 | node-sg |
| Inbound | SSH | 22 | nginx-sg |
| Outbound | HTTP | 80 | `0.0.0.0/0` (via NAT, for apt updates) |
| Outbound | HTTPS | 443 | `0.0.0.0/0` (via NAT, for apt updates) |

The only real difference from the L4 guide's security groups is that ports 5001/5002 are now referenced explicitly since Node1 and Node2 run on distinct ports — nginx's `http` upstream block needs to reach both by name and port, same as `stream` did.

## 4. Launch Instances

| Instance | Subnet | AMI | Type | Security Group |
|---|---|---|---|---|
| Nginx | public_subnet | Ubuntu 22.04 | t3.micro | nginx-sg |
| Node1 | private_subnet | Ubuntu 22.04 | t3.micro | node-sg |
| Node2 | private_subnet | Ubuntu 22.04 | t3.micro | node-sg |
| MySQL | private_subnet | Ubuntu 22.04 | t3.small | mysql-sg |

## 5. SSH Access to Private Instances

Nginx EC2 doubles as the jump host (no separate bastion in this topology). Use agent forwarding so private keys never touch the Nginx instance:

```bash
chmod 400 key.pem
eval $(ssh-agent)
ssh-add key.pem

ssh -A ubuntu@<nginx-public-ip>

# from inside Nginx EC2:
ssh ubuntu@10.0.2.x   # Node1
ssh ubuntu@10.0.2.x   # Node2
ssh ubuntu@10.0.2.x   # MySQL
```

## 6. MySQL Setup (10.0.2.12)

From Ngnix EC2, connect to the MySQL instance; install Docker and run the database container:

```
ssh ubuntu@10.0.2.x          # Enter into MySQL EC2
sudo apt update
sudo apt install docker.io
```

```bash
sudo docker run \
  --name mysql-cont \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=appdb \
  -e MYSQL_USER=appuser \
  -e MYSQL_PASSWORD=pass1212 \
  -p 3306:3306 \
  -v mysql_data:/var/lib/mysql \
  -d mysql:latest
```

We need to set the inbound security group to allow access on port 3306 from `node-sg` only.

## 7. Node.js App Setup (10.0.2.10 and 10.0.2.11 — identical steps on both)

```bash
sudo apt update && sudo apt install -y nodejs npm
mkdir nodeapp && cd nodeapp
npm init -y
npm install express mysql2 body-parser
```

`server.js` — same as the L4 guide, with one addition: since we're now behind an L7 proxy, the app logs the `X-Forwarded-For` header to demonstrate that nginx is preserving the real client IP (something the L4 guide could not do without `proxy_protocol`):

```javascript
const express = require('express');
const mysql = require('mysql2');
const bodyParser = require('body-parser');
const app = express();
const port = process.env.PORT || 5000;

const dbConfig = {
  host: '<mysql-instance-private-ip>',
  user: 'appuser',
  password: 'pass1212',
  database: 'appdb'
};

app.use(bodyParser.json());

function createConnection() {
  const connection = mysql.createConnection(dbConfig);
  connection.connect(error => {
    if (error) {
      console.error('Error connecting to the database:', error);
      return null;
    }
    console.log('Connected to MySQL database');
  });
  return connection;
}

function createTable() {
  const connection = createConnection();
  if (connection) {
    const createTableQuery = `
      CREATE TABLE IF NOT EXISTS users (
        id INT AUTO_INCREMENT PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) NOT NULL UNIQUE
      )
    `;
    connection.query(createTableQuery, (error, results) => {
      connection.end();
      if (error) {
        console.error('Error creating table:', error);
      } else {
        console.log('Table "users" ensured to exist');
      }
    });
  }
}

// Ensure the table is created when the server starts
createTable();

app.get('/', (req, res) => {
  const connection = createConnection();
  console.log(`Request from client IP: ${req.headers['x-forwarded-for'] || req.socket.remoteAddress}`);
  if (connection) {
    res.status(200).json({
      message: `Hello, from Node App on PORT: ${port} ! Connected to MySQL database.`,
      clientIp: req.headers['x-forwarded-for'] || req.socket.remoteAddress
    });
    connection.end();
  } else {
    res.status(500).json({ message: 'Failed to connect to MySQL database' });
  }
});

app.get('/users', (req, res) => {
  const connection = createConnection();
  if (connection) {
    connection.query('SELECT * FROM users', (error, results) => {
      connection.end();
      if (error) {
        return res.status(500).json({ message: 'Error fetching users', error });
      }
      res.json(results);
    });
  } else {
    res.status(500).json({ message: 'Failed to connect to MySQL database' });
  }
});

app.post('/users', (req, res) => {
  const connection = createConnection();
  const { name, email } = req.body;
  if (!name || !email) {
    return res.status(400).json({ message: 'Name and email are required' });
  }
  if (connection) {
    const query = 'INSERT INTO users (name, email) VALUES (?, ?)';
    connection.query(query, [name, email], (error, results) => {
      connection.end();
      if (error) {
        return res.status(500).json({ message: 'Error adding user', error });
      }
      res.status(201).json({ message: 'User added', userId: results.insertId });
    });
  } else {
    res.status(500).json({ message: 'Failed to connect to MySQL database' });
  }
});

app.listen(port, () => {
  console.log(`App running on port :${port}`);
});
```

Run it directly first to confirm it works before wiring up systemd:

```bash
export PORT=5001
node server.js
```

(Use `PORT=5000` on Node1 and `PORT=5001` on Node2, or whatever pair you prefer — just make sure the nginx `upstream` block in Section 8 points at the same ports.)

**(Optional for now)**
Systemd unit `/etc/systemd/system/nodeapp.service`:

```ini
[Unit]
Description=Node app
After=network.target

[Service]
EnvironmentFile=/etc/nodeapp/nodeapp.env
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

Repeat identically on Node2: the only difference between the two instances is the port and the hostname in the response, which is how you'll visually confirm load balancing in Section 9.

## 8. Nginx L7 Load Balancer Setup (10.0.1.14)

Connect to the Nginx instance and create an `nginx.conf` file and a `Dockerfile`. We also need Docker installed.

```bash
sudo apt update && sudo apt install -y nginx
mkdir Nginx
cd Nginx
touch nginx.conf
```

Paste this configuration:

```nginx
events {}

http {
    upstream node_backend {
        server 10.0.2.x:5001;  # Node-1
        server 10.0.2.x:5002;  # Node-2
    }

    server {
        listen 80;

        location / {
            proxy_pass http://node_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Passive health check — pulls an upstream out after repeated failures
            proxy_next_upstream error timeout http_502 http_503;
        }
    }
}
```

`NB:` **Replace the private IPs and ports of the Node.js apps according to your EC2 instances.**

This configuration sets up Nginx to act as an HTTP reverse proxy, distributing requests between the two Node.js applications. Key differences from the `stream` version:

- `http {}` instead of `stream {}` — nginx now parses full HTTP requests, not raw TCP bytes.
- `proxy_pass http://node_backend;` lives inside a `location` block and requires the `http://` scheme — this is what lets you later add `location /api { ... }`, `location /admin { ... }`, etc., each pointing at a different upstream if needed.
- `proxy_set_header` directives exist only in the `http` module — this is where the real client IP gets forwarded as `X-Forwarded-For`, which the app now logs.
- `proxy_next_upstream` defines what counts as a failure worth retrying on a different upstream — an HTTP-aware concept that doesn't exist in `stream`.

### Create & Configure Dockerfile for Nginx

```
FROM nginx:latest
COPY nginx.conf /etc/nginx/nginx.conf
```

Build and run:

```bash
docker build -t custom-nginx-l7 .
docker run -d -p 80:80 --name my_nginx_l7 custom-nginx-l7
```

## 9. Verification

**1. Basic load balancer check**

Visit `http://<nginx-public-ip>` in a browser. You should get a JSON response from one of the Node apps, including a `clientIp` field.

**2. Confirm the client IP is actually forwarded (the L7-specific check)**

```bash
curl -s http://<nginx-public-ip>/ | grep clientIp
```

The `clientIp` returned should be your real public IP, not Nginx's private IP (`10.0.1.14`) — this is the concrete proof that L7 header forwarding is working, and the main practical thing this guide adds over the L4 version.

**3. `/users` endpoints**

- `GET http://<nginx-public-ip>/users` — fetch all users
- `POST http://<nginx-public-ip>/users` — add a new user

POST body (JSON):

```json
{
  "name": "Rifat Khan",
  "email": "khan1212@gmail.com"
}
```

Using Postman for the `POST`: set method to `POST`, URL to `http://<nginx-public-ip>/users`, Body → raw → JSON, paste the payload above, and send. Then confirm the write with a `GET` to `/users`.

**4. Confirm load balancing across requests**

Refresh the browser or make multiple requests to observe the load balancing in action. You should see responses alternating between Node-app-1 and Node-app-2 running on different ports:
<img width="1038" height="257" alt="image" src="https://github.com/user-attachments/assets/ede16040-a6ee-4232-a903-ddb70b0fc76d" />
<img width="933" height="214" alt="image" src="https://github.com/user-attachments/assets/cca212ff-3290-42d8-95b5-d1652bd66891" />

**Validation checklist:**

- [ ] NAT Gateway state = `Available`
- [ ] Private RT default route points to NAT Gateway, public RT to IGW
- [ ] `nginx -t` passes with no syntax errors
- [ ] `docker ps` shows `my_nginx_l7` and `my_mysql` running
- [ ] Repeated curls to Nginx public IP alternate between Node1 and Node2 responses
- [ ] `curl .../` returns your real public IP in `clientIp`, not `10.0.1.14`
- [ ] `POST /users` via Postman returns `201` with a new `id`
- [ ] `GET /users` reflects users inserted via POST, regardless of which node served the GET
- [ ] MySQL instance has no public IP and is unreachable from the internet
- [ ] Stopping one Node app causes `proxy_next_upstream` to route all traffic to the surviving node

## 10. Quick Reference

| Item | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 |
| Private subnet | 10.0.2.0/24 |
| Nginx | 10.0.1.14 : 80 (public), `http` module, proxy_pass |
| Node1 / Node2 | 10.0.2.10 : 5001 / 10.0.2.11 : 5002 |
| MySQL | 10.0.2.12 : 3306 |
| LB config file | /etc/nginx/nginx.conf (`http` block) |
| LB method | round robin (default); `proxy_next_upstream` for passive failover |
| Forwarded headers | X-Real-IP, X-Forwarded-For, X-Forwarded-Proto |

## Key Takeaways

- `http {}` is nginx's L7 module — it parses full HTTP requests, unlike `stream {}` (L4), which forwards opaque TCP bytes.
- `proxy_pass` in L7 requires a scheme (`http://...`) and lives inside a `location` block — this is the mechanism that later enables path-based routing (`location /api`, `location /admin`, etc.).
- The single biggest practical gain in this guide over the L4 one: the backend now sees the real client IP via `X-Forwarded-For`, with no `proxy_protocol` setup needed on either side.
- L7 health checks can be request-aware (`proxy_next_upstream` on HTTP status codes); L4 can only react to TCP-level connection failures.
- Round robin is still the nginx upstream default in `http`, same as in `stream` — the load-balancing algorithm itself didn't change, only what nginx understands about each request.
- Same VPC, subnets, route tables, and SG-chaining pattern as the L4 guide — this guide is purely a swap of the Nginx layer, not the network.
