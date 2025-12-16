
# BMI & Health Tracker — AWS Ubuntu EC2 Deployment Guide

This comprehensive guide walks you through deploying the **BMI & Health Tracker** full-stack application (React + Node.js + PostgreSQL) on a **fresh AWS Ubuntu EC2 server** using **Nginx** as a reverse proxy and static file server.

## 🌟 Application Features

- **Health Metrics Calculation**: BMI, BMR (using Mifflin-St Jeor equation), and daily calorie needs
- **Custom Measurement Dates**: Track measurements with specific dates (not just "now") - great for historical data
- **30-Day BMI Trend Visualization**: Interactive charts showing your BMI progress over time
- **Multiple Activity Levels**: From sedentary to very active lifestyle calculations
- **Real-time Stats Dashboard**: Quick view of your current metrics and total measurements
- **Responsive Design**: Works seamlessly on desktop and mobile devices

---

## Prerequisites

- AWS Account with EC2 access
- Domain name (optional, but recommended for HTTPS)
- SSH client (Terminal, PuTTY, or similar)
- Basic knowledge of Linux commands

---

## Part 1: AWS EC2 Setup

### 1.1 Launch EC2 Instance

1. **Sign in to AWS Console** → Navigate to EC2 Dashboard
2. **Click "Launch Instance"**
3. **Configure Instance:**
   - **Name**: `bmi-health-tracker-server`
   - **AMI**: Ubuntu Server 22.04 LTS (or latest LTS)
   - **Instance Type**: `t2.micro` (free tier) or `t2.small` (recommended)
   - **Key Pair**: Create new or use existing `.pem` key (download and save securely)
   - **Network Settings**: 
     - Create security group or use existing
     - Allow SSH (port 22) from your IP
     - Allow HTTP (port 80) from anywhere (0.0.0.0/0)
     - Allow HTTPS (port 443) from anywhere (0.0.0.0/0) - if using SSL
   - **Storage**: 20 GB gp3 (minimum 10 GB recommended)
4. **Launch Instance** and wait for it to be in "Running" state

### 1.2 Configure Security Group

Add these inbound rules to your security group:

| Type  | Protocol | Port Range | Source    | Description           |
|-------|----------|------------|-----------|-----------------------|
| SSH   | TCP      | 22         | My IP     | SSH access            |
| HTTP  | TCP      | 80         | 0.0.0.0/0 | HTTP web traffic      |
| HTTPS | TCP      | 443        | 0.0.0.0/0 | HTTPS secure traffic  |

### 1.3 Connect to EC2 Instance

```bash
# Set permissions for your .pem file (first time only)
chmod 400 your-key.pem

# Connect via SSH
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace `YOUR_EC2_PUBLIC_IP` with your instance's public IP from AWS Console.

---

## Part 2: Server Preparation

### 2.1 Update System & Install Essential Tools

```bash
# Update package lists and upgrade existing packages
sudo apt update && sudo apt upgrade -y

# Install essential build tools and utilities
sudo apt install -y git curl wget build-essential nginx ufw ca-certificates gnupg
```

### 2.2 Install Node.js via NVM (Node Version Manager)

```bash
# Download and install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Load NVM into current session
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# Install Node.js LTS version
nvm install --lts

# Verify installation
node -v   # Should show v20.x.x or similar
npm -v    # Should show 10.x.x or similar
```

**Note**: If you disconnect and reconnect, NVM will auto-load via `.bashrc`.

---

## Part 3: PostgreSQL Database Setup

### 3.1 Install PostgreSQL

```bash
# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Start and enable PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verify it's running
sudo systemctl status postgresql
```

### 3.2 Create Database and User

```bash
# Switch to postgres user and create database user
sudo -u postgres createuser --interactive --pwprompt

# Enter details:
# Username: bmi_user
# Password: [ostad2025]
# Superuser: n
# Create databases: n
# Create roles: n

# Create database owned by bmi_user
sudo -u postgres createdb -O bmi_user bmidb

# Test connection
psql -U bmi_user -d bmidb -h localhost
# Enter password when prompted
# Type \q to quit
```

### 3.3 Configure PostgreSQL for Local Connections (if needed)

```bash
# Edit pg_hba.conf to ensure local connections work
sudo nano /etc/postgresql/*/main/pg_hba.conf

# Ensure this line exists (should be there by default):
# local   all             all                                     md5
# host    all             all             127.0.0.1/32            md5

# Restart PostgreSQL
sudo systemctl restart postgresql
```

---

## Part 4: Deploy the Application

### 4.1 Clone or Upload Project

**Option A: Clone from GitHub (recommended)**
```bash
cd /home/ubuntu
git clone https://github.com/YOUR_USERNAME/bmi-health-tracker.git
cd bmi-health-tracker
```

**Option B: Upload via SCP**
```bash
# On your local machine:
scp -i your-key.pem -r ./bmi-health-tracker ubuntu@YOUR_EC2_PUBLIC_IP:/home/ubuntu/

# Then SSH and navigate:
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
cd /home/ubuntu/bmi-health-tracker
```

**Option C: Upload as ZIP**
```bash
# On local machine:
zip -r bmi-health-tracker.zip bmi-health-tracker
scp -i your-key.pem bmi-health-tracker.zip ubuntu@YOUR_EC2_PUBLIC_IP:/home/ubuntu/

# On server:
sudo apt install -y unzip
unzip bmi-health-tracker.zip
cd bmi-health-tracker
```

### 4.2 Setup Backend

```bash
cd /home/ubuntu/bmi-health-tracker/backend

# Create environment file from example
cp .env.example .env

# Edit environment variables
nano .env
```

**Configure `.env` file:**
```env
PORT=3000
DATABASE_URL=postgresql://bmi_user:YOUR_DB_PASSWORD@localhost:5432/bmidb
NODE_ENV=production
```

Replace `YOUR_DB_PASSWORD` with the password you created for `bmi_user`.

**Install backend dependencies:**
```bash
npm install --production
```

**Run database migrations:**
```bash
# Apply migrations in order to create and update the measurements table

# Migration 001: Create measurements table
psql -U bmi_user -d bmidb -h localhost -f migrations/001_create_measurements.sql
# Enter password when prompted

# Migration 002: Add measurement_date column for custom date tracking
psql -U bmi_user -d bmidb -h localhost -f migrations/002_add_measurement_date.sql
# Enter password when prompted

# Verify migrations were successful
psql -U bmi_user -d bmidb -h localhost -c "\d measurements"
# You should see the measurements table structure with measurement_date column
```

**What these migrations do:**
- **Migration 001**: Creates the `measurements` table with all health metrics (BMI, BMR, calories, etc.)
- **Migration 002**: Adds the `measurement_date` column to allow users to specify when a measurement was taken (not just when it was recorded)

**Test backend locally (optional):**
```bash
npm start
# Should see: "Server started"
# Press Ctrl+C to stop
```

### 4.3 Setup Frontend

```bash
cd /home/ubuntu/bmi-health-tracker/frontend

# Install dependencies
npm install

# Build for production
npm run build
```

This creates a `dist/` folder with optimized static files.

**Deploy built files to Nginx web directory:**
```bash
# Create directory for frontend
sudo mkdir -p /var/www/bmi-health-tracker

# Copy built files
sudo cp -r dist/* /var/www/bmi-health-tracker/

# Set proper ownership
sudo chown -R www-data:www-data /var/www/bmi-health-tracker

# Verify files copied
ls -la /var/www/bmi-health-tracker
```

---

## Part 5: Process Management with PM2

PM2 keeps your Node.js backend running continuously and restarts it on crashes or server reboots.

### 5.1 Install PM2 Globally

```bash
npm install -g pm2
```

### 5.2 Start Backend with PM2

```bash
cd /home/ubuntu/bmi-health-tracker/backend

# Start the backend server
pm2 start src/server.js --name bmi-backend

# Check status
pm2 status

# View logs
pm2 logs bmi-backend

# Save PM2 process list
pm2 save
```

### 5.3 Configure Auto-Start on Reboot

```bash
# Generate startup script
pm2 startup systemd -u ubuntu --hp /home/ubuntu

# Run the command that PM2 outputs (it will be something like):
# sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/vXX.X.X/bin ...
# Copy and paste that exact command

# Verify auto-start is configured
sudo systemctl status pm2-ubuntu
```

### 5.4 Useful PM2 Commands

```bash
pm2 list                    # List all processes
pm2 restart bmi-backend     # Restart backend
pm2 stop bmi-backend        # Stop backend
pm2 delete bmi-backend      # Remove from PM2
pm2 logs bmi-backend        # View live logs
pm2 logs bmi-backend --lines 100  # Last 100 lines
pm2 monit                   # Monitor CPU/memory
```

---

## Part 6: Nginx Configuration

### 6.1 Create Nginx Site Configuration

```bash
sudo nano /etc/nginx/sites-available/bmi-health-tracker
```

**Paste this configuration:**

```nginx
server {
    listen 80;
    listen [::]:80;
    
    # Replace with your domain or EC2 public IP
    server_name YOUR_DOMAIN_OR_IP;

    # Frontend static files
    root /var/www/bmi-health-tracker;
    index index.html;

    # Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/x-javascript application/xml+rss 
               application/javascript application/json;

    # Frontend routing (React Router)
    location / {
        try_files $uri $uri/ /index.html;
        
        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # Backend API proxy
    location /api/ {
        proxy_pass http://127.0.0.1:3000/api/;
        proxy_http_version 1.1;
        
        # WebSocket support (if needed in future)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        
        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Disable caching for API
        proxy_cache_bypass $http_upgrade;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Hide nginx version
    server_tokens off;

    # Logs
    access_log /var/log/nginx/bmi-access.log;
    error_log /var/log/nginx/bmi-error.log;
}
```

**Replace `YOUR_DOMAIN_OR_IP` with:**
- Your domain name (e.g., `bmi.example.com`) if you have one
- Your EC2 public IP (e.g., `3.15.123.45`) if you don't have a domain

### 6.2 Enable the Site

```bash
# Create symbolic link to enable site
sudo ln -s /etc/nginx/sites-available/bmi-health-tracker /etc/nginx/sites-enabled/

# Remove default site (optional)
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# If test passes, reload Nginx
sudo systemctl reload nginx

# Check Nginx status
sudo systemctl status nginx
```

### 6.3 Verify Nginx is Running

```bash
# Start Nginx if not running
sudo systemctl start nginx

# Enable auto-start on boot
sudo systemctl enable nginx
```

---

## Part 7: Firewall Configuration (UFW)

Secure your server by enabling Ubuntu's firewall.

### 7.1 Configure UFW

```bash
# Check UFW status
sudo ufw status

# Allow SSH (CRITICAL - do this first!)
sudo ufw allow OpenSSH
# Or if using custom SSH port:
# sudo ufw allow 2222/tcp

# Allow HTTP traffic
sudo ufw allow 'Nginx HTTP'

# Allow HTTPS traffic (for future SSL)
sudo ufw allow 'Nginx HTTPS'

# Or allow specific ports:
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Verify rules
sudo ufw status verbose
```

**Expected output:**
```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx HTTP                 ALLOW       Anywhere
Nginx HTTPS                ALLOW       Anywhere
```

---

## Part 8: SSL/HTTPS Setup (Optional but Recommended)

Use Let's Encrypt to secure your site with free SSL certificates.

### 8.1 Prerequisites

- A domain name pointed to your EC2 instance's public IP
- DNS A record configured (wait 5-10 minutes for propagation)

### 8.2 Install Certbot

```bash
# Install Certbot and Nginx plugin
sudo apt install -y certbot python3-certbot-nginx
```

### 8.3 Obtain SSL Certificate

```bash
# Replace YOUR_DOMAIN with your actual domain
sudo certbot --nginx -d YOUR_DOMAIN

# Or for both www and non-www:
# sudo certbot --nginx -d example.com -d www.example.com
```

**Follow the prompts:**
1. Enter email address
2. Agree to terms
3. Choose whether to share email with EFF
4. Choose whether to redirect HTTP to HTTPS (recommended: Yes)

### 8.4 Test Auto-Renewal

```bash
# Test renewal process (dry run)
sudo certbot renew --dry-run

# Certificates auto-renew via systemd timer
sudo systemctl status certbot.timer
```

### 8.5 Verify HTTPS

Visit `https://YOUR_DOMAIN` in your browser. You should see a padlock icon.

---

## Part 9: Testing & Verification

### 9.1 Test Backend API

```bash
# Test health endpoint
curl http://localhost:3000/health
# Expected: {"status":"ok","environment":"production"}

# Test get all measurements
curl http://localhost:3000/api/measurements
# Expected: {"rows":[]} (empty array initially)

# Test creating a measurement with custom date
curl -X POST http://localhost:3000/api/measurements \
  -H "Content-Type: application/json" \
  -d '{
    "weightKg": 70,
    "heightCm": 175,
    "age": 30,
    "sex": "male",
    "activity": "moderate",
    "measurementDate": "2025-12-15"
  }'

# Expected response includes calculated values:
# {"measurement":{"id":1,"bmi":"22.9","bmi_category":"Normal","bmr":1732,"daily_calories":2685,...}}

# Test from public internet (replace with your IP/domain)
curl http://YOUR_EC2_PUBLIC_IP/api/measurements

# Test 30-day trends endpoint
curl http://localhost:3000/api/measurements/trends
# Returns BMI averages grouped by date for the last 30 days
```

### 9.2 Test Frontend

Open browser and navigate to:
- `http://YOUR_EC2_PUBLIC_IP` or
- `http://YOUR_DOMAIN` or
- `https://YOUR_DOMAIN` (if SSL configured)

**Check these features:**
1. ✅ Form displays with all 6 input fields:
   - **Measurement Date** (new feature! allows entering historical data)
   - Weight (kg)
   - Height (cm)
   - Age (years)
   - Biological Sex
   - Activity Level
2. ✅ Submit a measurement (defaults to today's date, but you can change it)
3. ✅ Measurement appears in the recent list with the specified date
4. ✅ Stats cards display current BMI, BMR, and daily calorie needs
5. ✅ Trend chart displays (shows 30-day BMI trends)
6. ✅ Check browser console for errors (F12)

**Test the measurement date feature:**
- Try entering a measurement from a past date (e.g., last week)
- Verify it appears with the correct date in the measurements list
- The date field has a max limit (cannot select future dates)

### 9.3 Test Database

```bash
# Connect to database
psql -U bmi_user -d bmidb -h localhost

# View all measurements with dates
SELECT id, weight_kg, height_cm, bmi, bmi_category, measurement_date, created_at 
FROM measurements 
ORDER BY measurement_date DESC;

# View 30-day BMI trends (same query the chart uses)
SELECT measurement_date AS day, AVG(bmi) AS avg_bmi 
FROM measurements
WHERE measurement_date >= CURRENT_DATE - interval '30 days' 
GROUP BY measurement_date 
ORDER BY measurement_date;

# Count total measurements
SELECT COUNT(*) FROM measurements;

# View table structure (verify measurement_date column exists)
\d measurements

# Exit
\q
```

**Note:** The `measurement_date` column allows users to track when measurements were actually taken, which is different from `created_at` (when the record was entered into the system).

### 9.4 Monitor Logs

**Backend logs:**
```bash
pm2 logs bmi-backend
pm2 logs bmi-backend --lines 50
```

**Nginx access logs:**
```bash
sudo tail -f /var/log/nginx/bmi-access.log
```

**Nginx error logs:**
```bash
sudo tail -f /var/log/nginx/bmi-error.log
```

**PostgreSQL logs:**
```bash
sudo tail -f /var/log/postgresql/postgresql-*-main.log
```

---

## Part 10: Maintenance & Troubleshooting

### 10.1 Update Application

```bash
# Navigate to project directory
cd /home/ubuntu/bmi-health-tracker

# Pull latest changes (if using Git)
git pull origin main

# Update backend
cd backend
npm install --production
pm2 restart bmi-backend

# Update frontend
cd ../frontend
npm install
npm run build
sudo rm -rf /var/www/bmi-health-tracker/*
sudo cp -r dist/* /var/www/bmi-health-tracker/
sudo chown -R www-data:www-data /var/www/bmi-health-tracker
```

### 10.2 Common Issues & Solutions

**Issue: Backend not accessible**
```bash
# Check PM2 status
pm2 status

# Restart backend
pm2 restart bmi-backend

# Check logs
pm2 logs bmi-backend --lines 100
```

**Issue: Database connection failed**
```bash
# Verify PostgreSQL is running
sudo systemctl status postgresql

# Test connection
psql -U bmi_user -d bmidb -h localhost

# Check backend .env file has correct DATABASE_URL
cat /home/ubuntu/bmi-health-tracker/backend/.env
```

**Issue: Nginx 502 Bad Gateway**
```bash
# Verify backend is running
pm2 status
curl http://localhost:3000/api/measurements

# Check Nginx error logs
sudo tail -100 /var/log/nginx/bmi-error.log

# Test Nginx config
sudo nginx -t
sudo systemctl restart nginx
```

**Issue: Cannot connect via HTTP**
```bash
# Check UFW firewall
sudo ufw status

# Ensure ports 80/443 are open
sudo ufw allow 'Nginx Full'

# Check Nginx is listening
sudo netstat -tlnp | grep nginx

# Check AWS Security Group has port 80/443 open
```

**Issue: Frontend shows blank page**
```bash
# Check if files were copied
ls -la /var/www/bmi-health-tracker/

# Check browser console for errors (F12)

# Verify Nginx is serving files
curl http://YOUR_EC2_PUBLIC_IP

# Check Nginx error logs
sudo tail -50 /var/log/nginx/bmi-error.log
```
### 10.3 Common Issues

**Problem: Measurement dates not showing correctly**
```bash
# Check if migration 002 was applied
psql -U bmi_user -d bmidb -h localhost -c "\d measurements" | grep measurement_date

# If column is missing, run migration 002:
cd /home/ubuntu/bmi-health-tracker/backend
psql -U bmi_user -d bmidb -h localhost -f migrations/002_add_measurement_date.sql

# Restart backend
pm2 restart bmi-backend
```

**Problem: Cannot select dates in the form**
```bash
# Verify frontend is properly deployed
ls -la /var/www/bmi-health-tracker/

# Check for JavaScript errors in browser console (F12)
# Rebuild and redeploy if needed:
cd /home/ubuntu/bmi-health-tracker/frontend
npm run build
sudo cp -r dist/* /var/www/bmi-health-tracker/
sudo chown -R www-data:www-data /var/www/bmi-health-tracker
```

**Problem: Backend not accepting measurementDate field**
```bash
# Check backend code is current
cd /home/ubuntu/bmi-health-tracker/backend
grep -r "measurementDate" src/routes.js

# If not found, ensure you have the latest code
# Then restart:
pm2 restart bmi-backend
```
### 10.3 Backup Database

```bash
# Create backup directory
mkdir -p /home/ubuntu/backups

# Backup database
pg_dump -U bmi_user -h localhost bmidb > /home/ubuntu/backups/bmidb_$(date +%Y%m%d_%H%M%S).sql

# Restore database (if needed)
psql -U bmi_user -h localhost bmidb < /home/ubuntu/backups/bmidb_20251212_120000.sql
```

### 10.4 Automated Backups (Optional)

Create a backup script:
```bash
nano /home/ubuntu/backup-db.sh
```

Add:
```bash
#!/bin/bash
BACKUP_DIR="/home/ubuntu/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR
pg_dump -U bmi_user -h localhost bmidb > $BACKUP_DIR/bmidb_$TIMESTAMP.sql
# Keep only last 7 days
find $BACKUP_DIR -name "bmidb_*.sql" -mtime +7 -delete
```

Make executable and add to cron:
```bash
chmod +x /home/ubuntu/backup-db.sh

# Add to crontab (daily at 2 AM)
crontab -e
# Add this line:
0 2 * * * /home/ubuntu/backup-db.sh
```

### 10.5 Server Resource Monitoring

```bash
# Check disk space
df -h

# Check memory usage
free -h

# Check CPU usage
top
# Press 'q' to quit

# Check PM2 resource usage
pm2 monit

# Check system resources
htop  # (install with: sudo apt install htop)
```

---

## Part 11: Server Structure Overview

After successful deployment, your server structure looks like:

```
/home/ubuntu/
├── bmi-health-tracker/
│   ├── backend/
│   │   ├── src/
│   │   │   ├── server.js
│   │   │   ├── routes.js
│   │   │   ├── db.js
│   │   │   └── calculations.js
│   │   ├── migrations/
│   │   │   └── 001_create_measurements.sql
│   │   ├── .env                    # Environment variables
│   │   ├── package.json
│   │   └── node_modules/
│   └── frontend/
│       ├── dist/                   # Build output (copied to /var/www)
│       ├── src/
│       ├── package.json
│       └── node_modules/
│
/var/www/bmi-health-tracker/        # Production frontend files
├── index.html
├── assets/
│   ├── index-[hash].js
│   └── index-[hash].css
└── ...

/etc/nginx/
└── sites-available/
    └── bmi-health-tracker          # Nginx configuration

/var/log/nginx/
├── bmi-access.log                  # Access logs
└── bmi-error.log                   # Error logs
```

---

## Part 12: Security Best Practices

### 12.1 Keep System Updated

```bash
# Update regularly
sudo apt update && sudo apt upgrade -y

# Enable automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 12.2 Secure SSH

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Recommended changes:
# PermitRootLogin no
# PasswordAuthentication no
# Port 2222  # Change default port (optional)

# Restart SSH
sudo systemctl restart sshd
```

### 12.3 Use Strong Database Password

Ensure your PostgreSQL user has a strong password (at least 16 characters, mixed case, numbers, symbols).

### 12.4 Environment Variables

Never commit `.env` file to Git. Keep it secure with proper permissions:
```bash
chmod 600 /home/ubuntu/bmi-health-tracker/backend/.env
```

### 12.5 Regular Backups

- Backup database daily (see Part 10.4)
- Backup application code to Git
- Consider AWS snapshots for entire EC2 instance

---

## Part 13: Cost Optimization

### 13.1 EC2 Instance Sizing

- **Development/Testing**: t2.micro (1 vCPU, 1 GB RAM) - Free tier eligible
- **Low Traffic**: t2.small (1 vCPU, 2 GB RAM) - ~$17/month
- **Medium Traffic**: t2.medium (2 vCPU, 4 GB RAM) - ~$34/month

### 13.2 Use Reserved Instances

For production, consider 1-year Reserved Instances for ~40% savings.

### 13.3 Stop Instance When Not Needed

```bash
# From AWS Console or CLI
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
```

Note: You still pay for EBS storage when stopped.

---

## Deployment Complete! 🎉

Your BMI Health Tracker is now live and accessible!

### Quick Access Commands

```bash
# SSH to server
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP

# Check backend status
pm2 status

# View backend logs
pm2 logs bmi-backend

# Restart backend
pm2 restart bmi-backend

# Check Nginx status
sudo systemctl status nginx

# Reload Nginx config
sudo nginx -t && sudo systemctl reload nginx

# View database
psql -U bmi_user -d bmidb -h localhost
```

---

## Need Help?

**Common Resources:**
- AWS EC2 Documentation: https://docs.aws.amazon.com/ec2/
- Nginx Documentation: https://nginx.org/en/docs/
- PM2 Documentation: https://pm2.keymetrics.io/docs/
- PostgreSQL Documentation: https://www.postgresql.org/docs/

**Checklist:**
- [ ] EC2 instance running
- [ ] Security group configured (ports 22, 80, 443)
- [ ] Node.js and PostgreSQL installed
- [ ] Database created and migrated
- [ ] Backend running via PM2
- [ ] Frontend built and deployed
- [ ] Nginx configured and running
- [ ] Firewall enabled
- [ ] Application accessible via browser
- [ ] SSL certificate installed (optional)
- [ ] Backups configured

---

**Last Updated**: December 16, 2025  
**Version**: 2.1  
**Changes**: Updated to include measurement_date feature (Migration 002), enhanced testing procedures, and current project state
