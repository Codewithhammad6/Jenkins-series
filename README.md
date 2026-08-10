# Jenkins Installation on AWS EC2

This guide explains how to install and configure **Jenkins LTS** on an Ubuntu AWS EC2 instance, starting from SSH connection and ending with accessing Jenkins from a web browser.

---

## 1. Prerequisites

### Recommended EC2 Configuration

For learning and small projects:

| Resource |          Recommended |
| -------- | -------------------: |
| RAM      |                4 GB+ |
| Storage  |               50 GB+ |
| OS       | Ubuntu 22.04 / 24.04 |
| Java     |           OpenJDK 21 |
| Jenkins  |                  LTS |

> Jenkins can run with less hardware, but 4 GB RAM is recommended for a comfortable learning environment.

---

# 2. Create an AWS EC2 Instance

Go to the AWS EC2 Console and create a new instance.

Recommended settings:

* **AMI:** Ubuntu Server
* **Instance type:** At least 2 GB RAM; 4 GB RAM preferred
* **Storage:** 20–50 GB
* **Key pair:** Create/select your `.pem` key
* **Security Group:** Allow SSH

For initial setup, you need:

```text
SSH
TCP
Port: 22
Source: My IP
```

You will later add Jenkins:

```text
Custom TCP
Port: 8080
Source: My IP
```

For a production server, don't expose Jenkins `8080` publicly to `0.0.0.0/0` unless you have a specific reason.

---

# 3. SSH into the EC2 Instance

Open your terminal/PowerShell and move to the directory containing your `.pem` file.

Example:

```bash
cd Downloads
```

Then connect:

```bash
ssh -i "your-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

```bash
ssh -i "jenkins-key.pem" ubuntu@54.xx.xx.xx
```

If the connection is successful, you should see something similar to:

```text
Welcome to Ubuntu
```

You are now inside the EC2 instance.

---

# 4. Verify the Server

Check the current user:

```bash
whoami
```

Expected:

```text
ubuntu
```

Check the operating system:

```bash
lsb_release -a
```

Check system resources:

```bash
free -h
```

Check disk space:

```bash
df -h
```

---

# 5. Update Ubuntu

Before installing Jenkins, update the package lists:

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt upgrade -y
```

---

# 6. Install Java 21

Jenkins requires Java.

Install Java 21 and Fontconfig:

```bash
sudo apt install fontconfig openjdk-21-jre -y
```

Check Java:

```bash
java -version
```

Expected output will look similar to:

```text
openjdk 21.x.x
OpenJDK Runtime Environment
OpenJDK 64-Bit Server VM
```

Check where Java is installed:

```bash
which java
```

---

# 7. Add the Jenkins LTS Repository

Jenkins provides an official Debian/Ubuntu repository for its LTS releases.

Create the keyrings directory if necessary:

```bash
sudo mkdir -p /etc/apt/keyrings
```

Download the Jenkins repository signing key:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

Add the Jenkins LTS repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Update package information:

```bash
sudo apt update
```

---

# 8. Install Jenkins

Install Jenkins:

```bash
sudo apt install jenkins -y
```

The Jenkins service should be installed automatically.

---

# 9. Check Jenkins Status

Check whether Jenkins is running:

```bash
sudo systemctl status jenkins
```

If everything is working correctly, you should see:

```text
Active: active (running)
```

The important part is:

```text
active (running)
```

Press:

```text
q
```

to exit the status screen.

---

# 10. Start Jenkins Manually

If Jenkins is not running:

```bash
sudo systemctl start jenkins
```

Then check:

```bash
sudo systemctl status jenkins
```

---

# 11. Enable Jenkins at Boot

To make Jenkins start automatically whenever the EC2 instance is rebooted:

```bash
sudo systemctl enable jenkins
```

You can verify:

```bash
sudo systemctl is-enabled jenkins
```

Expected:

```text
enabled
```

---

# 12. Useful Jenkins Service Commands

### Start Jenkins

```bash
sudo systemctl start jenkins
```

### Stop Jenkins

```bash
sudo systemctl stop jenkins
```

### Restart Jenkins

```bash
sudo systemctl restart jenkins
```

### Check Jenkins status

```bash
sudo systemctl status jenkins
```

### Enable Jenkins at boot

```bash
sudo systemctl enable jenkins
```

### Disable Jenkins at boot

```bash
sudo systemctl disable jenkins
```

---

# 13. Check Jenkins Port

By default, Jenkins runs on:

```text
Port 8080
```

Check whether Jenkins is listening on port 8080:

```bash
sudo ss -lntp | grep 8080
```

You may see:

```text
LISTEN 0 50 0.0.0.0:8080
```

This means Jenkins is listening on port `8080`.

---

# 14. Configure AWS Security Group

Now go to:

```text
AWS Console
    ↓
EC2
    ↓
Instances
    ↓
Select Jenkins Instance
    ↓
Security
    ↓
Security Groups
    ↓
Inbound Rules
```

Add a new inbound rule:

```text
Type: Custom TCP
Port: 8080
Source: My IP
```

For example:

```text
Custom TCP | TCP | 8080 | YOUR_IP/32
```

### Why port 8080?

Jenkins uses port `8080` by default.

Your browser will connect to:

```text
http://EC2_PUBLIC_IP:8080
```

---

# 15. Find EC2 Public IP

From AWS EC2 Console, find:

```text
Public IPv4 address
```

Example:

```text
54.xx.xx.xx
```

You can also check from the server:

```bash
curl ifconfig.me
```

---

# 16. Open Jenkins in Browser

Open:

```text
http://YOUR_EC2_PUBLIC_IP:8080
```

Example:

```text
http://54.xx.xx.xx:8080
```

You should see:

```text
Unlock Jenkins
```

---

# 17. Get the Initial Jenkins Admin Password

Jenkins stores the initial administrator password here:

```bash
/var/lib/jenkins/secrets/initialAdminPassword
```

Run:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Example:

```text
8f3cxxxxxxxxxxxxxxxxxxxxxxxx
```

Copy this password.

Paste it into the **Unlock Jenkins** page.

---

# 18. Install Jenkins Plugins

After unlocking Jenkins, you will see plugin installation options.

Choose:

```text
Install suggested plugins
```

Jenkins will install commonly required plugins.

Examples include plugins for:

* Git
* Pipeline
* Credentials
* SCM
* Build tools
* Jenkins UI

You can install additional plugins later.

---

# 19. Create Jenkins Admin User

Jenkins will ask you to create an administrator account.

Example:

```text
Username: admin
Password: ********
Full name: Your Name
Email: your-email@example.com
```

Use a strong password.

Then continue.

---

# 20. Jenkins URL

Jenkins may ask you to confirm the Jenkins URL.

For a basic EC2 setup:

```text
http://YOUR_EC2_PUBLIC_IP:8080/
```

Example:

```text
http://54.xx.xx.xx:8080/
```

For production environments, you would normally use a domain name and HTTPS instead.

---

# 21. Jenkins Dashboard

After setup, you should reach:

```text
Jenkins Dashboard
```

At this point Jenkins is successfully installed.

---

# 22. Verify Jenkins Installation from Terminal

Check Jenkins service:

```bash
sudo systemctl status jenkins
```

Check Java:

```bash
java -version
```

Check Jenkins port:

```bash
sudo ss -lntp | grep 8080
```

Check Jenkins process:

```bash
ps aux | grep jenkins
```

---

# 23. Check Jenkins Logs

If Jenkins is not working, check its logs:

```bash
sudo journalctl -u jenkins
```

To see recent logs:

```bash
sudo journalctl -u jenkins -n 50
```

To follow logs live:

```bash
sudo journalctl -u jenkins -f
```

Press:

```text
Ctrl + C
```

to stop following the logs.

---

# 24. Troubleshooting Jenkins

## Jenkins is not running

Check:

```bash
sudo systemctl status jenkins
```

Try restarting:

```bash
sudo systemctl restart jenkins
```

Then check:

```bash
sudo systemctl status jenkins
```

---

## Port 8080 is not listening

Run:

```bash
sudo ss -lntp | grep 8080
```

If nothing appears, check logs:

```bash
sudo journalctl -u jenkins -n 100
```

---

## Browser cannot access Jenkins

Check these three things:

### 1. Jenkins is running

```bash
sudo systemctl status jenkins
```

### 2. Jenkins is listening on 8080

```bash
sudo ss -lntp | grep 8080
```

### 3. AWS Security Group allows port 8080

Make sure the inbound rule contains:

```text
TCP 8080
```

---

# 25. Check Jenkins Configuration

Jenkins configuration is generally stored under:

```text
/var/lib/jenkins
```

Check:

```bash
sudo ls -la /var/lib/jenkins
```

Important locations include:

```text
/var/lib/jenkins
/var/lib/jenkins/secrets
/var/lib/jenkins/jobs
/var/lib/jenkins/workspace
```

---

# 26. Jenkins Workspace

Jenkins uses workspaces to download and build projects.

The default workspace location is:

```text
/var/lib/jenkins/workspace
```

For example:

```bash
sudo ls -la /var/lib/jenkins/workspace
```

---

# 27. Important Jenkins Concepts

After installing Jenkins, learn these concepts:

```text
Jenkins
│
├── Jobs
├── Builds
├── Pipeline
├── Jenkinsfile
├── Credentials
├── Agents
├── Nodes
├── Plugins
├── Workspace
├── Webhooks
└── CI/CD
```

---

# 28. First Jenkins Pipeline

After Jenkins is working, create a new:

```text
New Item
```

Select:

```text
Pipeline
```

Give it a name:

```text
first-pipeline
```

Then add a simple pipeline:

```groovy
pipeline {
    agent any

    stages {

        stage('Build') {
            steps {
                echo 'Building application...'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying application...'
            }
        }
    }
}
```

Run:

```text
Build Now
```

If successful, Jenkins will execute:

```text
Build
  ↓
Test
  ↓
Deploy
```

---

# 29. Jenkins CI/CD Architecture

A typical DevOps workflow will eventually look like:

```text
Developer
    |
    v
   Git
    |
    v
  GitHub
    |
    | Webhook
    v
 Jenkins
    |
    v
 Build
    |
    v
 Test
    |
    v
 Docker Build
    |
    v
 Docker Image
    |
    v
 Docker Registry
    |
    v
 Kubernetes / AWS
```

---

# 30. What to Learn Next

After successfully installing Jenkins, learn Jenkins in this order:

### Step 1 — Jenkins Basics

Learn:

```text
Jobs
Builds
Plugins
Nodes
Agents
Workspace
Credentials
```

### Step 2 — Freestyle Jobs

Learn how to:

```text
Clone GitHub repository
    ↓
Install dependencies
    ↓
Run tests
    ↓
Build application
```

### Step 3 — Pipeline

Learn:

```text
Jenkins Pipeline
Jenkinsfile
Declarative Pipeline
Scripted Pipeline
Stages
Steps
Agents
Environment Variables
```

### Step 4 — GitHub Integration

Learn:

```text
GitHub
    ↓
Webhook
    ↓
Jenkins
    ↓
Pipeline
```

### Step 5 — Docker Integration

Learn:

```text
Jenkins
    ↓
Docker Build
    ↓
Docker Image
    ↓
Docker Hub
```

### Step 6 — Kubernetes Deployment

Eventually build:

```text
GitHub
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Hub
   ↓
Kubernetes
   ↓
Deployment
   ↓
Service
   ↓
Ingress
```

---

# 31. Quick Command Reference

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install Java
sudo apt install fontconfig openjdk-21-jre -y

# Check Java
java -version

# Add Jenkins key
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

# Add Jenkins repository
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update repositories
sudo apt update

# Install Jenkins
sudo apt install jenkins -y

# Check Jenkins
sudo systemctl status jenkins

# Start Jenkins
sudo systemctl start jenkins

# Restart Jenkins
sudo systemctl restart jenkins

# Enable Jenkins at boot
sudo systemctl enable jenkins

# Check port 8080
sudo ss -lntp | grep 8080

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Check Jenkins logs
sudo journalctl -u jenkins -n 50

# Follow Jenkins logs
sudo journalctl -u jenkins -f
```

---

# 32. Final Verification Checklist

Before starting Jenkins learning, verify:

```text
[✓] EC2 instance created
[✓] SSH connection successful
[✓] Ubuntu updated
[✓] Java 21 installed
[✓] Jenkins LTS repository added
[✓] Jenkins installed
[✓] Jenkins service running
[✓] Jenkins enabled at boot
[✓] Port 8080 configured
[✓] AWS Security Group configured
[✓] Jenkins accessible from browser
[✓] Initial password obtained
[✓] Suggested plugins installed
[✓] Admin user created
[✓] Jenkins Dashboard accessible
```

---

# Conclusion

Jenkins is now installed and running on the AWS EC2 instance.

The basic setup is:

```text
AWS EC2
   │
   ├── Ubuntu
   │
   ├── Java 21
   │
   └── Jenkins LTS
          │
          └── Port 8080
                 │
                 ▼
        Jenkins Web Dashboard
```

The next practical step is to connect **Jenkins → GitHub → Docker → Docker Hub → Kubernetes**, which will turn this Jenkins instance into a complete CI/CD environment.
