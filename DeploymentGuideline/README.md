# 🚀 3-Tier Web Application — AWS Deployment Guide

> **Project:** BMI Health Tracker
> **Repository:** https://github.com/md-sarowar-alam/single-server-3tier-webapp
> **Stack:** React (Vite) + Node.js + PostgreSQL + Nginx + PM2
> **Infrastructure:** 3 Separate EC2 Instances on AWS

---

## 🏗️ Architecture Overview

```
                     ┌──────────────────────────────┐
                     │           INTERNET            │
                     └──────────────┬───────────────┘
                                    │ HTTP (Port 80)
                     ┌──────────────▼───────────────┐
                     │       FRONTEND SERVER         │
                     │    EC2 — Ubuntu 22.04         │
                     │    Nginx + React (Vite)       │
                     │    Publicly Accessible        │
                     └──────────────┬───────────────┘
                                    │ Reverse Proxy /api → Port 3000
                     ┌──────────────▼───────────────┐
                     │       BACKEND SERVER          │
                     │    EC2 — Ubuntu 22.04         │
                     │    Node.js + PM2              │
                     │    Private IP Only            │
                     └──────────────┬───────────────┘
                                    │ PostgreSQL → Port 5432
                     ┌──────────────▼───────────────┐
                     │       DATABASE SERVER         │
                     │    EC2 — Ubuntu 22.04         │
                     │    PostgreSQL 16              │
                     │    Private IP Only            │
                     └──────────────────────────────┘
```

---

## 📋 Prerequisites

- An AWS Account with EC2 access
- A Key Pair `.pem` file (create once, reuse for all 3 servers)
- Basic Linux/terminal knowledge

---

## 🖥️ PHASE 1 — AWS Infrastructure Setup

### Step 1: Launch 3 EC2 Instances

Go to **AWS Console → EC2 → Launch Instance** and repeat 3 times:

| # | Name | AMI | Instance Type |
|---|---|---|---|
| 1 | `frontend-server` | Ubuntu 22.04 LTS | t2.micro |
| 2 | `backend-server` | Ubuntu 22.04 LTS | t2.micro |
| 3 | `database-server` | Ubuntu 22.04 LTS | t2.micro |

> ⚠️ **Important:** Launch all three in the **same VPC** and with the **same Key Pair**.

---

### Step 2: Create 3 Separate Security Groups

#### 🔵 `sg-frontend` — Frontend Security Group

| Type | Protocol | Port | Source | Why? |
|---|---|---|---|---|
| SSH | TCP | 22 | 0.0.0.0/0 | Remote terminal access |
| HTTP | TCP | 80 | 0.0.0.0/0 | Allow browser traffic |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Secure browser traffic |

#### 🟡 `sg-backend` — Backend Security Group

| Type | Protocol | Port | Source | Why? |
|---|---|---|---|---|
| SSH | TCP | 22 | 0.0.0.0/0 | Remote terminal access |
| Custom TCP | TCP | 3000 | `sg-frontend` (SG ID) | Only Frontend can reach the API |

> 💡 Use the **Security Group ID** as the source — not an IP address.
> This way, even if the Frontend server's IP changes, it will always have access.

#### 🔴 `sg-database` — Database Security Group

| Type | Protocol | Port | Source | Why? |
|---|---|---|---|---|
| SSH | TCP | 22 | 0.0.0.0/0 | Remote terminal access |
| Custom TCP | TCP | 5432 | `sg-backend` (SG ID) | Only Backend can connect to DB |

---

### Step 3: Note Down IP Addresses

Go to **EC2 → Instances** and click each instance to find its IPs:

```
frontend-server  → Public IP: _______________  Private IP: _______________
backend-server   → Public IP: _______________  Private IP: _______________  ← needed
database-server  → Public IP: _______________  Private IP: _______________  ← needed
```

> ⚠️ The **Private IPs** of Backend and Database servers are critical.
> Servers communicate with each other using Private IPs within the VPC.

---

## 🗄️ PHASE 2 — Database Server Setup

### SSH into the Database Server

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<DATABASE-PUBLIC-IP>
```

---

### Step 4: Update the System

```bash
sudo apt update && sudo apt upgrade -y
# Updates all installed packages — gets latest security patches and bug fixes
```

---

### Step 5: Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
# postgresql        → the main database engine
# postgresql-contrib → extra utilities and extensions
```

```bash
sudo systemctl start postgresql@16-main
# Use postgresql@16-main — this is the REAL service
# "postgresql" alone is just a wrapper and does NOT actually start the database cluster!

sudo systemctl enable postgresql@16-main
# Auto-starts PostgreSQL after every system reboot
```

```bash
# Verify — port 5432 should appear
sudo ss -tlnp | grep 5432
# Expected: LISTEN 127.0.0.1:5432
```

---

### Step 6: Create Database and User

```bash
sudo -u postgres psql
# Switch to the postgres superuser and open the psql shell
# All admin-level database operations must be done through this user
```

Run these commands inside the psql prompt:

```sql
-- Create a dedicated app user
CREATE USER bmi_user WITH PASSWORD 'strongpassword';
-- Why a separate user? Never run apps as a superuser.
-- A dedicated user only accesses what it needs — principle of least privilege.

-- Create the application database
CREATE DATABASE bmidb OWNER bmi_user;

-- Grant full access to the user
GRANT ALL PRIVILEGES ON DATABASE bmidb TO bmi_user;

-- Verify — bmidb should appear in the list
\l

-- Exit psql
\q
```

---

### Step 7: Enable Remote Connections

This is the biggest difference from a single-server setup.
By default, PostgreSQL only accepts connections from localhost.
Since the Backend lives on a different server, remote access must be enabled.

#### postgresql.conf — Tell the server which interfaces to listen on

```bash
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/16/main/postgresql.conf
# The '#' means the line was commented out (disabled)
# '*' means listen on ALL network interfaces
# Before: only 127.0.0.1:5432 (localhost only)
# After:  0.0.0.0:5432 (all interfaces — remote connections allowed)
```

```bash
# Verify — the line should have no leading '#'
sudo grep listen_addresses /etc/postgresql/16/main/postgresql.conf
# Expected output: listen_addresses = '*'
```

#### pg_hba.conf — Define who is allowed to connect

```bash
echo "host    bmidb    bmi_user    10.0.0.0/16    md5" | sudo tee -a /etc/postgresql/16/main/pg_hba.conf
# host         → TCP/IP connection type
# bmidb        → only this database
# bmi_user     → only this user
# 10.0.0.0/16  → entire VPC IP range (all servers in the same VPC)
# md5          → authenticate using a password
```

> ⚠️ **Critical Mistake to Avoid:**
> Never write pg_hba rules inside `postgresql.conf`. They are two completely different files:
> - `postgresql.conf` → Server configuration (port, memory, logging, listen address)
> - `pg_hba.conf` → Access control (who can connect, from where, using what method)

```bash
# Verify the rule was added correctly
sudo tail -3 /etc/postgresql/16/main/pg_hba.conf
```

---

### Step 8: Restart PostgreSQL

```bash
sudo systemctl restart postgresql@16-main
# Applies all configuration changes

# Verify — should now show 0.0.0.0:5432 instead of 127.0.0.1:5432
sudo ss -tlnp | grep 5432
# Expected: LISTEN 0.0.0.0:5432  ← Remote connections are ready!
```

---

### Step 9: Run Database Migrations

```bash
sudo apt install -y git

git clone https://github.com/md-sarowar-alam/single-server-3tier-webapp.git

cd single-server-3tier-webapp/backend
# Must be in the backend/ folder — NOT inside migrations/
# Running from inside migrations/ would double the path and cause a "file not found" error
```

```bash
cat migrations/*.sql | psql -U bmi_user -d bmidb -h localhost -W
# cat migrations/*.sql → reads all SQL migration files in order
# psql -U bmi_user    → connects as bmi_user
# -d bmidb            → targets the bmidb database
# -h localhost        → connects on this server (the DB server itself)
# -W                  → prompts for password
# Password: strongpassword
```

Expected output:
```
CREATE TABLE
CREATE INDEX
Migration 001 completed successfully ✅
Migration 002 completed successfully ✅
```

---

### Step 10: Test Remote Connection

```bash
psql -U bmi_user -d bmidb -h <DATABASE-PRIVATE-IP> -W
# Tests connectivity using the server's own Private IP
# If this succeeds, the Backend server will also be able to connect
# Type \q to exit
```

**✅ Database Server — Complete!**

---

## ⚙️ PHASE 3 — Backend Server Setup

### Open a New Terminal and SSH into the Backend Server

```bash
ssh -i your-key.pem ubuntu@<BACKEND-PUBLIC-IP>
```

---

### Step 11: Update System and Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
```

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs git
# Node.js 18 LTS — stable, long-term supported production version
# npm is included automatically with nodejs

node -v   # v18.x.x
npm -v    # 9.x.x
```

```bash
sudo npm install -g pm2
# PM2 = Process Manager 2
# Why PM2?
#   Without it → closing the terminal kills the Node.js app
#   With PM2   → app runs in the background, auto-restarts on crash
# -g means globally installed — pm2 command works from any directory
```

---

### Step 12: Clone Repository and Install Packages

```bash
git clone https://github.com/md-sarowar-alam/single-server-3tier-webapp.git
cd single-server-3tier-webapp/backend

npm install
# Installs all dependencies listed in package.json
# Creates the node_modules/ folder

mkdir -p logs
# PM2 writes log files to logs/err.log and logs/out.log
# These paths are defined in ecosystem.config.js
# The folder must exist before starting PM2, or it may throw an error
```

---

### Step 13: Fix the Wrong Path in ecosystem.config.js

```bash
cat ecosystem.config.js
# You will see: cwd: '/home/ubuntu/bmi-health-tracker/backend'
# This is the OLD/WRONG repository name — it must be fixed!
```

```bash
sed -i "s|/home/ubuntu/bmi-health-tracker/backend|/home/ubuntu/single-server-3tier-webapp/backend|g" ecosystem.config.js
# Replaces the old repo name with the correct folder name
# Without this fix, PM2 will throw: "Error: Script not found"

# Verify
grep cwd ecosystem.config.js
# Expected: cwd: '/home/ubuntu/single-server-3tier-webapp/backend' ✅
```

---

### Step 14: Create Environment Variables File

```bash
nano .env
```

Paste the following content:

```env
PORT=3000
DATABASE_URL=postgresql://bmi_user:strongpassword@<DATABASE-PRIVATE-IP>:5432/bmidb
NODE_ENV=production
FRONTEND_URL=http://<FRONTEND-PUBLIC-IP>
```

> 💡 **Why the Database Private IP?**
> Both Backend and Database are in the same VPC.
> Private IP communication is faster, completely free within AWS, and more secure than using a Public IP.

```bash
# Save and exit: Ctrl+X → Y → Enter

# Verify
cat .env
```

---

### Step 15: Start Backend with PM2

```bash
pm2 start ecosystem.config.js
# Uses the ecosystem config which defines the app name, script path,
# environment variables, log file locations, and restart behavior
```

```bash
pm2 status
# ┌──────────────┬─────────┬────────┐
# │ bmi-backend  │ cluster │ online │  ← must show "online" ✅
# └──────────────┴─────────┴────────┘
```

```bash
pm2 logs bmi-backend --lines 20
# View real-time logs to confirm the app started correctly
# You should see:
# ✅ Database connected successfully at: 2026-...
# 🚀 Server running on port 3000
```

> ❌ If you see `Database connection failed: ECONNREFUSED`:
> 1. Verify PostgreSQL is running on the Database Server
> 2. Check that Port 5432 is allowed from `sg-backend` in the Database Security Group
> 3. Confirm the `DATABASE_URL` in `.env` has the correct Private IP

---

### Step 16: Configure PM2 to Auto-start on Reboot

```bash
pm2 startup
# Generates a sudo command — copy it and run it exactly as shown
# Example output:
# sudo env PATH=$PATH:/usr/bin /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

```bash
# After running the generated command above:
pm2 save
# Saves the current process list to disk
# PM2 will automatically restore bmi-backend after any system reboot
```

---

### Step 17: Test the Backend Locally

```bash
curl http://localhost:3000
# Should return a JSON response from the backend ✅
```

**✅ Backend Server — Complete!**

---

## 🌐 PHASE 4 — Frontend Server Setup

### Open a New Terminal and SSH into the Frontend Server

```bash
ssh -i your-key.pem ubuntu@<FRONTEND-PUBLIC-IP>
```

---

### Step 18: Update System and Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
```

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs git nginx
# nodejs → needed only to build the frontend (not required after the build)
# nginx  → serves the built static files and proxies API requests to the Backend
```

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
# start  → run Nginx immediately
# enable → auto-start Nginx after every system reboot

sudo systemctl status nginx
# Expected: Active: active (running) ✅
```

---

### Step 19: Clone Repository and Build Frontend

```bash
git clone https://github.com/md-sarowar-alam/single-server-3tier-webapp.git
cd single-server-3tier-webapp/frontend

npm install
# Installs React, Vite, and all frontend dependencies
```

```bash
npm run build
# Vite compiles and optimizes the React app for production
# Output directory: dist/
# Important: React (CRA) outputs to build/ — Vite outputs to dist/ — they are DIFFERENT!
# All JS and CSS files are minified and bundled for best performance
```

```bash
ls dist/
# Should show: assets/  index.html ✅
# This dist/ folder is exactly what Nginx will serve to the browser
```

---

### Step 20: Configure Nginx as a Reverse Proxy

```bash
sudo rm /etc/nginx/sites-enabled/default
# Removes the default "Welcome to Nginx" page
# Without this, the default page appears instead of your app
```

```bash
sudo nano /etc/nginx/sites-available/webapp
```

Paste the following complete configuration:

```nginx
server {
    listen 80;
    server_name _;
    # listen 80    → accept HTTP connections on port 80
    # server_name _ → respond to any IP address or domain name

    # ✅ TIER 1: Frontend — serve static files from the Vite build output
    location / {
        root /home/ubuntu/single-server-3tier-webapp/frontend/dist;
        index index.html;
        try_files $uri $uri/ /index.html;
        # root         → points to Vite's dist/ output folder
        # try_files    → required for React Router (Single Page App support)
        #               Without this, refreshing /about returns 404
        #               With this, Nginx always falls back to index.html
    }

    # ✅ TIER 2: Backend API — forward all /api requests to the Backend server
    location /api {
        proxy_pass http://<BACKEND-PRIVATE-IP>:3000;
        # Single Server:  proxy_pass http://localhost:3000
        # Multi  Server:  proxy_pass http://<BACKEND-PRIVATE-IP>:3000  ← KEY DIFFERENCE

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        # Required for WebSocket support

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # Forwards the real client IP to the Backend
        # Without this, all requests appear to come from Nginx's IP

        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
# Enable the site by creating a symbolic link
sudo ln -s /etc/nginx/sites-available/webapp /etc/nginx/sites-enabled/
# sites-available → where all config files are stored
# sites-enabled   → what Nginx actually reads (linked via symlink)
```

```bash
# Fix file permissions so Nginx can read the dist/ folder
sudo chmod 755 /home/ubuntu
sudo chmod -R 755 /home/ubuntu/single-server-3tier-webapp/frontend/dist
# Nginx runs as the www-data user
# It needs read access to the home directory and the dist/ folder
# Without this, Nginx returns 403 Forbidden
```

```bash
# Test Nginx configuration for syntax errors
sudo nginx -t
# Expected:
# nginx: the configuration file syntax is ok
# nginx: configuration file test is successful ✅
```

```bash
# Apply the new configuration
sudo systemctl reload nginx
# reload  → applies new config without dropping existing connections (preferred)
# restart → briefly interrupts all connections (avoid in production)
```

**✅ Frontend Server — Complete!**

---

## ✅ PHASE 5 — Final Verification

### Check the Status of All Three Servers

**On Database Server:**
```bash
sudo systemctl status postgresql@16-main --no-pager | grep Active
sudo ss -tlnp | grep 5432
# Active: active (running)  ✅
# LISTEN 0.0.0.0:5432       ✅  (remote connections enabled)
```

**On Backend Server:**
```bash
pm2 status
pm2 logs bmi-backend --lines 5
sudo ss -tlnp | grep 3000
# bmi-backend  online                    ✅
# ✅ Database connected successfully     ✅
# LISTEN *:3000                          ✅
```

**On Frontend Server:**
```bash
sudo systemctl status nginx --no-pager | grep Active
sudo ss -tlnp | grep 80
# Active: active (running)  ✅
# LISTEN 0.0.0.0:80         ✅
```

---

### End-to-End Connection Test

```bash
# From the Frontend Server — verify the Backend is reachable
curl http://<BACKEND-PRIVATE-IP>:3000
# Should return a JSON response ✅
```

```
# Open in browser:
http://<FRONTEND-PUBLIC-IP>        → BMI Health Tracker UI ✅
http://<FRONTEND-PUBLIC-IP>/api    → Backend API response  ✅
```

---

## 🔧 Troubleshooting Guide

### Problem 1: PM2 — "Script not found" Error
```
Cause: ecosystem.config.js contains the old/wrong repository path
```
```bash
grep cwd ecosystem.config.js
# Shows: /home/ubuntu/bmi-health-tracker/backend  ← wrong

sed -i "s|/home/ubuntu/bmi-health-tracker/backend|/home/ubuntu/single-server-3tier-webapp/backend|g" ecosystem.config.js
pm2 restart bmi-backend
```

---

### Problem 2: Backend — "ECONNREFUSED" on Database Connection
```
Cause: PostgreSQL is not accepting remote connections
```
```bash
# Check on Database Server:
sudo ss -tlnp | grep 5432
# If it shows 127.0.0.1:5432 → listen_addresses is still set to localhost only

sudo grep listen_addresses /etc/postgresql/16/main/postgresql.conf
# If it still shows 'localhost' or is commented out:
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/16/main/postgresql.conf
sudo systemctl restart postgresql@16-main
```

---

### Problem 3: PostgreSQL Shows "active (exited)" — Fake Running
```
Cause: "postgresql.service" is only a wrapper service
       It does NOT start the actual database cluster
```
```bash
# Wrong — misleading status
sudo systemctl status postgresql

# Correct — always use the cluster-specific service name
sudo systemctl status postgresql@16-main
sudo pg_ctlcluster 16 main start
```

---

### Problem 4: Nginx Returns 403 Forbidden
```
Cause: Nginx (runs as www-data) cannot read the dist/ folder
```
```bash
sudo chmod 755 /home/ubuntu
sudo chmod -R 755 /home/ubuntu/single-server-3tier-webapp/frontend/dist
sudo systemctl reload nginx
```

---

### Problem 5: Nginx Returns 502 Bad Gateway
```
Cause: Backend is not running or proxy_pass IP/port is incorrect
```
```bash
pm2 status                      # check if bmi-backend is "online"
pm2 restart bmi-backend         # restart if "errored"
sudo grep proxy_pass /etc/nginx/sites-available/webapp
# Confirm the Backend Private IP and port 3000 are correct
```

---

### Problem 6: Migration Fails — "No such file or directory"
```
Cause: Running the migration command from inside the migrations/ folder
       causes the path to be doubled
```
```bash
# Wrong — running from inside migrations/
cd migrations/
cat migrations/*.sql | psql ...   # ❌ resolves to migrations/migrations/*.sql

# Correct — always run from the backend/ folder
cd ~/single-server-3tier-webapp/backend
cat migrations/*.sql | psql -U bmi_user -d bmidb -h localhost -W  # ✅
```

---

### Problem 7: postgresql.conf — "Invalid Line" Error
```
Cause: A pg_hba.conf rule was accidentally written into postgresql.conf
       These are two completely separate files with different purposes:
         postgresql.conf → server settings (port, memory, listen_addresses, logging)
         pg_hba.conf    → access control (who can connect, from where, and how)
```
```bash
# Remove the wrong line from postgresql.conf
sudo sed -i '/^host.*bmidb.*bmi_user.*md5/d' /etc/postgresql/16/main/postgresql.conf

# Add the rule to the correct file
echo "host    bmidb    bmi_user    10.0.0.0/16    md5" | sudo tee -a /etc/postgresql/16/main/pg_hba.conf

sudo systemctl restart postgresql@16-main
```

---

## 📊 Single Server vs 3 Separate Servers — Key Differences

| Configuration | Single Server | 3 Separate Servers |
|---|---|---|
| DB Host in `.env` | `localhost` | `<DB-PRIVATE-IP>` |
| Nginx `proxy_pass` | `http://localhost:3000` | `http://<BACKEND-PRIVATE-IP>:3000` |
| PostgreSQL `listen_addresses` | `127.0.0.1` (localhost only) | `0.0.0.0` (all interfaces) |
| `pg_hba.conf` | local connections only | VPC CIDR `10.0.0.0/16` |
| Security Groups | One shared group | Separate group per tier |
| Debugging | Easy — 1 terminal | Requires 3 terminals simultaneously |
| Security | Basic | Production-grade |
| Scalability | Cannot scale tiers independently | Each tier scales independently |
| Cost | Lowest | 3× instance cost |

---

## 📸 Screenshots Checklist (Assignment Submission)

- [ ] EC2 Console — All 3 instances in **Running** state
- [ ] Security Groups — Inbound rules for all 3 groups (`sg-frontend`, `sg-backend`, `sg-database`)
- [ ] **Database Server:** `sudo systemctl status postgresql@16-main`
- [ ] **Database Server:** `sudo ss -tlnp | grep 5432` — must show `0.0.0.0:5432`
- [ ] **Backend Server:** `pm2 status` — must show `online`
- [ ] **Backend Server:** `pm2 logs bmi-backend` — must show `✅ Database connected`
- [ ] **Frontend Server:** `sudo systemctl status nginx`
- [ ] **Browser:** `http://<FRONTEND-PUBLIC-IP>` — App UI is fully visible

---

## 🧠 Key Concepts Covered

| Concept | What You Learned |
|---|---|
| EC2 & Security Groups | Server-to-server access control using Security Group chaining |
| Nginx Reverse Proxy | Single entry point routing traffic to multiple backend services |
| PM2 Process Manager | Keeping Node.js processes alive with automatic restart |
| PostgreSQL Remote Access | Configuring `listen_addresses` and `pg_hba.conf` for remote access |
| `postgresql.conf` vs `pg_hba.conf` | Server configuration vs access control — two distinct files |
| `postgresql.service` vs `postgresql@16-main` | Wrapper service vs the real database cluster service |
| `dist/` vs `build/` | Vite output folder vs Create React App output folder |
| Private IP Communication | Faster, free, and more secure within the same VPC |
| `try_files /index.html` | React Router SPA support in Nginx configuration |
| Security Group Chaining | `sg-frontend → sg-backend → sg-database` layered security model |

---

## 👨‍💻 Author

**Md. Salauddin**
Software Developer

---

*Last Updated: April 2026*