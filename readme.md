# Node HelloWorld on AWS EC2 with Nginx and GitHub Actions CI/CD

This project demonstrates how to deploy a minimal **Node.js “Hello World” web application** on an **AWS EC2 instance**, use **Nginx as a reverse proxy**, secure it with **HTTPS**, and implement **CI/CD using GitHub Actions**.

The pipeline automatically deploys new code to EC2 whenever changes are pushed to the `main` branch.

---

# Project Objectives

The goal of this setup is to understand, end-to-end, how to:

* Host a Node.js application on an EC2 Linux server.
* Use Nginx as a reverse proxy with a custom domain and SSL.
* Automate deployments using GitHub Actions over SSH.

---

# Architecture Overview

```
Client Browser
      │
      │ HTTPS Request
      ▼
Custom Domain (DuckDNS / Domain Provider)
      │
      ▼
Nginx Reverse Proxy (Port 80 / 443)
      │
      │ forwards request
      ▼
Node.js Application (Port 3000)
      │
      ▼
AWS EC2 Server
```

### Flow

1. The **client sends an HTTPS request** to the domain.
2. **Nginx receives the request on port 443** and handles SSL termination.
3. Nginx **forwards the request to the Node.js server running on port 3000**.
4. The **Node.js app responds with Hello World**.
5. **GitHub Actions automatically deploys updates** whenever code is pushed.

---

# Step 1: Create the Node.js Application

We start with a minimal HTTP server.

### index.js

```javascript
const http = require("http");

const PORT = 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hello, World from Node.js on AWS!");
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

This simple application allows us to easily verify deployments.

---

# Step 2: Push Code to GitHub

Initialize Git and push the repository.

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

GitHub will store the code and trigger CI/CD workflows.

---

# Step 3: Provision an AWS EC2 Instance

Launch an EC2 instance from the AWS console.

Recommended configuration:

* Instance type: **t2.micro**
* OS: **Amazon Linux**
* Key pair: **Create a new `.pem` key**
* Security group ports:

```
22  → SSH
80  → HTTP
443 → HTTPS
```

Connect to the instance using:

```bash
ssh -i your-key.pem ec2-user@<ec2-public-ip>
```

---

# Step 4: Install Required Software

Inside the EC2 instance install:

### Update packages

```bash
sudo yum update -y
```

### Install Node.js

```bash
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install -y nodejs
```

### Install Git

```bash
sudo yum install git -y
```

### Install Nginx

```bash
sudo yum install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

# Step 5: Clone Repository and Run App

Clone the GitHub repository:

```bash
cd ~
git clone https://github.com/<your-username>/<your-repo>.git Node-HelloWorld
cd Node-HelloWorld
node index.js
```

The application will run on:

```
http://localhost:3000
```

---

# Step 6: Configure Nginx Reverse Proxy

Create an Nginx configuration.

```bash
sudo nano /etc/nginx/conf.d/nodeapp.conf
```

Add:

```nginx
server {
    listen 80;
    server_name your-domain.duckdns.org;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

Now requests to the server will be forwarded to the Node app.

---

# Step 7: Configure HTTPS with Let's Encrypt

Install Certbot:

```bash
sudo yum install certbot python3-certbot-nginx -y
```

Run certificate generation:

```bash
sudo certbot --nginx -d your-domain.duckdns.org
```

Certbot will:

* Generate SSL certificates
* Configure Nginx automatically
* Enable HTTPS redirection

Now your application will run securely on:

```
https://your-domain.duckdns.org
```

---

# Step 8: Generate SSH Key for GitHub Actions

Create a dedicated key:

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions-ec2" -f ec2-github-actions -N ""
```

Add the public key to EC2:

```bash
cat ec2-github-actions.pub >> ~/.ssh/authorized_keys
```

---

# Step 9: Add GitHub Secrets

In the GitHub repository:

```
Settings → Secrets → Actions
```

Create these secrets:

| Secret   | Value                                      |
| -------- | ------------------------------------------ |
| EC2_HOST | EC2 public IP                              |
| EC2_USER | ec2-user                                   |
| EC2_KEY  | contents of ec2-github-actions private key |

These are securely used by GitHub Actions.

---

# Step 10: GitHub Actions CI/CD Pipeline

Create the workflow:

```
.github/workflows/deploy.yml
```

### deploy.yml

```yaml
name: Deploy to EC2

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Add SSH key
        run: |
          echo "${EC2_KEY}" > ec2-key.pem
          chmod 600 ec2-key.pem
        env:
          EC2_KEY: ${{ secrets.EC2_KEY }}

      - name: Deploy over SSH
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USER: ${{ secrets.EC2_USER }}
        run: |
          ssh -o StrictHostKeyChecking=no -i ec2-key.pem ${EC2_USER}@${EC2_HOST} '
            cd ~/Node-HelloWorld &&
            git pull origin main &&
            pkill node || true &&
            setsid nohup node index.js > app.log 2>&1 < /dev/null &
          '
```

---

# How the CI/CD Pipeline Works

1. Code is pushed to the `main` branch.
2. GitHub Actions workflow starts automatically.
3. The workflow connects to EC2 via SSH.
4. Latest code is pulled from GitHub.
5. Existing Node process is stopped.
6. Application restarts in the background.

This ensures **automatic deployment on every push**.

---

# Final Result

The application is accessible at:

```
https://your-domain.duckdns.org
```

Features implemented:

* Node.js Hello World web server
* AWS EC2 hosting
* Nginx reverse proxy
* HTTPS with Let's Encrypt
* CI/CD with GitHub Actions
* Automatic deployment

---

# What I Learned

* How to deploy Node.js applications on AWS EC2.
* Configuring Nginx as a reverse proxy.
* Setting up HTTPS with Let's Encrypt.
* Managing SSH authentication securely.
* Implementing CI/CD pipelines using GitHub Actions.

---

# Future Improvements

* Use **PM2** for better process management.
* Add **Docker containerization**.
* Implement **load balancing with AWS ALB**.
* Add monitoring with **CloudWatch or Prometheus**.

---

# Author

Vinay
DevOps Deployment Project
