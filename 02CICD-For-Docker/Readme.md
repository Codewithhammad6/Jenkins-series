# Django Notes App — Jenkins CI/CD with Docker

This project demonstrates a CI/CD workflow for a containerized Django Notes Application using:

* GitHub
* Jenkins
* Docker
* Docker Compose
* Docker Hub
* Nginx
* MySQL
* Gunicorn

The application contains a React frontend, Django backend with multiple instances, Nginx reverse proxy, and MySQL database.

---

# Architecture

```text
                         GitHub
                            |
                            | Push
                            v
                    GitHub Webhook
                            |
                            v
                     +-------------+
                     |   Jenkins   |
                     |    hammi    |
                     +-------------+
                            |
                            v
                     Docker Compose
                            |
                     Docker Build
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
          Frontend       Backend        Nginx
           Image          Image         Image
              |             |             |
              +-------------+-------------+
                            |
                            v
                       Docker Hub
                            |
                            v
                    Docker Compose Up
                            |
                            v
                   Running Application
```

---

# Project Structure

```text
Django-Notes-App-Containerize-with-Docker/
│
├── backend/
│   ├── Dockerfile
│   ├── manage.py
│   ├── requirements.txt
│   └── ...
│
├── mynotes/
│   ├── Dockerfile
│   ├── src/
│   ├── public/
│   └── ...
│
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf
│
├── docker-compose.yml
│
└── README.md
```

---

# Application Architecture

```text
                         Nginx
                           |
                 +---------+---------+
                 |                   |
                 v                   v
             Frontend             Backend
                                     |
                    +----------------+----------------+
                    |                |                |
                    v                v                v
                Django App 1     Django App 2     Django App 3
                    |                |                |
                    +----------------+----------------+
                                     |
                                     v
                                  MySQL
```

The application contains three Django backend instances:

```text
django_app1
django_app2
django_app3
```

Nginx works as the reverse proxy and load-balancing layer.

---

# Docker Images

The project uses the following Docker Hub images:

```text
hammadch123/notes-frontend-jenkins:latest
hammadch123/notes-backend-jenkins:latest
hammadch123/notes-nginx-jenkins:latest
```

MySQL uses the official image:

```text
mysql
```

---

# Docker Compose Services

The `docker-compose.yml` contains:

```text
nginx
frontend
django_app1
django_app2
django_app3
db
```

All services communicate through the Docker network:

```text
notes-app-nw
```

---

# Frontend

The frontend uses:

```text
hammadch123/notes-frontend-jenkins:latest
```

Container:

```text
frontend_cont
```

The frontend communicates with the backend through Nginx.

---

# Backend

The Django backend uses:

```text
hammadch123/notes-backend-jenkins:latest
```

Three backend containers are created:

```text
django_cont-1
django_cont-2
django_cont-3
```

Gunicorn is used to run Django:

```bash
gunicorn notesapp.wsgi --bind 0.0.0.0:5000
```

Each backend instance can use a different environment variable:

```text
SERVER_NAME=server1
SERVER_NAME=server2
SERVER_NAME=server3
```

---

# Database

The application uses MySQL:

```yaml
image: mysql
```

Database configuration:

```text
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=test_db
```

MySQL data is persisted using:

```text
./data/mysql/db:/var/lib/mysql
```

This allows database data to remain available when the container is recreated.

---

# Nginx

Nginx uses:

```text
hammadch123/notes-nginx-jenkins:latest
```

Container:

```text
nginx_cont
```

Port mapping:

```text
8080:80
```

Application URL:

```text
http://SERVER-IP:8080
```

---

# Jenkins CI/CD

Jenkins runs on an agent with the label:

```text
hammi
```

The CI/CD pipeline performs:

```text
1. Clone GitHub repository
2. Build Docker images
3. Login to Docker Hub
4. Tag Docker images
5. Push images to Docker Hub
6. Start application using Docker Compose
```

---

# Important Docker Compose Configuration

If Jenkins is expected to **build Docker images**, the `docker-compose.yml` must contain both `build:` and `image:`.

Example:

```yaml
services:

  frontend:
    build:
      context: ./mynotes
    image: smc-frontend:latest

  backend:
    build:
      context: ./backend
    image: smc-django_app1:latest

  nginx:
    build:
      context: ./nginx
    image: smc-nginx:latest
```

Then Jenkins can execute:

```bash
docker compose build
```

and create:

```text
smc-frontend:latest
smc-django_app1:latest
smc-nginx:latest
```

### Important

If the Compose file only contains:

```yaml
image: ...
```

and does not contain:

```yaml
build:
```

then:

```bash
docker compose build
```

will not build a new Docker image.

---

# Jenkins Pipeline

Example Jenkinsfile:

```groovy
pipeline {

    agent {
        label "hammi"
    }

    stages {

        stage('Clone Repository') {
            steps {
                echo 'Cloning code from GitHub'

                git branch: 'main',
                    url: 'https://github.com/Codewithhammad6/Django-Notes-App-Containerize-with-Docker'
            }
        }

        stage('Build Images') {
            steps {
                echo 'Building Docker images'

                sh '''
                    docker compose build
                    docker images
                '''
            }
        }

        stage('Docker Login') {
            steps {
                echo 'Logging in to Docker Hub'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhubcred',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                        -u "$DOCKER_USERNAME" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Tag Images') {
            steps {
                echo 'Tagging frontend, backend and nginx images'

                sh '''
                    docker tag smc-frontend:latest \
                    hammadch123/notes-frontend-jenkins:latest

                    docker tag smc-django_app1:latest \
                    hammadch123/notes-backend-jenkins:latest

                    docker tag smc-nginx:latest \
                    hammadch123/notes-nginx-jenkins:latest
                '''
            }
        }

        stage('Push Images') {
            steps {
                echo 'Pushing images to Docker Hub'

                sh '''
                    docker push hammadch123/notes-frontend-jenkins:latest

                    docker push hammadch123/notes-backend-jenkins:latest

                    docker push hammadch123/notes-nginx-jenkins:latest
                '''
            }
        }

        stage('Run Application') {
            steps {
                echo 'Starting application with Docker Compose'

                sh 'docker compose up -d'
            }
        }
    }
}
```

---

# Jenkins Credentials

Create Docker Hub credentials in Jenkins:

```text
Jenkins
    ↓
Manage Jenkins
    ↓
Credentials
    ↓
Global
    ↓
Add Credentials
```

Credential type:

```text
Username with password
```

Configuration:

```text
Username: hammadch123
Password: Docker Hub Access Token
ID: dockerhubcred
```

The Jenkinsfile references:

```groovy
credentialsId: 'dockerhubcred'
```

---

# Docker Build Process

Jenkins runs:

```bash
docker compose build
```

This builds the Docker images defined with `build:` in `docker-compose.yml`.

Expected local images:

```text
smc-frontend:latest
smc-django_app1:latest
smc-nginx:latest
```

Check them with:

```bash
docker images
```

---

# Docker Image Tagging

After building, Jenkins tags the local images with Docker Hub repository names.

### Frontend

```bash
docker tag smc-frontend:latest \
hammadch123/notes-frontend-jenkins:latest
```

### Backend

```bash
docker tag smc-django_app1:latest \
hammadch123/notes-backend-jenkins:latest
```

### Nginx

```bash
docker tag smc-nginx:latest \
hammadch123/notes-nginx-jenkins:latest
```

---

# Docker Hub Push

Jenkins pushes the images:

```bash
docker push hammadch123/notes-frontend-jenkins:latest

docker push hammadch123/notes-backend-jenkins:latest

docker push hammadch123/notes-nginx-jenkins:latest
```

After a successful push, the updated images are available on Docker Hub.

---

# GitHub Webhook

GitHub Webhook automatically triggers Jenkins whenever code is pushed.

Workflow:

```text
Developer
    |
    | git push
    v
GitHub
    |
    | Webhook
    v
Jenkins
    |
    v
Pipeline
```

---

# Jenkins Webhook Configuration

Open:

```text
Jenkins
→ Pipeline Job
→ Configure
→ Build Triggers
```

Enable:

```text
GitHub hook trigger for GITScm polling
```

Save the configuration.

---

# Jenkinsfile Webhook Trigger

You can also define:

```groovy
triggers {
    githubPush()
}
```

Example:

```groovy
pipeline {

    agent {
        label "hammi"
    }

    triggers {
        githubPush()
    }

    stages {
        ...
    }
}
```

---

# GitHub Webhook Configuration

Open:

```text
GitHub Repository
→ Settings
→ Webhooks
→ Add webhook
```

Payload URL:

```text
http://YOUR-JENKINS-PUBLIC-IP:8080/github-webhook/
```

Content type:

```text
application/json
```

Events:

```text
Just the push event
```

Enable:

```text
Active
```

Then click:

```text
Add webhook
```

---

# AWS Security Group

Because Jenkins is running on AWS EC2, port `8080` must be accessible.

Open:

```text
AWS
→ EC2
→ Security Groups
→ Inbound Rules
```

Add:

```text
Type: Custom TCP
Port: 8080
```

For testing:

```text
0.0.0.0/0
```

For production, access should be restricted and Jenkins should preferably be exposed through HTTPS/reverse proxy.

---

# Complete CI/CD Workflow

```text
Developer
    |
    | git push
    v
GitHub
    |
    | Webhook
    v
Jenkins
    |
    v
Clone Repository
    |
    v
Docker Compose Build
    |
    +-------------------+
    |                   |
    v                   v
Frontend Image      Backend Image
    |                   |
    +---------+---------+
              |
              v
          Nginx Image
              |
              v
        Docker Hub Login
              |
              v
          Tag Images
              |
              v
         Push Images
              |
              v
       Docker Hub Updated
              |
              v
      Docker Compose Up
              |
              v
      Application Running
```

---

# Manual Testing

Start the application:

```bash
docker compose up -d
```

Check Compose services:

```bash
docker compose ps
```

Check all containers:

```bash
docker ps
```

Check logs:

```bash
docker compose logs
```

Nginx logs:

```bash
docker compose logs nginx
```

Frontend logs:

```bash
docker compose logs frontend
```

Backend logs:

```bash
docker compose logs django_app1
```

---

# Stop Application

Stop the application:

```bash
docker compose down
```

The MySQL data stored in:

```text
./data/mysql/db
```

remains unless it is manually removed.

---

# Verify Docker Images

Check local images:

```bash
docker images
```

Expected tagged images:

```text
hammadch123/notes-frontend-jenkins
hammadch123/notes-backend-jenkins
hammadch123/notes-nginx-jenkins
```

---

# Webhook Testing

After configuring the GitHub webhook:

```bash
git add .
git commit -m "test webhook"
git push origin main
```

GitHub sends the webhook to Jenkins.

Jenkins should automatically start the pipeline.

Expected Jenkins log:

```text
Started by GitHub push
```

---

# Final Result

The final CI/CD workflow is:

```text
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Hub
   ↓
Docker Compose
   ↓
Running Application
```

The developer only needs to push code to GitHub.

Jenkins automatically handles:

* Source code checkout
* Docker image building
* Docker image tagging
* Docker Hub authentication
* Docker Hub image push
* Application startup

---

# Technologies Used

| Technology     | Purpose                        |
| -------------- | ------------------------------ |
| GitHub         | Source Code Management         |
| Git            | Version Control                |
| Jenkins        | CI/CD Automation               |
| Docker         | Containerization               |
| Docker Compose | Multi-container Application    |
| Docker Hub     | Container Image Registry       |
| Nginx          | Reverse Proxy / Load Balancing |
| Django         | Backend                        |
| Gunicorn       | Django Application Server      |
| MySQL          | Database                       |
| AWS EC2        | Jenkins / Application Server   |

---

# Conclusion

This project demonstrates how Jenkins can automate a Docker-based application deployment.

Whenever code is pushed to the `main` branch:

1. GitHub sends a webhook to Jenkins.
2. Jenkins starts the pipeline.
3. Jenkins clones the latest source code.
4. Docker images are built.
5. Jenkins logs into Docker Hub.
6. Images are tagged.
7. Images are pushed to Docker Hub.
8. Docker Compose starts the updated application.

This provides a basic automated CI/CD workflow for the Django Notes Application.
