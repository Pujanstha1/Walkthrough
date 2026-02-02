# Setting Up HTTPS on EC2 with NGINX

**Complete Guide for `pujanshrestha1.com.np`**

This guide walks you through setting up a secure HTTPS website on AWS EC2 using NGINX and Let's Encrypt SSL certificates (`certbot`). The process for both **Amazon Linux 2023** and **Ubuntu** is presented side by side so you can see the differences.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Domain Setup (Route 53)](#1-domain-setup-route-53)
3. [NGINX Installation](#2-nginx-installation)
4. [Configure NGINX Server Blocks](#3-configure-nginx-server-blocks)
5. [Install Certbot (Let's Encrypt)](#4-install-certbot-lets-encrypt)
6. [Obtain SSL Certificate](#5-obtain-ssl-certificate)
7. [Troubleshooting](#6-troubleshooting)
8. [Verify HTTPS](#7-verify-https)
9. [Auto-Renewal Setup](#8-auto-renewal-setup)
10. [Optional: Canonical Redirect](#9-optional-canonical-redirect)
11. [Summary of Differences](#summary-of-os-differences)

---

## Prerequisites

Before you begin, make sure you have:

- An AWS account
- An EC2 instance running (Amazon Linux 2023 OR Ubuntu)
- An Elastic IP attached to your EC2 instance
- A domain name registered (i am taking the example of one of my domain i.e. `pujanshrestha1.com.np`)
- SSH access to your EC2 instance
- Security Group configured to allow:
  - Port 22 (SSH)
  - Port 80 (HTTP)
  - Port 443 (HTTPS)

---

## 1. Domain Setup (Route 53)

This step is **identical** for both Amazon Linux and Ubuntu since it's done in the AWS Console, not on your server.

### Step 1.1: Create a Hosted Zone

1. Go to **AWS Route 53** console
2. Click **"Hosted zones"** → **"Create hosted zone"**
3. Enter your domain name: `pujanshrestha1.com.np`
4. Click **"Create hosted zone"**

### Step 1.2: Note Your Name Servers

AWS will provide 4 Name Server (NS) values. They'll look like:

```
ns-86.awsdns-10.com
ns-750.awsdns-29.net
ns-1777.awsdns-30.co.uk
ns-1496.awsdns-59.org
```

### Step 1.3: Update Your Domain Registrar (This points your domain to AWS Route 53.)

1. Go to your domain registrar (where you bought `pujanshrestha1.com.np`)
2. Find the DNS or Name Server settings
3. Replace the existing name servers with **at least 2** of the NS values from AWS
4. Save the changes

> ⏱️ **Wait time**: DNS propagation takes 5-15 minutes (sometimes up to 48 hours)

### Step 1.4: Create A Records

Back in Route 53, add these records:

| Record Name | Type | Value |
|-------------|------|-------|
| `pujanshrestha1.com.np` | A | Your EC2 Elastic IP |
| `www.pujanshrestha1.com.np` | A | Your EC2 Elastic IP |

**How to add:**
1. Click **"Create record"**
2. Enter the record name (leave blank for root domain, enter `www` for www subdomain)
3. Select **"A – Routes traffic to an IPv4 address"**
4. Enter your EC2 Elastic IP
5. Click **"Create records"**

### Step 1.5: Test DNS Resolution

From your local computer (not the EC2 instance):

```bash
ping pujanshrestha1.com.np
ping www.pujanshrestha1.com.np
```

> **Note**: Ping might fail if ICMP is blocked in your security group, but that's okay. The important thing is that DNS resolves to your IP.

---

## 2. NGINX Installation

Now we connect to our EC2 instance. The installation commands differ between operating systems.

### SSH into Your EC2 Instance

```bash
ssh -i your-key.pem ec2-user@your-ec2-ip     # Amazon Linux
# OR
ssh -i your-key.pem ubuntu@your-ec2-ip        # Ubuntu
```

### Install NGINX

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

```bash
# Update package lists
sudo dnf update -y

# Install NGINX
sudo dnf install nginx -y

# Start NGINX
sudo systemctl start nginx

# Enable NGINX to start on boot
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

</td>
<td>

```bash
# Update package lists
sudo apt update -y

# Install NGINX
sudo apt install nginx -y

# Start NGINX
sudo systemctl start nginx

# Enable NGINX to start on boot
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

</td>
</tr>
</table>

**Expected output**: You should see `active (running)` in green.

### Test NGINX in Browser

Open your browser and visit:
```
http://your-ec2-elastic-ip
```

You should see the default NGINX welcome page.

### Understanding Default Directories

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

**Default web root:**
```
/usr/share/nginx/html
```

**Config file:**
```
/etc/nginx/nginx.conf
```

**Additional configs:**
```
/etc/nginx/conf.d/
```

</td>
<td>

**Default web root:**
```
/var/www/html
```

**Config file:**
```
/etc/nginx/nginx.conf
```

**Site configs:**
```
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
```

</td>
</tr>
</table>

---

## 3. Configure NGINX Server Blocks

We need to configure NGINX to handle our domain name and prepare for HTTPS.

### Step 3.1: Edit NGINX Configuration

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

```bash
# Option 1: Edit main config
sudo nano /etc/nginx/nginx.conf

# Option 2: Create new config (recommended)
sudo nano /etc/nginx/conf.d/pujanshrestha1.conf
```

</td>
<td>

```bash
# Edit the default site configuration
sudo nano /etc/nginx/sites-available/default
```

</td>
</tr>
</table>

### Step 3.2: Add Server Block Configuration

Add this configuration (works for both OS):

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name pujanshrestha1.com.np www.pujanshrestha1.com.np;
    
    # Temporary location for initial setup
    root /usr/share/nginx/html;  # Amazon Linux
    # root /var/www/html;         # Ubuntu - uncomment this and comment above
    
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

> **Important**: The `server_name` directive **must** include both the `root domain` and the `www` subdomain.

### Step 3.3: Test Configuration

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

```bash
# Test NGINX syntax
sudo nginx -t

# If successful, reload NGINX
sudo systemctl reload nginx
```

</td>
<td>

```bash
# Test NGINX syntax
sudo nginx -t

# If successful, reload NGINX
sudo systemctl reload nginx
```

</td>
</tr>
</table>

**Expected output**:
```
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Step 3.4: Test Your Domain

Visit in browser:
```
http://pujanshrestha1.com.np
http://www.pujanshrestha1.com.np
```

You should see the NGINX welcome page.

---

## 4. Install Certbot (Let's Encrypt)

Certbot automates SSL certificate installation.

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

```bash
# Install Certbot and NGINX plugin
sudo dnf install certbot python3-certbot-nginx -y

# Verify installation
certbot --version
```

</td>
<td>

```bash
# Install Certbot and NGINX plugin
sudo apt install certbot python3-certbot-nginx -y

# Verify installation
certbot --version
```

</td>
</tr>
</table>

**Expected output**: Something like `certbot 2.x.x`

### Key Differences Explained

| Feature | Amazon Linux 2023 | Ubuntu |
|---------|-------------------|--------|
| Package Manager | `dnf` | `apt` |
| Extra Repositories | Not needed | Not needed |
| Installation Command | `dnf install` | `apt install` |

> Modern versions of both OS have Certbot in their default repositories, so no special setup is needed!

---

## 5. Obtain SSL Certificate

Now we'll get a free SSL certificate from Let's Encrypt.

### Step 5.1: Run Certbot

**Same command for both OS:**

```bash
sudo certbot --nginx -d pujanshrestha1.com.np -d www.pujanshrestha1.com.np
```

### Step 5.2: Follow the Prompts

You'll be asked several questions:

1. **Enter email address**: Used for urgent renewal and security notices
   ```
   Enter email address (used for urgent renewal and security notices): your-email@example.com
   ```

2. **Agree to Terms of Service**: Type `Y` and press Enter
   ```
   Please read the Terms of Service at https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf
   (A)gree/(C)ancel: A
   ```

3. **Share email with EFF** (optional): Type `Y` or `N`
   ```
   Would you be willing to share your email address with EFF? (Y)es/(N)o: N
   ```

4. **Redirect HTTP to HTTPS**: Choose option 2 (recommended)
   ```
   Please choose whether or not to redirect HTTP traffic to HTTPS
   1: No redirect
   2: Redirect - Make all requests redirect to secure HTTPS access
   Select the appropriate number [1-2] then [enter]: 2
   ```

### Step 5.3: Success!

You should see:

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/pujanshrestha1.com.np/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/pujanshrestha1.com.np/privkey.pem
```

---

## 6. Troubleshooting

### Issue: "Could not automatically find a matching server block for www.pujanshrestha1.com.np"

This means Certbot couldn't find both domains in your NGINX config.

**Solution:**

1. Edit your NGINX config again:

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

```bash
sudo nano /etc/nginx/conf.d/pujanshrestha1.conf
```

</td>
<td>

```bash
sudo nano /etc/nginx/sites-available/default
```

</td>
</tr>
</table>

2. Make sure `server_name` includes both domains:

```nginx
server_name pujanshrestha1.com.np www.pujanshrestha1.com.np;
```

3. Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

4. Retry Certbot installation:

```bash
sudo certbot install --cert-name pujanshrestha1.com.np
```

### Issue: Port 80 or 443 not accessible

Check your AWS Security Group:

1. Go to EC2 → Security Groups
2. Select your instance's security group
3. Click "Edit inbound rules"
4. Make sure you have:
   - Port 80 (HTTP) from 0.0.0.0/0
   - Port 443 (HTTPS) from 0.0.0.0/0

---

## 7. Verify HTTPS

### Step 7.1: Test in Browser

Open these URLs:

```
https://pujanshrestha1.com.np
https://www.pujanshrestha1.com.np
http://pujanshrestha1.com.np   (should redirect to HTTPS)
http://www.pujanshrestha1.com.np   (should redirect to HTTPS)
```

### Step 7.2: Check for the Lock Icon

You should see:
- 🔒 A padlock icon in the address bar
- "Connection is secure" when you click the padlock
- Certificate valid for both `pujanshrestha1.com.np` and `www.pujanshrestha1.com.np`

### Step 7.3: Test SSL Configuration

Use SSL Labs to test your configuration:

```
https://www.ssllabs.com/ssltest/analyze.html?d=pujanshrestha1.com.np
```

You should get an **A** or **A+** rating!

---

## 8. Auto-Renewal Setup

Let's Encrypt certificates expire every 90 days. Certbot sets up automatic renewal.

### Check Renewal Configuration

**Same for both OS:**

```bash
# Test renewal process (doesn't actually renew)
sudo certbot renew --dry-run
```

**Expected output**:
```
Congratulations, all simulated renewals succeeded
```

### Understanding Auto-Renewal

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

Certbot creates a systemd timer:

```bash
# Check renewal timer
sudo systemctl status certbot-renew.timer

# View timer details
sudo systemctl list-timers
```

</td>
<td>

Certbot creates a systemd timer:

```bash
# Check renewal timer
sudo systemctl status certbot.timer

# View timer details
sudo systemctl list-timers
```

</td>
</tr>
</table>

> You don't need to do anything! Certbot will automatically renew certificates when they're close to expiring.

---

## 9. Optional: Canonical Redirect

If you want all traffic to go to the non-www version (e.g., `https://pujanshrestha1.com.np`), add this configuration.

### Step 9.1: Edit NGINX Config

<table>
<tr>
<th>Amazon Linux 2023</th>
<th>Ubuntu</th>
</tr>
<tr>
<td>

```bash
sudo nano /etc/nginx/conf.d/pujanshrestha1.conf
```

</td>
<td>

```bash
sudo nano /etc/nginx/sites-available/default
```

</td>
</tr>
</table>

### Step 9.2: Add Redirect Block

Add this **before** your main server block:

```nginx
# Redirect www to non-www (HTTP)
server {
    listen 80;
    listen [::]:80;
    server_name www.pujanshrestha1.com.np;
    return 301 https://pujanshrestha1.com.np$request_uri;
}

# Redirect www to non-www (HTTPS)
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name www.pujanshrestha1.com.np;
    
    ssl_certificate /etc/letsencrypt/live/pujanshrestha1.com.np/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pujanshrestha1.com.np/privkey.pem;
    
    return 301 https://pujanshrestha1.com.np$request_uri;
}

# Main server block (non-www only)
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name pujanshrestha1.com.np;
    
    ssl_certificate /etc/letsencrypt/live/pujanshrestha1.com.np/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pujanshrestha1.com.np/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    
    root /usr/share/nginx/html;  # Amazon Linux
    # root /var/www/html;         # Ubuntu
    
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Step 9.3: Test and Reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Now all URLs redirect to `https://pujanshrestha1.com.np`!

---

## Summary of OS Differences

| Feature | Amazon Linux 2023 | Ubuntu |
|---------|-------------------|--------|
| **Package Manager** | `dnf` | `apt` |
| **Update Command** | `sudo dnf update -y` | `sudo apt update -y` |
| **Install NGINX** | `sudo dnf install nginx -y` | `sudo apt install nginx -y` |
| **Install Certbot** | `sudo dnf install certbot python3-certbot-nginx -y` | `sudo apt install certbot python3-certbot-nginx -y` |
| **Default Web Root** | `/usr/share/nginx/html` | `/var/www/html` |
| **Main Config** | `/etc/nginx/nginx.conf` | `/etc/nginx/nginx.conf` |
| **Additional Configs** | `/etc/nginx/conf.d/*.conf` | `/etc/nginx/sites-available/` |
| **Default User** | `ec2-user` | `ubuntu` |
| **Certbot Repo Setup** | Not needed | Not needed |

### What's the Same?

- Certbot commands
- NGINX configuration syntax
- SSL certificate locations
- Testing and verification steps
- Auto-renewal setup

---

## Quick Reference Commands

### NGINX Commands (Same for Both)

```bash
# Test configuration
sudo nginx -t

# Reload NGINX (without downtime)
sudo systemctl reload nginx

# Restart NGINX (with downtime)
sudo systemctl restart nginx

# Stop NGINX
sudo systemctl stop nginx

# Start NGINX
sudo systemctl start nginx

# Check status
sudo systemctl status nginx

# View error logs
sudo tail -f /var/log/nginx/error.log

# View access logs
sudo tail -f /var/log/nginx/access.log
```

### Certbot Commands (Same for Both)

```bash
# Get/renew certificate
sudo certbot --nginx -d pujanshrestha1.com.np -d www.pujanshrestha1.com.np

# Test renewal
sudo certbot renew --dry-run

# List certificates
sudo certbot certificates

# Renew certificates manually
sudo certbot renew

# Delete a certificate
sudo certbot delete --cert-name pujanshrestha1.com.np
```

---

## Congratulations!

You now have a fully secure HTTPS website running on AWS EC2 with NGINX!

### What You've Accomplished

- Configured DNS in Route 53
- Installed and configured NGINX
- Obtained a free SSL certificate
- Set up automatic HTTPS redirects
- Configured automatic certificate renewal

### Next Steps

1. Upload your website files to the web root directory
2. Configure additional NGINX features (caching, compression, etc.)
3. Set up monitoring and backups
4. Consider adding a firewall (UFW on Ubuntu, firewalld on Amazon Linux)

---

## Additional Resources

- 📚 [Certbot Documentation](https://certbot.eff.org/docs/)
- 📚 [NGINX Documentation](https://nginx.org/en/docs/)
- 📚 [AWS Route 53 Documentation](https://docs.aws.amazon.com/route53/)
- 📚 [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- 📚 [SSL Labs Testing Tool](https://www.ssllabs.com/ssltest/)

---

## Troubleshooting Common Issues

### Issue: "Failed to connect" errors

**Cause**: Security group not configured properly

**Solution**: 
1. Go to EC2 → Security Groups
2. Add inbound rules for ports 80 and 443
3. Allow from `0.0.0.0/0` (anywhere)

### Issue: "502 Bad Gateway"

**Cause**: Your application isn't running or NGINX can't connect to it

**Solution**:
```bash
# Check NGINX logs
sudo tail -f /var/log/nginx/error.log

# Restart NGINX
sudo systemctl restart nginx
```

### Issue: Certificate not found after installation

**Cause**: Certbot didn't update NGINX config

**Solution**:
```bash
# Manually install certificate to NGINX
sudo certbot install --cert-name pujanshrestha1.com.np
```

### Issue: DNS not resolving

**Cause**: DNS propagation delay or incorrect NS records

**Solution**:
1. Wait up to 48 hours for full propagation
2. Verify NS records at your registrar match Route 53
3. Test with: `nslookup pujanshrestha1.com.np`

---

## Support

If you run into issues:

1. Check the logs: `sudo tail -f /var/log/nginx/error.log`
2. Verify DNS: `nslookup pujanshrestha1.com.np`
3. Test SSL: https://www.ssllabs.com/ssltest/
4. Review security groups in AWS Console

---
