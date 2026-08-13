# Jenkins Shared Library

Haan. Jenkins mein **Shared Library** bana kar aap pipeline ka reusable code Git repository mein rakh sakte ho. Phir har Jenkinsfile mein sirf library import karke functions call karoge.

Aapke current Docker pipeline ke liye main simple aur practical structure suggest karunga.

## 1. Shared Library ka Git repo

Example:

```text
jenkins-shared-library/
│
├── vars/
│   ├── dockerBuild.groovy
│   ├── dockerPush.groovy
│   └── deployApp.groovy
│
└── src/
```

`vars/` mein woh functions honge jo Jenkinsfile se directly call kiye ja sakte hain.

---

## 2. `dockerBuild.groovy`

```groovy
def call() {
    echo 'Building Docker images'

    sh '''
        docker compose build
    '''
}
```

---

## 3. `dockerPush.groovy`

Isko thora reusable bana dete hain:

```groovy
def call(String username, String credentialId, List<String> images) {

    echo 'Logging in to Docker Hub'

    withCredentials([
        usernamePassword(
            credentialsId: credentialId,
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

    echo 'Pushing Docker images'

    images.each { image ->
        sh "docker push ${username}/${image}:latest"
    }
}
```

---

## 4. `deployApp.groovy`

```groovy
def call() {

    echo 'Starting application with Docker Compose'

    sh '''
        docker compose up -d
    '''
}
```

---

## 5. Ab aapka Jenkinsfile kaafi short ho jayega

```groovy
@Library('my-shared-library') _

pipeline {

    agent {
        label 'hammi'
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
                dockerBuild()
            }
        }

        stage('Push Images') {
            steps {
                dockerPush(
                    'hammadch123',
                    'dockerhubcred',
                    [
                        'notes-frontend-jenkins',
                        'notes-backend-jenkins',
                        'notes-nginx-jenkins'
                    ]
                )
            }
        }

        stage('Deploy') {
            steps {
                deployApp()
            }
        }
    }
}
```

Lekin **ek cheez missing hai**: build ke baad images ko Docker Hub names ke saath tag karna. Isliye better architecture ye hoga ke tagging bhi Shared Library mein daal dein.

---

## 6. `dockerBuild.groovy` ko complete bana dein

```groovy
def call() {

    echo 'Building Docker images'

    sh '''
        docker compose build
    '''

    echo 'Tagging Docker images'

    sh '''
        docker tag smc-frontend:latest \
            hammadch123/notes-frontend-jenkins:latest

        docker tag smc-django_app1:latest \
            hammadch123/notes-backend-jenkins:latest

        docker tag smc-nginx:latest \
            hammadch123/notes-nginx-jenkins:latest
    '''
}
```

Phir Jenkinsfile:

```groovy
@Library('my-shared-library') _

pipeline {

    agent {
        label 'hammi'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Codewithhammad6/Django-Notes-App-Containerize-with-Docker'
            }
        }

        stage('Build & Tag') {
            steps {
                dockerBuild()
            }
        }

        stage('Push Images') {
            steps {
                dockerPush(
                    'hammadch123',
                    'dockerhubcred',
                    [
                        'notes-frontend-jenkins',
                        'notes-backend-jenkins',
                        'notes-nginx-jenkins'
                    ]
                )
            }
        }

        stage('Deploy') {
            steps {
                deployApp()
            }
        }
    }
}
```

---

## 7. Jenkins mein Shared Library configure karni hai

Jenkins:

**Manage Jenkins → System → Global Trusted Pipeline Libraries**

Example:

```text
Name:
my-shared-library

Default version:
main

Retrieval method:
Modern SCM

Source Code Management:
Git

Project Repository:
https://github.com/YOUR_USERNAME/jenkins-shared-library.git
```

Phir Jenkinsfile mein:

```groovy
@Library('my-shared-library') _
```

Bas.

---

## 8. Update kaise automatically hogi?

Ye Shared Library ka **main benefit** hai.

Suppose aapke Git repo mein:

```text
jenkins-shared-library
        │
        └── vars/
             └── dockerBuild.groovy
```

Aaj:

```groovy
docker compose build
```

Aap kal library mein change kar dete ho:

```groovy
docker compose build --no-cache
```

Git mein:

```bash
git add .
git commit -m "Update docker build"
git push
```

Ab jis Jenkins pipeline mein:

```groovy
@Library('my-shared-library') _
```

hai, woh next build mein updated library use kar sakti hai.

Flow:

```text
Developer
    │
    │ git push
    ▼
Shared Library Git Repo
    │
    │ updated code
    ▼
Jenkins
    │
    │ @Library('my-shared-library')
    ▼
Latest Shared Library Code
    │
    ▼
Jenkins Pipeline
```

---

## Important Concept

Aapki **application repo**:

```text
Django-Notes-App-Containerize-with-Docker
```

aur **Jenkins Shared Library repo**:

```text
jenkins-shared-library
```

alag repositories hongi.

```text
GitHub
│
├── Django-Notes-App-Containerize-with-Docker
│       └── Jenkinsfile
│
└── jenkins-shared-library
        └── vars/
            ├── dockerBuild.groovy
            ├── dockerPush.groovy
            └── deployApp.groovy
```

Is approach se aap future mein **10–20 projects** ke liye same Docker build, Docker Hub login/push, deployment, notifications, SonarQube, Kubernetes deployment waghera ka reusable code rakh sakte ho.
