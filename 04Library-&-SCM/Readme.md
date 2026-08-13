# Jenkins Pipeline with Shared Library and Jenkinsfile from SCM

Haan, **ye bilkul sahi approach hai**. Agar tum Jenkinsfile ko GitHub repository mein rakhna chahte ho, to Jenkins ko **Pipeline script from SCM** se Jenkinsfile read karwana best hai.

Tumhara Jenkinsfile almost correct hai. Bas `dockerbuild()` ka naam tumhari Shared Library mein exactly jis naam se function bana hai, woh same hona chahiye. Agar file `vars/dockerBuild.groovy` hai, to normally `dockerBuild()` use karo.

---

## GitHub Repository Mein

Tumhari repo:

```text
Django-Notes-App-Containerize-with-Docker/
│
├── backend/
├── mynotes/
├── nginx/
├── docker-compose.yml
├── Jenkinsfile
└── README.md
```

`Jenkinsfile` root directory mein rakho.

---

## Jenkinsfile

```groovy
@Library("shared") _

pipeline {
    agent {
        label "hammi"
    }

    stages {

        stage('Clone Repository') {
            steps {
                clone(
                    "https://github.com/Codewithhammad6/Django-Notes-App-Containerize-with-Docker",
                    "main"
                )
            }
        }

        stage('Build Images') {
            steps {
                dockerBuild()
            }
        }

        stage('Tag Images') {
            steps {
                dockerTag(
                    'hammadch123',
                    [
                        'smc-frontend': 'notes-frontend-jenkins',
                        'smc-django_app1': 'notes-backend-jenkins',
                        'smc-nginx': 'notes-nginx-jenkins'
                    ]
                )
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

        stage('Run Application') {
            steps {
                echo 'Starting application using Docker Compose'
                sh 'docker compose up -d'
            }
        }
    }
}
```

---

## Jenkins Mein Kya Change Karna Hai

Jenkins Job → **Configure**

`Pipeline` section mein:

```text
Definition:
Pipeline script from SCM
```

Then:

```text
SCM:
Git
```

Repository URL:

```text
https://github.com/Codewithhammad6/Django-Notes-App-Containerize-with-Docker
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile
```

Phir:

```text
Save → Build Now
```

---

## Shared Library Configuration

Tumhare Jenkinsfile mein:

```groovy
@Library("shared") _
```

ka matlab Jenkins pehle se configured **Shared Library named `shared`** load karega.

Us library mein tumhare functions hone chahiye:

```text
shared/
└── vars/
    ├── clone.groovy
    ├── dockerBuild.groovy
    ├── dockerTag.groovy
    └── dockerPush.groovy
```

Aur Jenkins Shared Library configuration mein library ka naam exactly:

```text
shared
```

hona chahiye.

---

## Shared Library Functions

### `clone.groovy`

```groovy
def call(String repoUrl, String branch) {

    echo "Cloning repository: ${repoUrl}"

    git(
        branch: branch,
        url: repoUrl
    )
}
```

### `dockerBuild.groovy`

```groovy
def call() {

    echo 'Building Docker images'

    sh '''
        docker compose build
    '''
}
```

### `dockerTag.groovy`

```groovy
def call(String username, Map<String, String> images) {

    echo 'Tagging Docker images'

    images.each { localImage, dockerHubRepo ->

        sh """
            docker tag ${localImage}:latest \
                ${username}/${dockerHubRepo}:latest
        """
    }
}
```

### `dockerPush.groovy`

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

## Complete Architecture

```text
GitHub
│
├── Application Repository
│   │
│   ├── backend/
│   ├── mynotes/
│   ├── nginx/
│   ├── docker-compose.yml
│   └── Jenkinsfile
│
└── Shared Library Repository
    │
    └── vars/
        ├── clone.groovy
        ├── dockerBuild.groovy
        ├── dockerTag.groovy
        └── dockerPush.groovy
```

Jenkins:

```text
Jenkins
   │
   ├── Pipeline script from SCM
   │
   │       ↓
   │
   ├── GitHub Application Repository
   │       │
   │       └── Jenkinsfile
   │
   │
   └── @Library("shared")
           │
           ↓
      Shared Library
           │
           ├── clone()
           ├── dockerBuild()
           ├── dockerTag()
           └── dockerPush()
```

---

## Pipeline Flow

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── Read Jenkinsfile from SCM
   │
   ├── Load Shared Library
   │
   ├── Clone Repository
   │
   ├── Build Docker Images
   │
   ├── Tag Docker Images
   │
   ├── Login to Docker Hub
   │
   ├── Push Images to Docker Hub
   │
   └── Run Application
          │
          ▼
      Docker Compose
```

---

## Important Point

Agar Jenkinsfile repository ke andar hi hai, technically `Clone Repository` stage mein dobara same repository clone karna redundant ho sakta hai, kyunki **Pipeline script from SCM** Jenkins already repository checkout karta hai.

Lekin agar tum apni `clone()` Shared Library ko specifically practice kar rahe ho, to current structure **learning purpose ke liye theek hai**.

### Recommended Production Approach

Production mein duplicate checkout avoid karne ke liye `Clone Repository` stage remove kar sakte ho:

```groovy
@Library("shared") _

pipeline {

    agent {
        label "hammi"
    }

    stages {

        stage('Build Images') {
            steps {
                dockerBuild()
            }
        }

        stage('Tag Images') {
            steps {
                dockerTag(
                    'hammadch123',
                    [
                        'smc-frontend': 'notes-frontend-jenkins',
                        'smc-django_app1': 'notes-backend-jenkins',
                        'smc-nginx': 'notes-nginx-jenkins'
                    ]
                )
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

        stage('Run Application') {
            steps {
                sh 'docker compose up -d'
            }
        }
    }
}
```

Is case mein Jenkins **SCM se repository checkout** karega aur Shared Library sirf reusable pipeline functions provide karegi.
