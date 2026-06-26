# TravelMemory — 3-Tier MERN Stack Deployment on AWS

> Full-stack deployment of the [TravelMemory](https://github.com/UnpredictablePrashant/TravelMemory) application on AWS EC2 using a 3-tier architecture with Nginx, Application Load Balancers, and Cloudflare DNS.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Instance Reference](#instance-reference)
- [Task 1 — Backend EC2 Setup](#task-1--backend-ec2-setup)
- [Task 2 — Frontend EC2 Setup](#task-2--frontend-ec2-setup)
- [Task 3 — Scaling with Replica Instances](#task-3--scaling-with-replica-instances)
- [Task 4 — Load Balancing with AWS ALB](#task-4--load-balancing-with-aws-alb)
- [Task 5 — Domain Setup with Cloudflare](#task-5--domain-setup-with-cloudflare)
- [Final Verification](#final-verification)
- [Ports Reference](#ports-reference)
- [Best Practices](#best-practices)

---

## Project Overview

| Field | Details |
|---|---|
| Application | TravelMemory (MERN Stack) |
| Frontend | React.js |
| Backend | Node.js (Express) |
| Database | MongoDB Atlas |
| Cloud Provider | Amazon Web Services (AWS) |
| Region | ap-south-1 (Mumbai) |
| Domain | http://mern-travelmemory.duckdns.org |
| DNS Provider | Cloudflare |
| Load Balancer | AWS Application Load Balancer (ALB) |

The TravelMemory application follows a classic **3-tier architecture**:

| Tier | Technology | Role |
|---|---|---|
| Presentation (Frontend) | React.js on EC2 + Nginx | Renders the UI in the user's browser |
| Application (Backend) | Node.js on EC2 + Nginx | Handles API requests and business logic |
| Data (Database) | MongoDB Atlas | Stores and retrieves all application data |

---

## Architecture

```
User Browser
     |
     v
[Cloudflare DNS] --> mern-travelmemory.duckdns.org
     |
     v
[Frontend ALB: MERN-FE-LoadBalancer]
     |              |
     v              v
[MERN_FE]      [MERN_FE_2]       (React on :3000, Nginx proxy on :80)
     |
     v  (url.js --> back.mern-travelmemory.duckdns.org)
[Backend ALB: MERN-BE-LoadBalancer]
     |              |
     v              v
[MERN_BE]      [MERN_BE_2]       (Node.js on :3000, Nginx proxy on :80)
     |
     v
[MongoDB Atlas — travelmemory cluster]
```

---

## Instance Reference

| Server | Instance Name | Public IP | Private IP | Availability Zone |
|---|---|---|---|---|
| Backend Primary | MERN_BE | 13.203.67.29 | 10.0.3.198 | ap-south-1a |
| Backend Replica | MERN_BE_2 | 13.126.94.179 | 10.0.4.145 | ap-south-1b |
| Frontend Primary | MERN_FE | 13.232.57.171 | 10.0.3.217 | ap-south-1a |
| Frontend Replica | MERN_FE_2 | 13.206.185.103 | 10.0.4.22 | ap-south-1b |

---

## Task 1 — Backend EC2 Setup

### 1.1 Launch the Backend EC2 Instance

Navigate to **AWS Console → EC2 → Launch Instance** with these settings:

| Setting | Value |
|---|---|
| Instance Name | MERN_BE |
| AMI | Ubuntu Server 22.04 LTS (64-bit x86) |
| Instance Type | t2.micro (Free Tier eligible) |
| Key Pair | rahul-mumbai-keypair |
| Auto-assign Public IP | Enabled |

**User Data Script** (paste in *Advanced → User Data*):

```bash
#!/bin/bash
sudo apt-get update
sudo curl -s https://deb.nodesource.com/setup_18.x | sudo bash
sudo apt install -y nodejs
sudo apt update
cd /home/ubuntu/
sudo git clone https://github.com/UnpredictablePrashant/TravelMemory
```

This script automatically: updates packages → installs Node.js 18 → clones the TravelMemory repo on first boot.

> 📸 **Screenshot:** EC2 Launch Instance form with MERN_BE settings filled in

### 1.2 Configure the Backend `.env`

SSH into the backend instance and create the environment file:

```bash
cd /home/ubuntu/TravelMemory/backend
nano .env
```

Contents of `.env`:

```env
MONGO_URI='mongodb+srv://<db_user>:<password>@mern-cluster.zdgycnf.mongodb.net/'
PORT=3000
```

> ⚠️ **Never commit `.env` to version control.** Add it to `.gitignore`.

### 1.3 Install Dependencies and Start the Backend

```bash
npm install
sudo node index.js
```

Expected output:
```
Node.js v18.17.1
Server started at http://localhost:3000
```

Visiting `http://13.203.67.29:3000` should show `Cannot GET /` — confirming the Node.js server is running.

> 📸 **Screenshot:** Browser showing `Cannot GET /` on port 3000 (server is live)

### 1.4 Whitelist Backend IP in MongoDB Atlas

1. Log in to [cloud.mongodb.com](https://cloud.mongodb.com)
2. Navigate to your cluster → **Network Access**
3. Click **Add IP Address**
4. Add `13.203.67.29` (backend primary)
5. Add `13.126.94.179` (backend replica, when scaling)
6. Click **Confirm**

> 📸 **Screenshot:** MongoDB Atlas Network Access panel showing whitelisted IPs

### 1.5 Configure Nginx Reverse Proxy on Backend

```bash
sudo apt install nginx -y
sudo systemctl status nginx

# Disable default site
sudo unlink /etc/nginx/sites-enabled/default

# Create custom config
cd /etc/nginx/sites-available/
sudo nano custom_server.conf
```

Paste this into `custom_server.conf`:

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

Enable and restart:

```bash
sudo ln -s /etc/nginx/sites-available/custom_server.conf \
           /etc/nginx/sites-enabled/custom_server.conf

sudo service nginx configtest   # Expected: [ OK ]
sudo service nginx restart
```

Visiting `http://13.203.67.29` (port 80) should now forward to the Node.js backend.

> 📸 **Screenshot:** `sudo service nginx configtest` output showing `[ OK ]`

---

## Task 2 — Frontend EC2 Setup

### 2.1 Launch the Frontend EC2 Instance

Navigate to **AWS Console → EC2 → Launch Instance**:

| Setting | Value |
|---|---|
| Instance Name | MERN_FE |
| AMI | Ubuntu Server 22.04 LTS (64-bit x86) |
| Instance Type | t2.micro |
| Key Pair | rahul-mumbai-keypair |
| Auto-assign Public IP | Enabled |

Use the **same User Data script** from Task 1 to install Node.js and clone the repo.

Result: Public IP `13.232.57.171`, Private IP `10.0.3.217`, AZ: `ap-south-1a`

### 2.2 Configure Frontend to Connect to Backend

SSH into the frontend instance and update `src/url.js`:

```bash
cd /home/ubuntu/TravelMemory/frontend
nano src/url.js
```

Initial value (using backend's direct IP):

```js
export const baseUrl = "http://13.203.67.29"
```

> This will be updated to use the Cloudflare domain after the load balancer is configured (see Task 5).

### 2.3 Install Dependencies and Start the Frontend

```bash
cd /home/ubuntu/TravelMemory/frontend
npm install
npm start
```

Expected output:
```
Compiled successfully!
Local:   http://localhost:3000
On Your Network: http://10.0.3.217:3000
```

> 📸 **Screenshot:** TravelMemory app UI loaded in the browser at the frontend IP

### 2.4 Configure Nginx Reverse Proxy on Frontend

Apply identical steps from [Task 1.5](#15-configure-nginx-reverse-proxy-on-backend) on the frontend instance.

```bash
sudo apt install nginx -y
sudo unlink /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-available/custom_server.conf
```

Config (same as backend):

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/custom_server.conf \
           /etc/nginx/sites-enabled/custom_server.conf
sudo service nginx configtest && sudo service nginx restart
```

Visiting `http://13.232.57.171` should now serve the React app on port 80.

> 📸 **Screenshot:** TravelMemory UI accessible on port 80 via Nginx

---

## Task 3 — Scaling with Replica Instances

To ensure high availability, AMI snapshots of both instances were taken and replica instances launched in a different Availability Zone.

### 3.1 Create AMI Snapshots

**Backend AMI:**
1. Select `MERN_BE` in EC2 Console
2. Click **Actions → Image and templates → Create Image**
3. Image name: `MERN_BE`
4. Click **Create Image**
   - AMI ID: `ami-06d46ae66f734c3de`

**Frontend AMI:**
1. Select `MERN_FE` in EC2 Console
2. Click **Actions → Image and templates → Create Image**
3. Image name: `MERN_FE`
4. Click **Create Image**
   - AMI ID: `ami-078a42232bf2556a8`

> 📸 **Screenshot:** EC2 → AMIs list showing MERN_BE and MERN_FE AMIs

### 3.2 Launch Replica Instances from AMIs

**Backend Replica (MERN_BE_2):**
1. Go to **EC2 → AMIs → My AMIs**
2. Select `MERN_BE` → **Launch instance from AMI**
3. Name: `MERN_BE_2`, Type: `t2.micro`, Key: `rahul-mumbai-keypair`
4. Launch in `ap-south-1b` (different AZ for HA)
   - Result: Public IP `13.126.94.179`, Private IP `10.0.4.145`

> ⚠️ Add `13.126.94.179` to the MongoDB Atlas IP whitelist as well.

**Frontend Replica (MERN_FE_2):**
1. Select `MERN_FE` from My AMIs → **Launch**
2. Name: `MERN_FE_2`, launch in `ap-south-1b`
   - Result: Public IP `13.206.185.103`, Private IP `10.0.4.22`

> 📸 **Screenshot:** EC2 Instances list showing all four running instances (MERN_BE, MERN_BE_2, MERN_FE, MERN_FE_2)

---

## Task 4 — Load Balancing with AWS ALB

### 4.1 Create Target Groups

Navigate to **EC2 → Target Groups → Create target group**

**Frontend Target Group:**

| Setting | Value |
|---|---|
| Target type | Instances |
| Target group name | MERN-FE-TG |
| Protocol | HTTP |
| Port | 80 |
| Protocol version | HTTP1 |
| Registered targets | MERN_FE (ap-south-1a), MERN_FE_2 (ap-south-1b) |

**Backend Target Group:**

| Setting | Value |
|---|---|
| Target group name | MERN-BE-TG |
| Protocol | HTTP, Port 80 |
| Registered targets | MERN_BE (ap-south-1a), MERN_BE_2 (ap-south-1b) |

> 📸 **Screenshot:** Target groups list showing MERN-FE-TG and MERN-BE-TG with healthy targets

### 4.2 Create Application Load Balancers

Navigate to **EC2 → Load Balancers → Create load balancer → Application Load Balancer**

**Frontend Load Balancer:**

| Setting | Value |
|---|---|
| Name | MERN-FE-LoadBalancer |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| Availability Zones | ap-south-1a, ap-south-1b |
| Listener | HTTP : 80 |
| Default action | Forward to MERN-FE-TG |
| DNS Name | MERN-FE-LoadBalancer-185256895.ap-south-1.elb.amazonaws.com |

**Backend Load Balancer:**

| Setting | Value |
|---|---|
| Name | MERN-BE-LoadBalancer |
| Listener | HTTP : 80 |
| Default action | Forward to MERN-BE-TG |
| DNS Name | MERN-BE-LoadBalancer-872883685.ap-south-1.elb.amazonaws.com |

Verify both load balancers show **State = Active** in the AWS Console.

> 📸 **Screenshot:** Load Balancers list showing both ALBs in Active state

---

## Task 5 — Domain Setup with Cloudflare

### 5.1 Add CNAME Records in Cloudflare

Log in to [cloudflare.com](https://cloudflare.com) → select your domain → **DNS → Add Record**

| Type | Name | Content (Target) | Proxy Status |
|---|---|---|---|
| CNAME | `@` (root / mern-travelmemory) | `MERN-FE-LoadBalancer-185256895.ap-south-1.elb.amazonaws.com` | Proxied |
| CNAME | `back` | `MERN-BE-LoadBalancer-872883685.ap-south-1.elb.amazonaws.com` | Proxied |

Effect:
- `mern-travelmemory.duckdns.org` → Frontend Load Balancer → Frontend EC2 servers
- `back.mern-travelmemory.duckdns.org` → Backend Load Balancer → Backend EC2 servers

> 📸 **Screenshot:** Cloudflare DNS panel showing both CNAME records

### 5.2 Update `url.js` on Both Frontend Instances

Now that the backend has a domain name, update `src/url.js` on both frontend servers to use it.

**On MERN_FE (Primary):**

```bash
cd /home/ubuntu/TravelMemory/frontend
nano src/url.js
```

```js
export const baseUrl = "http://back.mern-travelmemory.duckdns.org"
```

**On MERN_FE_2 (Replica):**

```bash
cd /home/ubuntu/TravelMemory/frontend
nano src/url.js
```

```js
export const baseUrl = "http://back.mern-travelmemory.duckdns.org"
```

Using the domain name ensures the frontend is decoupled from raw backend IPs — if instances change, only the load balancer needs updating.

> 📸 **Screenshot:** `url.js` open in nano showing the domain-based `baseUrl`

---

## Final Verification

### Access the Application

Open a browser and navigate to: **http://mern-travelmemory.duckdns.org**

Expected result: TravelMemory landing page with the travel entry form (Trip Name, Trip Date, Hotels, Places Visited, etc.)

> 📸 **Screenshot:** TravelMemory application fully loaded via the Cloudflare domain

### End-to-End Data Flow Test

Submit a travel entry through the UI and verify it appears in MongoDB Atlas:

1. Fill in the form and click **Submit**
2. Log in to [cloud.mongodb.com](https://cloud.mongodb.com)
3. Navigate to: **cluster → Collections → travelmemory → tripdetails**
4. Confirm the document was created:

```json
{
  "tripName": "Rahul's Goa Trip",
  "startDateOfJourney": "2024-01-01",
  "endDateOfJourney": "2024-01-05",
  "nameOfHotels": "Taj Resort",
  "placesVisited": "Baga Beach, Dudhsagar",
  "totalCost": 25000,
  "tripType": "leisure",
  "featured": true
}
```

> 📸 **Screenshot:** MongoDB Atlas Collections view showing the submitted trip document

---

## Ports Reference

| Layer | Technology | Internal Port | Exposed Via |
|---|---|---|---|
| Frontend | React (npm start) | 3000 | Nginx → port 80 |
| Backend | Node.js (Express) | 3000 | Nginx → port 80 |
| Database | MongoDB Atlas | 27017 (internal) | MongoDB URI over internet |
| Load Balancer | AWS ALB | 80 | Public internet |
| DNS / Proxy | Cloudflare | 80/443 | mern-travelmemory.duckdns.org |

---

## Best Practices

### Security
- Database credentials stored in `.env` files — not hardcoded or committed to Git
- MongoDB Atlas IP whitelisting restricts database access to EC2 instances only
- SSH access secured via key pair — no password-based login
- Security groups restrict inbound traffic to necessary ports only
- Cloudflare proxying adds DDoS protection and hides origin server IPs

### Scalability
- AMI-based replication enables identical instances to spin up in minutes
- Application Load Balancers automatically distribute traffic to all healthy targets
- Instances deployed across multiple Availability Zones (`ap-south-1a` and `ap-south-1b`)
- Frontend and backend scale independently — each tier can be replicated without affecting the other
- MongoDB Atlas scales independently of EC2

### Resilience & High Availability
- Multi-AZ deployment prevents a single AZ failure from taking down the app
- ALB health checks automatically reroute traffic away from unhealthy instances
- Replica instances serve live traffic — they are not cold standbys
- Using a domain name in `url.js` decouples the frontend from raw backend IPs

---

## Repository

**GitHub:** https://github.com/UnpredictablePrashant/TravelMemory
