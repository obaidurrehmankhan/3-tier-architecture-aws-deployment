# ☁️ AWS 3-Tier Architecture – React + Node.js + MySQL on VPC, ALB, ASG, RDS, CloudFront

This project is a **full 3-tier web application** deployed on AWS:

- **Presentation Tier (Frontend)**  
  React app served via **NGINX** on EC2 instances behind an **internet-facing ALB**, fronted by **CloudFront** and a custom domain.

- **Application Tier (Backend)**  
  Node.js API running in private subnets, fronted by an **internal Application Load Balancer** and managed by an **Auto Scaling Group (ASG)**.

- **Data Tier**  
  **RDS MySQL** in private subnets, accessed securely only by the backend and a **bastion host**.

Infra is built around:

- A dedicated **VPC** with multiple subnets across **2 Availability Zones**
- Carefully designed **Security Groups** that enforce strict “who-can-talk-to-whom”
- **Launch Templates** (with versions) and **ASGs** for repeatable, auto-scaling compute
- **CloudWatch Logs** for backend log centralization
- **Route 53 + ACM + CloudFront** for a proper production-style domain + HTTPS + CDN.

This README documents **all steps from 0 → 6** in order, showing **how each part depends on previous steps**.

---

## 🔎 1. High-Level Architecture

**Layers:**

```text
User
  ↓
Route 53 (DNS)
  ↓
CloudFront (CDN, HTTPS)
  ↓
Presentation ALB (Internet-facing)
  ↓
Presentation EC2 (NGINX + React build)
  ↓ /api
Internal Application ALB (Private)
  ↓
Application EC2 (Node.js + PM2)
  ↓
RDS MySQL (Private DB in Data Tier)
````

**Ops / Admin paths:**

```text
Developer Laptop
  ↓ (SSH)
Bastion Host (Public EC2)
  ↓ (SSH / MySQL)
Private EC2 (App / Presentation) & RDS (via tunnel)
```

---

## 🧱 2. Step 0 – Network Foundation (VPC, Subnets, NAT, DNS)

### 2.1 Create Dedicated VPC

Create a **new VPC**:

* **Name:** `3-tier-architecture`
* **IPv4 CIDR:** `10.0.0.0/16`
* **Tenancy:** `Default`

This VPC is your **isolated network** for all 3 tiers.

---

### 2.2 Availability Zones & Subnets

Use **2 Availability Zones** for HA.

Within the VPC, create:

* **2 Public Subnets** (one in each AZ)
  Used for:

  * `bastion` EC2
  * internet-facing **Presentation ALB**
* **4 Private Subnets** (two per AZ)
  Split logically into:

  * **Application Tier private subnets** (backend EC2 + internal ALB)
  * **Data Tier private subnets** (RDS)

Example layout (conceptual):

```text
AZ A:
  - public-subnet-a          (bastion, public ALB)
  - app-private-subnet-a     (application EC2 + internal ALB)
  - db-private-subnet-a      (RDS)

AZ B:
  - public-subnet-b
  - app-private-subnet-b
  - db-private-subnet-b
```

---

### 2.3 NAT Gateway

Create a **NAT Gateway** in one public subnet, and route **private subnets**’ default traffic (`0.0.0.0/0`) through it.

Purpose:

* Private EC2 (backend/presentation) can **download packages** (yum, npm, git)
* They **cannot** be directly reached from the internet.

---

### 2.4 DNS Options in VPC

Enable:

* **DNS resolution**
* **DNS hostnames**

This allows:

* Internal DNS names (e.g. RDS endpoint, EC2 internal names) to resolve correctly.
* Some services (like RDS) to function properly.

---

## 🔐 3. Step 1 – Security Groups & Bastion Host

Security Groups = AWS **firewalls** attached to resources.

All SGs below must use the **same VPC**: `3-tier-architecture`.

---

### 3.1 Bastion Host SG – `bastion-host`

* **Inbound Rules:**

  * SSH (TCP 22) from `0.0.0.0/0` (for demo; restrict to your IP in real world)
* **Outbound Rules:**

  * Default: All traffic (`0.0.0.0/0`)

Used by: **bastion host EC2**
Purpose: allow you to SSH into one public instance, then hop into private instances / RDS.

---

### 3.2 Presentation Tier ALB SG – `presentation-tier-alb`

* **Inbound:**

  * HTTP (80) from `0.0.0.0/0`
* **Outbound:**

  * Default: All

Attached to: **internet-facing ALB** for the frontend.

---

### 3.3 Presentation Tier EC2 SG – `presentation-tier-ec2`

* **Inbound:**

  * SSH (22) from **`bastion-host` SG**
  * HTTP (80) from **`presentation-tier-alb` SG**
* **Outbound:**

  * Default: All

Attached to: **frontend EC2 instances** (NGINX + React build)

This forces:

* All user HTTP traffic via ALB → EC2
* All SSH via bastion → EC2

---

### 3.4 Application Tier ALB SG – `application-tier-alb`

* **Inbound:**

  * HTTP (80 or 3200) from **`presentation-tier-ec2` SG**
* **Outbound:**

  * Default: All

Attached to: **internal (private) Application ALB**

Only **frontend instances** can hit this ALB.

---

### 3.5 Application Tier EC2 SG – `application-tier-ec2`

* **Inbound:**

  * SSH (22) from **`bastion-host` SG**
  * Custom TCP (3200) from **`application-tier-alb` SG**
* **Outbound:**

  * Default: All

Attached to: **backend EC2 instances** (Node.js)

---

### 3.6 Data Tier SG – `data-tier`

* **Inbound:**

  * MySQL/Aurora (3306) from **`application-tier-ec2` SG**
  * MySQL/Aurora (3306) from **`bastion-host` SG**
* **Outbound:**

  * Default: All

Attached to: **RDS MySQL instance**

This ensures:

* Only backend EC2 and bastion can access DB.
* No public or frontend direct DB access.

---

### 3.7 Launch Bastion Host EC2

Create an EC2 instance:

* **Name:** `bastion-host`
* **AMI:** Amazon Linux
* **Subnet:** public subnet in `3-tier-architecture`
* **SG:** `bastion-host`
* **Key Pair:** your SSH key (e.g. `3-tier-architecture`)

You now have:

* One public EC2 acting as **gateway** into private network for SSH and DB tunneling.

---

## 🗄️ 4. Step 2 – Data Tier (RDS) – Subnet Group + Instance

### 4.1 DB Subnet Group

In RDS console:

* Create **DB Subnet Group**:

  * **VPC:** `3-tier-architecture`
  * Include **two private DB subnets** (one per AZ)

RDS will only place DBs into subnets that are in this group.

---

### 4.2 RDS MySQL Instance

Create new RDS instance:

* **Engine:** MySQL
* **Template:** Dev/Test
* **Public access:** `No`
* **VPC:** `3-tier-architecture`
* **DB Subnet Group:** the one you created above
* **Security Group:** `data-tier`

You set:

* **Master username/password**
* **DB identifier** (e.g. `dev-db-instance`)

Result:

* RDS MySQL is in **private subnets**, reachable only by:

  * Backend EC2 SG (`application-tier-ec2`)
  * Bastion SG (`bastion-host`)

---

## 🧩 5. Step 3 – DB Initialization Over SSH Tunnel

### 5.1 SSH Tunnel via Bastion

Because RDS is in private subnets, you connect via **bastion host**:

Conceptual SSH command (actual in your own scripts):

```bash
ssh -i 3-tier-architecture.pem \
    -L 3307:<rds-endpoint>:3306 \
    ec2-user@<bastion-public-ip>
```

Then in MySQL Workbench or CLI:

* Host: `127.0.0.1`
* Port: `3307`
* User: RDS master user
* Password: RDS master password

---

### 5.2 Create Application Database & User

Once connected:

```sql
CREATE DATABASE IF NOT EXISTS react_node_app
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'appuser'@'%' IDENTIFIED BY 'learnIT02#';

GRANT ALL PRIVILEGES ON react_node_app.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

Now the backend will use:

```env
DB_HOST=<your-rds-endpoint>
DB_PORT=3306
DB_USER=appuser
DB_PASSWORD=learnIT02#
DB_NAME=react_node_app
```

---

### 5.3 Load Application Data

Still via tunnel, in Workbench or CLI:

```sql
USE react_node_app;
SOURCE /path/to/your/schema-and-seed.sql;
```

Now DB is fully ready for the Node.js backend in later steps.

---

## ⚙️ 6. Step 4 – Application Tier (Backend) – LT v1, Internal ALB, ASG

### 6.1 Launch Template – `application-tier-lt` (v1)

Go to **EC2 → Launch Templates → Create launch template**.

* **Name:** `application-tier-lt`
* **AMI:** Amazon Linux
* **Instance type:** `t3.micro` (example)
* **Security Group:** `application-tier-ec2`
* **Network settings:** leave subnet to be chosen by ASG
* **User data:** backend bootstrap script below

```bash
#!/bin/bash
set -e

# 1) Update OS and install tools
yum update -y
yum install -y git

# 2) Install Node.js 18
curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
yum install -y nodejs

# 3) Install PM2 globally
npm install -g pm2

# 4) Define paths and repo
REPO_URL="https://github.com/learnItRightWay01/react-node-mysql-app.git"
BRANCH_NAME="feature/add-logging"

APP_DIR="/home/ec2-user/react-node-mysql-app"
BACKEND_DIR="$APP_DIR/backend"
ENV_FILE="$BACKEND_DIR/.env"
LOG_DIR="$BACKEND_DIR/logs"

# 5) Clone backend repo as ec2-user
cd /home/ec2-user
sudo -u ec2-user git clone "$REPO_URL"
cd "$APP_DIR"
sudo -u ec2-user git checkout "$BRANCH_NAME"

# 6) Create log directory and fix ownership
mkdir -p "$LOG_DIR"
chown -R ec2-user:ec2-user "$LOG_DIR"

# 7) Create .env file for backend (RDS credentials + log directory)
cat << EOF > "$ENV_FILE"
LOG_DIR=$LOG_DIR
DB_HOST=<your-rds-endpoint>
DB_PORT=3306
DB_USER=appuser
DB_PASSWORD=learnIT02#
DB_NAME=react_node_app
EOF

chown ec2-user:ec2-user "$ENV_FILE"

# 8) Install backend dependencies
cd "$BACKEND_DIR"
sudo -u ec2-user npm install

# 9) Start backend via npm (PM2 under the hood)
sudo -u ec2-user npm run serve

# 10) Persist PM2 process list and enable startup on reboot
sudo -u ec2-user pm2 save
sudo -u ec2-user pm2 startup systemd
```

Every new backend instance:

* Knows RDS endpoint/creds
* Runs Node.js app on port **3200**
* Uses PM2 for process management

---

### 6.2 Application Tier Target Group – `application-tier-tg`

Create a **Target Group**:

* **Name:** `application-tier-tg`
* **Target type:** Instances
* **Protocol:** HTTP
* **Port:** 3200
* **VPC:** `3-tier-architecture`
* **Health check path:** `/health`

This is where internal ALB will send traffic.

---

### 6.3 Internal Application ALB – `application-tier-alb`

Create an **Application Load Balancer**:

* **Name:** `application-tier-alb`
* **Scheme:** Internal
* **VPC:** `3-tier-architecture`
* **Subnets:** private app subnets (2 AZs)
* **Security Group:** `application-tier-alb`
  (Allows inbound HTTP from `presentation-tier-ec2` SG)

**Listener:**

* HTTP : 3200 (or 80 forwarded to target 3200)
* Forward → `application-tier-tg`

Now:

* Only Presentation EC2 instances can talk to internal ALB.
* Internal ALB chooses healthy backend instances in `application-tier-tg`.

---

### 6.4 Application Tier ASG – `application-tier-autoscaling-group`

Create an **Auto Scaling Group**:

* **Name:** `application-tier-autoscaling-group`
* **Launch Template:** `application-tier-lt` (v1)
* **VPC:** `3-tier-architecture`
* **Subnets:** private app subnets
* **Load Balancer:** attach `application-tier-alb` + `application-tier-tg`
* **Health checks:**

  * Type: `ELB`
  * Grace period: ~300s
* **Capacity:**

  * Desired: 3
  * Min: 1–2
  * Max: 3 (or more later)
* **Scaling policy:**

  * Target tracking: Average CPU ~50%

The backend is now:

* Private
* Auto-healing
* Auto-scaling
* Connected to RDS using `appuser`.

---

## 🖥️ 7. Step 5 – Presentation Tier (Frontend) – v1 & v2

### 7.1 Presentation Tier Launch Template – `presentation-tier-lt` (v1 – Metadata Demo)

First, we created **v1** to verify infra:

* **Name:** `presentation-tier-lt`
* **AMI:** Amazon Linux
* **Instance type:** `t3.micro`
* **Security Group:** `presentation-tier-ec2`
* **User data:** installs NGINX and writes instance metadata (Instance ID, AZ, Public IP) into `index.html` using IMDSv2.

This let us verify:

* ALB → EC2 routing
* Health checks
* ASG scale-out/scale-in

---

### 7.2 Presentation Tier Target Group – `presentation-tier-tg`

Create a Target Group:

* **Name:** `presentation-tier-tg`
* **Type:** Instances
* **Protocol:** HTTP
* **Port:** 80
* **VPC:** `3-tier-architecture`
* **Health check path:** `/health` (or `/` for initial demo)

---

### 7.3 Internet-Facing Presentation ALB – `presentation-tier-alb`

Create ALB:

* **Name:** `presentation-tier-alb`
* **Scheme:** Internet-facing
* **VPC:** `3-tier-architecture`
* **Subnets:** public subnets (2 AZs)
* **SG:** `presentation-tier-alb` (HTTP from `0.0.0.0/0`)

**Listener:**

* HTTP:80 → Forward → `presentation-tier-tg`

---

### 7.4 Presentation Tier ASG – `presentation-tier-asg`

Create ASG:

* **Name:** `presentation-tier-asg`
* **Launch Template:** `presentation-tier-lt` (v1)
* **VPC:** `3-tier-architecture`
* **Subnets:** public subnets
* **Load Balancer:** attach `presentation-tier-alb` + `presentation-tier-tg`
* **Health checks:** ELB
* **Capacity:** e.g. Desired=3, Min=2, Max=4
* **Scaling policy:** CPU ~50%

At this point, hitting the ALB DNS showed the **metadata HTML**.

---

### 7.5 Upgrade Presentation Tier to v2 – React Build + Reverse Proxy

Next, we improved the frontend by creating **Launch Template v2**:

* **Same base settings** (AMI, type, SG)
* **New user data**:

  * Installs Node.js + git + NGINX
  * Clones repo (`react-node-mysql-app`), branch `feature/add-logging`
  * Builds **React frontend** with Vite
  * Writes `.env` file:

    ```bash
    VITE_API_URL="/api"
    ```
  * Copies `dist` into `/usr/share/nginx/html/`
  * Writes NGINX config to:

    * Serve React SPA
    * Provide `/health`
    * Proxy `/api` → **internal Application ALB endpoint**

**Key NGINX server block** (core idea):

```nginx
server {
    listen 80;
    server_name <your-domain-or-subdomain>;
    root /usr/share/nginx/html/dist;
    index index.html index.htm;

    location /health {
        default_type text/html;
        return 200 "<!DOCTYPE html><p>Health check endpoint</p>\n";
    }

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://<internal-application-tier-alb-endpoint>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Why this matters:**

* Browser calls `/api/...` → NGINX reverse proxies to backend ALB.
* No hardcoded backend URL inside the frontend build.
* Keeps CORS simple (same origin).

---

### 7.6 Point Presentation ASG to Launch Template v2

To apply the new version:

1. Go to `presentation-tier-asg`
2. Edit ASG and select **launch template version = 2**
3. Set desired/min/max as needed (e.g. 2)
4. Terminate old instances → ASG creates new ones using v2.

Now:

* ALB serves **React app**
* `/api` calls go to backend via **internal ALB**

---

## 📊 8. Step 6 – Backend Logging with CloudWatch (LT v2)

### 8.1 IAM Role for Backend EC2

Create IAM Role:

* Trusted entity: **EC2**
* Policies:

  * `CloudWatchLogsFullAccess`
  * `CloudWatchAgentServerPolicy`
* Name: e.g. `application-tier-ec2-logging-role`

---

### 8.2 CloudWatch Log Group

Create log group:

* Name: `backend-node-app-logs`

This is where backend logs will go.

---

### 8.3 Update `application-tier-lt` to Version 2

Create **Launch Template version 2** with:

* IAM instance profile: `application-tier-ec2-logging-role`
* User data: same as v1 **plus** CloudWatch Agent install and config:

```bash
# Install CloudWatch agent
sudo yum install -y amazon-cloudwatch-agent

# Create CloudWatch agent configuration
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<EOL
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ec2-user/react-node-mysql-app/backend/logs/*.log",
            "log_group_name": "backend-node-app-logs",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y-%m-%d %H:%M:%S"
          }
        ]
      }
    }
  }
}
EOL

# Start CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

---

### 8.4 Update Application ASG to Use LT v2

* Edit `application-tier-autoscaling-group`
* Select `application-tier-lt` **version 2**
* Terminate existing backend instances → ASG recreates them using v2

Now backend logs from:

```bash
/home/ec2-user/react-node-mysql-app/backend/logs/*.log
```

are shipped to **CloudWatch Log Group** `backend-node-app-logs`
with one stream per instance (by `instance_id`).

---

## 🌍 9. Step 7 – CloudFront, ACM, Route 53 (Domain + HTTPS + CDN)

### 9.1 ACM – SSL Certificate

In **AWS Certificate Manager (ACM)**:

* Request a public certificate for:

  * `example.com`
  * `*.example.com` (or specific subdomain)
* Use **DNS validation** via Route 53.
* ACM will create CNAMEs in your hosted zone for validation.
* Once validated, certificate becomes **Issued**.

---

### 9.2 CloudFront Distribution

Create **CloudFront distribution**:

* **Origin:** `presentation-tier-alb` DNS
* **Viewer protocol policy:** `Redirect HTTP to HTTPS`
* **WAF:** Disabled (for this demo)
* **Alternate domain names (CNAMEs):**

  * `example.com`
  * `www.example.com` or `app.example.com`
* **SSL Certificate:** select the ACM cert (for your domain)

CloudFront now:

* Terminates HTTPS
* Uses your custom domain
* Forwards requests to Presentation ALB (HTTP as origin).

---

### 9.3 Route 53 – DNS Records

In your hosted zone:

1. **Root domain → CloudFront**

   * Record type: **A**
   * Name: `example.com`
   * Alias: `Yes`
   * Target: CloudFront distribution

2. **Subdomain → CloudFront**

   * Record type: **A**
   * Name: `www.example.com` or `app.example.com`
   * Alias: `Yes`
   * Target: same CloudFront distribution

Now:

```text
User → example.com (DNS: Route 53) → CloudFront (HTTPS)
     → Presentation ALB (HTTP)
     → Presentation EC2 (React + NGINX)
     → /api → Internal ALB → Backend EC2
     → RDS
     → Logs → CloudWatch
```

---

## ✅ 10. What You Have at the End

This project demonstrates:

* **Full 3-tier AWS architecture** with:

  * Strict network segmentation
  * Proper Security Groups per layer
* **Launch Templates with versioning**:

  * v1 for initial bootstraps
  * v2 for enhancements (React build, logging)
* **Auto Scaling Groups** for:

  * Frontend
  * Backend
* **Two ALBs**:

  * Internet-facing for frontend
  * Internal for backend
* **Managed RDS MySQL**:

  * Private, Multi-AZ capable
  * Proper DB user (`appuser`)
* **CloudWatch logging** for backend
* **CloudFront + Route 53 + ACM** for:

  * HTTPS
  * CDN
  * Custom domain

This is a **portfolio-grade project** you can show to recruiters as:

> “End-to-end 3-tier AWS deployment with React, Node.js, RDS, ALB, ASG, CloudFront, and centralized logging.”

You can easily extend it with:

* WAF
* CI/CD pipelines (GitHub Actions)
* Infrastructure as Code (CloudFormation / Terraform)
* Multi-env support (dev / stage / prod)




