# Ubuntu Server Setup & Deployment Guide

### Domain Mappings

| Domain             | Service        | Port |
| ------------------ | -------------- | ---- |
| mypick.com         | Frontend       | 3001 |
| api.mypick.com     | Backend API    | 5055 |
| admin.mypick.com   | Admin Panel    | 3002 |
| partner.mypick.com | Partner Portal | 3003 |
| socket.mypick.com  | Socket Service | 5088 |
| ai.mypick.com      | AI Service     | 8000 |


## 🚀 Initial System Setup

### 1. Start as Root
```bash
sudo -i
```

### 2. System Update & Upgrade
```bash
apt update && apt upgrade -y
```

### 3. Set Timezone (Asia/Qatar)
```bash
timedatectl set-timezone Asia/Qatar
```

### 4. Install Essential Tools
```bash
apt install -y htop nano curl wget git net-tools build-essential software-properties-common
```

### 5. Clean Up System
```bash
apt autoremove -y
apt clean
```

## 🔧 System Optimization

### 6. Configure Automatic Security Updates
```bash
apt install -y unattended-upgrades apt-listchanges
dpkg-reconfigure -plow unattended-upgrades
```

### 7. Optional: Configure Auto-updates
```bash
nano /etc/apt/apt.conf.d/50unattended-upgrades
```
Add these lines:
```bash
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
```

### 8. Create & Optimize Swap File 

```bash
# Check current swap
free -h

# Create 4GB swap file (16GB RAM) or
# Create 6GB swap file (32GB RAM)
fallocate -l 6G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Optimize swappiness
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf
sysctl -p

# Verify
swapon --show
free -h
```

## 🌐 NGINX Installation & Configuration

### 9. Install NGINX from Official Repository
```bash
# Remove old nginx if exists
apt remove --purge nginx nginx-common -y

# Add NGINX official repository
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list

# Install NGINX
apt update
apt install -y nginx

# Verify installation
nginx -v
systemctl status nginx
systemctl enable nginx
```

### 10. Optimize NGINX Configuration
```bash
# Backup default config
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

# Edit nginx.conf
nano /etc/nginx/nginx.conf

# Replace with optimized configuration:
```

```conf

user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    access_log off;
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;
    client_max_body_size 500M;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;

    # Load configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

```bash
# Test and restart NGINX
nginx -t
systemctl restart nginx
systemctl status nginx

reboot


### Create WebSocket Upgrade Map
sudo nano /etc/nginx/conf.d/websockets.conf

```
```nginx

map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```
```

```


---

## 🧩 NGINX Added Extra Settings Guide

Use this section when setting up or updating production NGINX for MyPick.  
These are the extra timeout, buffer, header, and WebSocket settings added after comparing production with demo.

### A. Global NGINX Extras

File:

```bash
/etc/nginx/nginx.conf
```

Add inside the `http {}` block:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

large_client_header_buffers 8 32k;
```

Recommended position:

```nginx
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ...
    client_max_body_size 500M;
    large_client_header_buffers 8 32k;
}
```

Command to add WebSocket map after `http {`:

```bash
sudo sed -i '/http {/a\    map $http_upgrade $connection_upgrade {\n        default upgrade;\n        '\'''\''      close;\n    }\n' /etc/nginx/nginx.conf
```

Command to add large client header buffers after `client_max_body_size 500M;`:

```bash
sudo sed -i '/client_max_body_size 500M;/a\    large_client_header_buffers 8 32k;' /etc/nginx/nginx.conf
```

---

### B. mypick-web Added Extra

File:

```bash
/etc/nginx/sites-available/mypick-web
```

Add inside `location /` after:

```nginx
proxy_pass http://localhost:3001;
```

Add:

```nginx
# Added Extra
proxy_connect_timeout 75s;
proxy_send_timeout 600s;
proxy_read_timeout 600s;
client_max_body_size 20m;
proxy_buffer_size 512k;
proxy_buffers 16 512k;
proxy_busy_buffers_size 1024k;
```

Command:

```bash
sudo sed -i '/proxy_pass http:\/\/localhost:3001;/a\        # Added Extra\n        proxy_connect_timeout 75s;\n        proxy_send_timeout 600s;\n        proxy_read_timeout 600s;\n        client_max_body_size 20m;\n        proxy_buffer_size 512k;\n        proxy_buffers 16 512k;\n        proxy_busy_buffers_size 1024k;' /etc/nginx/sites-available/mypick-web
```

---

### C. mypick-api Added Extra

File:

```bash
/etc/nginx/sites-available/mypick-api
```

Add inside `location /` after:

```nginx
proxy_cache_bypass $http_upgrade;
```

Add:

```nginx
# Added Extra
proxy_connect_timeout 75s;
proxy_send_timeout 300s;
proxy_read_timeout 300s;
client_max_body_size 25m;
proxy_buffer_size 256k;
proxy_buffers 8 256k;
proxy_busy_buffers_size 256k;
```

Command:

```bash
sudo sed -i '/proxy_cache_bypass \$http_upgrade;/a\        # Added Extra\n        proxy_connect_timeout 75s;\n        proxy_send_timeout 300s;\n        proxy_read_timeout 300s;\n        client_max_body_size 25m;\n        proxy_buffer_size 256k;\n        proxy_buffers 8 256k;\n        proxy_busy_buffers_size 256k;' /etc/nginx/sites-available/mypick-api
```

---

### D. mypick-admin Added Extra

File:

```bash
/etc/nginx/sites-available/mypick-admin
```

Add inside `location /` after:

```nginx
proxy_cache_bypass $http_upgrade;
```

Add:

```nginx
# Added Extra
proxy_buffer_size 256k;
proxy_buffers 8 256k;
proxy_busy_buffers_size 256k;
```

Command:

```bash
sudo sed -i '/proxy_cache_bypass \$http_upgrade;/a\        # Added Extra\n        proxy_buffer_size 256k;\n        proxy_buffers 8 256k;\n        proxy_busy_buffers_size 256k;' /etc/nginx/sites-available/mypick-admin
```

---

### E. mypick-ai Added Extra

File:

```bash
/etc/nginx/sites-available/mypick-ai
```

Recommended `location /`:

```nginx
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Added Extra
    proxy_read_timeout 300;
    proxy_send_timeout 300;
    proxy_connect_timeout 60;
}
```

Command to change `localhost` to `127.0.0.1`:

```bash
sudo sed -i 's/proxy_pass http:\/\/localhost:8000;/proxy_pass http:\/\/127.0.0.1:8000;/' /etc/nginx/sites-available/mypick-ai
```

Command to add timeout settings after `proxy_cache_bypass $http_upgrade;`:

```bash
sudo sed -i '/proxy_cache_bypass \$http_upgrade;/a\        # Added Extra\n        proxy_read_timeout 300;\n        proxy_send_timeout 300;\n        proxy_connect_timeout 60;' /etc/nginx/sites-available/mypick-ai
```

---

### F. mypick-socket Added Extra

File:

```bash
/etc/nginx/sites-available/mypick-socket
```

Because the global map was added:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Use this header in socket locations:

```nginx
proxy_set_header Connection $connection_upgrade;
```

Replace:

```nginx
proxy_set_header Connection "upgrade";
```

Command:

```bash
sudo sed -i 's/proxy_set_header Connection "upgrade";/proxy_set_header Connection $connection_upgrade;/' /etc/nginx/sites-available/mypick-socket
```

Keep existing Socket.IO timeout settings:

```nginx
proxy_read_timeout 86400;
proxy_send_timeout 86400;
proxy_connect_timeout 60;
```

Verify socket settings:

```bash
grep -n "Connection\|proxy_read_timeout\|proxy_send_timeout\|proxy_connect_timeout" /etc/nginx/sites-available/mypick-socket
```

Expected Connection lines:

```nginx
proxy_set_header Connection $connection_upgrade;
proxy_set_header Connection $connection_upgrade;
```

---

### G. mypick-partner

No extra change required.

Current partner config already matches demo:

```nginx
proxy_read_timeout 300;
proxy_connect_timeout 300;
proxy_send_timeout 300;
```

---

### H. Final Test and Reload

Always test before reload:

```bash
sudo nginx -t
```

Reload only if test is successful:

```bash
sudo systemctl reload nginx
```

One-line command:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

### I. Useful Verification Commands

Check global map:

```bash
grep -n "connection_upgrade" /etc/nginx/nginx.conf
```

Check global large header buffer:

```bash
grep -n "large_client_header_buffers" /etc/nginx/nginx.conf
```

Check all active buffer settings:

```bash
sudo nginx -T | grep -E "proxy_buffer_size|proxy_buffers|proxy_busy_buffers_size|large_client_header_buffers"
```

Check socket connection headers:

```bash
grep -n "Connection" /etc/nginx/sites-available/mypick-socket
```

Dump full active NGINX config:

```bash
sudo nginx -T
```

---


## 🗄️ MongoDB Installation

### 11. Check Ubuntu Version
```bash
cat /etc/lsb-release
# Should output:
# DISTRIB_ID=Ubuntu
# DISTRIB_RELEASE=24.04
# DISTRIB_CODENAME=noble
# DISTRIB_DESCRIPTION="Ubuntu 24.04.3 LTS"
```

### 12. Install MongoDB 8.2
```bash
# Install prerequisites
sudo apt-get install gnupg curl

# Import MongoDB public GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
   --dearmor

# Create MongoDB repository list file
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# Update package database and install MongoDB
sudo apt-get update
sudo apt-get install -y mongodb-org

# Start and enable MongoDB
sudo systemctl start mongod
sudo systemctl status mongod
sudo systemctl daemon-reload
sudo systemctl enable mongod
sudo systemctl stop mongod
sudo systemctl restart mongod

# MongoDB internal key and a proper rs0 replica-set setup
# 🧹 STEP 1 — Stop MongoDB
sudo systemctl stop mongod

# Create the directory
sudo mkdir -p /var/lib/mongo

# Set directory ownership
sudo chown mongodb:mongodb /var/lib/mongo
sudo chmod 700 /var/lib/mongo

# 🔐 STEP 2 — Generate a BRAND-NEW MongoDB key
sudo openssl rand -base64 756 | sudo tee /var/lib/mongo/mongod.key > /dev/null

# Set correct permissions (MANDATORY)
sudo chown mongodb:mongodb /var/lib/mongo/mongod.key
sudo chmod 600 /var/lib/mongo/mongod.key

# Verify
sudo ls -l /var/lib/mongo/mongod.key

# Expected:
# -rw------- 1 mongodb mongodb ...

# If you want to view the key content (again: don’t share it):
sudo cat /var/lib/mongo/mongod.key


# 🧾 STEP 3 — Configure MongoDB (mongod.conf) Edit config:

sudo nano /etc/mongod.conf

# ✅ FINAL, CORRECT CONFIG

storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1

processManagement:
  fork: false
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled
  keyFile: /var/lib/mongo/mongod.key

replication:
  replSetName: "rs0"


# Save and exit.

# ▶️ STEP 4 — Start MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod

# Check status
sudo systemctl status mongod

```
### 12. MongoDB User Authentication:

```bash

# Access MongoDB shell:
mongosh


# Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [{ _id: 0, host: "localhost:27017" }]
})

#List databases:
show dbs;

#Switch to the admin database:
use admin

#Create user:
db.createUser({
  user: "root",
  pwd: "MyPickDB@Pass",
  roles: [{ role: "root", db: "admin" }]
});

# Lits All Users
db.getUsers()

#Exit shell:
exit

# Reconnect Using Root Credentials
mongosh -u root -p 'MyPickDB@Pass' --authenticationDatabase admin


# Update User Password
use admin
db.updateUser("root", { pwd: "MyPickDB@Pass" })

#Exit shell:
exit


#Install Network Tools (If Not Installed)
sudo apt install -y net-tools


#Check running ports:
sudo netstat -ntlp

# Verify Login Again
mongosh -u root -p 'MyPickDB@Pass' --authenticationDatabase admin


#List databases:
show dbs;

exit

# MongoDB Compass Connection Using Privately
# 1. EC2 Must SSH Whitelisted Your Public Computer IP for Access with SSH

#Explanation:
| Part                        | Meaning                                                  |
| --------------------------- | -------------------------------------------------------- |
| `-i "C:/…/cart24-live.pem"` | Your EC2 private key                                     |
| `-L 27017:127.0.0.1:27017`  | Forwards your **local port 27017 → EC2 localhost:27017** |
| `ubuntu@52.71.50.242`       | EC2 username (`ubuntu`) and **public IP**                |

# First Connect from PC Using Terminal this Your SSH Tunnel Command for Localhost IP in MongoDB: 
"commandline": "ssh -i \"C:/Users/devom/OneDrive/data/Multimedia Qatar/Multimedia E-Marketing/Customer/cart24.com/aws-key/cart24-live.pem\" -L 27017:127.0.0.1:27017 ubuntu@52.71.50.242",

# Mac ssh Settings:
# MyPick MongoDB Tunnel Connect for MongoDB Localhost Connection
Host mypick-live-mongodb-tunnel
    HostName 52.45.203.24 
    User ubuntu
    IdentityFile "/Users/devomman/Library/CloudStorage/OneDrive-Personal/data/Multimedia Qatar/Multimedia E-Marketing/Customer/mypick/aws/mypick-live-lightsail-ssh-key.pem"                     
    LocalForward 27017 127.0.0.1:27017



# Connect MongoDB Compass
# 1. Keep Open the SSH Windows Powershell Terminal for MongoDB Tunnel 
# 2. Open MongoDB Compass → New Connection.
# 3. Use this connection string: 
mongodb://root:MyPickDB%40Pass@127.0.0.1:27017/?authSource=admin


# root → MongoDB username
# MyPickDB%40Pass → your password (@ must be %40)
# 127.0.0.1:27017 → local machine, forwarded via SSH tunnel
# authSource=admin → authenticate against admin database
# Click Connect → you should see your MongoDB databases.


# [Knowledge] 🔐 Common URL Encodings
# Character	Encoded
# @	        %40
# %	        %25
# :	        %3A
# /	        %2F




# ✅ Must show active (running)

# 🧠 STEP 5 — Initialize Replica Set (rs0)
mongosh -u root -p 'MyPickDB@Pass' --authenticationDatabase admin


# Inside mongosh:
rs.initiate()


# Verify:
rs.status()


# Expected:
# stateStr: "PRIMARY"
# set: "rs0"

# 🔍 STEP 6 — Confirm key is ACTIVE Run:
db.adminCommand({ getCmdLineOpts: 1 })



# You should see:

"security": {
  "authorization": "enabled",
  "keyFile": "/var/lib/mongo/mongod.key"
},
"replication": {
  "replSetName": "rs0"
}

```

## 6. Redis Installation & Configuration

### Install Redis

```bash
sudo apt install -y redis-server
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

### Redis Configuration

```bash
# Edit /etc/redis/redis.conf
sudo nano /etc/redis/redis.conf
```

Key settings (already configured):

```conf
bind 127.0.0.1 -::1
protected-mode yes
port 6379
```

### Verify Redis

```bash
redis-cli ping
# Should return: PONG
```


## 4. Environment Setup

### Create Application Directory

```bash
mkdir -p /home/ubuntu/deploy
cd /home/ubuntu/deploy
```

### Clone Repositories

```bash
# Backend API
git clone https://github.com/devomman/mypick-api.git mypick-api

# Frontend (Web)
git clone https://github.com/devomman/mypick-web.git mypick-web

# Admin Panel
git clone https://github.com/devomman/mypick-admin.git mypick-admin

# Partner Portal
git clone https://github.com/devomman/mypick-partner.git mypick-partner

# Socket Service (if separate repo, adjust accordingly)
git clone https://github.com/devomman/mypick-socket.git mypick-socket

# AI Server (if separate repo, adjust accordingly)
git clone https://github.com/devomman/mypick-ai.git mypick-ai
```



## 📦 Node.js & NVM Setup (Optional)

### 13. Install NVM and Node.js
```bash
# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash

# Load NVM
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# If any Issue After Install
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"


# Reload the shell configuration
source ~/.bashrc

nvm -v

# Install Node.js 18.20.6
nvm install 18.20.6
nvm alias default 18.20.6
nvm use 18.20.6
node -v
# Expected: v18.20.6
npm -v
which node

# Should output: v18.20.6
```

## 7. Node.js & PM2 Setup

### Install Node.js 24.x

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
which node
```

### Install PM2

```bash
sudo npm install -g pm2

# Setup PM2 startup script
pm2 startup systemd -u ubuntu --hp /home/ubuntu
# Run the command it outputs
```

### Install PM2 Monitoring (Optional)

```bash
pm2 install pm2-logrotate
```



### 14. Set User Permissions
```bash
sudo chown -R ubuntu:ubuntu /home/ubuntu
```

## 🔗 NGINX Site Configuration

### 15. Setup NGINX Site Directories
```bash
# Check if sites directories exist
ls /etc/nginx/

# Create directories if needed
sudo mkdir -p /etc/nginx/sites-available
sudo mkdir -p /etc/nginx/sites-enabled

# Verify NGINX configuration includes sites-enabled
sudo nano /etc/nginx/nginx.conf
# Add inside http block if not present:
# include /etc/nginx/sites-enabled/*;

# Test and restart NGINX
sudo nginx -t
sudo systemctl restart nginx
```

### 16. Create NGINX Server Blocks

**mypick-web:**
```bash
sudo nano /etc/nginx/sites-available/mypick-web
```
```conf
server {
    listen 80;
    listen [::]:80;
    server_name mypick.com www.mypick.com;

    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;
}
```

**mypick-admin:**
```bash
sudo nano /etc/nginx/sites-available/mypick-admin
```
```conf
server {
    listen 80;
    listen [::]:80;
    server_name admin.mypick.com www.admin.mypick.com;

    location / {
        proxy_pass http://localhost:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }

    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;
}
```

**mypick-partner:**
```bash
sudo nano /etc/nginx/sites-available/mypick-partner
```
```conf
server {
    listen 80;
    listen [::]:80;
    server_name partner.mypick.com www.partner.mypick.com;

    location / {
        proxy_pass http://localhost:3003;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;

        # Timeout settings for Next.js
        proxy_read_timeout 300;
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
    }

    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;
}

```

**mypick-api:**
```bash
sudo nano /etc/nginx/sites-available/mypick-api
```
```conf
server {
    listen 80;
    listen [::]:80;
    server_name api.mypick.com www.api.mypick.com;

    location / {
        proxy_pass http://localhost:5055;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }

    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;
}
```

**mypick-socket:**
```bash
sudo nano /etc/nginx/sites-available/mypick-socket
```
```conf
server {
    listen 80;
    listen [::]:80;
    server_name socket.mypick.com www.socket.mypick.com;

    # Socket.IO endpoint
    location /socket.io/ {
        proxy_pass http://localhost:5088;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings for WebSocket
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
        proxy_connect_timeout 60;
    }


    location / {
        proxy_pass http://localhost:5088;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;

        # Timeout settings for Socket
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;
}
```
**mypick-ai:**
```bash
sudo nano /etc/nginx/sites-available/mypick-ai
```
```conf
server {
    listen 80;
    listen [::]:80;
    server_name ai.mypick.com www.ai.mypick.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }

    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype >
}
```

### 17. Enable Sites
```bash
# Create symbolic links
sudo ln -s /etc/nginx/sites-available/mypick-web /etc/nginx/sites-enabled/mypick-web
sudo ln -s /etc/nginx/sites-available/mypick-admin /etc/nginx/sites-enabled/mypick-admin
sudo ln -s /etc/nginx/sites-available/mypick-partner /etc/nginx/sites-enabled/mypick-partner
sudo ln -s /etc/nginx/sites-available/mypick-api /etc/nginx/sites-enabled/mypick-api
sudo ln -s /etc/nginx/sites-available/mypick-socket /etc/nginx/sites-enabled/mypick-socket
sudo ln -s /etc/nginx/sites-available/mypick-ai /etc/nginx/sites-enabled/mypick-ai

# Verify links
cd /etc/nginx/sites-enabled/
ls

# Test configuration
sudo nginx -t
sudo systemctl restart nginx
```

## 🔐 SSL Certificate with Certbot

### 18. Install Certbot and SSL
```bash
# Install Certbot
sudo apt update
sudo apt install snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Obtain SSL certificates (automatic configuration)
sudo certbot --nginx
# Or manual configuration:
# sudo certbot --nginx -d mypick.com

# Follow prompts:
# 1. Enter email: admin@mypick.com
# 2. Agree to terms (Y)
# 3. Subscribe to emails (N)
# 4. Select all domains (Press Enter)

# Test automatic renewal
sudo certbot renew --dry-run

# Certbot sets up automatic renewal via systemd timer:
sudo systemctl list-timers | grep certbot
```

## 🚀 Application Deployment


### 20. Install PM2
```bash
sudo npm install -g pm2
pm2 -v
pm2 status

# Setup PM2 startup script
pm2 startup systemd -u ubuntu --hp /home/ubuntu
# Run the command it outputs

# Then enable startup:
pm2 startup
pm2 save
pm2 save --force
```

### Install "pnpm" for not to use "npm"
```bash
sudo npm i -g pnpm

```

### 22. Deploy Applications

**mypick-api:**
```bash
cd deploy/mypick-api
git status
git checkout live
git fetch origin
git pull origin live
sudo rm -rf node_modules
pnpm i
pnpm build:com
# pnpm run start:prod
pnpm pm2:start
# pm2 start pnpm --name "mypick-api" -- run start:prod

# Update process:
# git pull
# git stash - if any changes
# pnpm build (Or)
# pnpm build:com
# pm2 restart 0
# pm2 restart 0 --update-env (If any env update)
```

**mypick-web:**
```bash
cd deploy/mypick-web
git status
git checkout live
git fetch origin
git pull origin live
sudo rm -rf node_modules
pnpm i
pnpm build:com
pnpm pm2:start
pm2 start pnpm --name "mypick-web" -- start -p 3001


# Update process:
# git pull
# git stash - if any changes
# pnpm build (Or)
# pnpm build:com
# pm2 restart 1
# pm2 restart 1 --update-env (If any env update)
# pm2 start pnpm --name "mypick-web" -- start -p 3001
```

**mypick-admin:**
```bash
cd deploy/mypick-admin
git status
git checkout live
git fetch origin
git pull origin live
sudo rm -rf node_modules
pnpm i
pnpm build:com
pnpm pm2:start
pm2 start pnpm --name "mypick-admin" -- start -p 3002
# or "pnpm run pm2:start"

# Update process:
# git pull
# git stash - if any changes
# pnpm run build (Or)
# pnpm build:com
# pm2 status
# pm2 delete 5 - if needed
# pm2 restart 6
# pnpm run pm2:start
# pm2 restart 6 --update-env (If any env update)
```

**mypick-partner:**
```bash
cd deploy/mypick-partner
git status
git checkout live
git fetch origin
git pull origin live
sudo rm -rf node_modules
pnpm i
pnpm build:com
pnpm pm2:start
pm2 start pnpm --name "mypick-partner" -- start -p 3003

# Update process:
# git pull
# git stash - if any changes
# pnpm run build (Or)
# pnpm build:com
# pm2 status
# pm2 delete 5 - if needed
# pm2 restart 6
# pm2 start pnpm --name "mypick-partner" -- start -p 3003
# pm2 restart 6 --update-env (If any env update)
```

**mypick-socket:**
```bash
cd deploy/mypick-socket
git status
git checkout live
git fetch origin
git pull origin live
sudo rm -rf node_modules
pnpm i
pnpm build:com
pnpm pm2:start
pm2 start pnpm --name "mypick-socket" -- run start:prod

# Update process:
# git pull
# git stash - if any changes
# pm2 restart 3
# pm2 restart 3 --update-env (If any env update)
```

**mypick-ai:**
```bash
cd deploy/mypick-ai

git status
git checkout live
git fetch origin
git pull origin live
sudo apt install python3 pip
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates
sudo apt install -y python3.12
sudo apt install python3.12-venv
python3.12 -m venv venv
./venv/bin/pip install --upgrade pip
./venv/bin/pip install -r requirements.txt
chmod +x prod-pm2.sh && ./prod-pm2.sh

git status
git checkout live
git fetch origin
git pull origin live
./venv/bin/pip install --upgrade pip
./venv/bin/pip install -r requirements.txt
chmod +x prod-pm2.sh && ./prod-pm2.sh
pm2 restart 5 --update-env

# APP_DIR="$(pwd)"
# pm2 start "$APP_DIR/venv/bin/python" \
#   --name mypick-ai \
#   --cwd "$APP_DIR" \
#   --time \
#   -- -m uvicorn src.main:app --host 0.0.0.0 --port 8000
# pm2 save
# pm2 startup

# Update process:
# git pull
# git stash - if any changes
# pm2 restart 5
# pm2 restart 5 --update-env (If any env update)
```