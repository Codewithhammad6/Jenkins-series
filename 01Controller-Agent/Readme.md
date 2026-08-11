# Jenkins Controller-Agent Setup Using SSH

This guide explains how to connect a separate **AWS EC2 instance as a Jenkins Agent using SSH**.

## Architecture

```text
                    AWS
                     |
          +----------+----------+
          |                     |
          v                     v
 Jenkins Controller        Jenkins Agent
      EC2 Instance           EC2 Instance
          |                     |
          |         SSH          |
          +--------------------->|
                                |
                          Executes Pipeline
```

The **Jenkins Controller** manages Jenkins and schedules jobs, while the **Jenkins Agent** executes the actual pipeline commands.

---

# 1. Controller vs Agent

## Jenkins Controller

The Controller is responsible for:

* Managing Jenkins
* Managing jobs
* Scheduling builds
* Managing agents
* Storing Jenkins configuration

## Jenkins Agent

The Agent is responsible for:

* Executing pipeline commands
* Running shell commands
* Building applications
* Running tests
* Building Docker images
* Performing deployment tasks

---

# 2. Create a New EC2 Instance for Agent

Create a second EC2 instance from AWS.

Recommended configuration:

```text
AMI:
Ubuntu Server

Architecture:
64-bit (x86)

Instance:
2 GB RAM or more

Storage:
20 GB or more

Security Group:
SSH - TCP 22
```

The setup will contain two EC2 instances:

```text
Jenkins Controller
        EC2
         |
         | SSH
         v
Jenkins Agent
        EC2
```

---

# 3. SSH Into the Agent

From your local machine:

```bash
ssh -i "your-key.pem" ubuntu@AGENT_PUBLIC_IP
```

Example:

```bash
ssh -i "jenkins-agent.pem" ubuntu@3.xxx.xxx.xxx
```

Verify the current user:

```bash
whoami
```

Expected:

```text
ubuntu
```

---

# 4. Update the Agent

Run:

```bash
sudo apt update
```

Optionally:

```bash
sudo apt upgrade -y
```

---

# 5. Install Java

Jenkins Agent requires Java.

Install Java 21:

```bash
sudo apt install fontconfig openjdk-21-jre -y
```

Verify:

```bash
java -version
```

Expected:

```text
openjdk 21.x.x
```

---

# 6. Generate SSH Key on Jenkins Controller

SSH into the Jenkins Controller.

Check the `.ssh` directory:

```bash
ls -la ~/.ssh
```

Generate an Ed25519 SSH key:

```bash
ssh-keygen -t ed25519
```

When asked for the file location, press:

```text
Enter
```

For this lab setup, when asked for a passphrase, press:

```text
Enter
```

again to leave it empty.

This creates:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

---

# 7. Understand the SSH Keys

Two files are created:

```text
id_ed25519
id_ed25519.pub
```

### Private Key

```text
id_ed25519
```

This is the **private key**.

Never share it publicly.

### Public Key

```text
id_ed25519.pub
```

This is the **public key**.

The public key will be placed on the Jenkins Agent.

The private key will be stored in Jenkins Credentials.

```text
Jenkins Controller
       |
       | Private Key
       v
Jenkins Credentials

       +

Jenkins Agent
       |
       | Public Key
       v
~/.ssh/authorized_keys
```

---

# 8. Copy the Public Key

On the Jenkins Controller:

```bash
cat ~/.ssh/id_ed25519.pub
```

You will get something similar to:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI........ ubuntu@jenkins
```

Copy the **entire line**.

---

# 9. Add Public Key to the Agent

SSH into the Agent:

```bash
ssh -i "your-key.pem" ubuntu@AGENT_PUBLIC_IP
```

Create the `.ssh` directory:

```bash
mkdir -p ~/.ssh
```

Set permissions:

```bash
chmod 700 ~/.ssh
```

Open the authorized keys file:

```bash
nano ~/.ssh/authorized_keys
```

Paste the public key copied from the Jenkins Controller.

Save and exit.

Then:

```bash
chmod 600 ~/.ssh/authorized_keys
```

The Agent now trusts the Jenkins Controller's public key.

---

# 10. Test SSH From Controller

Go back to the Jenkins Controller.

Run:

```bash
ssh ubuntu@AGENT_PUBLIC_IP
```

Example:

```bash
ssh ubuntu@3.xxx.xxx.xxx
```

If everything is configured correctly, the Controller should connect to the Agent without requiring the original `.pem` file.

You should get something similar to:

```text
ubuntu@ip-xxx-xxx-xxx:~$
```

This confirms:

```text
Jenkins Controller
        |
        | SSH
        v
Jenkins Agent
```

is working.

---

# 11. Verify SSH Keys

On the Jenkins Controller:

```bash
ls -la ~/.ssh
```

You should see:

```text
id_ed25519
id_ed25519.pub
```

The private key:

```text
id_ed25519
```

must remain protected on the Controller.

The public key:

```text
id_ed25519.pub
```

was added to the Agent:

```text
~/.ssh/authorized_keys
```

---

# 12. Add Private Key to Jenkins Credentials

Open Jenkins:

```text
Jenkins Dashboard
        ↓
Manage Jenkins
        ↓
Credentials
        ↓
System
        ↓
Global credentials
        ↓
Add Credentials
```

Select:

```text
Kind:
SSH Username with private key
```

Configure:

```text
Username:
ubuntu
```

For the private key, select:

```text
Enter directly
```

On the Jenkins Controller run:

```bash
cat ~/.ssh/id_ed25519
```

Copy the entire private key.

It will look similar to:

```text
-----BEGIN OPENSSH PRIVATE KEY-----

xxxxxxxxxxxxxxxxxxxxxxxx

xxxxxxxxxxxxxxxxxxxxxxxx

-----END OPENSSH PRIVATE KEY-----
```

Paste it into the Jenkins private key field.

---

# 13. Credential Configuration

Example:

```text
Kind:
SSH Username with private key

Username:
ubuntu

Private Key:
<contents of id_ed25519>

ID:
hammi-agent-key

Description:
SSH key for hammi-agent
```

Click:

```text
Create
```

The important concept is:

```text
Private Key
     ↓
Jenkins Credentials
```

while:

```text
Public Key
     ↓
Agent ~/.ssh/authorized_keys
```

---

# 14. Create Jenkins Agent Node

Go to:

```text
Jenkins Dashboard
        ↓
Manage Jenkins
        ↓
Nodes
```

Click:

```text
New Node
```

Enter:

```text
Node Name:
hammi-agent
```

Select:

```text
Permanent Agent
```

Click:

```text
Create
```

---

# 15. Configure the Agent

## Name

```text
hammi-agent
```

## Description

```text
Ubuntu EC2 Jenkins Agent
```

## Number of Executors

For a basic lab:

```text
1
```

Executors determine how many builds the Agent can execute simultaneously.

---

# 16. Remote Root Directory

Set:

```text
/home/ubuntu/jenkins
```

A dedicated directory is preferable.

This directory is used by Jenkins for Agent-related files and workspaces.

---

# 17. Labels

Set:

```text
hammi
```

The label allows pipelines to specifically select this Agent.

Example:

```groovy
agent {
    label "hammi"
}
```

Our configuration:

```text
Node Name:
hammi-agent

Label:
hammi
```

The label does not have to be the same as the node name.

---

# 18. Usage

For this lab, select:

```text
Use this node as much as possible
```

or the equivalent option available in your Jenkins version.

---

# 19. Launch Method

Under:

```text
Launch method
```

Select:

```text
Launch agents via SSH
```

Jenkins will connect to the Agent using SSH.

---

# 20. Agent Host

Enter the Agent EC2 public IP:

```text
Host:
AGENT_PUBLIC_IP
```

Example:

```text
Host:
3.xxx.xxx.xxx
```

---

# 21. Select Credentials

Select the SSH credential created earlier:

```text
hammi-agent-key
```

This credential contains:

```text
Username:
ubuntu

Private Key:
id_ed25519
```

---

# 22. Host Key Verification

For a learning/lab environment, Jenkins may provide different host-key verification options depending on the Jenkins version.

For production environments, use proper SSH host-key verification.

Do not blindly disable SSH verification in production.

---

# 23. Save the Agent Configuration

Click:

```text
Save
```

Jenkins will attempt to connect:

```text
Jenkins Controller
        |
        | SSH
        v
hammi-agent
```

---

# 24. Check Agent Status

Go to:

```text
Manage Jenkins
        ↓
Nodes
```

You should see:

```text
Built-In Node
hammi-agent
```

The `hammi-agent` should be **online**.

You may also see information such as:

```text
Architecture:
Linux (amd64)

Clock Difference:
In sync

Free Disk Space:
...

Response Time:
...
```

If the Agent is online, Jenkins can execute jobs on it.

---

# 25. Verify the Agent Workspace

When a pipeline runs on the Agent, Jenkins creates a workspace.

For example:

```text
/home/ubuntu/workspace/CICD-Demo
```

The structure may look like:

```text
/home/ubuntu/
    |
    ├── workspace/
    │      |
    │      └── CICD-Demo/
    |
    └── jenkins/
```

The exact workspace location depends on the node configuration and Jenkins setup.

---

# 26. Create a Test Pipeline

Create a new Jenkins Pipeline:

```text
New Item
    ↓
Pipeline
```

Example name:

```text
CICD-Demo
```

Use this Jenkinsfile:

```groovy
pipeline {

    agent {
        label "hammi"
    }

    stages {

        stage('Hello') {
            steps {
                echo 'Hello World'
            }
        }

        stage('Create Folder') {
            steps {
                sh "mkdir -p devops"
            }
        }

        stage('Bye') {
            steps {
                echo 'Bye'
            }
        }
    }
}
```

---

# 27. Pipeline Explanation

## Agent

```groovy
agent {
    label "hammi"
}
```

This tells Jenkins:

```text
Run this pipeline on the node
whose label is "hammi".
```

Therefore:

```text
CICD-Demo
     |
     ↓
label = hammi
     |
     ↓
hammi-agent
```

---

# 28. Hello Stage

```groovy
stage('Hello') {
    steps {
        echo 'Hello World'
    }
}
```

This prints:

```text
Hello World
```

---

# 29. Create Folder Stage

```groovy
stage('Create Folder') {
    steps {
        sh "mkdir -p devops"
    }
}
```

This executes:

```bash
mkdir -p devops
```

The folder is created on:

```text
hammi-agent
```

**Not on the Jenkins Controller.**

This is an important Jenkins Controller-Agent concept.

---

# 30. Bye Stage

```groovy
stage('Bye') {
    steps {
        echo 'Bye'
    }
}
```

This prints:

```text
Bye
```

---

# 31. Expected Build Output

After running the pipeline, the output should look similar to:

```text
Started by user

[Pipeline] Start of Pipeline

[Pipeline] node

Running on hammi-agent in /home/ubuntu/workspace/CICD-Demo

[Pipeline] {
[Pipeline] stage
[Pipeline] { (Hello)

[Pipeline] echo
Hello World

[Pipeline] }

[Pipeline] stage
[Pipeline] { (Create Folder)

[Pipeline] sh
+ mkdir -p devops

[Pipeline] }

[Pipeline] stage
[Pipeline] { (Bye)

[Pipeline] echo
Bye

[Pipeline] }

[Pipeline] }

[Pipeline] End of Pipeline

Finished: SUCCESS
```

The important line is:

```text
Running on hammi-agent
```

This proves the pipeline is executing on the Agent.

---

# 32. Verify the Folder on Agent

SSH into the Agent:

```bash
ssh ubuntu@AGENT_PUBLIC_IP
```

Go to the workspace:

```bash
cd /home/ubuntu/workspace/CICD-Demo
```

Check the files:

```bash
ls
```

You should see:

```text
devops
```

This proves that:

```bash
mkdir -p devops
```

was executed on the Jenkins Agent.

---

# 33. Controller vs Agent

## Controller

```text
- Manages Jenkins
- Schedules jobs
- Manages agents
- Stores job configuration
- Sends work to agents
```

## Agent

```text
- Executes pipeline commands
- Builds applications
- Runs tests
- Creates Docker images
- Performs deployment tasks
```

Architecture:

```text
                  Jenkins Controller
                         |
                         |
                    SSH Connection
                         |
                         v
                     hammi-agent
                         |
                +--------+--------+
                |        |        |
                v        v        v
              Git      Docker    Tests
```

---

# 34. Why Use Jenkins Agents?

## Load Separation

```text
Controller
    |
    | Scheduling
    v
Agent
    |
    | Heavy Build
    v
CPU / RAM / Disk
```

The Controller does not need to perform every build.

## Different Environments

You can have different Agents for different requirements:

```text
Linux Agent
Windows Agent
Docker Agent
Kubernetes Agent
Java Agent
Node.js Agent
```

For example:

```text
Java Application
       ↓
Java Agent

Windows Application
       ↓
Windows Agent

Docker Build
       ↓
Docker Agent
```

## Parallel Builds

Multiple Agents allow multiple jobs to run simultaneously:

```text
                   Jenkins
                  /   |   \
                 /    |    \
                v     v     v
            Agent-1 Agent-2 Agent-3
              Job-A   Job-B   Job-C
```

---

# 35. Final Architecture

The completed setup looks like:

```text
                         AWS
                          |
               +----------+----------+
               |                     |
               v                     v
       Jenkins Controller        Jenkins Agent
            EC2 Instance           EC2 Instance
               |                     |
               |                     |
               | SSH                 |
               +-------------------->|
                                      |
                                 ubuntu user
                                      |
                                   Java 21
                                      |
                               Jenkins Agent
                                      |
                                      v
                              Pipeline Execution
                                      |
                           +----------+----------+
                           |          |          |
                           v          v          v
                         Git       Docker       Tests
```

---

# 36. Security Notes

Never share:

```text
id_ed25519
```

or the Jenkins private key.

The private key must remain secret.

Only the public key should be placed on the Agent:

```text
~/.ssh/authorized_keys
```

Recommended permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

The SSH private key should only be accessible to the appropriate user/Jenkins credential system.

---

# 37. Troubleshooting

## Agent Is Offline

Go to:

```text
Manage Jenkins
    ↓
Nodes
    ↓
hammi-agent
    ↓
Log
```

Look for SSH connection errors.

---

## SSH Permission Denied

On the Agent:

```bash
ls -la ~/.ssh
```

Check:

```bash
cat ~/.ssh/authorized_keys
```

Make sure the Controller's public key exists there.

Set permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## Test SSH Manually

From the Controller:

```bash
ssh ubuntu@AGENT_PUBLIC_IP
```

If manual SSH does not work, Jenkins will also have problems connecting to the Agent.

---

## Java Not Found

On the Agent:

```bash
java -version
```

If Java is missing:

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
```

---

## Agent Not Accessible

Check the AWS Security Group.

The Agent must allow:

```text
TCP 22
```

from the Jenkins Controller.

For better security, avoid exposing SSH to:

```text
0.0.0.0/0
```

when you can restrict access to the Controller's IP or appropriate security configuration.

---

# 38. Complete Setup Flow

```text
1. Create Jenkins Controller EC2
        ↓
2. Install Jenkins
        ↓
3. Create NEW EC2 instance
        ↓
4. Use it as Jenkins Agent
        ↓
5. SSH into Agent
        ↓
6. Install Java 21
        ↓
7. Generate SSH key on Controller
        ↓
8. Copy PUBLIC key
        ↓
9. Add PUBLIC key to Agent
        ~/.ssh/authorized_keys
        ↓
10. Keep PRIVATE key on Controller
        ↓
11. Add PRIVATE key to Jenkins Credentials
        ↓
12. Create hammi-agent Node
        ↓
13. Configure SSH launch method
        ↓
14. Add Agent IP
        ↓
15. Select SSH credentials
        ↓
16. Add label "hammi"
        ↓
17. Agent becomes Online
        ↓
18. Create Pipeline
        ↓
19. agent { label "hammi" }
        ↓
20. Jenkins sends job to hammi-agent
        ↓
21. Agent executes commands
        ↓
22. Pipeline SUCCESS
```

---

# 39. Result

Our pipeline uses:

```groovy
agent {
    label "hammi"
}
```

This selects:

```text
hammi-agent
```

Jenkins confirms this with:

```text
Running on hammi-agent
```

The command:

```bash
mkdir -p devops
```

is executed on the Agent.

The final result is:

```text
Finished: SUCCESS
```

Therefore, the **Jenkins Controller → Jenkins Agent SSH setup is working successfully**.

---

## Key Concept

The complete relationship is:

```text
                  Jenkins
                 Controller
                     |
                     | SSH
                     |
                     v
                hammi-agent
                  EC2
                     |
                     |
              Pipeline Commands
                     |
          +----------+----------+
          |          |          |
          v          v          v
        Build      Test      Deploy
```

The Controller **manages and schedules** the job, while the Agent **actually executes** the pipeline commands.
