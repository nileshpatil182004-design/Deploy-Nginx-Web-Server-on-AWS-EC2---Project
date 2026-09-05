# NGINX Web Server on AWS EC2 — Setup Guide

A step-by-step guide to launching an EC2 instance, installing NGINX, hosting a custom website, and monitoring logs.

---

## Table of Contents

- [Phase 1 — AWS EC2 Setup](#phase-1--aws-ec2-setup)
- [Phase 2 — NGINX Installation and Configuration](#phase-2--nginx-installation-and-configuration)
- [Prerequisites](#prerequisites)

---

## Prerequisites

- An AWS account (free tier works)
- A browser
- Basic terminal/SSH knowledge

---

## Phase 1 — AWS EC2 Setup

### Step 1.1 — Access AWS Console

1. Go to [https://aws.amazon.com](https://aws.amazon.com)
2. Sign in (or create a free tier account)
3. Select your preferred region from the top-right corner (e.g., `US East - N. Virginia`)

---

### Step 1.2 — Launch an EC2 Instance

1. Search for **EC2** in the AWS Console search bar
2. Click the orange **Launch Instance** button

![EC2 Instances Dashboard](images/ec2-instances.png)

> *EC2 Instances Dashboard — your running instance will show "Running" status with a green indicator, along with its Public IPv4 address.*

---

### Step 1.3 — Configure the Instance

**Name and OS**
- Name: `nginx-web-server`
- AMI: `Amazon Linux 2 AMI (HVM)` — Free tier eligible

**Instance Type**
- `t2.micro` — Free tier eligible (1 vCPU, 1 GB RAM)

**Key Pair**
- Click **Create new key pair**
- Name: `nginx-server-key`
- Type: `RSA`
- Format: `.pem` (Mac/Linux) or `.ppk` (Windows with PuTTY)
- Click **Create key pair** — file downloads automatically

> **Important:** Save this `.pem` file safely. You will need it for SSH access. It cannot be re-downloaded.

**Network Settings**
- Click **Edit** next to Network settings
- Auto-assign public IP: **Enable**
- Create a new security group named `nginx-webserver-sg`

| Rule | Type | Port | Source |
|------|------|------|--------|
| 1 | SSH | 22 | My IP (recommended) |
| 2 | HTTP | 80 | Anywhere `0.0.0.0/0` |
| 3 | HTTPS | 443 | Anywhere `0.0.0.0/0` |

**Storage**
- Leave default: 8 GB gp2

---

### Step 1.4 — Launch

1. Review settings in the **Summary** panel
2. Click **Launch instance**
3. Wait 1–2 minutes for the instance to start
4. Click **View all instances**

---

### Step 1.5 — Note Your Instance Details

Once Status Check shows `2/2 checks passed`:

- Copy your **Public IPv4 address** (e.g., `54.123.45.67`)
- This is referred to as `<YOUR-IP>` throughout this guide

---

## Phase 2 — NGINX Installation and Configuration

### Step 2 — Connect to Your EC2 Instance

**Option A: AWS Console (Easiest)**

1. In EC2 Dashboard, select your instance
2. Click **Connect** at the top
3. Go to the **EC2 Instance Connect** tab
4. Username: `ec2-user`
5. Click **Connect** — a browser terminal opens

**Option B: SSH from Your Computer**

```bash
# Mac/Linux
chmod 400 nginx-server-key.pem
ssh -i "nginx-server-key.pem" ec2-user@<YOUR-IP>

# Windows (Git Bash or WSL)
ssh -i "nginx-server-key.pem" ec2-user@<YOUR-IP>
```

---

### Step 3 — Install and Start NGINX

**3.1 Update system packages**

```bash
sudo yum update -y
```

**3.2 Install NGINX**

```bash
sudo yum install nginx -y
```

**3.3 Start NGINX**

```bash
# Start NGINX
sudo systemctl start nginx

# Enable auto-start on reboot
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

Expected output: `Active: active (running)` in green

**3.4 Test the web server**

Open a browser and go to:

```
http://<YOUR-IP>
```

You should see the default NGINX welcome page.

---

### Step 4 — Understand NGINX Configuration

**4.1 View the main config file**

```bash
cat /etc/nginx/nginx.conf
```

Key fields:

| Field | Description |
|-------|-------------|
| `user nginx;` | User NGINX runs as |
| `http { }` | HTTP configuration block |
| `server { }` | Virtual host configuration |
| `listen 80;` | Port NGINX listens on |
| `root /usr/share/nginx/html;` | Website files location |

**4.2 Test configuration syntax**

```bash
sudo nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**4.3 Common NGINX Commands**

```bash
sudo systemctl start nginx     # Start
sudo systemctl stop nginx      # Stop
sudo systemctl restart nginx   # Restart (downtime)
sudo systemctl reload nginx    # Reload config (no downtime)
sudo systemctl status nginx    # Check status
sudo nginx -t                  # Test config before applying
```

---

### Step 5 — Host a Custom Web Page

**5.1 Remove default files**

```bash
sudo rm -rf /usr/share/nginx/html/*
```

**5.2 Create website directory**

```bash
sudo mkdir -p /usr/share/nginx/html/mywebsite
```

**5.3 Create an HTML file**

```bash
sudo nano /usr/share/nginx/html/mywebsite/index.html
```

Paste this content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My First NGINX Site</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            text-align: center;
        }
        h1 { font-size: 3em; margin-bottom: 20px; }
        p { font-size: 1.2em; line-height: 1.6; }
        .container {
            background: rgba(255, 255, 255, 0.1);
            padding: 40px;
            border-radius: 10px;
            backdrop-filter: blur(10px);
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Hello from NGINX!</h1>
        <p>Welcome to my custom web server</p>
        <p>This page is served by NGINX on AWS EC2</p>
        <p><strong>Server IP:</strong> Replace-with-your-IP</p>
    </div>
</body>
</html>
```

Save: `Ctrl + X` → `Y` → `Enter`

**5.4 Set file permissions**

```bash
sudo chown -R ec2-user:ec2-user /usr/share/nginx/html
```

**5.5 Update NGINX config to point to your site**

```bash
sudo nano /etc/nginx/nginx.conf
```

Find the `server` block and make these changes:

```nginx
server {
    listen       80;
    listen       [::]:80;
    server_name  _;
    root         /usr/share/nginx/html/mywebsite;  # Updated path

    include /etc/nginx/default.d/*.conf;

    location / {
        try_files $uri $uri/ =404;  # Added line
    }
}
```

**5.6 Test and reload**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**5.7 View your custom site**

```
http://<YOUR-IP>
```

You should see the purple gradient page.

---

### Step 6 — Change the Default HTTP Port (Optional)

**6.1 Edit NGINX config**

```bash
sudo nano /etc/nginx/nginx.conf
```

Change:
```nginx
listen       80;
listen       [::]:80;
```

To:
```nginx
listen       8080;
listen       [::]:8080;
```

Save and exit.

**6.2 Update AWS Security Group**

1. Go to EC2 Dashboard → **Security Groups**
2. Select `nginx-webserver-sg`
3. Click **Edit inbound rules** → **Add rule**

| Type | Port | Source |
|------|------|--------|
| Custom TCP | 8080 | Anywhere `0.0.0.0/0` |

4. Click **Save rules**

**6.3 Reload NGINX**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**6.4 Test**

```
http://<YOUR-IP>:8080
```

> To revert: change `8080` back to `80` in `nginx.conf` and reload.

---

### Step 7 — Monitor NGINX Logs

**7.1 View access logs (live)**

```bash
sudo tail -f /var/log/nginx/access.log
```

Press `Ctrl + C` to stop.

**Access log format:**

```
54.123.45.67 - - [14/Feb/2026:10:30:45 +0000] "GET / HTTP/1.1" 200 512
```

| Part | Meaning |
|------|---------|
| `54.123.45.67` | Visitor's IP |
| `14/Feb/2026:10:30:45` | Timestamp |
| `GET /` | Requested page |
| `200` | HTTP status (success) |
| `512` | Response size in bytes |

**7.2 View error logs**

```bash
sudo tail -n 10 /var/log/nginx/error.log
```

**7.3 Test a 404 error**

Visit a non-existent page in browser:

```
http://<YOUR-IP>/this-page-does-not-exist
```

Then check:

```bash
sudo tail /var/log/nginx/error.log
```

You will see a `404 Not Found` entry.

---

## What You Have After This Guide

- A running AWS EC2 instance
- NGINX installed, configured, and serving a custom site
- Understanding of NGINX config structure and commands
- Log monitoring set up for access and error tracking

---

## Security Note

For production use:
- Restrict SSH access to `My IP` only, not `0.0.0.0/0`
- Set up HTTPS with a valid SSL certificate (e.g., Let's Encrypt)
- Regularly update system packages with `sudo yum update -y`
