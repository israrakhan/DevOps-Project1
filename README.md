# DevOps-Project1

<img width="959" height="715" alt="Devops1" src="https://github.com/user-attachments/assets/f34bb249-b91b-4a33-a825-7f86c6f58fbb" />
# DevOps Engineer Challenge – README

## Project Overview

This repository documents the completion of a full DevOps workflow for a React-based chat application, deployed on AWS EC2 with CI/CD, reverse proxy, security hardening, and containerization. [file:1]

The original application source is cloned from:  
`https://github.com/riyanuddin17/DevOpsChallenge.git` [file:1]

---

## Architecture

- AWS EC2 Ubuntu instance as the main host. [file:1]  
- React app built with Vite, deployed via **pm2** and also Dockerized. [file:1]  
- Nginx as reverse proxy for the React app. [file:1]  
- Jenkins for CI/CD pipeline (build, deploy, upload to S3). [file:1]  
- UFW firewall configured to allow only 22, 80, 443, 3000. [file:1]  

---

## Prerequisites

- AWS EC2 instance (Ubuntu). [file:1]  
- SSH access as `ubuntu` (or equivalent sudo user). [file:1]  
- AWS S3 bucket and credentials (for Jenkins S3 upload step). [file:1]  

---

## Directory Structure

- `/opt/checkout/chat-app` – Git checkout of the React app. [file:1]  
- `/opt/deployment/react` – Final built static files (`dist`). [file:1]  
- `/var/www/html` or Nginx default root – used via Nginx reverse proxy to pm2 / Docker app. [file:1]  

---

## Part 1: Create sudo user

1. Create user `DevOps`:
   ```bash
   sudo adduser DevOps
Add to sudo group:

bash
sudo usermod -aG sudo DevOps
Optional sudoers entry (passwordless for specific commands): [file:1]

bash
sudo visudo

DevOps  ALL=(ALL:ALL) ALL
jenkins ALL=(ALL) NOPASSWD: /usr/bin/pm2, /usr/bin/git, /usr/bin/npm
Part 2: React App – Checkout, Build, Deploy with pm2
2.1 Install Node.js and npm
Example (Node 18 on Ubuntu): [file:1]

bash
sudo apt update
sudo apt install -y nodejs npm
node --version
npm --version
2.2 Clone the project
bash
sudo mkdir -p /opt/checkout
cd /opt/checkout
sudo git clone https://github.com/riyanuddin17/DevOpsChallenge.git chat-app
cd chat-app
[file:1]

2.3 Build the React app
bash
sudo npm install
sudo npm run build      # generates dist/
[file:1]

2.4 Move build artifacts
bash
sudo mkdir -p /opt/deployment/react
sudo cp -r dist/* /opt/deployment/react/
sudo chown -R DevOps:DevOps /opt/deployment/react
[file:1]

2.5 Run with pm2
bash
sudo npm install -g pm2
cd /opt/deployment/react
pm2 serve . 3000 --name chat-app --spa
pm2 save
pm2 startup systemd
[file:1]

Part 3: Nginx Reverse Proxy and UFW
3.1 Install and enable Nginx
bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
[file:1]

3.2 Nginx proxy configuration
Create site config (e.g. /etc/nginx/sites-available/chat-app):

text
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
Enable and reload:

bash
sudo ln -s /etc/nginx/sites-available/chat-app /etc/nginx/sites-enabled/chat-app
sudo nginx -t
sudo systemctl reload nginx
Now visiting http://<EC2_PUBLIC_IP> shows the chat app. [file:1]

3.3 Configure UFW firewall
bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3000/tcp

sudo ufw enable
sudo ufw status verbose
[file:1]

Part 4: Jenkins CI/CD
4.1 Install Jenkins and Java
bash
sudo apt update
sudo apt install -y openjdk-11-jdk
java --version
[file:1]

Then install Jenkins (standard Debian/Ubuntu steps) and start service: [file:1]

bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
Access Jenkins at http://<IP_ADDRESS>:8080 and unlock with password from: [file:1]

bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
4.2 Jenkins plugins for S3
In Jenkins UI: Manage Jenkins → Plugins → Available/Installed.
Install AWS-related plugins (e.g. Amazon Web Services SDK v2, EC2, S3). [file:1]

4.3 Jenkins pipeline job
Create a Pipeline job chat-app-pipeline with “Pipeline script from SCM” and set Repo URL to https://github.com/riyanuddin17/DevOpsChallenge.git. [file:1]

Example Jenkinsfile:

groovy
pipeline {
    agent any

    environment {
        APP_DIR = "/opt/checkout/chat-app"
        DEPLOY_DIR = "/opt/deployment/react"
        S3_BUCKET = "your-s3-bucket-name"
        AWS_REGION = "ap-south-1"
    }

    stages {
        stage('Stop current deployment') {
            steps {
                sh '''
                  if pm2 list | grep -q chat-app; then
                    pm2 stop chat-app || true
                  fi
                '''
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/riyanuddin17/DevOpsChallenge.git'
                sh '''
                  sudo mkdir -p ${APP_DIR}
                  sudo rsync -a --delete ./ ${APP_DIR}/
                  sudo chown -R jenkins:jenkins ${APP_DIR}
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                  cd ${APP_DIR}
                  npm install
                  npm run build
                '''
            }
        }

        stage('Copy dist to deployment') {
            steps {
                sh '''
                  sudo mkdir -p ${DEPLOY_DIR}
                  sudo rm -rf ${DEPLOY_DIR}/*
                  sudo cp -r ${APP_DIR}/dist/* ${DEPLOY_DIR}/
                  sudo chown -R DevOps:DevOps ${DEPLOY_DIR}
                '''
            }
        }

        stage('Deploy with pm2') {
            steps {
                sh '''
                  cd ${DEPLOY_DIR}
                  if ! command -v pm2 >/dev/null 2>&1; then
                    sudo npm install -g pm2
                  fi
                  pm2 delete chat-app || true
                  pm2 serve . 3000 --name chat-app --spa
                  pm2 save
                '''
            }
        }

        stage('Upload dist to S3') {
            steps {
                withAWS(region: "${AWS_REGION}", credentials: 'aws-s3-credentials-id') {
                    s3Upload(
                        includePathPattern: '**/*',
                        workingDir: "${APP_DIR}/dist",
                        bucket: "${S3_BUCKET}"
                    )
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
aws-s3-credentials-id: Jenkins credential with IAM access to S3. [file:1]

Part 5: Dockerization and docker-compose
5.1 Dockerfile
text
FROM nginx:alpine

# Remove default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built files only (dist)
COPY dist /usr/share/nginx/html

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"]
[file:1]

Example nginx.conf:

text
server {
    listen 3000;
    server_name _;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri /index.html;
    }
}
[file:1]

5.2 Build and run container
bash
cd /opt/checkout/chat-app
sudo docker build -t chat-app:latest .
sudo docker images
[file:1]

5.3 docker-compose.yml
text
version: "3.8"

services:
  chat-app-container:
    image: chat-app:latest
    container_name: chat-app-container
    ports:
      - "3000:3000"
    restart: always
[file:1]

Run:

bash
sudo docker-compose up -d
sudo docker ps
