# AirBNB Clone — EC2 Deployment Guide
**Server:** AWS EC2 t3.medium · Ubuntu 24.04 LTS  
**Stack:** Angular 15 (frontend) · Node.js/Express/TypeScript (backend) · MongoDB Atlas · Nginx · PM2

> **Architecture note:** In production the Angular app uses relative URLs (`/api/` and same-origin sockets). This means the backend **must run on the same server**. Nginx acts as a reverse proxy — it serves the Angular static files and forwards `/api/` and `/socket.io/` requests to the Node.js backend running on port 3030.

---

## Table of Contents
1. [AWS Prerequisites](#1-aws-prerequisites)
2. [Connect to the Server](#2-connect-to-the-server)
3. [System Setup](#3-system-setup)
4. [Install Node.js 22](#4-install-nodejs-22)
5. [MongoDB — Atlas or Local](#5-mongodb--atlas-cloud-or-local)
6. [Install PM2 & Nginx](#6-install-pm2--nginx)
7. [Deploy the Backend](#7-deploy-the-backend)
8. [Deploy the Frontend](#8-deploy-the-frontend)
9. [Configure Nginx](#9-configure-nginx)
10. [PM2 Startup on Reboot](#10-pm2-startup-on-reboot)
11. [(Optional) SSL with Let's Encrypt](#11-optional-ssl-with-lets-encrypt)
12. [Sanity Checks](#12-sanity-checks)

---

## 1. AWS Prerequisites

### Security Group Inbound Rules
Open the following ports on your EC2 instance's Security Group before connecting:

| Type       | Port | Source    | Purpose                        |
|------------|------|-----------|--------------------------------|
| SSH        | 22   | Your IP   | Remote access                  |
| HTTP       | 80   | 0.0.0.0/0 | Web traffic                    |
| HTTPS      | 443  | 0.0.0.0/0 | SSL web traffic (if using SSL) |

> Port 3030 (the backend) should **not** be opened publicly — Nginx proxies to it internally.

### Note your EC2 Public IP / Domain
You'll need this in Step 9. Get it from the AWS console under **EC2 → Instances → Public IPv4 address** (e.g. `54.123.45.67`) or your custom domain if you've set one up.

---

## 2. Connect to the Server

From your local machine:

```bash
chmod 400 /path/to/your-key.pem
ssh -i /path/to/your-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>
```

---

## 3. System Setup

Update all packages first:

```bash
sudo apt update && sudo apt upgrade -y
```

Install essential build tools:

```bash
sudo apt install -y git curl build-essential
```

---

## 4. Install Node.js 22

Angular 15 requires Node.js 16+. We'll install Node 22 LTS via NodeSource:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify the installation:

```bash
node -v   # Should print v22.x.x
npm -v    # Should print 10.x.x or similar
```

---

## 5. MongoDB — Atlas (Cloud) or Local

The backend ships configured for **MongoDB Atlas**, but you can use a local MongoDB instance instead. Choose one option.

### Option A — MongoDB Atlas (recommended)

No installation needed. You'll update the connection string in Step 7c with your own Atlas credentials.

> Don't have a cluster yet? Create a free one at [cloud.mongodb.com](https://cloud.mongodb.com). After creating it, go to **Database Access** to add a user and **Network Access** to whitelist your EC2 IP (or `0.0.0.0/0` for open access). Copy the connection string — you'll need it in Step 7c.

### Option B — Local MongoDB

Install MongoDB Community Edition on the server:

```bash
# Import the MongoDB GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Add the MongoDB repo (Ubuntu 24.04 uses the 'noble' codename)
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Install
sudo apt update
sudo apt install -y mongodb-org
```

Start MongoDB and enable it on boot:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod
# Look for: Active: active (running)
```

Then in Step 7c, set `config/prod.ts` to:

```typescript
module.exports = {
  dbURL: 'mongodb://127.0.0.1:27017/stayDB',
  dbName: 'stayDB'
}
```

Same structure as the Atlas version — just swap the `mongodb+srv://...` URL for `mongodb://127.0.0.1:27017/stayDB`. No username or password needed for a local instance by default.

---

## 6. Install PM2 & Nginx

**PM2** keeps the Node.js backend running as a daemon and restarts it on crashes:

```bash
sudo npm install -g pm2
```

**Nginx** serves the frontend and proxies API calls to the backend:

```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

Test Nginx is up — open `http://<YOUR_EC2_PUBLIC_IP>` in a browser. You should see the default Nginx welcome page.

---

## 7. Deploy the Backend

### 7a. Clone the backend repository

```bash
cd ~
git clone https://github.com/emraay-devops/AirBNB-backend.git
cd AirBNB-backend
```

### 7b. Install dependencies

```bash
npm install
```

### 7c. Update the MongoDB Atlas connection string

The backend hardcodes the Atlas URL in `config/prod.ts`. Open it and replace it with your own Atlas connection string:

```bash
nano config/prod.ts
```

Replace the `dbURL` value:

```typescript
module.exports = {
  dbURL: 'mongodb+srv://<YOUR_USER>:<YOUR_PASSWORD>@<YOUR_CLUSTER>.mongodb.net/?retryWrites=true&w=majority',
  dbName: 'stayDB'
}
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

> **Don't have an Atlas cluster?** Create a free one at [cloud.mongodb.com](https://cloud.mongodb.com). After creating a cluster, go to **Database Access** to add a user and **Network Access** to whitelist your EC2 IP (or `0.0.0.0/0` for open access).

### 7d. Create the environment file

```bash
nano .env
```

Paste the following:

```env
PORT=3030
NODE_ENV=production
SECRET1=replace_with_a_long_random_string
```

> Generate a strong `SECRET1` with:
> ```bash
> node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
> ```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

### 7e. Start the backend with PM2

The backend is TypeScript — use `ts-node` (already installed as a dev dependency) to run it:

```bash
pm2 start server.ts --interpreter $(which ts-node) --name airbnb-backend
```

Confirm it's running:

```bash
pm2 status
# Should show airbnb-backend with status: online
```

Check logs for any startup errors:

```bash
pm2 logs airbnb-backend --lines 30
```

Quick API test (should return JSON, not an error):

```bash
curl http://localhost:3030/api/stay
```

---

## 8. Deploy the Frontend

### 8a. Clone the frontend repository

```bash
cd ~
git clone https://github.com/emraay-devops/AirBNB-Frontend-clone.git
cd AirBNB-Frontend-clone
```

### 8b. Install dependencies

```bash
npm install
```

> This may take 2–3 minutes on first run. The `postinstall` script runs `ngcc` (Angular compatibility compiler) which is expected.

### 8c. Install the Angular CLI globally

```bash
sudo npm install -g @angular/cli@15
```

Confirm:

```bash
ng version
```

### 8d. Build for production

```bash
ng build --configuration production
```

> This will take 3–5 minutes. The output goes to `dist/airbnb/`.

Confirm the build output exists:

```bash
ls dist/airbnb/
# You should see: index.html, main.*.js, styles.*.css, etc.
```

### 8e. Copy build output to Nginx web root

```bash
sudo mkdir -p /var/www/airbnb
sudo cp -r dist/airbnb/. /var/www/airbnb/
sudo chown -R www-data:www-data /var/www/airbnb
```

---

## 9. Configure Nginx

### 9a. Create the site config

```bash
sudo nano /etc/nginx/sites-available/airbnb
```

Paste the following (replace `YOUR_EC2_PUBLIC_IP_OR_DOMAIN` with your actual IP or domain):

```nginx
server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP_OR_DOMAIN;

    root /var/www/airbnb;
    index index.html;

    # Serve Angular app — fallback to index.html for client-side routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy REST API calls to the Node.js backend
    location /api/ {
        proxy_pass http://localhost:3030;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Proxy Socket.io (for real-time notifications)
    location /socket.io/ {
        proxy_pass http://localhost:3030;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }

    # Serve service worker at root (required for Angular PWA)
    location /ngsw.json {
        add_header Cache-Control "no-cache";
        try_files $uri =404;
    }

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 9b. Enable the site

```bash
# Enable our new config
sudo ln -s /etc/nginx/sites-available/airbnb /etc/nginx/sites-enabled/

# Remove the default site to avoid conflicts
sudo rm /etc/nginx/sites-enabled/default
```

### 9c. Test and reload Nginx

```bash
sudo nginx -t
# Expected: syntax is ok / test is successful
```

```bash
sudo systemctl reload nginx
```

---

## 10. PM2 Startup on Reboot

Make sure the backend auto-starts if the server reboots:

```bash
pm2 startup
```

This prints a command starting with `sudo env PATH=...` — **copy and run that exact command** from your terminal.

Then save the current PM2 process list:

```bash
pm2 save
```

---

## 11. (Optional) SSL with Let's Encrypt

If you have a domain name pointed at your EC2 IP, add free HTTPS:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow the interactive prompts. Certbot will automatically update your Nginx config for HTTPS and set up auto-renewal.

Test auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

## 12. Sanity Checks

Run these to confirm everything is wired up correctly:

```bash
# 1. Backend is running and responding
curl -s http://localhost:3030/api/stay | head -c 200

# 3. Nginx is running
sudo systemctl status nginx | grep Active

# 4. Nginx config is valid
sudo nginx -t

# 5. PM2 shows the backend as online
pm2 status

# 6. Check backend logs for errors
pm2 logs airbnb-backend --lines 50

# 7. The frontend is being served (should return HTML)
curl -s http://localhost | head -c 500

# 8. API is reachable through Nginx (the proxy works)
curl -s http://localhost/api/stay | head -c 200
```

Then open `http://<YOUR_EC2_PUBLIC_IP>` in a browser — you should see the AirBNB clone homepage.

---

## Troubleshooting

**Blank page / 404 on Angular routes**  
Make sure the `try_files $uri $uri/ /index.html;` line is in your Nginx config. This is required for Angular's client-side router.

**`/api/` returns 502 Bad Gateway**  
The backend isn't running. Check `pm2 status` and `pm2 logs airbnb-backend`.

**Socket.io not connecting**  
Confirm the `/socket.io/` proxy block is in your Nginx config with the `Upgrade` and `Connection` headers.

**`ng build` runs out of memory**  
t3.medium has 4GB RAM which should be sufficient, but if it fails:
```bash
export NODE_OPTIONS="--max_old_space_size=3072"
ng build --configuration production
```

**Nginx permission denied on `/var/www/airbnb`**  
```bash
sudo chown -R www-data:www-data /var/www/airbnb
sudo chmod -R 755 /var/www/airbnb
```
